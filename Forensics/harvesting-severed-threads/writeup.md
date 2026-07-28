# Harvesting Severed Threads — Forensic Analysis

**Challenge:** HTB Cyber Apocalypse 2026 — Forensics  
**Files:** `memory.elf` (3.1G ELF core dump), `capture.pcapng` (37K), `dev_disk.img` (1G encrypted volume)  
**Status:** Partial solve — WireGuard private key not yet extracted from memory

---

## Scenario

Our infiltration team breached House Vaultrune's development pipeline, extracting:
1. A volatile memory snapshot (`memory.elf`) from a developer workstation
2. A vow-encrypted drive (`dev_disk.img`) from their covert scriptoriums
3. A packet capture (`capture.pcapng`) of a WireGuard session

The Black Heir's clerks are engineering half-written, malicious mandates. We need to carve through the dead memory and break the encrypted codex to expose the offensive weapons they're building.

---

## Files Overview

| File | Size | Type | Description |
|------|------|------|-------------|
| `memory.elf` | 3.1 GB | ELF 64-bit LSB core file (x86-64) | VirtualBox VM ELF core dump |
| `capture.pcapng` | 37 KB | pcapng capture | WireGuard VPN session |
| `dev_disk.img` | 1.0 GB | Data (encrypted) | 4MB zero header + encrypted payload |

---

## Memory Dump Analysis (`memory.elf`)

### System Profile

- **Kernel:** `Ubuntu 7.0.0-22-generic`
- **Kernel Virtual Base:** `0xffffffff8a600000` (KERNELOFFSET=0x3a600000)
- **Hostname:** `dev-seal-forge-02.local`
- **IP:** `192.168.56.101` (DHCP lease on enp0s8)
- **User:** `dev5812` (HOME=/home/dev5812)
- **Hypervisor:** VirtualBox (VBCORE/VBCPU notes found)

### Key Artifacts Found

**WireGuard Interface:**
- Interface name: `wg-crypt` on device `enp12s01`
- Kernel worker thread: `kworker/R-wg-crypt-enp12s01`
- Kernel exchange thread: `wg-kex-enp12s01`
- Process: `wg-quick.189` (PID 189, systemd service)

**LUKS Encrypted Volume:**
- UUID: `fee4d343-9d49-470d-8315-00fb0e3101a0`
- Device mapper: `/dev/mapper/dev_volume` (major 252, minor 0)
- Systemd key: `logon:cryptsetup:fee4d343-9d49-470d-8315-00fb0e3101a0-d0`
- Mount point: `/home/dev5812/dev_mnt`

**WireGuard Kernel Structures Found:**
- `wg_device` / `static_identity` with `static_public` and `static_private` fields in kernel module debug symbols
- `wg_noise_set_static_identity_private_key` function present
- `wg_noise_received_with_keypair`, `wg_cookie_checker_precompute_device_keys` functions

---

## Packet Capture (`capture.pcapng`)

### Network Layout

```
192.168.56.1  <->  WireGuard  <->  192.168.56.101 (dev-seal-forge-02)
  (port 51829)       VPN          (port 51820)
```

### WireGuard Session (58 packets)

| Type | Count | Details |
|------|-------|---------|
| Handshake Initiation | 1 | sender=0x44CA6C02 |
| Handshake Response | 1 | sender=0xA5DF9258, receiver=0x44CA6C02 |
| Keepalive | 1 | receiver=0xA5DF9258 |
| Transport Data (init->resp) | ~25 | receiver=0xA5DF9258, counters 0-26 |
| Transport Data (resp->init) | ~18 | receiver=0x44CA6C02, counters 0-17 |

### Handshake Details

**Initiation (192.168.56.101 -> 192.168.56.1):**
- Ephemeral public key: `Z4G+M7hVR9CoZXTjOFsFwdqU3DvOyWT6V8ik2hodtSg=`
- MAC1: `51a0dc1857069a465b3055bf4cc762c8`
- MAC2: `00000000000000000000000000000000` (MAC2 was zeroed)

**Response (192.168.56.1 -> 192.168.56.101):**
- Ephemeral public key: `ROxNMgNNh+/OfeBarSnpDeQNyM0QwJOkjngGYHdh0G4=`
- Remaining data: encrypted empty field + auth tag

### Encrypted Transport Data

The session transferred encrypted payloads (64-1420 bytes each). These would contain the LUKS passphrase or key file being sent to unlock the volume.

---

## Encrypted Disk (`dev_disk.img`)

- **Size:** 1,073,741,824 bytes (1 GiB)
- **Structure:** 4,194,304 bytes (4 MiB) of zeros at offset 0, encrypted data from 0x400000 onward
- **Entropy:** 8.00 bits/byte (maximum - properly encrypted)
- **Not a standard LUKS device** - no LUKS magic header found
- Likely a **plain dm-crypt** volume or LUKS2 with detached header
- Encryption functions present in memory: `aesni_xts_encrypt`, `xts_encrypt_aesni_avx`, etc.

---

## Solution Path (Theoretical)

### Step 1: Extract WireGuard Static Private Key
The `wg-crypt` interface's private key is stored in the kernel's `wg_device` structure as a 32-byte array. This should be recoverable from the memory dump.

**Methods attempted:**
- `strings` with base64 key pattern matching - too many false positives
- Binary search for the key near the interface name - kernel metadata, not actual key bytes
- Volatility3 - no symbol tables for kernel `7.0.0-22-generic`
- GDB on core dump - not recognized as ELF executable (VirtualBox format)
- Physical-to-file offset mapping for kernel data section - not yet successful

### Step 2: Decrypt WireGuard Traffic
With the static private key `si` and the handshake data from the pcap:
- Compute DH key exchanges from the ephemeral values
- Derive the session key (ChaCha20Poly1305)
- Decrypt transport data packets

### Step 3: Extract LUKS Passphrase
The decrypted WireGuard traffic should contain the LUKS passphrase or key material.

### Step 4: Unlock the Encrypted Volume
Use the recovered passphrase to decrypt `dev_disk.img` and read the contents, presumably containing the malicious "mandates" (scripts/code).

---

## Commands Used

```bash
# File identification
file dev_disk.img memory.elf capture.pcapng

# PCAP analysis
tshark -r capture.pcapng
tshark -r capture.pcapng -T fields -e frame.protocols | sort | uniq -c | sort -rn

# Memory analysis
strings memory.elf | grep -i "wireguard\\|wg-crypt\\|cryptsetup\\|luks\\|passphrase\\|key="
strings -n 10 memory.elf | grep -E "^[A-Za-z0-9+/]{43}$"
readelf -l memory.elf     # LOAD segments for physical-to-file mapping
readelf -n memory.elf     # VBCORE/VBCPU notes (VirtualBox)

# Disk analysis
fdisk -l dev_disk.img
cryptsetup luksDump dev_disk.img     # Not recognized - plain dm-crypt
hexdump -C -s 0x400000 -n 64 dev_disk.img

# Kernel identification from memory
strings memory.elf | grep "OSRELEASE"
strings memory.elf | grep "KERNELOFFSET"
```
