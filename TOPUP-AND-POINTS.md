# Top up and Points — integration guide

For any site or app (Pentagon's own or a third party's) that wants to **show a
user's Points** and **let them top up**. You do not build a checkout, handle
cards, or hold balances: Pentagon does that at
[`pentagon.games/topup`](https://pentagon.games/topup). Your part is a link or
one function call, and reading one number.

Prerequisite: your users sign in with the pill or the sign-in popup. See
[PENTAGON-LOGIN-STANDARD.md](PENTAGON-LOGIN-STANDARD.md) and
[SIGN-IN-WITH-PENTAGON.md](SIGN-IN-WITH-PENTAGON.md) (that is also where you get
a client id).

## What Points are

- **Points** are a Pentagon account's top-up balance. One name everywhere:
  "Points" (not "PG Points", "NPC", "credits" or "$PC").
- They are bought with a card or crypto at `pentagon.games/topup` and spent
  inside Pentagon apps. They are **in-ecosystem only**: no cash value, not
  withdrawable, not bridgeable, not transferable between accounts.
- They are **not** the `$PC` token and not the chain's gas, even though they are
  recorded on Pentagon Chain. Where a conversion has to be shown, the unit is
  1 PC = 1,000 Points.
- Each account's Points sit in that account's own wallet (`aa_wallet_address`).
  You never send funds to it yourself; top-up does.

## 1. Show the user's Points

**Easiest: the pill shows them.** Once the user is signed in, the pill displays
their Points with the magenta Points mark. Nothing to build.

**Reading the number yourself** (for your own UI):

| You have | Call | Read |
|---|---|---|
| `ssoToken` (every site gets this from sign-in) | `POST https://api.account.pentagon.games/sso/walletinfo` with JSON `{"token": "<ssoToken>"}` | `result.npc_points` |
| the login `token` (Pentagon's own domains only) | `GET https://api.account.pentagon.games/user/walletinfo` with `Authorization: Bearer <token>` | `result.npc_points` |

- `npc_points` is the number to display. It is always present (0 when empty).
  Do not compute Points from any other field, and do not use `pc_balance`,
  `balance` or any legacy address field.
- A `401` means the token expired: sign the user in again.
- **CORS:** being registered for sign-in does not by itself allow your origin to
  call the account API from the browser. If the browser call is blocked, read
  Points on your server and hand the number to the pill:

```js
PCConnector.setPoints(npcPoints);   // null clears it
```

## 2. Let the user top up

### With the pill (recommended)

```js
PCConnector.topUp();
```

This opens `pentagon.games/topup` in a small window. When the user closes it,
the pill re-reads their Points on its own. Call it from a click (browsers block
windows that are not opened by a user action; if it is blocked, the pill
navigates to the page instead so the button never does nothing).

The pill's own panel already has a Top up button; `topUp()` is for your own
"Top up" / "Get more Points" buttons, so the page has one flow, not two.

### Without the pill

Link or open a window to:

```
https://pentagon.games/topup
```

Open it as a **top-level page or a popup window, never in an iframe**: it is a
payment page.

Optional parameters:

| Parameter | What it does |
|---|---|
| `?return_url=<https URL>` | After the top-up, the page shows "Continue →" back to this URL. URL-encode the value. It is honoured only for `https://` URLs on a Pentagon ecosystem domain (see below); anything else is ignored and the user simply gets "Done". |
| `#sso_token=<ssoToken>` | Hands over the user's sign-in so they are not asked to sign in again on the top-up page. It goes in the URL **fragment** (after `#`), which browsers do not send to servers. |
| `?points=<N>` | Makes that package the main button. `N` is one of `100`, `425`, `1000`, `2500`, `5500`; anything else is ignored and 1,000 leads. |
| `?points=<N>&checkout=1` | **Straight to checkout.** Skips the amount screen: as soon as the account is known, the window goes to the Stripe checkout for that package, then comes back, shows delivery, and continues to `return_url` by itself. See below. |

If you pass nothing, the page signs the user in itself (same Sign in with
Pentagon card) and carries on.

#### Which domains `return_url` accepts

Any subdomain of these, over `https://` — so you can check your own URL before
you ship rather than finding out it silently fell back to "Done":

`pentagon.games` · `pentaswap.io` · `etherfantasy.com` · `gunnies.io` ·
`bcsh.xyz` · `gemry.ai` · `rugpull.art` · `etherfamily.com` · `peg.gg`

The restriction is the point, not an inconvenience: an arbitrary `return_url`
would let a crafted link walk a user who has just paid straight off-site. If
your domain belongs on that list, ask — it is a one-line change on our side,
not a new feature.

Do **not** pass a wallet address, and do not try to choose the recipient: the
page resolves the signed-in account's own wallet. Points always go to the
account that is signed in on the top-up page.

### Choosing the amount, and going straight to checkout

Use these when your page has already told the user what they are buying
("Roll for $5"), so Pentagon's amount screen would only be a repeat.

**Pick the default amount** — the user still sees Pentagon's page, with your
amount as the big button:

```
https://pentagon.games/topup?points=100&return_url=https%3A%2F%2Fyour.site%2Fback
```

**Straight to the card checkout** — no amount screen at all:

```js
// from a click handler
const url = 'https://pentagon.games/topup?points=100&checkout=1'
          + '&return_url=' + encodeURIComponent('https://your.site/back')
          + (ssoToken ? '#sso_token=' + encodeURIComponent(ssoToken) : '');
PCConnector.topUp(url);            // or: window.open(url, 'pg-topup', 'width=520,height=760')
```

What happens: the window opens the Stripe checkout for that package, the user
pays, the window shows "payment → delivering → done" (about a minute), then goes
to your `return_url`. If they cancel at Stripe they land on the amount list with
"nothing was charged".

Rules that keep this safe, so you know what to expect:

- Your page must have shown the price and the amount before sending the user.
  Check the current price in the catalogue (`payments.pentagon.games/api/topup/packages`)
  rather than hard-coding it.
- The user must be signed in on the top-up page. Pass `#sso_token=` to avoid a
  second sign-in; otherwise the sign-in card shows first and checkout follows.
- It auto-starts at most once per tab per 30 minutes. Coming back with the
  browser's Back button, or reloading, shows the amount list rather than
  starting a second checkout. If you are testing `checkout=1` and it stops
  firing, that is this rule, not a broken link — use a new tab or wait it out.
- `checkout=1` without a valid `points` does nothing.

## 3. What the user sees

1. Their current Points, and the packages: 100, 425, 1,000, 2,500 or 5,500
   Points. Each shows the balance before and after.
2. Card payment in a Stripe window (card details never touch Pentagon), or
   "Pay with crypto" for the 1,000 Points package (PC on Pentagon Chain, USDC or
   $PC on Ethereum).
3. A progress view: payment, delivery, done. Delivery usually takes about a
   minute, occasionally a few. The user may close the page; delivery completes
   on its own.
4. A receipt at `payments.pentagon.games`.

Prices are set by Pentagon and can change; read them from the page, do not
hard-code them in your own copy.

## 4. Knowing when it finished

There is no callback to third-party pages. Do one of these:

- **Using the pill:** nothing. It re-reads Points when the top-up window closes;
  subscribe if your UI needs the new number:

```js
PCConnector.onChange(function (s) { render(s.points); });
```

- **Using your own window:** when the window you opened closes, read
  `npc_points` again (section 1). Points can land up to a few minutes after
  payment, so if the number has not changed, read again after a short wait
  rather than telling the user it failed.

Never mark an order as paid on your side because the top-up window closed.
Closing the window proves nothing; only the balance does.

## 5. Spending Points

Spending is **server-side only**. A page cannot move a user's Points: the keys
that can are held by Pentagon's backend, not by the browser. Your backend calls
the identity API on the user's behalf with your app key. The endpoints, the app
key and the rules are in the API reference:
<https://blockchainsuperheroes.github.io/pg-identity-docs/>

If your product takes Points as payment, talk to Pentagon before building it:
spends go through the sanctioned payment paths, not through ad-hoc transfers.

## 6. Words to use

| Say | Do not say |
|---|---|
| "Points", "Top up Points" | "NPC", "credits", "PG Points", "$PC" for Points |
| "Top up", "buy Points" | "earn Points" (they are bought, not earned) |
| "Spend Points in Pentagon apps" | "withdraw", "cash out", "send", "convert to cash" |
| "Points have no cash value" | anything implying an investment or a return |

Do not promise refunds in your own copy; refund wording is Pentagon's to give.

## 7. Do not

- Build your own Stripe checkout for Points, or call Pentagon's checkout
  endpoints directly. Send the user to `/topup`.
- Put `/topup` in an iframe.
- Show a Points number you computed yourself. Show `npc_points`.
- Store or forward the user's tokens to any origin other than
  `api.account.pentagon.games`.
- Tell a user their top-up failed because the number has not moved within a few
  seconds.

## Checklist

- [ ] Users sign in with the pill (client id registered for your exact origin).
- [ ] Points shown come from `npc_points` (pill, or section 1).
- [ ] "Top up" calls `PCConnector.topUp()` or opens `pentagon.games/topup` as a
      window or top-level page.
- [ ] After top-up you re-read the balance instead of assuming success.
- [ ] Copy says "Points", bought not earned, no cash value.
