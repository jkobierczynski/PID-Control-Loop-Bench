# PID Control Loop Bench

A single-page, client-side control-systems workbench: design a PID or arbitrary
Laplace-domain compensator, run it against any plant transfer function, and see
step/ramp/sine response, Bode magnitude/phase, Nyquist, root locus, and a
pole-zero map update live — with pan/zoom and rearrangeable panels.

Everything lives in **one HTML file** (`loop-bench.html`). There is no server,
no build step, and no network dependency beyond loading the two Google Fonts
(IBM Plex Sans / IBM Plex Mono) — the page works fine offline once fonts are
cached, and still functions (with fallback system fonts) with no network at all.

## Is the math actually happening in real time?

Yes. Every keystroke or control change (Kp/Ki/Kd, a Laplace expression, the
plant preset, the test signal, dragging a slider) fires a single JS function,
`computeAll()`, which re-parses both transfer functions from scratch and
re-runs the *entire* pipeline — root finding, state-space simulation, the
320-point frequency sweep, and the root-locus gain sweep — synchronously, in
the browser, before repainting all six canvases. There's no server round-trip,
no caching of "unchanged" results, and no debounce hiding latency: it's cheap
enough (a few milliseconds for typical system orders) to just redo it all,
every time. Panning/zooming a chart skips the recompute and only re-draws
from the last computed result, via a separate `redrawAll()` path, so dragging
stays smooth even though editing a field re-derives everything.

## Math and algorithms used

No math, plotting, or charting library is used anywhere — no math.js,
numeric.js, Chart.js, D3, Plotly, etc. Everything below is hand-written
vanilla JavaScript in the `<script>` block, operating on plain arrays and
`[re, im]` pairs.

- **Complex arithmetic** — a minimal set of functions (`cAdd`, `cSub`, `cMul`,
  `cDiv`, `cAbs`, `cArg`, `cScale`) operating on 2-element `[re, im]` arrays.
- **Polynomials** — represented as plain arrays of real coefficients in
  descending power order. `polyAdd`, `polyScale`, `polyMul` (convolution),
  and `polyTrim` implement the algebra needed to multiply/add transfer
  functions (e.g. forming the open-loop `L = C·P` and closed-loop
  `T = L/(1+L)`).
- **Laplace expression parser** — a small hand-written tokenizer plus a
  recursive-descent parser (grammar: expression → term → unary → power →
  atom) that turns text like `(s+1)/(s^2+2s+1)` directly into two polynomial
  coefficient arrays. It supports `+ - * ^ ( )`, implicit multiplication
  (`2s`, `(s+1)(s+2)`), and exactly one top-level `/` to split
  numerator/denominator. There is no symbolic engine underneath — it builds
  the coefficient arrays as it parses.
- **Root finding** — the **Durand–Kerner (Weierstrass) method**, a
  simultaneous-iteration algorithm that finds *all* complex roots of a
  polynomial at once from a fixed set of spread-out initial guesses. Used for
  open- and closed-loop poles/zeros, and re-run at every one of the ~90 gain
  steps in the root locus sweep. Capped at degree 12 for numerical
  reliability.
- **State-space simulation** — the closed-loop `T(s)` is converted to
  **controllable canonical form** (A, B, C, D matrices) and integrated with a
  classic 4th-order **Runge–Kutta (RK4)** stepper to produce the step/ramp/sine
  time response, with the step count scaled to the simulated duration.
- **Frequency response** — `L(jω)` is evaluated directly via Horner's method
  in complex arithmetic over a 320-point logarithmically spaced sweep from
  0.01 to 1000 rad/s, feeding the Bode magnitude/phase and Nyquist plots.
- **Phase unwrapping** — a standard cumulative ±360° correction pass so the
  Bode phase curve is continuous instead of wrapping at ±180°.
- **Gain/phase margins** — found by scanning the swept frequency arrays for
  the first sign change across 0 dB and −180° and linearly interpolating
  between the bracketing samples.
- **Root-locus branch continuity** — the Durand–Kerner solver returns an
  *unordered* set of roots at every gain step, so a greedy nearest-neighbor
  match against the previous step's roots is used to stitch them into
  continuous branches for drawing.
- **Rendering** — all six panels are hand-drawn on `<canvas>` 2D contexts:
  axis/grid drawing, log and linear scales, equal-aspect complex-plane
  scaling for Nyquist/root-locus/pole-zero, and the pan/zoom/reset
  interaction (wheel-to-zoom anchored at the cursor, drag-to-pan, per-panel
  view state) are all custom, not a charting library feature.

## What's in the zip

- `loop-bench.html` — the complete application.
- `README.md` — this file.

Just open `loop-bench.html` in any modern browser (Chrome, Firefox, Safari,
Edge) — no installation or server required.

## Known limitations

- Laplace entries accept only polynomial arithmetic in `s` plus a single
  top-level division (e.g. `numerator / denominator`); nested division isn't
  supported since the parser builds polynomial coefficients directly rather
  than a general symbolic expression tree.
- Polynomial degree is capped at 12 for numerical stability of the root
  finder.
- Gain/phase margins report only the *first* crossing found while scanning
  upward in frequency — an unusual system with multiple gain- or
  phase-crossover frequencies will only show one of them.
- The frequency sweep and root-locus gain range are fixed (0.01–1000 rad/s;
  K up to 1000) rather than adapting to each system, so extreme systems may
  need the panel's zoom to see the interesting region clearly.
