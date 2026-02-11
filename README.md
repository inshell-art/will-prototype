# WILL Prototype (PRNG + Trace Map)

This repo contains two standalone HTML pages:

- `demo_grid_buy_2^n_prng.html` — interactive evolutionary color grid with deterministic PRNG + trace controls.
- `WILLMAP.html` — tree map viewer that visualizes the full trace history as a branching graph.

Both pages are self‑contained (no build step).

## Quick Start

1. Open `demo_grid_buy_2^n_prng.html` in a browser.
2. Enter an **Init Color** (e.g. `#176ac1`) and click **Reset**.
3. Click a mutant tile, then click **Will** to advance.
4. Click **Buy** to double the grid size (2^n refinement).

To view the map:

1. Open `WILLMAP.html` in the same browser.
2. Click **Load From PRNG Storage**.
   - If localStorage is not available (e.g. file:// separation), the map will request the trace directly from the demo page.

## Determinism

The system is deterministic when the **Init Color** and **trace sequence** are the same.
Randomness comes from a PRF keyed by:

```
seed (Init Color) + generation + iteration + candidate index + label
```

So the same trace always reproduces the same results.

## Trace Rules (G‑I‑P)

Each trace row is: **Generation (G), Iteration (I), Pick (P)**.

Rules used by the demo and map:

- `G0 I0 Pick=Genesis` is the starting parent (no PRF applied).
- Candidate mutants for iteration **I** are generated using `iter = I`.
- `Pick` is written into the **current waiting row**.
- **Will** applies the pick and appends a new waiting row.
- **Buy** creates a `Sold` row at `I0` for the new generation and a new waiting row at `I1`.

## Current + Mutants (G‑I‑P Labels)

At any moment on the demo page:

- **Center tile (Current)** = the **last applied pick** row `(G, I, P)`.
  - `P = Genesis` for the very first parent.
  - `P = Sold` when a Buy created the new generation parent.
- **Mutant tiles** are generated from that parent using the **current waiting row’s** `(G, I)` and the **candidate index** `P`.
  - PRF context for a mutant is: `seed + G + I + candidate + label`.

## Trace JSON Format

Exported JSON looks like:

```json
{
  "version": 1,
  "settings": {
    "sigma": 0.05,
    "structProb": 0.5,
    "cellStroke": 0
  },
  "initColor": "#176ac1",
  "rows": [
    {"gen":"0","iter":"0","pick":"Genesis"},
    {"gen":"0","iter":"1","pick":"5"},
    {"gen":"0","iter":"2","pick":""}
  ]
}
```

Multiple branches can be exported as:

```json
{
  "version": 1,
  "settings": { ... },
  "groups": [
    { "initColor":"#176ac1", "rows":[...] },
    { "initColor":"#c01a5b", "rows":[...] }
  ]
}
```

**Fork branches JSON format:** `groups` is an array of independent traces.  
Each group is what you see in one branch panel of the demo UI.

## Controls

### Demo

- **Click** a mutant tile to select it.
- **Will** advances iteration using the selected mutant.
- **Buy** doubles the grid resolution (2^n) using the current parent.
- **Fork** duplicates the current trace branch immediately.
- **Delete** removes the current branch (or resets if it is the only one).
- **Export / Import** a trace JSON.
- **Reset** returns to a single `Genesis` + waiting row.

Keyboard:

- `W` → Will
- `B` → Buy

### Map

- **Load From PRNG Storage** to read the live trace from the demo.
- **Import Trace JSON** to visualize a trace file.
- Hover a node to see the zoomed preview.

## Mutant Probability Design

Each mutant is produced in two stages:

1. **Per‑cell mutation**
   - Each cell mutates with probability  
     `pCell = clamp(0.18 + 1.3*sigma, 0.06, 0.95)`
   - If it mutates, H/S/L deltas are uniform in a ± range:
     - `dh = uniform(-1, 1) * sigma * 0.45`
     - `ds = uniform(-1, 1) * sigma * 0.35`
     - `dl = uniform(-1, 1) * sigma * 0.35`

2. **Structural mutation (optional, once per genome)**
   - Happens with probability `structProb`.
   - If it happens, exactly one of these occurs:
     - **Shift row/column** (50% of structural events)
     - **Swap two cells** (25%)
     - **Duplicate one cell into another** (25%)

## Notes / Limitations

- `sigma` is locked once Init Color is set, to keep a trace consistent.
- If the map cannot access localStorage (e.g. file:// isolation), it will request the trace directly from the demo page using `postMessage`.
- Both pages must be in the same folder for the direct request to work.
