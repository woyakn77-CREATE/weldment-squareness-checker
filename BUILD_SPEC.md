# Build Spec: Weldment Squareness Check Tool

**Audience:** An AI assistant building this tool for the Kenall welding team.
**Owner:** Kenall Quality Engineering
**Output:** One self-contained HTML page (HTML, CSS, and JavaScript in a single file, no external dependencies except optional web fonts).
**Primary use:** Desktop browser. Must also work on a phone.

---

## 1. Purpose

Welders need to confirm a rectangular weldment (frame) is square. They measure the two cross-corner (diagonal) dimensions with a tape. This tool converts the frame's length, width, and allowed corner angle tolerance into:

1. The **nominal** diagonal (perfectly square, 90.000 degrees).
2. The **minimum** and **maximum** diagonal that keep the corner within tolerance.
3. A **dual scale**: degrees off square on top, matching diagonal dimension on the bottom.
4. Optional: a pass/fail verdict for a measured diagonal.

---

## 2. Inputs

| Field | ID | Default | Unit | Rules |
|---|---|---|---|---|
| Units toggle | `u-in` / `u-mm` | Inches | in or mm | Switching converts the length, width, and measured values (x25.4 or /25.4) |
| Length (L) | `len` | 48 | in/mm | Must be > 0 |
| Width (W) | `wid` | 24 | in/mm | Must be > 0 |
| Angle tolerance (±) | `tol` | 0.5 | degrees | Must be > 0 and < 45. Hint text: "Default ±0.5° unless the print says otherwise." |
| Measured diagonal | `meas` | **Blank** | in/mm | Optional. Locked until Length and Width are valid (see below) |

Length, width, and tolerance recalculate the readouts and scale live on every keystroke (`input` event). There is no Calculate button.

### 2.0 Measured diagonal behavior

1. Starts blank and disabled, with placeholder "Enter length and width first" and a dashed border.
2. When the user leaves the Length or Width box (`change` event) and both hold valid numbers above zero, enable it. The placeholder changes to "e.g. 53 3/4" (in) or "e.g. 1363.5" (mm).
3. If Length or Width is cleared or made invalid, disable it again and ignore its value.
4. The verdict and measured marker update only on `change` (Done button, Enter, or clicking off the box), not on every keystroke. This stops the verdict from flashing while a value like `53 3/` is half typed.
5. Hint text: "Measure corner to corner. Check both diagonals and enter each one. The check runs when you tap Done or click off the box."

### 2.1 Accepted input formats (inches)

Length, width, and measured diagonal accept decimals, fractions, and mixed numbers. Show the hint "Decimal or fraction: 48.5, 48 1/2, or 48-1/2" under Length while in inches.

| Typed | Parsed value |
|---|---|
| `48` / `48.5` / `.5` | 48 / 48.5 / 0.5 |
| `3/4` | 0.75 |
| `48 3/4` | 48.75 |
| `48-3/4` | 48.75 |
| `48 3/4"` or `48 3/4in` | 48.75 (trailing inch mark ignored) |
| `48,5` | 48.5 (comma as decimal) |
| `53 3/` (partial while typing) | Invalid; treat as no value, no crash |

Parser rules (regex, after trimming and removing a trailing `"` or `in`):
1. Decimal: `^(?:\d+\.?\d*|\.\d+)$`
2. Fraction: `^(\d+)\s*/\s*(\d+)$` (denominator must not be 0)
3. Mixed: `^(\d+)(?:\s+|\s*-\s*)(\d+)\s*/\s*(\d+)$`

**Phone keyboard:** in inches mode set `inputmode="text"` on these three fields so the space, slash, and dash keys are available. In mm mode set `inputmode="decimal"`. Also set `autocomplete="off"`, `autocapitalize="off"`, `spellcheck="false"`.

---

## 3. The Math

### 3.1 Formula

The law of cosines gives the diagonal across a corner with angle θ:

```
d² = L² + W² - 2·L·W·cos(θ)
```

Let the corner be off square by δ degrees, so θ = 90° + δ. Then cos(90° + δ) = -sin(δ), and:

```
d(δ) = sqrt( L² + W² + 2·L·W·sin(δ) )
```

| Output | Formula |
|---|---|
| Nominal | d(0) = sqrt(L² + W²) |
| Max diagonal | d(+T) (corner at 90 + T) |
| Min diagonal | d(-T) (corner at 90 - T) |

### 3.2 Reverse: measured diagonal to degrees off square

```
δ = asin( (D² - L² - W²) / (2·L·W) )     (result in degrees)
```

- If the asin argument is outside -1 to +1, the measurement is physically impossible. Show "Check entry" and tell the user to re-measure and check L and W.
- Pass if |δ| ≤ T. Otherwise fail.

### 3.3 Note on the two diagonals

On an out-of-square frame, one corner is 90 + δ and the adjacent corner is 90 - δ. One diagonal reads long and the other reads short by about the same amount. The tool checks one diagonal at a time. Instructions tell the welder to enter both.

### 3.4 Display precision

- Inches: 3 decimals
- Millimeters: 2 decimals
- Degrees: 2 decimals on the scale, 3 decimals in the verdict text
- Fractions (inches mode only): to 1/32, reduced (16/32 shows as 1/2), with a trailing `"`

### 3.4.1 Fraction rounding rule

| Value | Rounding | Why |
|---|---|---|
| Nominal, scale ticks, measured value | Nearest 1/32 | General reference |
| **MIN** limit | **Round up** to the next 1/32 | Keeps the fractional limit inside tolerance |
| **MAX** limit | **Round down** to the previous 1/32 | Keeps the fractional limit inside tolerance |

Rounding the limits to the nearest 1/32 can place the fraction just outside the true limit. A welder who measures exactly that fraction would then pass a part that is slightly out of tolerance. Pass/fail is always calculated from the exact decimal value, not the fraction.

### 3.5 Test cases (use these to verify the build)

| L | W | T | Expected Nominal | Expected Min | Expected Max |
|---|---|---|---|---|---|
| 48 in | 24 in | 0.5° | 53.666 (53 21/32") | 53.478 (53 1/2") | 53.853 (53 27/32") |
| 72 1/8 in | 42-1/2 in | 0.5° | 83.715 (83 23/32") | 83.395 (83 13/32") | 84.034 (84 1/32") |
| 1219.2 mm | 609.6 mm | 0.5° | 1363.11 | 1358.34 | 1367.86 |

| L | W | T | Measured | Expected δ | Expected verdict |
|---|---|---|---|---|---|
| 48 | 24 | 0.5° | 53.750 | +0.225° | In tolerance |
| 48 | 24 | 0.5° | 53 3/4 (also 53-3/4) | +0.225° | In tolerance |
| 72 1/8 | 42-1/2 | 0.5° | 83 5/8 | -0.141° | In tolerance |
| 48 | 24 | 0.5° | 53.950 | +0.761° | Out of tolerance |
| 48 | 24 | 0.5° | 53.400 | -0.707° | Out of tolerance |
| 48 | 24 | 0.5° | 80.000 | Not possible | Check entry |

---

## 4. Output Layout

### 4.1 Readouts

Three tiles in a row (stack to one column on narrow screens):

| Tile | Value | Sub-line |
|---|---|---|
| Min | d(-T), then fraction to the right (e.g. `53.478  53 1/2"`) | "-0.188 in (corner 89.50°)" |
| Nominal | d(0), then fraction to the right (e.g. `53.666  53 21/32"`) | "Corner at 90.00°" (highlighted tile, Kenall Blue background, white text) |
| Max | d(+T), then fraction to the right (e.g. `53.853  53 27/32"`) | "+0.187 in (corner 90.50°)" |

Fraction styling: same line as the decimal (flex row, baseline aligned, wraps if narrow), about 19 px, semibold, 2767 80% (`#33507D`); 2767 30% (`#B2BDCE`) on the blue Nominal tile. Hide the fractions in mm mode.

### 4.2 Verdict bar

| State | Pill text | Color |
|---|---|---|
| No measured value | No reading | Gray |
| Within tolerance | In tolerance | Green |
| Outside tolerance | Out of tolerance | Red, plus text: "Hold the part and contact the Quality team." |
| Impossible value | Check entry | Red |

Verdict sentence format: "Measured 53.750 in (53 3/4") is +0.084 in from nominal: +0.225° off square (limit ±0.5°)." (Leave out the fraction in mm mode.)

### 4.3 Dual scale (SVG)

This is the key visual. Draw it as inline SVG, `viewBox="0 0 1000 262"`, width 100%.

1. **Range:** -1.5T to +1.5T degrees. The axis maps linearly in diagonal length (bottom values), so convert each degree tick with d(δ).
2. **Center bar:** horizontal band 18 px tall.
   - Green between d(-T) and d(+T).
   - Red outside on both ends.
3. **Ticks:** every T/4 degrees (13 ticks total). Major ticks every T/2 are longer and labeled.
4. **Top labels:** degrees off square, signed (e.g. "-0.50°", "0.00°", "+0.25°").
5. **Bottom labels:** matching diagonal dimension for the same tick.
   - **Inches mode:** a second line under each decimal shows the 1/32 fraction (14 px). Use the Section 3.4.1 rounding: up at -T, down at +T, nearest elsewhere.
6. **Captions under the bottom labels:** "MIN" at -T, "NOMINAL" at 0, "MAX" at +T. Put them below the fraction row in inches mode and directly under the decimals in mm mode.

Vertical positions (bar center yA = 108): top degree labels at y 74, bottom decimals at y 154, fractions at y 175, captions at y 198 (in) or 174 (mm), bottom axis title at y 252.
7. Make the 0 and ±T labels bold in Kenall Blue. Make other labels regular gray.
8. **Axis titles:** "DEGREES OFF SQUARE" above, "DIAGONAL (IN)" or "DIAGONAL (MM)" below.
9. **Measured marker:** vertical Kenall Blue line with a downward triangle above the bar. If the value is off the scale, pin it to the end and label it "Off scale above" or "Off scale below".
10. Wrap the SVG in a container with `overflow-x: auto` and give the SVG `min-width: 640px` so the labels never overlap on a phone.
11. Add a legend below: green = within tolerance, red = out of tolerance, blue = measured diagonal.

### 4.4 How to measure (instructions panel)

1. Measure both diagonals, corner to corner, at the same reference points (outside corners or marked centers).
2. Enter each diagonal in the Measured field and read the angle above the marker.
3. **Both readings in the green band:** the corner is square within tolerance.
4. **Either reading in the red zone:** hold the part, tag it, and contact the Quality team before any rework.

---

## 5. Branding (Kenall Brand Identity Manual)

### 5.1 Colors

| Token | Use | Hex | Source |
|---|---|---|---|
| Kenall Blue (PMS 2767) | Header band, headings, nominal tile, marker, active toggle | `#00245D` | p.12 |
| 2767 80% | Unit labels | `#33507D` | p.12 |
| 2767 50% | Input borders | `#7F91AE` | p.12 |
| 2767 30% | Secondary text on blue | `#B2BDCE` | p.12 |
| 2767 10% | Page background | `#E5E9EF` | p.12 |
| 90% Black | Body text, scale ticks | `#404041` | p.12 |
| 50% Black | Neutral pill | `#939598` | p.12 |
| 20% Black | Panel borders | `#D1D3D4` | p.12 |
| Pass green (Controls market color) | In-tolerance band and pill | `#41AD49` | p.12 |
| Fail red (Exit & Emergency market color) | Out-of-tolerance band and pill | `#E31823` | p.12 |

### 5.2 Typography

- Brand font is **Acumin Pro** (p.13). Use it if it is installed or licensed in your environment.
- Web fallback: **Archivo** from Google Fonts (variable width). Use condensed widths (`font-stretch: 75-80%`) for headings and labels to match Acumin Pro Condensed.
- Use tabular numerals (`font-variant-numeric: tabular-nums`) for all numbers.
- Uppercase labels get letter-spacing of about 0.14em.

### 5.3 Logos

| Logo | Placement | Rules |
|---|---|---|
| Kenall logo, reversed white | Header, on the Kenall Blue band | Only approved colors: PMS 2767, black, 90% black, or reversed white on 2767/black (p.9). Clear space on all sides = 1/9 of logo width (p.8). Minimum width 3/4 in. Never stretch, recolor, or add effects (p.11). |
| "A brand of Legrand" colored lockup | Footer, on white | Black symbol and text, Red 032 "legrand" (p.18-19). Keep clear space. |

- Get the logo files from Kenall Marketing. Do not redraw the logo.
- Embed logos as base64 data URIs so the page stays one file.
- **Approval:** Per p.3, anything with the Kenall logo needs written approval from Kenall Marketing (marketing@kenall.com) before distribution.

### 5.4 Theme

Light theme only. The logo rules allow Kenall Blue only on white backgrounds, so do not build a dark mode.

### 5.5 Page structure

1. **Header band:** Kenall Blue, white logo on the left, "QUALITY ENGINEERING / Welding inspection tool" on the right.
2. **Title:** "Weldment Squareness Check" plus a one-sentence description.
3. **Two columns** (stack under 820 px wide): Inputs panel (about 340 px) on the left, Results panel on the right.
4. **How to measure** panel, full width.
5. **Footer:** white with a 4 px Kenall Blue top border. Left: "KENALL QUALITY ENGINEERING", "Questions or out-of-tolerance parts: contact the Quality team.", and the formula. Right: Legrand lockup.

---

## 6. Required Behaviors and Known Bugs to Avoid

| # | Requirement | Why |
|---|---|---|
| 1 | Add a **Done** button inside each input field that shows only while the field has focus (`.field:focus-within .done { display: block; }`). Clicking it calls `blur()` on the input. | **Bug found in testing:** on iPhone, `inputmode="decimal"` opens a number pad with no return key, and users could not close it. |
| 2 | Set `enterkeyhint="done"` on every input. Pressing Enter blurs the field. | Desktop and keyboards that have a return key |
| 3 | Tapping anywhere outside an input field blurs the active input. | Second way to close the keypad on a phone |
| 4 | On the Done button, call `preventDefault()` on `pointerdown` before blurring. | Stops focus from jumping back to the input |
| 5 | Use `inputmode`, not `type="number"` (text in inches mode, decimal in mm mode; see 2.1). Accept a comma as the decimal separator. | `type="number"` rejects mixed numbers like `48 3/4` and adds spinner arrows |
| 5a | In each input row, give the flex container `min-width: 0` and the input `flex: 1 1 0; width: 0; min-width: 0`. | **Bug found in testing:** without this, the inputs' built-in width pushed the fields past the edge of the Inputs panel |
| 6 | Wrap the `<form>` submit in `preventDefault()`. | Stops page reloads |
| 7 | Remember the unit choice in `localStorage`, wrapped in try/catch. | Convenience only; the page must work without it |
| 8 | No text may overlap in any chart or table. Wrap text, or scroll inside its own container. | Readability standard |
| 9 | Show inline error text for invalid L, W, or T. Never use `alert()`. | Clear feedback |
| 10 | Visible keyboard focus outline on every control. | Accessibility |

---

## 7. Acceptance Checklist

- [ ] All Section 3.5 test cases give the expected values, including the fractions.
- [ ] `48 3/4`, `48-3/4`, `3/4`, and `48 3/4"` all parse correctly. A partial entry like `48 3/` does not crash the page.
- [ ] On a phone in inches mode, the keyboard has space and slash keys.
- [ ] Measured diagonal loads blank. It stays locked while Length or Width is blank, and unlocks after both are filled and the user clicks off.
- [ ] Typing in Measured diagonal does not change the verdict until the user clicks off or taps Done.
- [ ] MIN fraction rounds up and MAX fraction rounds down.
- [ ] Input fields stay inside the Inputs panel at desktop and phone widths.
- [ ] Unit toggle converts the values and the scale relabels to MM.
- [ ] Changing the tolerance updates the green band width and all tick labels.
- [ ] Measured marker lands at the right spot. Off-scale values pin to the edge with a label.
- [ ] On a phone: tap a field, type, tap Done, and the keypad closes.
- [ ] On a phone: no horizontal page scrolling (only the scale scrolls inside its box).
- [ ] Colors match Section 5.1 hex values.
- [ ] Logos come from official files, keep clear space, and are not distorted.
- [ ] No emojis and no em dashes anywhere in the page text.
- [ ] Kenall Marketing approval obtained before sharing outside Quality.
