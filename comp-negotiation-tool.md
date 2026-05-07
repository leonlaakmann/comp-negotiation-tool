# Compensation Negotiation Tool

> **Maintenance note:** This file is kept in sync with the interactive widget in the Claude.ai conversation.
> It is regenerated automatically after every change to the widget.
> To rebuild after an update: copy this file into your Claude Code project and run
> `claude "implement the spec in comp-negotiation-tool.md as index.html"`.

---

## Stack

- Single `index.html` — no build step, no framework, no dependencies except Chart.js via CDN
- Vanilla JS with a global state object and full DOM re-render on every `onchange`
- Chart.js 4.4.1: `https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.js`
- All styling in a single `<style>` block — no external CSS

---

## File structure

```
index.html    ← entire app
```

---

## UI layout (top to bottom)

1. **Tab bar + view toggle row** — package tabs left-aligned, gross/net pill toggle right-aligned
2. **Tax settings bar** — visible only when view = `net`: Steuerklasse select (I / III / IV), Kirchensteuer checkbox, Kinderlosenzuschlag checkbox
3. **Two-column card grid**
   - Left: **Cash card** — base salary input (always gross), bonus % input, net breakdown waterfall (net mode only)
   - Right: **VSOP/Equity card** — options count, strike price, fair value per share, vesting years; paper value + annual equivalent shown below inputs
4. **Benefits card** (full width) — four monthly inputs: bAV, Mobility, Home office, Other; annual total shown below
5. **Package meta row** — editable name input + Remove button (hidden on Current tab) + total comp summary right-aligned
6. **Comparison table** — all packages side by side
7. **Negotiation gap block** — per offer: base increase needed for parity
8. **Stacked horizontal bar chart** — one bar per package
9. **"Ask Claude" button** — copies a formatted summary to clipboard
10. **Disclaimer** — one line

---

## State model

```js
const state = {
  selected: 0,           // index of active tab
  view: 'gross',         // 'gross' | 'net'
  netOpts: {
    sk: 1,               // Steuerklasse: 1 | 3 | 4
    kirche: false,       // Kirchensteuer toggle
    childless: false     // Kinderlosenzuschlag on PV
  },
  packages: [
    {
      label:      'Current',
      base:       90000,   // annual gross base €
      bonusPct:   10,      // bonus as % of base
      vsopOpts:   0,       // number of options / virtual shares
      vsopStrike: 0,       // strike price per share €
      vsopFMV:    0,       // estimated fair market value per share €
      vsopYears:  4,       // vesting period in years
      bav:        50,      // bAV contribution €/month
      mobility:   100,     // mobility budget €/month
      home:       50,      // home office allowance €/month
      other:      0        // other monetary benefits €/month
    }
  ]
}
```

---

## Computed values per package

```
totalGross  = base × (1 + bonusPct / 100)
cash        = view === 'net' ? netOf(totalGross, netOpts).net : totalGross
vsopUpside  = max(0, (vsopFMV − vsopStrike) × vsopOpts)
vsopAnnual  = vsopYears > 0 ? vsopUpside / vsopYears : 0
benefits    = (bav + mobility + home + other) × 12
total       = cash + vsopAnnual + benefits
```

---

## German gross-to-net engine

### Tax parameters (2024/25)

| Parameter | Value |
|---|---|
| Grundfreibetrag | €11,604 |
| Zone 2 ceiling | €17,005 |
| Zone 3 ceiling | €66,760 |
| Zone 4 ceiling | €277,825 |
| BBG Rentenversicherung (West) | €90,600/yr |
| BBG Krankenversicherung | €66,150/yr |
| RV employee rate | 9.3% |
| ALV employee rate | 1.3% |
| KV employee rate | 8.15% (7.3% base + 0.85% avg Zusatzbeitrag) |
| PV employee rate (with children) | 1.7% |
| PV employee rate (childless 23+) | 2.3% |
| Werbungskostenpauschbetrag | €1,230 |
| Sonderausgaben-Pauschbetrag | €36 |
| Soli threshold | €18,130 Lohnsteuer |
| Soli Milderungszone start | €16,956 Lohnsteuer |
| Soli rate (above threshold) | 5.5% of Lohnsteuer |
| Soli rate (Milderungszone) | 11.9% of excess over €16,956 |
| Kirchensteuer rate | 9% of Lohnsteuer |

### Income tax formula (§32a EStG)

```
Zone 1  zvE ≤ 11,604:            tax = 0
Zone 2  11,605 – 17,005:         y = (zvE − 11604) / 10000
                                  tax = (979.18 × y + 1400) × y
Zone 3  17,006 – 66,760:         w = (zvE − 17005) / 10000
                                  tax = (192.59 × w + 2397) × w + 966.53
Zone 4  66,761 – 277,825:        tax = 0.42 × zvE − 10602.13
Zone 5  > 277,825:               tax = 0.45 × zvE − 18936.88
```

### `netOf(gross, opts)` — returns net salary and breakdown

```
rv       = min(gross, 90600) × 0.093
alv      = min(gross, 90600) × 0.013
kv       = min(gross, 66150) × 0.0815
pv       = min(gross, 66150) × (childless ? 0.023 : 0.017)
sozial   = rv + alv + kv + pv

zvE      = max(0, gross − 1230 − 36 − rv − kv − pv)   // simplified Vorsorgepauschale

lohnsteuer:
  if sk === 3:  0.5 × taxF(2 × zvE)    // Ehegattensplitting
  else:         taxF(zvE)               // SK I and SK IV

soli:
  if lohnsteuer > 18130:   lohnsteuer × 0.055
  elif lohnsteuer > 16956: (lohnsteuer − 16956) × 0.119
  else:                    0

ki       = kirche ? lohnsteuer × 0.09 : 0
net      = gross − sozial − lohnsteuer − soli − ki
rate     = (1 − net / gross) × 100    // effective deduction rate

returns: { net, sozial, lohnsteuer, soli, ki, rate }
```

### `findGross(targetNet, bonusPct, opts)` — binary search (60 iterations)

```
lo = 0, hi = 600000
repeat 60 times:
  mid = (lo + hi) / 2
  if netOf(mid × (1 + bonusPct / 100), opts).net < targetNet → lo = mid
  else → hi = mid
return round((lo + hi) / 2)
```

Used in net mode to calculate the gross base an offer needs to reach parity with current net total comp.

---

## Net breakdown panel

Shown inside the Cash card when `view === 'net'`. Waterfall layout:

```
Gross cash (base + bonus)         €XX,XXX
  − Social contributions          −€XX,XXX
  − Income tax (Lohnsteuer)       −€XX,XXX
  − Solidarity surcharge          −€XX,XXX
  − Kirchensteuer  [if enabled]   −€XX,XXX
  ──────────────────────────────────────────
  Net cash                         €XX,XXX
  Effective deduction rate: XX.X%
```

---

## Comparison table

| Column | Width | Notes |
|---|---|---|
| Package | 22% | Bold if active tab |
| Cash / Net cash | 19% | Column header changes label in net mode |
| VSOP/yr | 17% | Annual equivalent, pre-tax |
| Benefits | 17% | Annual total |
| Total | 17% | Bold |
| Δ | 8% | Green if ≥ 0, red if < 0, em-dash for Current |

Use `table-layout: fixed` to prevent overflow at 760px.

---

## Negotiation gap block

Shown below the comparison table when at least one offer exists.

**Gross mode — offer below current:**
```
requiredBase = (current.total − offer.vsopAnnual − offer.benefits) / (1 + offer.bonusPct / 100)
"Offer X: base needs +€XX,XXX (→ €XX,XXX) to match"
```

**Net mode — offer below current:**
```
targetNetCash = current.total − offer.vsopAnnual − offer.benefits
grossNeeded   = findGross(targetNetCash, offer.bonusPct, netOpts)
"Offer X: needs gross base +€XX,XXX (→ €XX,XXX gross) to match in net terms"
```

**Offer already above current:**
```
"✓ Offer X already exceeds current by €XX,XXX"   [green text]
```

---

## Stacked horizontal bar chart (Chart.js)

```
type:      'bar'
indexAxis: 'y'
datasets:
  Cash     background #1D9E75  — netOf(totalGross, netOpts).net in net mode, else totalGross
  VSOP/yr  background #7F77DD  — vsopAnnual
  Benefits background #BA7517  — annual benefits

x-axis ticks: '€XXk'  (v / 1000, 0 decimal places)
y-axis: package labels, no gridlines
Legend: custom HTML above canvas (coloured 10×10px squares + label text)
Canvas wrapper height: max(90, packages.length × 52 + 30) px
Destroy instance with try/catch before each re-render
```

---

## Number formatting

```js
fmt(n)    → '€\u202f' + Math.round(n).toLocaleString('de-DE')
fmtD(n)   → (n >= 0 ? '+' : '') + fmt(n)
rate(r)   → r.toFixed(1) + '%'
```

Narrow no-break space (`\u202f`) between € sign and digits.

---

## Interactions

| Trigger | Behaviour |
|---|---|
| Click tab | Switch `state.selected` |
| Click "+ Add offer" | Push blank package, select it; max 4 total |
| Click "Remove" | Splice current package; clamp `selected`; hidden on Current tab |
| Any number input `onchange` | Parse with `parseFloat(v) \|\| 0`, update state, re-render |
| Package name input `onchange` | Update label, re-render |
| Gross / Net toggle | Set `state.view`, re-render |
| Tax settings `onchange` | Update `state.netOpts`, re-render |
| "Ask Claude" button | Build summary string (see below), copy to clipboard, show "Copied ✓" for 2s |

Use `onchange` (not `oninput`) for all number inputs to avoid focus loss on keystroke re-renders.

---

## "Ask Claude" clipboard text

```
Compensation comparison ({view} view):

{for each package}
{label}: gross base €{base}, bonus {bonusPct}%[, net cash €{net} ({rate}%)]
  VSOP: {vsopOpts} options · strike €{vsopStrike} · FMV €{vsopFMV}
        → paper value €{vsopTotal} (€{vsopAnnual}/yr, pre-tax)
  Benefits: €{benefits}/yr
  → Total {gross|net} comp: €{total}/yr{[ · Δ {fmtD(delta)}] for offers}

Negotiation gap:
{gap lines, or "All offers meet or exceed current."}

Help me think through the negotiation strategy — what to push on, what
trade-offs matter, and how to frame the ask.
```

---

## Dark theme tokens

| Token | Value |
|---|---|
| Page background | `#0f1117` |
| Card background | `#1a1d27` |
| Input background | `#0f1117` |
| Border | `rgba(255,255,255,0.08)` |
| Text primary | `#e8eaf0` |
| Text secondary | `rgba(232,234,240,0.55)` |
| Text tertiary | `rgba(232,234,240,0.35)` |
| Accent green (Cash) | `#1D9E75` |
| Accent purple (VSOP) | `#7F77DD` |
| Accent amber (Benefits) | `#BA7517` |
| Success | `#5DCAA5` |
| Danger | `#e24b4a` |

Font: `system-ui, -apple-system, sans-serif`
Border radius: `10px` cards, `6px` inputs/buttons/tabs

---

## Edge cases

- `vsopFMV ≤ vsopStrike` → paper value = 0 (floor, never negative)
- `vsopYears = 0` → annual equivalent = 0 (guard divide-by-zero)
- `gross = 0` → net = 0, effective rate = 0
- All number inputs: `parseFloat(value) || 0`
- Chart `destroy()` in `try/catch` (canvas may be gone before destroy runs)
- Package default labels: Current, Offer 1, Offer 2, Offer 3

---

## Disclaimer

```
Net calculation: German 2024/25 approximation (§32a EStG). Actual net may differ based on
exact Vorsorgepauschale, individual deductions, and KV Zusatzbeitrag. VSOP figures are
pre-tax. Verify with a Lohnrechner for binding figures.
```
