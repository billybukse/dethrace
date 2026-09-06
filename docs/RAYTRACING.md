# Ray tracing (hybrid) in the OpenGL renderer

Dethrace can augment the OpenGL renderer (`glrend`) with ray traced lighting:

* **Sun shadows** with soft edges, cast by all opaque geometry
* **Ambient occlusion** (contact shadows under cars, at wall bases, etc.)
* **Reflections** on cars and other moving objects (and on environment mapped surfaces)
* **Street lights**: lamp posts are detected automatically and become shadow-casting point lights with glowing fixtures
* **Car lights**: every car gets headlight spot cones and a red tail light glow
* **Colour bleeding** from bright surfaces (lit windows, neon, headlights) onto nearby geometry, plus a soft bloom around them

Enable it with `--raytracing` on the command line, or `RayTracing = 1` in the
`[General]` section of `dethrace.ini`. Ray tracing implies OpenGL mode
(`Emulate3DFX`). It needs an OpenGL 3.1 context with texture buffer objects,
which every desktop GPU since about 2010 provides.

## How it works

The rasteriser draws into an offscreen framebuffer with a G-buffer attached
(view-space position and normal, plus the instance index). Every stored model
group gets a small bounding volume hierarchy (BLAS) when it is uploaded; at the
end of each scene a top-level hierarchy (TLAS) is built over the visible
instances and everything is uploaded as texture buffers. A full-screen fragment
shader (`rt_trace.frag.glsl`) then traces, per pixel, one jittered shadow ray,
four short ambient occlusion rays and, on reflective surfaces, one reflection
ray. The noisy terms are accumulated over time: each pixel is reprojected into
the previous frame (via the recovered world transform) and blended with the
history where depth agrees, with less history on moving objects. A second
pass (`rt_composite.frag.glsl`) then filters with a depth- and normal-aware
blur and composites onto the rasterised colour. Sky and world detection carry
hysteresis across frames so classification cannot flip and cause flicker.

Everything runs in view space, because that is the only space BRender exposes
to the driver. World space is recovered from the transform shared by the most
triangles (the static track), which is used for the fixed sun direction and
to tell moving objects (reflective) from scenery. Geometry that spans the
whole scene with few triangles is treated as the sky box: it neither casts
shadows nor receives lighting. Tall, thin, low-poly moving objects are lamp
posts: each gets a point light just under its top, and its bright material
groups are rendered as emitters.

Fake blob shadows (`RenderShadows`) are skipped while ray tracing is on.

## Tuning (environment variables)

| Variable | Default | Meaning |
|---|---|---|
| `DETHRACE_RT_SCALE` | 0.5 | Trace resolution relative to the framebuffer (0.2 - 1.0) |
| `DETHRACE_RT_AMBIENT` | 0.55 | Light level in full shadow |
| `DETHRACE_RT_SUN_INTENSITY` | 1.0 | Extra brightness on sunlit (moonlit) surfaces, added to ambient |
| `DETHRACE_RT_LAMP_INTENSITY` | 1.4 | Street light brightness |
| `DETHRACE_RT_LAMP_R/G/B` | 1.0 / 0.93 / 0.78 | Street light colour (soft warm white) |
| `DETHRACE_RT_HEADLIGHTS` | 3.0 | Headlight intensity (0 disables car lights) |
| `DETHRACE_RT_BLEED` | 1.2 | Colour bleeding strength from bright surfaces |
| `DETHRACE_RT_BLOOM` | 0.5 | Glow strength around bright sources |
| `DETHRACE_RT_FALLOFF` | 3.0 | Falloff exponent for omni lights; higher = more concentrated |
| `DETHRACE_RT_GLOW` | 1.0 | Visible glare drawn at each light source |
| `DETHRACE_RT_LAMP_RADIUS` | 3.5 | Street light reach (world units; a car is about 0.9 long) |
| `DETHRACE_RT_AO_RADIUS` | 1.5 | Ambient occlusion ray length (world units) |
| `DETHRACE_RT_AO_STRENGTH` | 0.75 | How dark ambient occlusion gets |
| `DETHRACE_RT_REFLECTION` | 0.45 | Reflection blend strength |
| `DETHRACE_RT_SUN_JITTER` | 0.02 | Sun cone radius (softness of shadow edges) |
| `DETHRACE_RT_SUN_X/Y/Z` | 0.35 / 1.0 / 0.25 | World-space direction towards the sun |
| `DETHRACE_RT_SUN_FROM_LIGHT` | 0 | Use the track's BRender directional light instead |
| `DETHRACE_RT_DEBUG` | 0 | 1 shadow, 2 AO, 3 normals, 4 reflections, 7 combined (r/g/b), 9 street light contribution, 10 raw rasterised colour |
| `DETHRACE_RT_STATS` | 0 | Log frame times every 120 frames |
| `DETHRACE_RT_DUMP` | | Directory: write a PPM screenshot every 60 frames |
| `DETHRACE_AUTOKEYS` | | Scripted key presses for testing, e.g. `5:return,20:up:3` |

## Files

* `lib/BRender-v1.3.2/drivers/glrend/rt.c`, `rt.h` — framebuffers, BLAS/TLAS build, upload, passes
* `rt.vert.glsl`, `rt_trace.frag.glsl`, `rt_composite.frag.glsl` — shaders
* `brender.frag.glsl` — writes the G-buffer (extra colour attachments)
* `v1model.c` — classifies each material group (occluder / reflective / none)
* `gstored.c` — builds and frees the per-group BLAS
* `renderer.c`, `devpmglf.c` — scene begin/end hooks and presentation

## Related fixes in the OpenGL driver

FLI animations (HUD flics after collisions, menus) poke single palette entries.
`glrend` used to treat any palette change as a reason to re-convert every 8-bit
texture, so after a crash the road and buildings turned red for the rest of the
race, with or without ray tracing. `devclut.c` now only bumps the texture palette
revision on whole-palette sets, and in OpenGL mode the game only forwards the
render palette to the device (`PDSetPalette` in `allsys.c`); 2D content is
converted on the game side anyway.

The world also turned red (or another tint) at random, sometimes from the first
frame, sometimes mid-race, depending on the run. `StoredGLRenderGroup` read the
scene clear colour from `renderer->pixelmap->asBack.clearColour`, but that
pixelmap is the screen, whose active union member is `asFront`: the "colour"
was the bit pattern of a function pointer, and the vertex shader adds it to the
colour of every lit material. With address randomisation the value differs per
process, hence the randomness. `v1model.c` now only reads it from a real
offscreen buffer.
