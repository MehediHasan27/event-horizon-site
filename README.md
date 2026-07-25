# Event Horizon

A black hole rendered by ray-marching null geodesics, with the page's type
gravitationally lensed around it.

One self-contained HTML file. No build step, no dependencies, no network
requests at runtime — fonts are embedded and the score is synthesised in the
browser. Open `index.html` and it runs.

## What it does

**The field** is integrated, not painted. For every pixel a ray is fired from
the camera and stepped through curved space under the Schwarzschild null
geodesic acceleration:

```
a = -1.5 · h² · r⃗ / r⁵
```

with `h` the photon's angular momentum, conserved and so computed once at
launch. Each step tests whether the ray crossed the disk plane; if the crossing
radius lies inside the disk, that patch's emission is added. Rays that fall
inside `r_s` return black; rays that escape sample the starfield along their
final deflected direction.

Because the crossing test runs every step, one ray can pick up the disk two or
three times — in front, arching over the top, and beneath. Seen nearly edge-on
that closes into a halo. The arch is not drawn; it falls out of the integration.

**The disk flows Keplerian.** Angular speed goes as `r^-3/2`, so the inner edge
laps the outer and the texture shears rather than spinning rigidly. Filaments
are built from integer harmonics of the azimuth so they wrap cleanly at ±π — a
noise field indexed by angle leaves a visible seam at the branch cut.
Large-scale turbulence is sampled in the rotating disk plane instead, which is
seamless by construction.

**The type** is lensed by a cheaper, closed-form map — the point-mass lens,
applied per glyph:

```
θ = ½ ( β + √( β² + 4θ_E² ) )
```

Each character's rest position is a source at radius `β`, drawn at `θ`. Far out,
`θ → β` and type sits where it was set. Close in, every glyph is pushed onto the
Einstein radius, so words crossing behind the hole pile onto a ring instead of
disappearing into it — nothing can land inside the shadow, because the map
forbids it. Each glyph is also sampled a fraction of an em to either side; the
offset between where those two samples land gives the local tangent and stretch,
which is what leans letters into the curve rather than stepping them around it
upright.

This is deliberately not `shape-outside`. A CSS float is anchored to one edge
and only pushes inward from it, so text can wrap one side of an object, never
both, and the glyphs stay rigid. A radial lens has no preferred side.

## Controls

Drag the hole anywhere on the page — it records your offset from its drift path
and wanders on from where you let go. The panel sets mass (pixels per
Schwarzschild radius), spin, inclination (face-on ring through to edge-on),
the Einstein radius the type responds to, disk brightness, drift speed and
render resolution.

The panel collapses to a small tab, and starts collapsed at 640px and below,
where expanded it would cover a third of the viewport (36% down to 5%). Your
choice sticks once you've made it.

`Score` plays an original ambient piece, off by default.

## Performance

The integrator is the expensive half — 112 adaptive steps per pixel, with a
per-pixel step jitter so the fixed cadence does not alias into concentric rings.

It renders at **full resolution by default**. If that is too slow on your
hardware, drag `Res` down, or all the way to its lowest position for `auto`,
which hands the scale to a tuner driven by measured frame time and reports what
it settles on. The device pixel ratio is capped at 1.5, so "100%" means full CSS
resolution rather than full retina — visually near-identical here for a soft
render, at a little over half the cost.

The glyph pass is cheap by comparison: measured at 2.9 ms/frame for ~1200
lensed glyphs, and it does no style writes at all while the page is idle.

`prefers-reduced-motion` stops the disk turning and the hole drifting. The
lensing is spatial rather than temporal, so the warp itself is kept.

## Licensing

Code, shader, copy and score in this repository are original — license them as
you like.

Two embedded fonts are third-party, both under the SIL Open Font License 1.1,
with full license text in `licenses/`:

| Font | Copyright | Source |
| --- | --- | --- |
| Instrument Serif | © 2022 The Instrument Serif Project Authors | [Instrument/instrument-serif](https://github.com/Instrument/instrument-serif) |
| Manrope | © 2018 The Manrope Project Authors | [sharanda/manrope](https://github.com/sharanda/manrope) |

Latin subsets, via Google Fonts. Neither declares a Reserved Font Name. The OFL
requires the license travel with the font software, which is why `licenses/`
ships alongside `index.html` — keep it if you redistribute.

The score is **not** the Interstellar theme, and no part of that recording or
composition is included. It is generated in the page from additive organ
partials over a drone and a soft pulse, through a procedurally built reverb.
The visual is a homage to a well-known depiction of a lensed accretion disk,
built from published physics rather than from any frame of the film.

## Hosting

A single static file, so any static host works with no configuration. See the
notes in the parent conversation for GitHub Pages setup.
