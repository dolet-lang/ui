# Dolet UI

`ui` is the platform-neutral rendering and interaction foundation for Dolet
applications. It does not create windows or read native events. A framework or
engine supplies a pixel surface and an input snapshot; `ui` records and renders
the interface deterministically.

Current version: **0.8.0**

## Responsibilities

- BGRA8 software surfaces that can own memory or wrap external memory
- colors, rectangles, hit testing, and alpha-aware drawing primitives
- cached A8 coverage-font text through the independent `fonts` package
- subpixel anti-aliased circles, rounded corners, and thin lines
- growable draw-command buffers with owned text and nested clipping
- platform-neutral mouse, keyboard-navigation, and text-input snapshots

Window creation, event polling, swapchains, and framebuffer presentation belong
to higher layers such as `eqoi`, game engines, or native backend adapters.

```text
application / eqoi / engine UI
              |
              v
      input snapshots + commands
              |
              v
             ui
              |
              v
   owned or externally wrapped surface
```

## Direct rendering

```dolet
import ui

surface: UISurface = ui_surface_create(640, 360)
commands: UIDrawList = ui_draw_list_create(256)

commands.push_clear(640, 360, ui_rgb(18, 22, 28))
commands.push_clip(20, 20, 600, 320)
commands.push_fill_rounded_rect(32, 32, 220, 72, 8, ui_rgba(65, 130, 220, 230))
commands.push_text(52, 62, "Dolet UI", ui_rgb(255, 255, 255))
commands.pop_clip()

canvas: DrawCanvas = surface.canvas()
commands.render(canvas)

# Present surface.pixels using a window, Vulkan, or engine adapter.

commands.destroy()
surface.destroy()
```

`UIDrawList` grows when required and copies command text into its own arena, so
transient application strings remain safe until the list is cleared. Nested
clip rectangles are intersected while recording.

## External surfaces

Use `ui_surface_wrap(pixels, width, height, pitch)` when another system owns the
framebuffer. Destroying the wrapper frees its font data but never frees the
external pixels.

The default surface font is a cached 14-pixel coverage face. Widgets should use
`surface.text_width(value)` and `surface.text_height()` instead of assuming a
fixed 8x8 glyph size. The lower-level `draw_canvas_from_font_face(...)`
constructor also accepts custom A8 atlas faces supplied by the `fonts` package.

`surface.use_font_face(face)` borrows a caller-owned face without copying its
atlas. Keep that face alive until the surface is destroyed or receives another
face. Surface creation uses the ownership-transferring internal path for its
built-in fallback. Per-glyph advance tables are preserved, so loaded TrueType
faces use their proportional spacing in drawing, measurement, carets, and
widget centering.

## Input snapshots

`UIInputState.begin_frame(...)` receives state collected by a native platform or
engine. Widgets can then use transition helpers such as `left_pressed()`,
`left_released()`, pointer deltas, wheel values, typed text, and navigation keys
without importing an operating-system API.

## Package layout

```text
types.dlt       colors, points, rectangles
color.dlt       drawing color helpers
canvas.dlt      software rasterizer
surface.dlt     owned and wrapped surfaces
input.dlt       platform-neutral input snapshots
commands.dlt    growable command recording and replay
```

This separation is intentional: `ui` stays reusable on Windows, Linux,
bare-metal targets, game engines, and future GPU renderers.


## Text is UTF-8

Drawing and measuring walk codepoints, not bytes. A canvas resolves a
codepoint through two glyph blocks — the face's primary range and an optional
second one — so a single face can carry Latin and Arabic without rasterizing
everything in between.

This is what makes non-ASCII text possible at all: drawing per byte turns each
two-byte letter into two wrong glyphs, and measuring per byte puts every
centred label and text caret in the wrong place.
