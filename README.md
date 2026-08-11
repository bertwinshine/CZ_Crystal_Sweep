# CZ melt — crystal rotation study

Reference CFD cases for the Czochralski silicon melt, varying the crystal
rotation. Eleven steady 2D axisymmetric swirl cases from 4 to 20 rpm, everything
else held fixed. Meant as ground-truth data for the surrogate/PINN model.

Each case is a full converged field on a common mesh, exported as
`r, z, u_r, u_z, u_swirl, p, T` at 8181 nodes. Every case is checked against the
boundary conditions it should have been run with — `certify_case.py` does that
and all eleven pass.

The 8 rpm case is the shared baseline. It is the same field, node for node, as
the −3 rpm case in the crucible study and the 1745 K case in the temperature
study, so the three sweeps intersect at a single common point.

## What varies

Only the crystal rotation. The crucible stays at its baseline −3 rpm
(ω = −0.3140 rad/s) and the crucible wall stays at 1745 K.

The free surface spans from the crystal edge to the crucible wall, so its
prescribed swirl ramps from the crystal edge speed down to the crucible wall
speed. On this axis it is the **inner** end of that ramp that has to follow the
swept parameter, with the outer end held at −0.094200 m/s.

| rpm | ω (rad/s) | free-surface swirl at r = 0.18 |
|---:|---:|---:|
| 4 | 0.418879 | 0.075398 |
| 5 | 0.523599 | 0.094248 |
| 6 | 0.628319 | 0.113097 |
| 7 | 0.733038 | 0.131947 |
| 8 | 0.837758 | 0.150796 |
| 10 | 1.047198 | 0.188496 |
| 12 | 1.256637 | 0.226195 |
| 14 | 1.466077 | 0.263894 |
| 16 | 1.675516 | 0.301593 |
| 18 | 1.884956 | 0.339292 |
| 20 | 2.094395 | 0.376991 |

Free-surface swirl expression, with the inner value taken from the table:

```
INNER [m/s] + ((RadialCoordinate - 0.18 [m]) / (0.30 [m] - 0.18 [m]))
              * (-0.094200 [m/s] - INNER [m/s])
```

## Case setup

Geometry: crucible radius 0.30 m, crystal radius 0.18 m, melt height 0.15 m.

| Zone | Thermal | Momentum |
|---|---|---|
| Crystal face | 1685 K | rotational, ω per case |
| Crucible wall | 1745 K | rotational, −0.3140 rad/s |
| Crucible bottom | `1700 + (r/0.30)·(1745−1700)` K | rotational, −0.3140 rad/s |
| Free surface | `1685 + ((r−0.18)/0.12)·(1745−1685)` K | swirl ramp, see above |
| Axis | — | axis |

Solver: 2D axisymmetric swirl, laminar, energy on, Boussinesq with operating
temperature 1717 K, gravity in x = −9.81, coupled pressure–velocity, second-order
discretisation, automatic pseudo time step. ANSYS Fluent 2026 R1 Student.

## What the sweep shows

Two things, and they sit close together.

**A velocity minimum near 6 rpm.**

| crystal rpm | mean \|u\| (m/s) | max \|u\| (m/s) | p range (Pa) |
|---:|---:|---:|---:|
| 4 | 0.002918 | 0.012003 | 15.78 |
| 5 | 0.002509 | 0.009704 | 16.24 |
| **6** | **0.001795** | **0.006368** | 16.53 |
| 7 | 0.001966 | 0.006376 | 14.95 |
| 8 | 0.002598 | 0.009519 | 14.94 |
| 10 | 0.003841 | 0.016147 | 15.18 |
| 12 | 0.005000 | 0.023949 | 15.63 |
| 14 | 0.006136 | 0.031871 | 16.00 |
| 16 | 0.007242 | 0.039854 | 16.07 |
| 18 | 0.008349 | 0.047855 | 16.40 |
| 20 | 0.009616 | 0.055834 | 17.74 |

**A reversal of the meridional circulation between 6 and 7 rpm.** The
cross-case correlation of the axial velocity between adjacent cases:

| step | u_z | u_r |
|---|---:|---:|
| 4 → 5 | +0.995 | +0.992 |
| 5 → 6 | +0.924 | +0.965 |
| **6 → 7** | **+0.135** | **+0.779** |
| 7 → 8 | +0.942 | +0.886 |
| 8 → 10 | +0.934 | +0.841 |
| 10 → 12 | +0.968 | +0.950 |
| 12 → 14 | +0.979 | +0.978 |
| 14 → 16 | +0.983 | +0.988 |
| 16 → 18 | +0.986 | +0.993 |
| 18 → 20 | +0.993 | +0.997 |

Every step sits above 0.92 except one. Across 6 → 7 the axial velocity almost
completely decorrelates, and 4 rpm correlates at −0.67 with 20 rpm — the two ends
of the sweep are opposed flow structures, not scaled versions of each other.

The reading is that crystal rotation drives a pumping cell near the crystal which
opposes the buoyancy-driven cell. Below about 6 rpm buoyancy wins; above it the
crystal-driven cell takes over and grows steadily. The velocity minimum is where
the two most nearly cancel, and the circulation flips direction just above it.

The swirl field, by contrast, is smooth throughout — every adjacent correlation
is above 0.995. The reversal is in the meridional flow only.

The 5 and 7 rpm cases were run specifically to resolve this after the first pass
(4, 6, 8, 10, …) showed the decorrelation but could not place it.

## Certification

`certify_case.py` reads the rpm from the filename, re-derives every boundary
condition from it, and checks the exported field against them.

```
python certify_case.py steady/*.csv
```

All eleven pass. Typical margins:

| check | typical |
|---|---|
| crystal face swirl | 0.061 % of edge speed |
| free-surface ramp, point by point | max err ~1.5e-4 m/s |
| free-surface fitted inner end | within 0.04 % of target |
| crucible wall / bottom swirl | < 1e-4 % of target |
| free-surface / bottom thermal ramps | < 0.03 K |
| axis symmetry, max \|u_r\| on axis | 0.08 – 0.50 % of field max |

The fitted-inner-end check is worth singling out. On this axis the free-surface
ramp is the boundary that carries the swept parameter, and a ramp left at a
neighbouring case's value is still smooth, monotonic and entirely physical on its
own. It only shows up when the ramp is fitted back to its inner end and compared
against the rotation the case is labelled with. Comparing point by point alone is
not enough, because the two ramps differ by less than the ramp's own span.

Two things the checks are deliberately tolerant about, both understood:

- **The corner node at r = 0.18** is shared between the crystal face and the free
  surface, and the solver blends the two conditions there. It is excluded from
  the ramp checks rather than being treated as an error.
- **The node-based export** interpolates wall values, so a linear fit of the
  crystal face returns ω about 4e-5 relative below the prescribed value. The
  swirl checks are relative and print the percentage rather than hiding it.

## Reduced-order model

A POD + RBF surrogate over the eleven cases, validated leave-one-out, as % of
each field's range:

| held out | u_r | u_z | u_swirl | p | T |
|---:|---:|---:|---:|---:|---:|
| 5 | 0.393 % | 0.624 % | 0.080 % | 1.148 % | 0.873 % |
| 6 | 2.167 % | 5.214 % | 0.154 % | 2.273 % | 2.410 % |
| 7 | 2.600 % | 5.905 % | 0.092 % | 3.142 % | 2.746 % |
| 8 | 1.305 % | 2.060 % | 0.086 % | 1.477 % | 1.869 % |
| 10 | 0.687 % | 0.987 % | 0.123 % | 0.448 % | 1.727 % |
| 12 | 0.329 % | 0.462 % | 0.055 % | 0.330 % | 1.253 % |
| 14 | 0.184 % | 0.317 % | 0.056 % | 0.385 % | 0.978 % |
| 16 | 0.179 % | 0.258 % | 0.063 % | 0.522 % | 0.816 % |
| 18 | 0.103 % | 0.171 % | 0.047 % | 0.723 % | 0.760 % |

Above 10 rpm the model is accurate to a few tenths of a percent, comparable to
the crucible sweep. Around the reversal it is an order of magnitude worse.

That is not a sampling deficiency that more cases will fully remove. POD builds a
basis from the snapshots and the interpolator blends them, so predicting a point
between two opposed circulations produces something close to their average — a
weak, structureless field that resembles neither. Adding 5 and 7 rpm cut the
interior u_z error from 3.07 % to 1.78 %, but a linear reduced-order model
crossing a structural change will keep a residual there. **Treat surrogate output
between 6 and 8 rpm as indicative only.**

The model interpolates inside 4 to 20 rpm and should not be used outside it —
holding out an endpoint gives 2.4 % (4 rpm) and 1.8 % (20 rpm) rather than
tenths.

## Scope and limits

- Laminar, Boussinesq, steady, prescribed free surface. No turbulence model, no
  radiation, no Marangoni convection, no species transport, no magnetic field.
- The free surface is a prescribed wall, not a solved meniscus.
- One parameter varies. Crystal rotation is swept; hot-wall temperature and
  crucible rotation are held at 1745 K and −3 rpm. Nothing here says anything
  about how those parameters interact — that needs cases where more than one
  moves at a time.
- Student-licence mesh, 8181 nodes.

This is a benchmark dataset for method development, not an industrially
representative model of a real puller.

## Files

```
steady/            11 certified cases, r,z,u_r,u_z,u_swirl,p,T
certify_case.py    boundary-condition and sanity checks
```

The ANSYS modelling, the sweep design and the validation are mine. The Python was
written with AI assistance — I understand what it does and can walk through it.
