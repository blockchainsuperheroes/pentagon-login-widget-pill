# Pentagon Games Identity Standard — the login pill

**Status: required for every Pentagon front-end.** This supersedes every
hand-rolled login form, every "sign in with wallet" button, and the older
guidance in this repo and in `pg-identity-docs` that told you to build your own
form. If something you maintain has its own Pentagon login, it is out of date.

**Pentagon login is now the pill**, and the pill is the **gateway to the
wallets** — ours and everyone else's. One component gives a site: Pentagon
sign-in, the user's Points, and optionally a connected web3 wallet (PGAI,
MetaMask, Rabby, or a Pentagon AI app approving from a phone). You do not
implement any of it, and there is no separate "sign in with wallet" button to
build: connecting a wallet *is* one of the ways in.

This is a deliberate simplification. Sign-in, balance display and wallet
connection used to be three separate integrations that every site built
differently. They are now one step.

```html
<div data-pc-connector></div>
<script src="https://pentagon.games/connector/pc-connector.js"
        data-client-id="YOUR_CLIENT_ID" defer></script>
```

On `pentagon.games` omit `data-client-id`. Everywhere else it is required, and
your **exact** origin must be registered first (scheme + host; `www` and every
subdomain are separate). Ask nftprof.

**See every state live: [pentagon.games/connector/](https://pentagon.games/connector/)**
— the shipping pill rendered in each state this document describes, from
fixture data, so you can see what your users will see without owning the
account, wallet or phone each state needs. Pages that move funds pin a
versioned build with SRI instead of the URL above: see
[CONNECTOR.md](https://github.com/blockchainsuperheroes/pentagon-games-website/blob/main/docs/CONNECTOR.md).

---

## Why a pill and not a login button

**Because a login button alone leaves the user unable to see what they own.**

- MetaMask will not show `$PC` on Pentagon Chain unless the user has manually
  added chain 3344 as a custom network. Almost nobody has.
- No wallet can ever show **Points**. Points are an account balance, not a
  token — there is nothing for a wallet to display.

So a user connects MetaMask, sees nothing, and concludes they hold nothing.
Showing balances is therefore **not a nice-to-have in this design — it is the
reason the component exists.** A site that takes the login and drops the
balances has reimplemented the problem.

---

## Normative requirements

1. **MUST use the pill for Pentagon sign-in.** No site may collect a Pentagon
   password on its own origin. Passwords are typed on `pentagon.games` only.
2. **MUST display the pill's content** — balances included. You **MAY**
   re-theme it (the pill inherits your CSS custom properties). You **MUST NOT**
   take the login and discard the balance display.
   And the pill is the **only** login widget on the page: you **MUST NOT**
   render a second account chip, Points display, Log in / Sign up button or
   Top up button beside it. Signed in with no wallet the pill is just the
   Points ("3,690 Pts ▾") — the account name lives in the panel, not inline on
   the pill; it reads "Log in" when signed out, and carries sign-in, Top up and
   sign-out. Two widgets showing the same balance is the drift this standard
   exists to stop — pentagon.games itself shipped exactly that, and removed it.
   Other prompts on your page that ask someone to log in call
   `PCConnector.signIn()`; they do not open a login of their own.
   The pill shows at **every width** — on a phone it is often the only way in.
3. **MUST NOT redirect users to the wallet app to log in.** Sign-in is a popup
   over your page. If the popup is blocked, re-prompt from a button; never
   fall back to navigating away.
4. **MUST re-validate server-side** before granting anything that matters.
   Pass the token to your backend, call `GET /user/info`, and build your
   session from that response — not from what the popup reported.
5. **MUST NOT pass a Pentagon login token to another origin.**
6. **SHOULD call the identity API from your server, not the browser.** Being
   registered for sign-in does **not** put your origin in the API's CORS
   allowlist — they are separate lists and they differ. A server-side read also
   means your page shows a balance it cannot forge. Hand the Points you read to
   the pill with `PCConnector.setPoints(n)`, so requirement 2 holds on an
   origin the API will not answer.
7. **MUST sign out through the pill.** If your page has its own account menu,
   its Sign out calls `PCConnector.signOut()`. That clears both `pg_token` and
   `pg_sso_token` and fires `pg:auth`. A second, hand-rolled sign-out that
   clears only one of them leaves the user signed in to every other Pentagon
   site on that machine while your page says they left.

### The one exception: native and in-app surfaces

A popup is wrong inside a native app or a webview, and a user already signed
into the host app must not be asked again. Those surfaces take a **session
handoff from the host app** (postMessage / an injected global), not the pill.
`ar.etherfantasy.com` is the current example. If you are unsure which you are,
ask before building.

---

## The account model this standard assumes

| | What it is | Shown as |
|---|---|---|
| **Pentagon account** | The identity. Works with no wallet at all. | Sign-in, Points |
| **Points (PG Balance)** | Custodial, non-transferable, spend-only in-ecosystem. Bought, not earned. | A balance in the pill |
| **Connected web3 wallet** | The user's own, optional | `$PC` gas on Pentagon Chain, `$PC` on Ethereum (the figure, with **Bridge to Pentagon Chain** as the main action), and **the connected network, named on the pill** |
| **PGAI wallet** | Self-custody, for users who have no wallet. **Pentagon Chain 3344 only.** | Its Pentagon Chain gas. **Never** an Ethereum prompt: PGAI cannot sign on Ethereum, so the pill never sends it to the bridge |

**The account is the baseline; a web3 wallet is optional.** A user with no
wallet must be able to sign in, see their Points, and spend them. Sites that
require a wallet connection to do anything have the model backwards.

---

## Two entry orders, both required

**Sign in first, then connect a wallet.** After connecting, the pill checks the
wallet against the account's own addresses (`mm_address`, `penai_address`).
If it is neither, the pill **flags it and carries on** — this is not an error.
Web3 functions still work (mining can be paid from any wallet); the pill simply
must not imply that wallet belongs to the account, and Points stay with the
account regardless.

**Connect a wallet first, then find its account.** The wallet signs Pentagon's
message and identity returns the account that owns that address. If no account
exists, the pill offers to create one and connect it. This **is** the old "sign
in with wallet", with one fewer click — the user connects, and sign-in follows.
In the pill this is **one button**: a signed-out visitor sees only *Connect
wallet* and *Log in with Pentagon*. Connect connects, then asks the wallet to
sign Pentagon's login message; if an account owns the wallet they are logged
in, and if they decline they simply stay connected. There is no separate
"connect & sign in" button to choose between.

There is deliberately **no address→account lookup**. The signature is what
proves control, so you can only learn a wallet has a Pentagon account if you
own that wallet. An open lookup would let anyone link wallets to Pentagon
membership.

---

## Roaming ("approve in my Pentagon AI app")

Offered **only when the backend says the account has a Pentagon AI wallet**
(`user/penai/history`). A wallet on a phone or in Telegram is invisible to
EIP-6963 — the browser genuinely cannot detect it — so this cannot be guessed.
It fails closed: on any error the option is hidden rather than dead-ending
someone who never installed one.

**The phone wallet's balance works today.** When a signed-in account has a
Pentagon AI wallet (`penai_address` on `user/info`) and no wallet is connected,
the pill offers "Show my Pentagon AI wallet": it reads that address's balances
directly — no provider, no connection, no signature — names the wallet and its
network, and labels itself read-only. No PGAI address on the account, no offer.

**Roaming sign-in works today. Roaming *signing* — approving a transaction on
your phone from a third-party page — does not exist yet and is deferred.**
This is canonical (nftprof, 2026-09-28): deferred, not cancelled. It is
securely buildable; see `PILL-AND-WALLETS.md` in the website repo for the
conditions. Do not design a flow that depends on it.

---

## Migrating an existing login

1. Ask nftprof to register your exact origin; you get a client id.
2. Add the two lines at the top of this document.
3. Point your existing login click at the pill. Keep your own form only as a
   popup-blocked fallback.
4. Re-validate the token server-side, then create your session as you do today.
5. Delete your captcha, email-verification and password-reset code. The pill
   covers password login (email / username / PNS name), approve-from-your-app,
   email-only sign-up, forgot password, and the first-login set-password step.
6. Leave wallet-signature login and your web3 connect flow alone if you have
   them — those are a separate axis.

---

## Open question: should the agent live in the pill?

**Recommendation: no — keep it in the wallet, with at most one entry point in
the pill.**

The pill ships on every page of every Pentagon site. It must stay small,
and safe to embed anywhere; it never moves anything and never sends a
transaction (its signatures are Pentagon's login message and, on pentagon.games
only, a confirmed one-time wallet-link message), and that property is what makes it uncontroversial to drop into
a partner's nav. An agent is the opposite: stateful, conversational, and able
to take actions, which needs the wallet's trust context and per-request
approval. Putting it in the pill would put an acting surface on every partner
page and grow a component whose whole value is that it is small.

If the agent should be reachable from anywhere, the right shape is a single
button in the pill that opens it in the wallet — not the agent itself.

This is a recommendation, not a decision. nftprof's call.
