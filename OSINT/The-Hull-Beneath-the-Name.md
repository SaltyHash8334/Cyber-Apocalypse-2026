# The Hull Beneath the Name — OSINT Write-Up

**Challenge:** The Hull Beneath the Name
**Category:** OSINT
**Target:** 154.57.164.75:32457
**Flag Format:** HTB{VESSELNAME_IMO_BERTH}

---

## How We Solved It — Reasoning

### Overview

A dock scribe at Eastreach Harbour copied an MMSI before dawn from a grey cargo vessel whose cargo office called her "BRINE WALKER" but whose stern letters looked shorter. The Eastreach cargo seal was EC-4418, and the vessel was gone by midday. Our task was to reconstruct the vessel's true identity, IMO number, discharge berth, and previous port from the registry and harbor ledger.

### Step 1: Reconnaissance of the Challenge Interface

Connecting to `http://154.57.164.75:32457/` revealed a React-based desktop application called "Court-Eaves Console" running a maritime investigation game. The application has six main windows:

- Mission Briefing
- Tideglass Browser (internal registry info)
- Registry Search (Stormcoast Maritime Register)
- Harbor Ledger (Eastreach Port Authority)
- Evidence Satchel
- Oath Submission (validation endpoint)

All data was embedded client-side in the JavaScript bundle (`/assets/index-D3nBSKxN.js`), which we extracted and analyzed.

### Step 2: Identifying the Vessel via Registry

The **Mission Briefing** gave us the key starting point:
- **MMSI:** 257771420
- **Cargo office name:** BRINE WALKER
- **Stern letters looked shorter** (i.e., the actual registered name on the hull was shorter than "BRINE WALKER")
- **Seal:** EC-4418

Searching the embedded Registry database for MMSI 257771420 returned:

| Field | Value |
|-------|-------|
| **Current Name** | **BRINEWALKER** |
| Former Name | BRINE WALKER (renamed 2025-11-08) |
| **IMO** | **9384728** |
| MMSI | 257771420 |
| Flag | Stormcoast Maritime Register |
| Type | General Cargo Vessel |
| Year Built | 2007 |
| Registered Owner | Saltmere Navigation AS |
| ISM Manager | Greywater Shipmanagement Ltd |

**Key insight:** The vessel was renamed from "BRINE WALKER" (two words, 11 chars) to "BRINEWALKER" (one word, 11 chars but visually shorter without the space). The cargo office still called her by the old name, but the stern had been repainted with the new shorter name.

### Step 3: Harbor Ledger — Finding the Port Call

Looking at the Harbor Ledger port call records for IMO 9384728 (BRINEWALKER):

| Field | Value |
|-------|-------|
| **Port Call ID** | EPA-2026-0717-044 |
| Vessel | BRINEWALKER |
| IMO | 9384728 |
| Arrival (Anchorage) | 2026-07-16 22:18 UTC (before dawn ✅) |
| Berthed | 2026-07-17 03:44 UTC |
| **Berth** | **E-06** |
| **Previous Port** | **Saltmere Roads** |
| Departure | 2026-07-17 12:10 UTC (gone by midday ✅) |
| Customs Ref | **EC-4418** ✅ (matches the seal) |

### Step 4: Cargo Declaration

The cargo declaration for EC-4418 revealed:
- **Shipper:** Saltmere Ritual Supply Cooperative
- **Consignee:** Ash & Wick Provisioners Ltd
- **Commodities:** Ceremonial lamp oil, treated bell cord, mineral binding compound
- **Inspection:** Documentary only (waived under Protocol 7-C)

### Step 5: Validation

Submitting the four answers to the Oath Submission endpoint (`POST /api/validate`):

```json
{
  "q1": "BRINEWALKER",
  "q2": "9384728",
  "q3": "E-06",
  "q4": "Saltmere Roads"
}
```

Response:
```json
{
  "allCorrect": true,
  "results": {
    "q1": true, "q2": true, "q3": true, "q4": true
  }
}
```

### Step 6: Assembling the Flag

Flag format: `HTB{VESSELNAME_IMO_BERTH}`

| Component | Value |
|-----------|-------|
| VESSELNAME | BRINEWALKER |
| IMO | 9384728 |
| BERTH | E06 |

**Flag:** `HTB{BRINEWALKER_9384728_E06}`

---

## Flags Summary

| Flag | Value |
|------|-------|
| HTB{BRINEWALKER_9384728_E06} | Main challenge flag |

## Tools Used

- curl — HTTP requests to the challenge target
- grep / python3 — JS bundle analysis and data extraction

## Caveats

- All application data was embedded client-side in the React JS bundle — no server-side API queries needed beyond the validation endpoint
- The MMSI (257771420) was the critical starting clue linking the dock scribe's note to the registry record
- The "stern letters looked shorter" clue was resolved by the name change from "BRINE WALKER" to "BRINEWALKER" — same letters, visually shorter without the space
- Berth E-06 had been recently restored to service (per EPA-NTC-2026-082), making it available for this vessel's call