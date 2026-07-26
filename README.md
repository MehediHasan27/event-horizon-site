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
the Einstein radius the type responds to, disk brightness, drift speed,
relativistic `Physics`, and render resolution.

The panel collapses to a small tab, and starts collapsed at 640px and below,
where expanded it would cover a third of the viewport (36% down to 5%). Your
choice sticks once you've made it.

`Score` plays an original ambient piece, off by default.

## Jets

Bipolar jets are accumulated volumetrically along the same rays that carry the
disk, which means their base is lensed by the same curved space as everything
else rather than being composited on top. A narrow cone opens with height, wound
with helical filaments, and a bright knot is launched along the axis **every ten
seconds** and travels outward.

Lighting them extends the volume well past the disk, so the ray exit test has to
reach further while they are on. With `Jets` at 0 it snaps back to the tight
bound and most sky pixels retire on their first step again.

The framing was pulled back to suit them: the jets run to 24 Schwarzschild radii
and at the previous scale their tips fell off the screen.

## Haptics

There is no JavaScript haptics API doing this work. The trick — from
[web-haptics](https://github.com/lochie/web-haptics) by Lochie (MIT),
reimplemented inline here because this page ships as one file with no bundler —
is to let the **operating system** react to a native control being toggled.

A `<input type="checkbox" switch>` is rendered as a real system switch, and
toggling one produces a tap: the Taptic Engine on an iPhone, and the Force Touch
trackpad on a Mac. Clicking its label programmatically is enough to fire it. On
Android, `navigator.vibrate` runs alongside, with each pulse chopped into on/off
slices so a duty cycle stands in for the intensity that API will not expose.

The detail that decides whether any of it works: **the switch has to be really
rendered**. `display:none` or `visibility:hidden` leaves no control for the OS to
act on and nothing is felt. It lives at the bottom-left, small and faint at 22%
opacity, deliberately outside the collapsible panel body — that body is
`display:none` when the panel is shut, which would silently kill it. It flips
visibly when a jet fires.

A toggle is also a single very short tap, so sustaining a pulse means firing a
train of them, roughly every 16ms and wider apart for weaker steps.

Opt-in, off by default, and verified never to fire before you turn it on.

## Relativistic beaming## Relativistic beaming

A disk orbiting this close is moving at a large fraction of `c`, so the side
rotating toward you is beamed and blueshifted while the receding side is dimmed
and reddened. At each disk crossing the render computes the Doppler factor for
a Keplerian orbiter and the gravitational redshift climbing out of the well:

```
beta  = sqrt(M/r)                       Keplerian orbital speed, M = rs/2
dopp  = 1 / (gamma * (1 - beta.n))      n = source -> observer
grav  = sqrt(1 - rs/r)
shift = dopp * grav
```

Brightness scales as the cube of that shift, and the colour ramp is walked by
it too — an approaching element reads hotter and whiter, a receding one slides
into deep red.

The `Physics` slider mixes it in. At 0 the disk is near-symmetric, which is the
look the film chose, having deliberately dialled the asymmetry out on the
grounds that a lopsided disk reads as a mistake rather than as physics. At 1 it
is fully beamed. Measured across the two flanks: a brightness ratio of 1.11 at
0, and 2.98 at 1.

## Why not Three.js

The lensing is already WebGL — a fragment shader marching null geodesics. Three
would add a scene graph, cameras, materials and loaders for a single fullscreen
quad that uses none of them, run the identical shader at the identical cost,
and add well over the page's entire current weight. It would earn its place if
this needed real geometry, orbit controls, shadow maps or a post-processing
chain. It needs none.

The type stays in the DOM for the same kind of reason. Warping it on the GPU
means rasterising it to a texture, which costs selection, copy-paste, screen
readers, SEO and crisp scaling. The glyph pass costs 2.9 ms/frame, so there is
nothing to reclaim.

## Performance

The integrator is the expensive half, and it is where all the tuning went.

Rays are marched with a per-pixel step jitter, so the fixed cadence does not
alias into concentric rings. The single biggest saving is the exit test: a ray
that is outbound and already past the disk's outer radius cannot hit anything
else, and the deflection still to come is negligible, so it stops there. Most
sky pixels now retire on their first step instead of marching out to `r = 32`.
Along with a 96-step cap, a larger adaptive `dt`, one `inversesqrt` in place of
a `sqrt` and a divide, and fewer noise octaves, that measured **1.39x** faster
per draw at identical resolution (2702ms -> 1942ms under software
rasterisation, 800x560).

On top of that the field renders at **30 Hz** while the glyph pass keeps 60.
The disk flows slowly and the hole drifts about ten pixels a second, so drift
across one skipped frame is sub-pixel. That is roughly another 2x, for about
**2.8x less field work per second**.

Render scale is **auto by default**, and the signal it tunes against matters.
Because the field runs at 30 Hz, frames alternate cheap (glyphs only) and
expensive (glyphs plus field), and averaging the two hides exactly the frames
that miss vsync. So field frames are timed on their own and the scale is aimed
at fitting one inside a single vsync. A frame that runs badly over budget drops
the scale immediately rather than waiting for an average, since the slower the
device the fewer samples it can gather. The panel reports both the scale and
the measured field-frame cost. Pin it with the `Res` slider; its lowest
position hands control back to the tuner. Full resolution measured 11 fps on
real hardware before this pass, which is why it is no longer the default.
Device pixel ratio is capped at 1.5, so "100%" means full CSS resolution rather
than full retina — visually near-identical for a render this soft.

The glyph pass is cheap by comparison: 2.9 ms/frame for ~1200 lensed glyphs,
and no style writes at all while the page is idle.

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
