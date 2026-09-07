# doodle fit

Draw a curve. Fit a model to it.

A single static page: sketch a curve with a mouse, trackpad, pen, or finger, and a
model is fitted to your ink and drawn over it — with its parameters, R², RMSE, and
the actual equation. Two models, chosen from a dropdown:

- **polynomial** — least squares, degree 1–15, with an auto-degree mode.
- **logistic** — the S-curve `ŷ = y₀ + span·σ(β₁x + β₀)`, fitted by maximum likelihood.

Switching models re-fits the same doodle in place, so you can flip between them and
watch which one your curve actually wants to be.

No dependencies, no build step, no network calls. One `index.html` (~34 KB) that runs
straight from the filesystem.

## Run it

```sh
open index.html            # macOS — or just double-click the file
python3 -m http.server 8000   # then visit http://localhost:8000
```

Deploying is a file copy: any static host (GitHub Pages, Netlify, S3) serves it as-is.

## The interaction

1. **Land** — graph paper, one instruction, and a faint dashed curve hinting at the
   gesture. `draw one for me` sketches an example if you'd rather watch first.
2. **Draw** — ink follows the pointer. Coalesced pointer events keep the line smooth
   on high-refresh screens.
3. **Release** — the fit is computed instantly and sweeps in left to right. It's solid
   across the span you drew and dashed beyond it, so extrapolation always looks like
   extrapolation.
4. **Tune** — pick the model from the dropdown. For a polynomial the degree slider
   re-fits live and **auto** picks a degree for you; for a logistic curve the panel
   shows the midpoint (where the S crosses halfway) and the steepness instead. The
   equation, R², and RMSE update in place either way.
5. **Keep** — `copy` puts the equation on your clipboard, `save` writes a PNG with
   the equation, degree, R², and RMSE stamped into the image. Drawing again replaces
   the old doodle.

Keyboard: `M` model · `[` / `]` degree · `A` auto · `C` clear · `S` save.

Works in light and dark (follows the system theme), on phones, and respects
`prefers-reduced-motion`.

## How the fits work

Both models are fitted on the same resampled stroke and scored the same way, so their
R² and RMSE are directly comparable — that's what makes flipping between them useful.

### Shared

- **Resampling.** The raw stroke is resampled to ~220 points evenly spaced along its
  arc length, so slow, dense parts of the stroke don't outvote fast ones.
### Polynomial

- **Conditioning.** x is mapped to `t = (x − x̄)/x_half_range` ∈ [−1, 1] before the
  Vandermonde matrix is built. Fitting in raw pixel units falls apart well before
  degree 15; in `t` it stays stable.
- **Solve.** Householder QR least squares on the Vandermonde matrix — no normal
  equations, so the conditioning isn't squared.
- **Auto degree.** Generalized cross-validation, `GCV = m·SSE / (m − d − 1)²`, over
  degrees 1–12, with two guards that keep it honest: residuals below half a pixel are
  treated as hand and screen quantization rather than signal, and the simplest degree
  within 5% of the best score wins. A curve traced exactly on `0.05x³ − 0.75x² +
  3.3x − 4` comes back as degree 3 with those coefficients, not as degree 9.
- **Display.** The curve is evaluated in the stable `t` basis; coefficients are only
  converted back to the plain `x` basis for the printed equation.

### Logistic

- **Targets.** y is rescaled to a probability across the stroke's own y-range, so the
  fit doesn't care where on the canvas you drew. The fitted curve maps back out to
  `ŷ = y₀ + span·σ(β₁x + β₀)`.
- **Solve.** Maximum likelihood on the Bernoulli log-likelihood with fractional
  targets, by Newton/IRLS — two parameters, a handful of iterations, with a small
  ridge term on the Hessian so separable strokes don't blow up.
- **Readouts.** Midpoint `−β₀/β₁` (where the S crosses halfway) and steepness `β₁`,
  both in plain x units.
- Extrapolation is bounded by the asymptotes, so the dashed tails flatten out instead
  of running off the canvas the way a high-degree polynomial does.

The canvas is a real coordinate system: x spans 0–10 left to right, y is centered on
the axis at the same scale, so the equation on screen is the equation of what you drew.

## Limits

Both models are functions of x — one y per x. Loop back on yourself and the fit splits
the difference; the panel says so when it detects it. High polynomial degrees hug your
ink and go wild just outside it, which the dashed extrapolation makes visible on
purpose. A logistic curve is a single monotonic S, so it flattens any doodle that turns
around — the panel points that out and suggests the polynomial instead.
