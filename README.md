# Predicting FIFA Penalty Trajectories

Interactive supplement to our **UPC 2026, Problem B** entry:
*Predicting FIFA Penalty Trajectories — reconstructing the flight of a penalty kick from three early positions.*

**Live demo:** https://tian99-max.github.io/Predicting-FIFA-Penalty-Trajectories/

---

## What this is

A single-file browser application that predicts the first crossing of the goal plane
(`x = 11 m`, `v_x > 0`) from only three early ball positions
(`t = 0`, `0.05`, `0.10 s`), together with an uncertainty estimate for that prediction.

Everything — trajectory integration, parameter inversion, Monte-Carlo propagation and
all statistics — is computed at run time in the browser. **No result is precomputed or
hard-coded.** Open the page source and you will find the integrator and the optimiser,
not a table of answers.

There is no build step, no server, no Python and no external library: `index.html` is the
whole application.

---

## Physics

| Model | Equations | Fitted parameters |
|---|---|---|
| **0** Constant acceleration | `r = r0 + v0 t + ½ a t²`, from three-point finite differences | none (analytic) |
| **1** Gravity only | `dv/dt = g` | `v0x, v0y, v0z` |
| **2** Non-spinning drag | `dv/dt = g − K C_D(0) u v`, `C_D(0) = 0.15` | `v0x, v0y, v0z` |
| **3** Spin, frozen coefficients | `dv/dt = g − K C_D u v + K C_L u² n` | `v0x, v0y, v0z, S0, φ` |
| **4** Spin, updated coefficients | same, but `C_D`, `C_L` re-evaluated from `S(t)` at every step | `v0x, v0y, v0z, S0, φ` |

with `u = |v|`, `K = ρA/(2m)`, `n = (ω × v)/|ω × v|` and the dimensionless spin
parameter `S = R|ω|/|v|`.

**Spin parameterisation.** With `e = v0/|v0|`, a reference vector `q` (switched from
`(0,0,1)` to `(0,1,0)` when `|e·q| > 0.95`), `e1 = (e × q)/|e × q|` and `e2 = e × e1`:

```
ω = (S0 |v0| / R) [ cos φ e1 + sin φ e2 ]
```

The angular velocity is constant in the ground frame; only the direction of the
Magnus force changes as the velocity turns.

---

## Aerodynamic coefficient table

A piecewise-linear table indexed by `S = R|ω|/|v|`, **team-supplied**:

| S | 0 | 0.03 | 0.05 | 0.075 | 0.10 | 0.125 | 0.15 | 0.20 | 0.25 | 0.30 |
|---|---|---|---|---|---|---|---|---|---|---|
| C_D | 0.15 | 0.17 | 0.18 | 0.20 | 0.22 | 0.24 | 0.25 | 0.26 | 0.27 | 0.28 |
| C_L | 0.00 | 0.04 | 0.17 | 0.21 | 0.23 | 0.25 | 0.27 | 0.29 | 0.31 | 0.33 |

- Interpolation is linear between nodes and reproduces the node values exactly.
- **No extrapolation.** The domain is `S ∈ [0, 0.30]`; model 4 terminates with the status
  `outside_spin_table` when the bound is reached.
- This table is **not independently calibrated**. It is used as supplied by the team.

---

## Inverse problem

For the spinning models the unknown vector is

```
θ = (v0x, v0y, v0z, S0, φ)
```

and we minimise the weighted residual

```
J(θ) = Σ_i || [r_model(t_i; θ) − r_obs(t_i)] / σ ||²
```

with a bounded Levenberg–Marquardt solver (forward-difference Jacobian), velocities
initialised from the analytic constant-acceleration solution and 12 multi-starts over
`S0 ∈ {0.04, 0.12, 0.22}` × `φ ∈ {−2.4, −0.8, 0.8, 2.4}`.

The page reports the per-coordinate residuals, the RMSE, `RMSE/σ` and the number of
**distinct solutions** — parameter sets that reach equally good residuals. For the given
observations this number is 12, which is a direct numerical demonstration that six scalar
observations cannot identify five aerodynamic parameters uniquely.

| Bound | min | max |
|---|---|---|
| `v0x` | 5 m/s | 60 m/s |
| `v0y` | −30 m/s | 30 m/s |
| `v0z` | 0.05 m/s | 30 m/s |
| `S0` | 1e-6 | 0.299 |
| `φ` | −π | π (periodic) |

---

## Uncertainty

Observation perturbation with refitting (`σ = 1 cm` by default, seed `2026`): the six
coordinates of the second and third positions are perturbed with independent Gaussian
noise, every trial is refitted and integrated to its first event, and the resulting
`t_g, y_g, z_g` are summarised by their 2.5 / 50 / 97.5 % quantiles.

An *approximate 95 % Gaussian ellipse* (`χ² = 5.991`, 2 d.o.f.) is drawn as a visual aid.
It is **not** a validated coverage region and the page never claims an achieved coverage.

Failures are counted by category and never mixed with physical misses:

`goal_plane`, `ground_first`, `outside_spin_table`, `fit_failed`,
`no_crossing_by_time_limit`, `crossing_before_last_observation`.

---

## Reproducibility checks

The implementation was checked against analytic and reference values:

| Check | Result |
|---|---|
| Interpolation at the table nodes | exact (deviation 0) |
| Model 0 vs analytic reference | `t_g = 0.405008 s`, `(y_g, z_g) = (2.494824, 1.920367) m` |
| Model 1 equal-weight fit | `v0 = (29.68, 8.68, 6.801) m/s` |
| Step-size convergence (5e-4 vs 1e-4) | `y_g` differs by 2.6e-8 m, `z_g` by 1.6e-7 m |
| Runtime | nominal prediction ≈ 0.6 s; 100 Monte-Carlo trials ≈ 4.6 s |

Integration: fixed-step RK4 (`dt = 5e-4` for fitting, `dt = 2e-4` for event finding,
bisection refinement of event times).

---

## Running locally

Download `index.html` and open it in any modern browser (Chrome, Edge, Firefox).
Nothing else is required. Works offline.

---

## Limitations

- The coefficient table is **team-supplied and not independently calibrated**; it is not
  a validated Teamgeist performance model.
- Ball mass (0.43 kg) and radius (0.11 m) are **assumed** values, not measurements of the
  particular ball used.
- The spin axis is assumed perpendicular to the initial velocity; this is a modelling
  choice, not a result.
- Coefficients are used only for `S ≤ 0.30`; extrapolation is refused.
- No aerodynamics beyond drag and Magnus is modelled; wind, panel seams, spin decay,
  goalkeeper behaviour and frame collisions are outside the scope.
- The reported in-goal fraction is a sample proportion conditional on valid crossings, not
  a scoring probability.

---

## Citation

```bibtex
@misc{upc2026predictor,
  author       = {{UPC 2026 Problem B team}},
  title        = {Penalty Kick Predictor: interactive implementation of the
                  trajectory models for UPC 2026 Problem B},
  year         = {2026},
  howpublished = {\url{https://tian99-max.github.io/Predicting-FIFA-Penalty-Trajectories/}},
  note         = {Aerodynamic coefficient table supplied by the team; not
                  independently calibrated.}
}
```

---

## References

- Asai, T., Seo, K., Kobayashi, O., Sakashita, R. (2007).
  *Fundamental aerodynamics of the soccer ball.* Sports Engineering **10**, 101–110.
- NASA Glenn Research Center, *Aerodynamics of Soccer*,
  https://www1.grc.nasa.gov/beginners-guide-to-aeronautics/aerodynamics-of-soccer/
