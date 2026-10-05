# Sign in with Pentagon

One script. Your users sign in with their Pentagon Games account without leaving your site, and you get a token you can use straight away.

This is the recommended integration. The hand-rolled login form in this repo still works, but with this you don't handle passwords, captchas, sign-up, password resets or the Pentagon AI app approval flow yourself — and you get new sign-in methods as they ship.

```html
<script src="https://pentagon.games/pgai/web-local-app/pg-signin.js"
        data-client-id="YOUR_CLIENT_ID"></script>
```

```js
document.getElementById('login').addEventListener('click', function () {
  PGSignIn.open().then(function (r) {
    if (!r.ok) return                // r.reason: 'cancelled' | 'popup-blocked'
    startSession(r.token || r.ssoToken)
  })
})
```

Call it from a real click. Browsers block popups that aren't opened from a user action, and you'll get `{ok: false, reason: 'popup-blocked'}`.

## Getting a client id

Email nftprof@pentagon.games with your **exact site origin** (scheme + host, e.g. `https://app.example.com`) and we register it. One entry per origin: `https://example.com` and `https://www.example.com` are different, and so is every subdomain.

Until your origin is registered, the sign-in window shows "… isn't approved to use Sign in with Pentagon. Nothing was shared." and closes. Nothing leaks either way.

You still need an **App Key** (`X-PG-App-Key`) for your own API calls. Same request.

## What you get back

```js
{ ok: true, token: '<login JWT>', ssoToken: '<site-scoped token>' }
```

| | What it is | Use it for |
|---|---|---|
| `ssoToken` | Site-scoped, 24h. Every site gets this. | `POST /sso/walletinfo`, `/sso/validate`, `/sso/user_roles` |
| `token` | The login access token — the same one `POST /user/login` returns. **Pentagon's own domains only** (see below). | `GET /user/info`, `GET /user/walletinfo`, `POST /user/aa/execute`, everything else that takes a Bearer token |

Both are also written to your own site's `localStorage` (`pg_sso_token`, `pg_token`), so a reload keeps the session. Read them with `PGSignIn.ssoToken()` / `PGSignIn.token()`.

**Which domains get the login token.** "Pentagon's own domains" means these
five **and every subdomain of them**: `pentagon.games`, `gunnies.io`,
`etherfantasy.com`, `etherfantasy.io`, `nftmining.com`. So
`tcg.etherfantasy.com` and `mine.pentagon.games` receive the login token;
an unrelated domain receives only `ssoToken`. Your origin must still be
registered either way.

There is no refresh token. When a call returns 401, clear it and call `PGSignIn.open()` again.

### ⚠️ Calling the API from the browser needs your origin in the CORS allowlist

Getting a client id registers you for **sign-in**. It does **not** add your origin to the
identity API's CORS allowlist — those are two separate lists, and several registered sites
are not on the second one. If they differ, a browser `fetch` to
`api.account.pentagon.games` fails preflight even though sign-in worked perfectly.

Check yours before you build:

```bash
curl -si -X OPTIONS https://api.account.pentagon.games/user/walletinfo   -H "Origin: https://your.site"   -H "Access-Control-Request-Method: GET"   -H "Access-Control-Request-Headers: authorization" | grep -i access-control-allow-origin
```

No header back means no browser access. **Prefer calling the API from your own server
anyway** — pass the token to your backend, call identity there, and return only what the page
needs. It avoids CORS entirely, and the page then shows a balance it cannot forge. If you
genuinely need browser-side calls, ask for a CORS entry when you request your client id.

## API

| Call | Does |
|---|---|
| `PGSignIn.open(opts?)` | Overlay on pentagon.games, popup elsewhere. Returns `Promise<{ok, token?, ssoToken?, reason?}>` |
| `PGSignIn.openPopup(opts?)` | Always a popup |
| `PGSignIn.token()` | Login token on this site, or `null` |
| `PGSignIn.ssoToken()` | Site-scoped token, or `null` |
| `PGSignIn.signOut()` | Forgets both on this site |

`opts.clientId` overrides `data-client-id`.

## Arriving already signed in (`#sso=`)

From pg-signin.js 2026-10-05 (fragment hand-off with audience check): when the
Pentagon AI wallet opens your site for a signed-in user, it appends
`#sso=<token>` to your URL. The script picks it up on load, so the user is not
asked to log in again. You don't write any code for this.

What the script does, in order:
1. Removes `sso` / `sso_token` from the URL before any network call. Other
   fragment params stay. The token never sits in history or leaves in a Referer.
2. Checks the token with `POST /sso/validate`, and accepts it only if the
   token's client is your `data-client-id` and its registered origin is your
   page's origin. A token minted for another site is refused.
3. Stores it as `pg_sso_token` and fires the same `pg:auth` event as a popup
   login, with `detail: { ssoToken, via: 'fragment' }`.

What you need:
- `data-client-id` set on the script tag. Without it, hand-offs are refused.
- Your page listed in the wallet's hand-off list. Ask Pentagon to add it.

The token is only ever in the fragment (`#`), never in the query string.
pentagon.games' own pages are skipped: there the session is already in
storage, and /topup reads its own hand-off codes.

## What the user sees

One window covering every way into a Pentagon account:

- email, username or PNS name + password
- **Approve from my Pentagon AI app** — no password: they type their account name, your page shows a 2-digit number, and they tap the matching number in the Pentagon AI app where they're already signed in (phone, Telegram, or the Chrome extension). The account's seed never moves.
- sign up (email only — we send them a link; Pentagon usernames are `user<id>` and a real name comes from a [PNS name](https://id.peg.gg))
- forgot password, and the first-login "set your password" step

## How it stays safe

Worth knowing, because it's why this is the only supported way to embed Pentagon sign-in:

- **Your origin is checked before anything is shown.** The window calls `POST /sso/authorize` with your client id and the origin that opened it. Not registered, not shown.
- **The result goes to your origin only.** It's posted with `targetOrigin` set to your exact origin, and the script checks it came from the window it opened, carrying a one-time random `state`. A page on any other origin receives nothing.
- **Popup, never an iframe on your site.** The sign-in page refuses to be framed anywhere but pentagon.games (`frame-ancestors 'self'`), so no site can wrap it in its own chrome and phish a password.
- **The sign-in window never asks for a seed phrase, a private key or a signature.** If anything claiming to be Pentagon sign-in does, it isn't ours.
- **Don't proxy or re-host the script or the sign-in page**, and don't pass the login token to another origin.

## Troubleshooting

| Symptom | Cause |
|---|---|
| `popup-blocked` | Not called from a click, or the browser blocked it. Re-prompt from a button. |
| "isn't approved…" | Origin not registered, or it doesn't match exactly (www, subdomain, http vs https). |
| `ok: true` but no `token` | Expected on third-party sites: use `ssoToken` with the `/sso/*` endpoints. |
| 401 on `/user/*` | Token expired. Clear it and sign in again. |
| CORS error calling the API | Your origin isn't in the API's CORS allowlist (separate from the client id). Call identity from your server instead — see above. |

Questions: nftprof@pentagon.games. API reference: https://blockchainsuperheroes.github.io/pg-identity-docs/
