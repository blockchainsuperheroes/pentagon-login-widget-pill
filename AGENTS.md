# AGENTS.md — putting the Pentagon pill on your front-end

This guide is for any agent or developer adding Pentagon login and wallet display
to a Pentagon surface: games, the bridge, staking, payments, the vault, partner
sites. Read it top to bottom once. The normative rules are in
[PENTAGON-LOGIN-STANDARD.md](PENTAGON-LOGIN-STANDARD.md). This file covers how to
integrate the pill and why it looks and behaves the way it does.

**Short version: you should not need to change the pill.** Load it, give it a
place in your nav, and wire your site's own login and wallet prompts to its API.
Everything below explains that, plus the few cases where you do need something
more.

- Live reference: <https://pentagon.games/home2> — the pill in the top-right nav.
- Every state, from fixture data: <https://pentagon.games/connector/>.
- Source of truth: `site/connector/pc-connector.js` in
  `blockchainsuperheroes/pentagon-games-website`, served at
  `https://pentagon.games/connector/pc-connector.js`. The `pill/` folder here is
  a read-only mirror of the current release for review. **Never serve it.**
- Owner: the pentagon.games website session (nftprof decides design).

---

## 1. Install (two lines)

```html
<div data-pc-connector></div>
<script src="https://pentagon.games/connector/pc-connector.js"
        data-client-id="YOUR_CLIENT_ID" defer></script>
```

- `data-pc-connector` marks where the pill goes, normally the right end of the
  nav. It renders in Shadow DOM, so your CSS can't break it and its CSS can't
  leak into your page.
- `data-client-id` is **required on any origin other than pentagon.games**.
  First ask nftprof to register your *exact* origin with identity. Without a
  client id, the pill still shows wallet balances but hides every sign-in entry,
  so it never offers a login that would be refused.
- Optional: `data-points-label="…"` relabels Points, both in the panel and in
  the pill's short form. The defaults are `Points` and `Pts`. See §8 for the
  copy rules.
- Optional: `<div data-pc-guide></div>` shows the inline "what should I do next"
  panel, for onboarding pages.

### Your site must also have

1. **An RPC allowlist entry.** Balances come from `rpc.pentagon.games`, which
   only answers allow-listed origins. Ask the cli-control-ui session to add your
   exact origin (dashboard → RPC CORS), then confirm it:
   ```sh
   curl -s -D - -o /dev/null -X POST -H "Origin: https://your.site" \
     -H "Content-Type: application/json" \
     --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' \
     https://rpc.pentagon.games | grep -i access-control-allow-origin
   ```
   Until that works, the pill says it couldn't read balances.
2. **CSP entries**, if you send a CSP:
   - `script-src https://pentagon.games`
   - `connect-src https://rpc.pentagon.games https://api.account.pentagon.games https://api.peg.gg https://ethereum-rpc.publicnode.com https://eth.llamarpc.com https://cloudflare-eth.com`

   All brand marks are inlined, so no image rule is needed.
3. **Pop-ups allowed from a click.** Sign-in (off pentagon.games), Top up and
   Bridge each open a window. The pill opens them inside the user's click, so
   browsers allow them. If you call the API yourself, do it inside a click
   handler too (see §4).
4. **A referrer.** Do not send `Referrer-Policy: no-referrer`. The bridge
   popup uses the referrer to decide whether it may report back (§7). The
   browser default, `strict-origin-when-cross-origin`, is what pentagon.games
   uses.

---

## 2. The one-widget rule

**The pill is the only login and wallet widget on the page.** nftprof's
wording: "u don't need two thing that display points, u just need the single
widget pill".

- Do **not** add a separate "Log in" button, "Connect wallet" button, Points
  badge or balance chip in your nav. home2 removed all of its own when the pill
  landed.
- Page content can still say "Log in to claim" or "Top up". Route those
  buttons to the pill's API (§4), so there is still only one login.
- If your app already has a wallet stack (wagmi, RainbowKit, web3modal), keep
  it and **attach** it to the pill (§4). Don't render two Connect buttons.

---

## 3. What the pill shows, and why

The pill is compact: numbers and marks, no words it doesn't need. The panel
that opens when you click it spells everything out.

### The two marks

| Mark | Means | Rule |
|---|---|---|
| **Pink pentagon** | Points: the account's in-ecosystem balance | The mark comes before the number. On the pill it replaces the word "Points". nftprof: "don't show Points word, just show the pink pentagon". |
| **Neon green pentagon** | $PC / PC where the wallet is now | On Ethereum it reads `$PC` (the ERC-20). On Pentagon Chain it reads `PC` (the gas coin). |

### Pill states

| State | Pill | Notes |
|---|---|---|
| Signed out, no wallet | `[PC logo] Log in ▾` | Reads `Connect wallet` on a site with no client id. |
| Signed in, no wallet | `[pink] 3,690 Pts ▾` | Just the Points. The username is in the panel. nftprof: "when only login no wallet connect then show only pink 3,690 Pts". |
| Wallet connected, logged out | `[neon] 8.781 $PC │ Ethereum ▾`<br>`Rabby Wallet · 0x1d187a…d3017a` | No Points segment. The panel offers "Not logged in to Pentagon · Log in with Pentagon". |
| Signed in **and** connected | `[pink] 3,690 · [neon] 8.781 $PC │ Ethereum ▾`<br>`Rabby Wallet · 0x1d187a…d3017a` | Points, then PC where you are, then the network. nftprof: "when login and connected should show both". |
| Connected on another chain | `[chain] Polygon │ Switch ▾` | That chain holds no PC. The panel offers the switch. |
| Pentagon AI wallet (phone, read-only) | Same layout; row 2 reads `Pentagon AI · 0x…` | Shown only when our backend says the account has one. It is never offered to someone who can't use it. |

**Second row = the wallet's own address format**, lowercase `0x` + 6 … 6. The
panel shows the *same string*. nftprof: "so when user click wallet they get
comfort and understanding seeing two are the same rather then guessing".

The caret `▾` is always there. It is what tells people the pill opens.

### The panel (click the pill)

Three boxes:

1. **Account** — "Signed in as **name**", with the Points on a row below
   (pink mark, number, "Points"), and **Top up**. Logged out: "Not logged in to
   Pentagon · Log in with Pentagon".
   - **PNS (1.1.0):** when the account has a PNS name, it is the name shown,
     with a `PNS` tag: `Signed in as nftprof [PNS]`. With none: `Get PNS ↗`,
     opening pns.pentagon.games **in a new tab**. With no wallet connected, the
     same appears under "Hi name —".
   - A name counts only when it is minted **and** spatially bound to the
     account's wallet (`mm_address`), the rule login resolution uses. There is
     no on-chain reverse lookup, so the pill asks PNS's index
     (`api.peg.gg/api/nft/pegnames/owner/<mm_address>`, the same call "My
     Names" on pns.pentagon.games makes), then confirms each binding on-chain
     (`spatialBinding(uint256)` on the registry
     `0xf97EB9f8293D1FD5587a809Eb74518c300738d07`, Pentagon Chain).
   - **Fails closed:** if the index or the chain can't be read (for example
     api.peg.gg refusing your origin), neither the tag nor "Get PNS" shows, so
     nobody who has a name is told to get one.
2. **NOW** — the connected wallet (name, address → explorer link), the PC on
   the chain it is on now, with the big neon mark and that chain's main action:
   - on Ethereum, **Bridge to Pentagon Chain**;
   - on Pentagon Chain, "you're set" or top-up guidance.

   Also here: **Switch to Ethereum / Switch to Pentagon Chain**, and
   **Disconnect wallet**.
3. **ALSO** — the PC on the *other* chain, dimmed. It lights up when the NOW
   chain has 0 and the other chain has some, because then it matters.

**Link this wallet (1.1.0, pentagon.games only):** signed in, a wallet connected,
and identity says the account has **no** linked wallet (`mm_address` empty) → the
NOW box offers "Your account has no linked wallet yet. **Link this one…**". It opens
a confirmation: "⚠ One-time link — pick carefully", what it means (the wallet can
log you in; PNS names bind to it), and **"You can't change or unlink it yourself"**
— unlinking takes a support ticket on Pentagon's Discord, gated members only.
"I understand this link is permanent" must be ticked before one free signature of
identity's bind message (`POST user/bind_metamask`). **This is the one place a
wallet gets linked**: the Pentagon AI apps show the linked wallet read-only and
send "Link a wallet" to pentagon.games (products-wallet-rn SPEC §25c-ii). Don't
build a bind flow on your own site.

**Footer (signed in):** **Account & privacy ↗** opens `pentagon.games/account` in a
new tab, on its own row above **Sign out** / **Disconnect wallet**. Privacy decides
what *other people* see when they look you up (pump.pentagon.games, Friends).
`/account` is the one address for account settings: the old settings page today,
forwarded to the Pentagon AI app's settings once the app can take them
(products-wallet-rn SPEC §25). Don't build account settings on your own site; link
there.

Signed out, with no wallet, the panel has **one Connect** plus **Log in with
Pentagon**. Connect logs the visitor in by itself when the wallet belongs to an
account: a single login-message signature, never a transaction. Nothing is sold
before login, so the signed-out panel has no "Pay by card". nftprof: "they are
not buying anything yet".

---

## 4. Wiring your site to the pill (API)

`window.PCConnector` exists once the script has run. With `defer`, that is
before `DOMContentLoaded`.

| Call | Use it for |
|---|---|
| `PCConnector.signIn()` | Every "Log in" prompt on your page. It opens the pill's own sign-in. **Call it inside a click handler**, because off pentagon.games the sign-in is a popup. |
| `PCConnector.signOut()` | Your account menu's sign-out. It clears both tokens and fires `pg:auth`. |
| `PCConnector.topUp(href?)` | Every "Top up" link. It opens pentagon.games/topup in a popup and re-reads Points when the popup closes. It keeps `?return_url=…` on any `https://pentagon.games/topup…` href. If the popup is blocked, it navigates instead. |
| `PCConnector.setPoints(n)` | Pass Points your **server** read, when the browser can't read them (§6). `null` clears. |
| `PCConnector.attach(provider, address, { connect, name, icon })` | Hand your own wallet connection to the pill. Call it on connect, on account change and on chain change. |
| `PCConnector.detach()` | Your user disconnected. |
| `PCConnector.state()` | `{ account, chainId, pcGas, ethPc, eth, points, recommendation }` |
| `PCConnector.onChange(fn)` | Called whenever that state changes. |
| `PCConnector.connect()` / `switchToPentagonChain()` | Open the pill's connect, or request the chain switch. |
| `PCConnector.mount(el)` / `unmount(el)` | SPAs that re-render the nav: `unmount` before the element goes, `mount` when it is back. |
| `PCConnector.version` | `'1.1.0'` |

**Event:** `window` receives `pg:auth` (`CustomEvent`, `detail.ok` /
`detail.signedOut`) on sign-in and sign-out. Listen for it and your page state
stays in step with the pill. The pill also listens, so a sign-in your page
performs shows up in the pill.

Routing every Top up link through the popup, as home2 does:

```js
document.addEventListener('click', e => {
  const a = e.target.closest('a[href*="/topup"]');
  if (!a || !window.PCConnector) return;
  e.preventDefault();
  PCConnector.topUp(a.href);
});
```

Attaching an existing wagmi / RainbowKit connection:

```js
watchAccount(config, { onChange: async ({ address, connector }) => {
  if (!address) return PCConnector.detach();
  PCConnector.attach(await connector.getProvider(), address, {
    connect: () => openConnectModal(),   // the pill's Connect opens YOUR modal
    name: connector.name, icon: connector.icon,
  });
}});
```

While attached, the pill follows your provider and never asks the wallet for
anything itself, except Switch / Add Pentagon Chain on that same provider. It
also shows no Disconnect of its own: disconnecting is yours.

---

## 5. Responsive behaviour: compact is built in

The pill compacts itself by viewport width, and your nav doesn't need to do
anything:

| Width | What drops |
|---|---|
| ≤ 560px | the username and the `·` separators, and the padding tightens |
| ≤ 480px | the network name and the wallet name on row 2; the address stays |
| ≤ 359px | the Points segment when a wallet is connected (the PC figure is what matters there), and the `Pts` word |

Measured in home2's nav, the pill fits at 320px (40px tall, inside a 60px
nav). The panel is `min(310px, 100vw − 24px)` wide, anchored right.

**Mobile browsers.** Phones usually have no extension wallet. There the pill
offers **Log in with Pentagon** and, only when our backend says the account
has one, the **Pentagon AI** phone wallet (read-only balances). Nothing is
offered that the phone can't do. Wallet in-app browsers (MetaMask or Rabby
mobile) announce their provider like the extensions do, so they work as
"connected".

**If you need it more compact than this**, for example a crowded mobile nav at
a width where the pill doesn't compact yet:

1. First give the pill room. Shrink your logo to its mark, hide nav text
   links behind the menu. home2 does both at ≤430px.
2. If that isn't enough, ask for a host option, such as a
   `data-compact` attribute. Ask in the website session, or open a PR against
   `pentagon-games-website/site/connector/pc-connector.js` (§9). **Do not
   restyle it from outside or fork a copy.** Every surface should keep reading
   the same way.

**Theming.** The pill takes your `--pg-*` CSS variables when you define them,
and otherwise uses the Pentagon neon palette. Tokens it reads:
`--pg-accent`, `--pg-accent-soft`, `--pg-accent-text`, `--pg-on-accent`,
`--pg-text`, `--pg-text-muted`, `--pg-surface`, `--pg-surface-raised`,
`--pg-border`, `--pg-border-strong`, `--pg-danger-text`, `--pg-warning`,
`--pg-radius`, `--pg-radius-small`, `--pg-font-body`, `--pg-font-display`.
The two brand marks (pink Points, neon $PC) are fixed and never themed.

---

## 6. Points on your origin

Reading Points calls `api.account.pentagon.games`. **Registering for sign-in
does not add your origin to that API's CORS allowlist.** The two lists are
separate. If the read is refused, the pill drops the Points segment instead of
showing a misleading `0`.

- Ask for your origin to be added to the API allowlist, **or**
- read Points server-side (`GET /user/walletinfo` with the user's token) and
  call `PCConnector.setPoints(n)`.

---

## 7. Caveats: things that open another site

### Bridge (Ethereum $PC → Pentagon Chain)

- **Bridge to Pentagon Chain** opens `https://bridge.pentagon.games/?mode=popup`
  in a **popup window**. The panel says so under the button: "Opens
  bridge.pentagon.games in a window. Your wallet may ask to connect there the
  first time."
- It is a window, not an in-page frame. Wallets don't reliably give a provider
  to a cross-origin frame, and approvals belong on the bridge's own origin.
- The pill passes two **hints**: `wallet=<rdns>`, so the bridge connects the
  same wallet, and `amount=<$PC seen, rounded down to 6 dp>`. The bridge's
  terms, review and every wallet prompt still run. **A second wallet connect
  can't be fully avoided.** The bridge is another origin, so the wallet may ask
  to connect once there.
- The bridge reports progress back (`{type:'pg:bridge', status}`) **only to
  its allowlisted openers**: `pentagon.games`, `www.pentagon.games`,
  `getpc.pentagon.games` and `bridge.pentagon.games`, matched against
  `document.referrer`. On any other origin the pill still re-reads balances
  when the window closes. It just doesn't get live progress. To be added,
  ask the bridge2-frontend session.
- If the popup is blocked, the pill navigates to the bridge instead.
- **Money pages** (the bridge itself, staking, checkout) must load a
  **pinned** build (§10), not the auto-updating URL.

### Top up (card → Points)

- Opens `pentagon.games/topup` in a popup. The purchase happens on
  pentagon.games. When the window closes, the pill re-reads Points.
- It needs the visitor's pentagon.games session. A visitor without one is asked
  to sign in *inside that window*.
- **Cancel top up** is in there. It checks first that no payment went through.
- **All five amounts sell** — 100 / 425 / 1,000 / 2,500 / 5,500 Points. 1,000
  leads and the rest fold under "More amounts"; `?points=<N>` promotes another
  one (see [TOPUP-AND-POINTS.md](TOPUP-AND-POINTS.md)).
  This paragraph used to say the opposite — that only one amount was safe and
  the other four would charge without delivering. That was true when the page
  trusted Stripe's `metadata.sku_id`, which still carries `0` on four of the
  five products. /topup no longer reads it: it maps each Stripe `priceId` to
  its fulfilment SKU itself, so all five deliver. Payments confirmed delivery
  on all five.

### Other

- **GetPC** ("Lock on GetPC") is built but switched off (`GETPC_CTA = false`)
  until it is cleared.
- **Faucet** links are paused on home2 while the faucet is down. They come
  back when nftprof says so.
- **Pentagon AI (phone) wallet** balances are read-only. Roaming *signing*
  ("approve in my Pentagon AI app") is not built yet. The Pentagon AI app
  itself does not show PC on Ethereum; the pill does, with Bridge as the main
  CTA.
- **Account switches in Rabby** and other wallets that switch without an
  event are picked up by polling `eth_accounts` / `eth_chainId` every 4s
  while the tab is visible. Wallets that emit `accountsChanged` update
  instantly.
- **Native and in-app surfaces** (iOS/Android apps, game clients) are the
  standard's one exception. See the standard.

---

## 8. Copy rules (legal: don't paraphrase)

Points are an **in-ecosystem balance**: bought, not earned; spend-only. Never
describe them as convertible, withdrawable or bridgeable, and never say they
have a "value" or are "worth" anything. The pill's copy already follows this.
If you relabel with `data-points-label`, keep it neutral.

The pill **never moves anything**. Its only wallet requests are connect,
add/switch network, one signature of Pentagon's login message, and (1.1.0,
pentagon.games only) the one-time "link this wallet" signature of identity's
bind message — only after the user confirms it is permanent.

---

## 9. When you really do need to change the pill

1. Open a PR against **`blockchainsuperheroes/pentagon-games-website`**,
   file `site/connector/pc-connector.js`. Don't edit the `pill/` mirror here,
   and don't copy the file into your repo.
2. Run the checks there:
   ```sh
   node scripts/pill-probe.mjs      # end-to-end: demo, home2, three boxes, wallet switching, top up, bridge
   ./scripts/gate.sh                # guardrails + render audit
   ```
   Both need `PW_CHROMIUM_PATH` pointing at a Chromium.
3. Bump the `version`, cut `site/connector/pc-connector-X.Y.Z.js`
   (byte-identical), record its sha384 in `docs/CONNECTOR.md`, and bump `?v=`
   where the pill is loaded.
4. Changes to anything a surface depends on (API, events, the two-row layout,
   marks) go to nftprof first.

---

## 10. Money pages: pin a version with SRI

```html
<script src="https://pentagon.games/connector/pc-connector-1.1.0.js"
        integrity="sha384-7D5DZJE9+If+3I6DD2vFDIZaODA2753ilrKXIipBIGVDqKLbedCVYXB9RKpZMkhH"
        crossorigin="anonymous" data-client-id="YOUR_CLIENT_ID" defer></script>
```

Pinned files never change. A new release gets a new filename and hash. The
version table with every hash is `docs/CONNECTOR.md` in pentagon-games-website.
Upgrading is your decision, made when you want what a newer row adds.

---

## 11. Adoption checklist

- [ ] Origin registered with identity; `data-client-id` set
- [ ] Origin on the RPC allowlist (curl check passes)
- [ ] Points readable (API allowlist), **or** `setPoints()` from the server
- [ ] PNS readable: `api.peg.gg` answers your origin (else the PNS tag / Get PNS link simply don't show)
- [ ] CSP entries added (if you send a CSP)
- [ ] Old login buttons, Points badges and balance chips removed from the nav
- [ ] Page "Log in" prompts call `PCConnector.signIn()` in the click
- [ ] Top up links go through `PCConnector.topUp(href)`
- [ ] An existing wallet stack is **attached**, not duplicated
- [ ] Account-menu sign-out calls `PCConnector.signOut()`; page listens for `pg:auth`
- [ ] Money pages load the **pinned** build with SRI
- [ ] Checked at 320 / 390 / 768 / 1280px: signed out, signed in, connected, both
