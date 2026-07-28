# Archonyx — Cyber Apocalypse 2026 (Web)

- **Target:** `154.57.164.76:31498`
- **Source:** `/home/davey/CTF/CApoc/Archonyx/`
- **Category:** Web
- **Status:** In-progress (all three known approaches tested, path traversal blocked)

---

## Challenge Narrative

> House Veyr & Co. runs the convoy ledger the whole coast trusts. Veyr cooks the numbers: stalling Damas Marrowcairn's cargo, leaking his routes, clearing its own convoys first. Break into the ledger, trace the false delays, and dig out the one convoy Veyr buried deep — the shipment that proves it all.

Goal: RCE via `/readflag` (SUID binary) to read `/flag.txt`.

---

## App Architecture

- **Express 5.2.1 + EJS 6.0.1** — view engine, JWT auth via `jsonwebtoken@9.0.3`
- **Routes:**
  - `/` → auth (login/register/verify)
  - `/api` → API (file upload, download, convoys, dispatches, relay keys)
  - `/ledgermaster` → admin (requires `role: 'ledgermaster'`)
  - `/` → pages (ledger, report, file-convoy, profile, vault, widgets)
- **Bot:** Puppeteer (`puppeteer@25.1.0`), visits any URL from `/report` form submission, cookie set with `SameSite` features disabled
- **CSP:** `script-src 'nonce-...'` — no inline scripts, no external

---

## Key Dependencies (vulnerable targets)

| Package | Version | Notes |
|---------|---------|-------|
| `decompress` | 4.2.1 | Orchestrator — proxies to sub-plugins |
| `decompress-tar` | 4.1.1 | **PATCHED** — has `safeMakeDir`, `preventWritingThroughSymlink` |
| `decompress-unzip` | 4.0.1 | **UNPATCHED** — no path validation, but yauzl (2.10.0) blocks `..` |
| `download` | 8.0.0 | Uses `decompress` for extraction |
| `unzipper` | 0.12.3 | Used by manifest upload (separate path from download) |
| `less` | 4.2.0 | `@plugin` directive can load local JS files as plugins (RCE!) |
| `ejs` | 6.0.1 | Template engine |
| `yauzl` | 2.10.0 | Used by decompress-unzip; has `validateFileName()` blocking `..` |

---

## Findings & Proofs

### ✅ CONFIRMED WORKING

#### 1. CSRF via cross-origin form submission

Puppeteer is launched with `--disable-features=SameSiteByDefaultCookies,CookiesWithoutSameSiteMustBeSecure`, meaning the bot's cookie behaves like `SameSite=None` (included in cross-origin POSTs).

**Chain:**
1. Host an HTML page with auto-submitting `<form>` targeting `http://target:31498/api/fetch`
2. Bot visits our page → form auto-submits → `POST /api/fetch` with `url=...`
3. Bot's cookie is included → `resolveAuth` finds bot as caller → proceeds
4. Server downloads and extracts the archive from our URL

**Evidence:** Python HTTP server logs show the bot connecting to our localtunnel (`archonyx.loca.lt`):
- `20:24:26` — `GET /` (CSRF page)
- `20:24:29` — `GET /malicious.zip` (archive fetch)
- `20:28:09` — `GET /malicious2.zip`
- Repeated for `.tar.gz`, `.png`, `.zip` payloads — all confirmed fetched.

#### 2. Less `@plugin` RCE (local PoC)

```less
@plugin '/tmp/evil';
```

`evil.js` top-level code executes during `less.render()`. Verified locally with:
```
LESS PLUGIN LOADED!
Plugin installed
```

This is the intended RCE vector — but requires `role: 'ledgermaster'` to reach `/ledgermaster/render`.

#### 3. Same-origin file download works for non-archive files

When `archiveType(data)` returns null (e.g., raw PNG), the `download` package writes the file directly to `uploads/imports/<drawsId>/filename`. `validateExtractedFiles` runs asynchronously and removes non-images.

---

### ❌ BLOCKED PATHS

#### 1. Path traversal via decompress-tar (tar.gz archives)

`decompress-tar@4.1.1` has three protections:
- `safeMakeDir()` — resolves `realpath()` of parent dir, checks it starts with output path
- `preventWritingThroughSymlink()` — checks if destination itself is a symlink
- `realpath` check on file destination dir

```
Error: Refusing to create a directory outside the output path.
```

#### 2. Path traversal via decompress-unzip (ZIP archives)

`decompress-unzip@4.0.1` itself has NO path validation. However, it uses `yauzl@2.10.0` which has `validateFileName()`:

```javascript
function validateFileName(fileName) {
  if (fileName.indexOf("\\\\") !== -1) return "invalid characters";
  if (/^[a-zA-Z]:/.test(fileName) || /^\\//.test(fileName)) return "absolute path";
  if (fileName.split("/").indexOf("..") !== -1) return "invalid relative path";
  return null;
}
```

```
Error: invalid relative path: ../../../views/petition.ejs
```

The validation runs when `decodeStrings = true` (default). No bypass through options.

#### 3. Unzipper path check (manifest upload)

`unzipper@0.12.3` Extract checks:
```javascript
const extractPath = path.join(opts.path, entry.path.replace(/\\/g, '/'));
if (extractPath.indexOf(opts.path) != 0) { return cb(); }
```

This catches `..` via `path.join` normalization. Central-dir vs local-header filename mismatch was tested but extraction-side check still blocks.

#### 4. JWT forgery
- `alg: "none"` blocked — jsonwebtoken 9.x requires explicit `algorithms: ['none']`
- Empty/null secret rejected — "secretOrPrivateKey must have a value"
- Algorithm confusion not applicable (HMAC-only, no public key exposed)
- Secret is `crypto.randomBytes(32).toString('hex')` from `.env` file

#### 5. Prototype pollution via qs
- `qs@6.15.3` strips `__proto__` and `constructor[prototype]` keys
- Node 22 `JSON.parse` creates plain objects (no prototype pollution)

#### 6. EJS SSTI via query params
- `_OPTS_PASSABLE_WITH_DATA` exists (`delimiter`, `scope`, `context`, etc.) but requires `data.delimiter` to be set on render data — user-controlled data is nested under `data.query`, not at root
- No unescaped EJS output (`<%-`) in any template

---

## What Works / Exploit Chain Pieces

| Step | Status |
|------|--------|
| Bot visits attacker URL | ✅ Confirmed via `/report` |
| CSRF form POST to `/api/fetch` | ✅ Cookie included (SameSite disabled) |
| Server downloads attacker file | ✅ Archive fetched from localtunnel |
| Archive extracted via decompress | ✅ (non-traversal files succeed) |
| Path traversal via `..` in ZIP | ❌ yauzl `validateFileName` blocks |
| Path traversal via `..` in tar.gz | ❌ decompress-tar zip-slip protections |
| Admin Less `@plugin` RCE | ❌ Need `role: 'ledgermaster'` JWT |
| JWT forgery (admin) | ❌ Random 32-byte secret, no known attack |

---

## Tools / Infrastructure Used

- **localtunnel:** `npx lt --port 8888 --subdomain archonyx` → `https://archonyx.loca.lt`
- **HTTP server:** `python3 -m http.server 8888` (served from `/tmp/www/`)
- **Payloads:** malicious `.tar.gz`, `.zip` (path traversal), `.png` (non-archive test)

---

## Remaining Questions & Exploration Ideas

1. Could the vault-downloaded file be referenced as a Less plugin path via web-served URL (`/api/cargo/<filename>/raw`)?
2. Does unzipper's `Open.buffer` vs `Extract` local-header parsing differ enough to allow confusion that bypasses the prefix check?
3. Is there a race condition in `validateExtractedFiles` (setImmediate) that allows accessing files before cleanup?
4. Are there any Express 5 `app.render` internals that expose EJS options to user-controlled data differently?
5. Could the Info-ZIP Unicode Path Extra Field in yauzl create a validated-path / extracted-path mismatch?
