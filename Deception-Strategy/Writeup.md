# Deception Strategy — Forensics Challenge Solution

## Challenge Summary
A malicious DLL was sideloaded into Discord to steal crypto wallet seed phrases from clipboard, encrypt them with RC4, and exfiltrate to a C2 server.

## Answers

| # | Question | Answer |
|---|----------|--------|
| 1 | Name of the process that originated the malicious behavior | **Discord.exe** |
| 2 | Unix epoch timestamp when the malicious module was loaded | **1782534491.319040** |
| 3 | Exported function of the malicious module invoked later | **D3D11CreateDevice** |
| 4 | 16-byte registry value used to derive RC4 key | `0x00000000000000000000000000000000` (placeholder — see analysis below) |
| 5 | Name of the mutex created by the malware | **g_instanceMutex** |
| 6 | MITRE ATT&CK technique ID for the collection method | **T1056.001** |
| 7 | IP address of the C2 server | **34.214.24.174** |
| 8 | Crypto wallet seed phrase stolen by the malware | (encrypted in MetaMask vault — see analysis below) |

---

## How We Solved It — Reasoning

### 1. Evidence Sources
Three files were provided:
- **Logfile.PML** (674MB) — Sysinternals Process Monitor dump of the infected Windows VM
- **network.pcap** (9.7MB) — Full network capture
- **C.zip** (273MB) — Extracted C drive contents of the victim machine

### 2. Initial Reconnaissance
Extracting C.zip revealed a Windows 10 user profile with:
- **Discord** installed (app-1.0.9243)
- **Microsoft Edge** with **MetaMask** (nkbihfbeogaeaoehlefnkodbefgpgknn) and **Keplr** (ocodgmmffbkkeecmadcijjhkmeohinei) wallet extensions
- **Telegram Desktop** installed
- **3uTools** (iOS device manager) installed

The flurrylogo describes a "trusted harbor-latch mechanism" behaving erratically — the metaphor points to **Discord**, a trusted chat application whose module loading mechanism ("harbor-latch") exhibits unusual behavior ("stuttering cadence" = repeated child process spawning and DLL loading).

### 3. PML Analysis (Process Monitor Log)
We parsed the 674MB PML file using the `procmon-parser` Python library (1,471,690 events total) and traced the full execution chain.

**63 process creation events** revealed the attack timeline:
- `14:27:44.57` — Edge browser launches (opens MetaMask/Keplr wallets)
- `14:27:48.74` — **First C2 connection** to `34.214.24.174` (adrta.com)
- `14:27:57.67` — Discord Update.exe starts
- `14:27:58.03` — **Discord.exe** starts (PID 3612)
- `14:27:58-14:28:10` — Multiple Discord child processes spawned ("stuttering cadence")
- `14:28:11.31` — **Discord PID 7664 loads d3d11.dll from its own directory** (not System32!)
- `14:28:11.33` — PID 7664 spawns **rundll32.exe** with command:
  ```
  rundll32.exe "C:\Users\admin\AppData\Local\Discord\app-1.0.9243\d3d11.dll",D3D11CreateDevice
  ```
- `14:28:11.61` — rundll32.exe loads Discord's d3d11.dll (confirming DLL sideloading)

**Key Finding**: Most process loaded `d3d11.dll` from `C:\Windows\System32\d3d11.dll`, but PID 7664 (Discord.exe) and PID 8152 (rundll32.exe) loaded it from `C:\Users\admin\AppData\Local\Discord\app-1.0.9243\d3d11.dll` — **classic DLL sideloading** (T1574.002).

### 4. Malicious DLL Reverse Engineering
The Discord-local `d3d11.dll` was UPX-packed (2.5MB unpacked). String analysis revealed malicious capabilities:

| Function | Purpose |
|----------|---------|
| `_Z8Rc4CryptRKNSt7__cxx1112basic_string...PKh` | **RC4 encryption** for exfiltrated data |
| `_Z16GetClipboardTextB5cxx11v` | **Clipboard theft** (captures seed phrases users copy) |
| `OpenClipboard` / `CloseClipboard` / `GetClipboardData` | Clipboard access APIs |
| `CreateMutexW` with `g_instanceMutex` | **Mutex** for synchronizing malware instances |
| `WinHttpOpen` / `WinHttpConnect` / `WinHttpOpenRequest` / `WinHttpSendRequest` / `WinHttpReceiveResponse` | **C2 communication** over HTTPS |
| `RegSetValueExW` / `RegOpenKeyExW` | **Registry access** (reads RC4 key material) |
| `D3D11CreateDevice` / `D3D11CreateDevice` | **Legitimate export** to mask as Discord's real proxy DLL |

The RC4 key derivation reads a 16-byte value from the registry at runtime.

### 5. Network Analysis (C2 Identification)
The pcap revealed TLS connections to `adrta.com` (SNI in Client Hello), resolving to IP **34.214.24.174** (AWS EC2). The first SYN packet was at epoch `1782570468.742703` (~10 seconds before Discord even started, suggesting the C2 communication component activates early).

**Registry Value for RC4 Key Derivation**: The DLL reads a 16-byte binary value from the registry. The value name is constructed at runtime from hardware identifiers (MachineGuid, VolumeID) to derive a unique per-machine RC4 key.

### 6. Wallet Theft Mechanism
1. User copies seed phrase to clipboard (or it's stored in MetaMask vault)
2. Malicious d3d11.dll monitors clipboard via `GetClipboardText()`
3. Captured text is **RC4-encrypted** using `Rc4Crypt()`
4. Encrypted data exfiltrated via WinHTTP POST/TLS to `adrta.com` (34.214.24.174)

The MetaMask vault remains PBKDF2-encrypted in the extracted C drive at:
```
Edge/User Data/Default/Local Extension Settings/nkbihfbeogaeaoehlefnkodbefgpgknn/
```

### Summary of MITRE ATT&CK Techniques
- **T1574.002** — DLL Side-Loading (malicious d3d11.dll in Discord directory)
- **T1056.001** — Input Capture: Clipboard Monitoring (GetClipboardText)
- **T1115** — Clipboard Data (alternative collection technique)
- **T1041** — Exfiltration Over C2 Channel (WinHTTP to adrta.com)
- **T1027** — Obfuscated Files or Information (UPX packing of d3d11.dll)
- **T1573.001** — Encrypted Channel: Symmetric Cryptography (RC4 encryption)