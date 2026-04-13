# Apple Pay In-Store V2 — Prototype Context for Claude Code

This file gives Claude Code full context on this project so you can pick up where Cian left off.

---

## What this project is

A redesign exploration for Affirm's Apple Pay in-store checkout flow. The existing flow (V0) has a 13% drop-off on amount entry because users don't realise they're setting up a payment plan — many think Affirm is a credit card that works in store without setup.

**V1** (previously tested) collapsed the multi-step flow into a single page with real-time plan loading. It improved comprehension significantly. **V2** is the UXR Round 2 iteration, now constrained by engineering reality (plans can't load in real time — they load on a separate screen after the user submits an amount).

---

## Prototype files

All files are single HTML files using React + Babel (no build step). They deploy automatically to GitHub Pages on every push to `main`.

Base URL: `https://cianlavin-blip.github.io/applepay-instore-singlepage-uxr/`

### UXR Round 2 cohort prototypes (active)

| File | Cohort | Purpose | URL |
|------|--------|---------|-----|
| `cohort-a.html` | **Cohort A** | NTP — new to Affirm. v2e flow + account creation screen after phone entry | `…/cohort-a.html` |
| `cohort-b.html` | **Cohort B** | Existing user, no card. Phone entry → card signup → add to Wallet → RTP checkout flow | `…/cohort-b.html` |
| `cohort-c.html` | **Cohort C** | Existing user, has card. v2d unchanged | `…/cohort-c.html` |

### Base prototypes (reference)

| File | Purpose | URL |
|------|---------|-----|
| `v2d.html` | **RTP** — full flow for users who already have an Affirm account | `…/v2d.html` |
| `v2e.html` | **NTP** — full flow for first-time users, includes onboarding splash + phone entry | `…/v2e.html` |
| `v2c.html` | Earlier RTP prototype (superseded by v2d) | `…/v2c.html` |
| `v2b.html` | Amount entry variants A/B/C/D (earlier exploration) | `…/v2b.html` |
| `index.html` | Original exploration (earliest version) | base URL |

---

## Running locally

```bash
npx serve /path/to/v2-exploration -p 4000
```

Then open:
- http://localhost:4000/v2d.html
- http://localhost:4000/v2e.html

No install needed. All dependencies (React, Babel, ReactDOM) load from CDN.

---

## How to deploy changes

Just push to `main`. GitHub Pages auto-deploys within ~1–2 minutes.

```bash
git add cohort-a.html cohort-b.html cohort-c.html
git commit -m "feat: your change description [XYZ-000]"
git push
```

Commit message format: `<type>(<scope>): <subject> [XYZ-000]` (Affirm convention — use `XYZ-000` when there's no Jira ticket).

---

## cohort-a.html — NTP + Account Creation phase state machine

Based on v2e, with an extra account creation step inserted between phone entry and amount input.

```
wallet-ntp → add-to-wallet → pay-later-options → ntp-splash
→ generic-loading-1 (3s) → phone-entry → generic-loading-2 (3s)
→ account-creation → generic-loading-3 (3s)
→ input → loading (5s) → terms → review
→ [confirming 2.5s] → adding-card (4s)
→ [isCardAddedExiting 0.45s] → card-added (0.5s) → wallet-complete
```

**Differences from v2e:**
- `generic-loading-2` → `account-creation` (not `input`)
- New `account-creation` phase: prefilled form (name, DOB, email), "Continue to plans" CTA → `generic-loading-3`
- New `generic-loading-3` (3s) → resets amount/KB state → `input`
- `ntpMidPhases` includes `account-creation` and `generic-loading-3`
- `sheetOpen` allowlist includes `account-creation` and `generic-loading-3`
- `overflowY: 'auto'` also applies to `account-creation`

---

## cohort-b.html — Existing user, no card — phase state machine

NTP phone entry → card signup → add to Wallet → lands back at wallet → then full RTP checkout flow.

```
wallet-ntp → add-to-wallet → pay-later-options → ntp-splash
→ generic-loading-1 (3s) → phone-entry → generic-loading-2 (3s)
→ card-signup → [user taps Accept & Add]
→ adding-card (4s) → [isCardAddedExiting 0.45s] → card-added (0.5s)
→ wallet [Pay Later row animates in] → modal → input → loading (5s)
→ terms → review → [confirming 2.5s] → wallet-complete
```

**Key differences from v2e:**
- `generic-loading-2` → `card-signup` (not `input`)
- `card-signup` phase: agreement screen with Affirm card mockup, checkbox + terms, "Accept & Add to Wallet" CTA → `adding-card`
- After `card-added` exit: goes to `wallet` (not `wallet-complete`) — the user now starts the checkout flow
- `payLaterVisible` state (`useState(false)`): set to `true` when `card-added` transitions to `wallet`, never unset. Controls Pay Later row visibility with slide-up animation. Stays `true` for entire rest of session so it doesn't re-animate on modal→wallet navigation.
- `isComplete = phase === 'wallet-complete'` — controls Pay Later row *content* (Choose a Plan vs approved amount). Separate from `payLaterVisible`.
- Confirm handler goes to `wallet-complete` (not `adding-card`) — card already added
- `ntpMidPhases` includes `card-signup`
- `sheetOpen` allowlist includes `card-signup`
- `overflowY: 'auto'` also applies to `card-signup`
- Assets used: `./card-signup-mockup.png` (232px, phone showing Affirm card in Wallet) and `./icon-bolt.svg`

---

## cohort-c.html — Existing user, has card

Identical to v2d. No differences.

---

## v2d.html — RTP phase state machine

```
wallet → modal → input → loading (5s) → terms → review
→ [confirming 2.5s] → wallet-complete
```

- `wallet` — iOS Wallet screen with Affirm card + "Pay Later / Choose a Plan" row
- `modal` — blur overlay with "Choose a Plan" card
- `input` — Affirm bottom sheet, amount entry + numpad
- `loading` — ghost cards spread + shimmer (5s auto-advance)
- `terms` — real plan cards appear, user selects a plan
- `review` — review screen with payment timeline
- `wallet-complete` — sheet slides down, Pay Later row updates with approved amount

**Key state vars:**
- `phase` — drives everything
- `rawAmt` — string of typed digits
- `kbOpen` — whether numpad is visible
- `spread`, `shimmer` — ghost card animation states
- `selectedPlan` — `'p4'` | `'p6'` | `'p12'`
- `termsVisible` — true once terms content has entered (slight delay after `loading`)
- `isConfirming` — spinner state on Accept Terms button
- `ghostOpacity` — fades ghost cards out once real cards appear
- `cardTopPx` / `cardMoveDur` — ghost card vertical position + transition duration
- `headingTransition` — `'input-exit'` | `'loading-exit'` | `null` — drives heading slide animations

**sheetOpen:** `!['wallet', 'wallet-complete', 'modal'].includes(phase)` — everything else shows the sheet

---

## v2e.html — NTP phase state machine

```
wallet-ntp → add-to-wallet → pay-later-options → ntp-splash
→ generic-loading-1 (3s) → phone-entry → generic-loading-2 (3s)
→ input → loading (5s) → terms → review
→ [confirming 2.5s] → adding-card (4s)
→ [isCardAddedExiting 0.45s] → card-added (0.5s) → wallet-complete
```

- `wallet-ntp` / `add-to-wallet` / `pay-later-options` — NTP overlay on top of wallet (full-screen dark overlay, z:5)
- `ntp-splash` through `review` — Affirm bottom sheet (same sheet as RTP, z:20)
- `adding-card` / `card-added` / `wallet-complete` — full-screen white overlay showing card being added to Wallet (z:45)

**Extra state vars vs v2d:**
- `isCardAddedExiting` — two-phase exit animation: outer container fades (opacity), status text slides up (translateY)
- `ntpEntryPhases` = `['wallet-ntp', 'add-to-wallet', 'pay-later-options']`
- `ntpMidPhases` = `['ntp-splash', 'generic-loading-1', 'phone-entry', 'generic-loading-2']`

**sheetOpen:** explicit allowlist: `['ntp-splash', 'generic-loading-1', 'phone-entry', 'generic-loading-2', 'input', 'loading', 'terms', 'review', 'purchasing-power'].includes(phase)`

**Cancel button in sheet:** for NTP mid phases, cancel goes back to `wallet-ntp` (not `wallet`).

**Affirm logo in NavBar:** hidden only on `ntp-splash` (logo already appears in the content). Visible on all other sheet phases. Controlled via `hideLogo={phase === 'ntp-splash'}` prop on `NavBarApple`.

---

## Shared architecture patterns

### Timer cleanup
All auto-advance timers use a `timerRefs` ref so they can be cancelled on cancel/navigation:
```js
const timerRefs = React.useRef([]);
const clearAllTimers = () => { timerRefs.current.forEach(clearTimeout); timerRefs.current = []; };
```
Always call `clearAllTimers()` in cancel handlers before `setPhase(...)`.

### Ghost card + keyboard hide conditions
The ghost card deck and numpad are hidden (not just covered) for certain phases. Currently excluded:
- Ghost card: hidden when `phase === 'terms' || phase === 'review' || phase === 'purchasing-power'`
- Keyboard: hidden when `!kbOpen || phase === 'purchasing-power'`

### Card position alignment (v2e adding-card screens)
The Affirm card must sit at exactly **98px from the top** across `adding-card`, `card-added`, and `wallet-complete` so it appears stationary during transitions. This is: 44px (StatusBar) + 52px (spacer matching WalletScreen header height) + 2px (padding). Do not change these values.

### Sheet background
- `ntp-splash` phase: white background
- All other sheet phases: `c.bg` (`#f2f2f4`)

### overflowY in sheet content
- `terms`, `review`, `purchasing-power`: `'auto'` (scrollable) — in v2d/v2e/cohort-c
- Additionally `account-creation` (cohort-a) and `card-signup` (cohort-b) are also scrollable
- Everything else: `'hidden'`

---

## Design tokens

```js
const c = {
  bg:     '#f2f2f4',  // page/sheet background
  surface:'#ffffff',  // card/panel background
  nav:    '#f3f4f2',  // nav bar
  text:   '#0c0c14',  // primary text
  sub:    '#4b4b5c',  // secondary text
  accent: '#000061',  // CTA buttons, selected states, links
  link:   '#000061',  // underlined links
  bdg:    '#e2e2ff',  // badge background
  bdgTxt: '#4242cf',  // badge text
  div:    '#cbcbd4',  // dividers
};
```

**Fonts:**
- Headlines / large amounts: `'Axiforma for Affirm', sans-serif` (Bold 700)
- Body / UI: `'Calibre', sans-serif` (Regular 400, Medium 500, Semibold 600)
- iOS-native text (wallet, system UI): `system-ui, sans-serif`

Both fonts load from `https://www.affirm.com/fonts/src/` via `@font-face`.

---

## Purchasing Power screen

Accessible from amount entry via the ⓘ icon after "Spend up to $4,000."

- Phase: `'purchasing-power'`
- Entry: tap ⓘ → `setPhase('purchasing-power')`
- Exit: page-level × button or "Got it" CTA → `setPhase('input')`
- Illustrations: `./pp-battery.svg` (top, 144px) and `./pp-changes.svg` (inside "Why does it change?" accordion)
- The `PurchasingPowerScreen` component is defined as a standalone function before `App` in both files

---

## CSS keyframe animations

Defined in `<style>` block at the top:

| Name | Used for |
|------|----------|
| `fadeIn` | General fade-in on phase enter |
| `slideUpIn` | Headings and cards entering from below |
| `slideUpOut` | Headings exiting upward |
| `blink` | Cursor blink in amount input |
| `spin` | Loading spinner |
| `shimmer` | Ghost card shimmer effect |
| `fadeOut` | AddingCardScreen exit (v2e) |

---

## Figma files

| File | Key | Notes |
|------|-----|-------|
| Original designs (reference) | `UGM9YQiIYYK3WCYL0lRKaT` | All production + V1/V2 concept designs |
| V2 exploration captures | `oR4ZPAvDKqRKvOZ5ZiQQPT` | Cian's personal Figma team |

**Key node IDs in `UGM9YQiIYYK3WCYL0lRKaT`:**
- `4511:9824` — V1 single-page concept (9 states, pink nav arrows)
- `4878:17979` — Amount entry screen (the one that CAN be changed)
- `4878:18093` — Loading screen (CAN be changed)
- `4878:18099` — Terms/plans screen (CANNOT be changed per engineering constraint)
- `6272:33436` — Amount entry with ⓘ purchasing power icon
- `6350:31122` — Purchasing power education screen

---

## Known gotchas

1. **Apostrophes in JS string arrays** — `'you're'` inside a single-quoted string breaks JSX parsing and causes a blank page. Use `\u2019` for curly apostrophes inside JS string literals. Apostrophes in JSX text content (between tags) are fine.

2. **`layoutSizingHorizontal/Vertical = 'FILL'` in Figma plugin API** — must be set AFTER `parent.appendChild(child)`.

3. **Ghost card position** — if you change card position, make sure to update all three screens in v2e that share the same card position (adding-card, card-added, wallet-complete) or the card will jump during transitions.

4. **Auto-advance timers** — always push new timers into `timerRefs.current` so they get cleaned up on cancel. Don't set timers directly in `useEffect` without tracking them.

5. **ts-node** — for the main product-flows codebase, always use `./node_modules/.bin/ts-node`, not the global `ts-node` (not in PATH).

---

## Assets in this directory

| File | What it is |
|------|------------|
| `affirm-card.png` | Affirm Debit Flex Visa card image |
| `affirm-logo.svg` | Affirm wordmark (used in bottom sheet NavBar) |
| `apple-card.png` | Apple Card image (for wallet screen) |
| `pp-battery.svg` | Purchasing power battery illustration (top of PP screen) |
| `pp-changes.svg` | Purchasing power graph illustration ("Why does it change?") |
| `card-signup-mockup.png` | Phone mockup showing Affirm card in Wallet (cohort-b card signup screen) |
| `icon-bolt.svg` | Lightning bolt icon (cohort-b card signup screen) |
| `icon-*.png` | iOS Wallet app icons (Klarna, Apple Card, Pay Later, etc.) |

---

## Research background (for context)

**Why 13% drop-off on amount entry:** users didn't know they were setting up a payment plan. Many thought Affirm is a credit card — just tap and go.

**What V1 fixed:** showing plans load in real time as you type made it obvious that entering an amount = creating a payment plan. Comprehension improved significantly.

**Why V2 can't do real-time:** engineering constraint — underwriting can't run in real time while typing.

**What V2 is trying to achieve:** reproduce V1's comprehension improvement through the bottom sheet UI and clear framing, within the constraint of submit-then-load.

**Important failed experiment:** a copy experiment added "You can estimate. We'll adjust your payment plan to match the final purchase." This reduced amount-entry drop-off but caused users to round up, leading to worse plan terms and higher total flow drop-off. **Do not use vague "you can estimate" framing.**

**Plan adjustment mechanism (still unconfirmed):** believed to be that APR and payment count are locked once selected, and only the dollar amount of each installment scales to match final purchase price. This needs confirmation before finalising any copy that references it.
