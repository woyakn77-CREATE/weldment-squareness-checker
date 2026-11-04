# Weldment Squareness Check

Legrand Quality Engineering tool for the welding team. Enter a weldment's length, width, and corner angle tolerance (default ±0.5°). The tool returns the nominal, minimum, and maximum cross-corner (diagonal) dimensions, in decimals and to the nearest 1/32", on a scale that pairs degrees off square with diagonal length.

## Use

Open `index.html` in any browser. It is one self-contained file and needs no install or internet connection, except for the web font, which falls back to Arial when offline.

## Language

Use the EN | ES toggle in the header to switch between English and Spanish. The choice is remembered on each computer.

## Inputs

| Field | Accepts |
|---|---|
| Length, Width | Decimals (48.5), fractions (3/4), mixed numbers (48 3/4 or 48-3/4) |
| Angle tolerance | Degrees, default 0.5 |
| Measured diagonal | Optional. Unlocks after Length and Width are filled |

## Math

d = sqrt(L² + W² + 2·L·W·sin(δ)), where δ is the corner's deviation from 90°.
MIN fraction rounds up and MAX fraction rounds down, so both limits stay inside tolerance.

## Files

| File | Purpose |
|---|---|
| `index.html` | The tool |
| `BUILD_SPEC.md` | Full specification for rebuilding or changing the tool |

## Branding

Legrand branded (Process Black, Red 032, and the colored Legrand logo). Confirm logo use with Legrand Marketing before distributing outside Quality.

Owner: Legrand Quality Engineering
