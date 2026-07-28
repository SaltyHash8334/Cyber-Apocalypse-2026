# The Compressed Truth — Registry Forensics PoC

**Challenge:** HTB Cyber Apocalypse 2026 — The Compressed Truth
**Target:** Windows 11 machine image of Veylen Marr's inner node
**Objective:** Recover what CROWQUILL staged and exfiltrated via 7-Zip
**Status:** Solved

---

## Summary

### What Happened

An operative (CROWQUILL) authenticated as user `vmarr` using stolen credentials. They brought KeeFarce onto the machine to dump the master KeePass database, extracted the Shard Reference custody chain and key material, then staged everything into a tar archive (`shardchain.tar`) for exfiltration. Source files and tools were wiped.

### Critical Forensic Evidence

- **7-Zip ArcHistory** confirms the exact archive created: `C:\Users\Public\Pictures\shardchain.tar`
- **7-Zip FolderHistory** reveals every directory browsed and staged — the complete Registry document tree
- **UserAssist** identifies every tool executed: KeeFarce extraction, KeePass access, 7-Zip, KAPE
- **ComDlg32 OpenSavePidlMRU** confirms the save action of `shardchain.tar`
- **KeeFarce** was extracted to `C:\Users\vmarr\AppData\Local\Temp\writ\KeeFarce\`
- **CopyHistory** shows files copied from `C:\Users\Public\Music\saltwork\`

### What Was Taken

The `shardchain.tar` archive contained the entire **C:\Users\vmarr\Documents\Registry\** directory tree:

| Directory/File | Contents |
|----------------|----------|
| `shard_references\` | Shard reference documents |
| `shard_storage\ShardKeepas_FirstMark\` | KeePass databases containing shard keys |
| `shard_ref\` | Shard reference files (e.g., `shard_ref_011_ember_court_piece.txt`) |
| `internal_reports\` | Internal operational reports |
| `custody_chains\` | Shard Reference 7 custody chain records |
| `oath_records_cinderbound_vol2.zip` | Archived oath records (with `saltoaths_secretive\` subfolder) |
| `C:\Users\Public\Music\saltwork\` | Additional data staged via copy operation |

### Security Implications — Critical

- Stolen credentials allowed full system access without forced entry
- KeeFarce extracted the master KeePass database, compromising every shard key the Registry had catalogued
- The complete Registry document structure was exfiltrated — custody chains, shard references, internal reports, and oath records
- 7-Zip was used without encryption headers (EncryptHeaders=0), meaning the archive contents are unprotected
- The archive was staged in `C:\Users\Public\Pictures\` — a non-obvious location, likely for exfiltration through channels that "do not ask questions"

---

## 1. Information Gathering (Registry Forensics)

### 1.1 — Registry Hive Location

The machine image was provided as a forensic copy with registry hives preserved:
- `C/Users/vmarr/NTUSER.DAT` (1,835,008 bytes)
- `C/Users/vmarr/ntuser.dat.LOG1` (606,208 bytes)
- `C/Users/cyberjunkie/NTUSER.DAT` (1,835,008 bytes)
- `C/Windows/System32/config/DEFAULT` (262,144 bytes)

### 1.2 — 7-Zip Compression History

The most critical artifact. The 7-Zip registry key at `Software\7-Zip\Compression` records what was last compressed.

```bash
reglookup -p "Software/7-Zip" C/Users/vmarr/NTUSER.DAT
```

**ArcHistory** revealed the archive:
```
Software/7-Zip/Compression/ArcHistory
Value: C:\Users\Public\Pictures\shardchain.tar
```

**Compression Settings:**
| Setting | Value |
|---------|-------|
| Archiver | tar |
| Level | 5 (Normal) |
| EncryptHeaders | Off (0) |
| ShowPassword | Off |

### 1.3 — 7-Zip File Manager FolderHistory (Browsed Directories)

The FolderHistory key at `Software\7-Zip\FM\FolderHistory` records every directory browsed in 7-Zip File Manager. This reveals the complete reconnaissance path:

```
c:\users\vmarr\desktop\working\                          # CROWQUILL's working directory
c:\users\public\music\saltwork\                          # Additional staged data
c:\users\vmarr\appdata\Roaming\KeePass\                  # KeePass database location
c:\users\vmarr\appdata\Roaming\                          # User appdata roaming
c:\users\vmarr\appdata\                                  # User appdata root
c:\users\vmarr\appdata\Local\                            # User appdata local
C:\Users\vmarr\Documents\Registry\shard_references\      # Shard reference documents
C:\Users\vmarr\Documents\Registry\                       # Registry root
C:\Users\vmarr\Documents\Registry\shard_storage\         # Shard storage
C:\Users\vmarr\Documents\Registry\shard_storage\ShardKeepas_FirstMark\  # KeePass shard vault
C:\Users\vmarr\Documents\Registry\shard_ref\             # Shard reference files
C:\Users\vmarr\Documents\Registry\internal_reports\      # Internal reports
C:\Users\vmarr\Documents\Registry\custody_chains\        # Custody chain records
C:\Users\vmarr\Documents\Registry\oath_records_cinderbound_vol2.zip      # Oath records archive
    ↳ oath_records_cinderbound_vol2\                     # Inside the zip
    ↳ oath_records_cinderbound_vol2\saltoaths_secretive\ # Salted oath records
C:\Users\vmarr\Documents\                                # User documents
C:\Users\vmarr\Documents\Personal\                       # Personal documents
C:\Users\vmarr\                                          # User home
C:\Users\vmarr\Downloads\                                # Downloads
C:\Users\                                                  # Users root
C:\                                                       # Drive root
Computer\                                                  # Shell namespace
```

### 1.4 — CopyHistory (Files Staged via Copy)

```bash
Software/7-Zip/FM/CopyHistory:
c:\users\public\music\saltwork\
```

### 1.5 — Extraction History (KeeFarce Extraction)

```bash
Software/7-Zip/Extraction/PathHistory:
C:\Users\vmarr\AppData\Local\Temp\writ\KeeFarce\
```

The operator extracted KeeFarce tool to perform in-memory KeePass credential dumping.

### 1.6 — UserAssist (Program Execution Evidence)

Decoded from ROT13 encoding:

```bash
reglookup -p "Software/Microsoft/Windows/CurrentVersion/Explorer/UserAssist/{CEBFF5CD-ACE2-4F4F-9178-9926F41749EA}/Count" C/Users/vmarr/NTUSER.DAT
```

| Program | Path | Count | Last Run |
|---------|------|-------|----------|
| 7-Zip File Manager | `7zFM.exe` | 1 | 2026-06-18 |
| 7-Zip | `7zG.exe` | 1 | 2026-06-18 |
| KeePass Password Safe 2 | `KeePass.exe` | 1 | 2026-06-18 |
| Windows Terminal | `WindowsTerminal.exe` | 3 | 2026-06-18 |
| Windows PowerShell ISE | `PowerShell_ISE.exe` | 1 | 2026-06-18 |
| KAPE | `gkape.exe` | 1 | 2026-06-18 |
| Microsoft Edge | `MSEdge` | 2 | 2026-06-18 |
| File Explorer | `Microsoft.Windows.Explorer` | 6 | 2026-06-18 |
| Notepad | `WindowsNotepad` | 2 | 2026-06-18 |
| StickyNotes | `Microsoft.StickyNotes` | 2 | 2026-06-18 |
| Paint | `Microsoft.Paint` | 5 | 2026-06-18 |
| Calculator | `WindowsCalculator` | 8 | 2026-06-18 |

**Downloaded executables:**
```
C:\Users\vmarr\Downloads\7z2601-x64.exe     # 7-Zip 26.01 installer
C:\Users\vmarr\Downloads\KeePass-2.61.1-Setup.exe  # KeePass installer
C:\Users\vmarr\Desktop\KAPE\KAPE\gkape.exe  # KAPE forensic collector
```

### 1.7 — RecentDocs (File Access Artifacts)

```bash
reglookup -p "Software/Microsoft/Windows/CurrentVersion/Explorer/RecentDocs" C/Users/vmarr/NTUSER.DAT
```

MRU order (most recent first):
1. `Registry` folder
2. `custody_chains` folder
3. `shard_ref` folder
4. `internal_reports` folder
5. `current_working` folder (from desktop\working)
6. `Desktop` folder
7. `shard_references` folder
8. `shardchain.tar` (the archive)
9. `Capture` (screenshot evidence?)
10. `KAPE` (Kroll Artifact Parser and Extractor)
11. `marr_working_notes_lowtide.txt` (Veylen Marr's working notes)
12. `Personal` folder
13. `shard_ref_011_ember_court_piece.txt` (specific shard reference)
14. `Documents` folder

### 1.8 — ComDlg32 (Save Dialog Evidence)

The last save operation confirmed:

```bash
reglookup -p "Software/Microsoft/Windows/CurrentVersion/Explorer/ComDlg32/OpenSavePidlMRU/*" C/Users/vmarr/NTUSER.DAT
```

Last saved file: **shardchain.tar**
Location: `C:\Users\Public\Pictures\`
Application used: **7zG.exe** (confirmed by LastVisitedPidlMRU)

---

## 2. Vulnerability Assessment

| Finding | Vulnerability | Risk |
|---------|--------------|------|
| Stolen credentials accepted without MFA | Single-factor authentication bypass | Critical |
| KeePass database accessible from user context | Master password vault exposed to user-level access | Critical |
| Registry document store world-readable | No additional access controls on shard data | High |
| 7-Zip archive unencrypted | Data-in-transit unprotected | High |
| No EDR detection of KeeFarce execution | No behavioural monitoring | High |

---

## 3. Exploitation (Credential Theft Narrative)

```
CROWQUILL
  ├── Stolen password → authenticate as vmarr
  ├── Install 7-Zip (7z2601-x64.exe)
  ├── Download KeeFarce (extracted to Temp\writ\KeeFarce\)
  ├── Use KeeFarce to dump KeePass credentials
  │   └── From: C:\Users\vmarr\AppData\Roaming\KeePass\
  │   └── Extracted: All stored KeePass entries
  └── Access Registry document store
      └── Documents\Registry\shard_references\
      └── Documents\Registry\shard_storage\ShardKeepas_FirstMark\
      └── Documents\Registry\custody_chains\
      └── Documents\Registry\internal_reports\
      └── Documents\Registry\oath_records_cinderbound_vol2.zip\
```

---

## 4. Post-Exploitation (Data Staging)

### Staging Process

1. **Browse** — Navigate the Registry document structure in 7-Zip FM
2. **Copy** — Stage data from `C:\Users\Public\Music\saltwork\`
3. **Compress** — Create `C:\Users\Public\Pictures\shardchain.tar` (tar format, level 5, no encryption)
4. **Clean** — Wipe source files and removal tools

### Timeline (from registry metadata)

| Timestamp | Event |
|-----------|-------|
| 2026-06-18 13:15:15 | 7-Zip extraction path set (KeeFarce) |
| 2026-06-18 13:18:12 | 7-Zip FM column configuration |
| 2026-06-18 13:24:41 | Save dialog opened for shardchain.tar |
| 2026-06-18 13:24:55 | shardchain.tar saved (via 7zG.exe) |
| 2026-06-18 13:25:06 | Compression settings finalized |
| 2026-06-18 13:26:47 | Last 7-Zip FM activity |
| 2026-06-18 13:42:24 | RecentDocs last updated |
| 2026-06-18 13:42:32 | UserAssist last activity |

### Files Wiped (confirmed by absence in image)

- `C:\Users\Public\Pictures\shardchain.tar` — the archive (exfiltrated)
- `C:\Users\vmarr\Documents\Registry\` — entire document tree
- `C:\Users\vmarr\Downloads\7z2601-x64.exe` — 7-Zip installer
- `C:\Users\vmarr\Downloads\KeePass-2.61.1-Setup.exe` — KeePass installer
- `C:\Users\vmarr\AppData\Local\Temp\writ\KeeFarce\` — KeeFarce tool
- `C:\Users\vmarr\Desktop\KAPE\` — KAPE forensic tool

---

## 5. Commands Used

### Full forensic toolset:

```bash
# Parse vmarr's NTUSER.DAT for 7-Zip artifacts
reglookup -p "Software/7-Zip" C/Users/vmarr/NTUSER.DAT

# Parse vmarr's UserAssist for executed programs
reglookup -p "Software/Microsoft/Windows/CurrentVersion/Explorer/UserAssist/{CEBFF5CD-ACE2-4F4F-9178-9926F41749EA}/Count" C/Users/vmarr/NTUSER.DAT

# Parse RecentDocs for file/folder access
reglookup -p "Software/Microsoft/Windows/CurrentVersion/Explorer/RecentDocs" C/Users/vmarr/NTUSER.DAT

# Parse ComDlg32 for save/open dialog evidence
reglookup -p "Software/Microsoft/Windows/CurrentVersion/Explorer/ComDlg32/OpenSavePidlMRU" C/Users/vmarr/NTUSER.DAT
reglookup -p "Software/Microsoft/Windows/CurrentVersion/Explorer/ComDlg32/LastVisitedPidlMRU" C/Users/vmarr/NTUSER.DAT
```

---

## Conclusion

The registry of Veylen Marr's machine tells the complete story of the CROWQUILL operation. Through 7-Zip artifacts alone — ArcHistory, FolderHistory, Extraction History, and CopyHistory — every action taken on the machine is recoverable:

- **Who**: CROWQUILL, operating as user `vmarr`
- **How**: Stolen credentials, KeeFarce for KeePass extraction
- **What**: Complete Registry document tree including Shard Reference 7 custody chain, shard keys (KeePass), internal reports, oath records, and saltwork data
- **Output**: `shardchain.tar` — an unencrypted tar archive staged in `C:\Users\Public\Pictures\`
- **Destination**: Exfiltrated through covert channels

"7-Zip does not forget which folders were opened, which paths were browsed, and where the staging began." — The registry remembered everything.
