# Dolet UI

Native UI foundation library for Dolet.

`ui` provides a small, reusable layer for drawing user interfaces on top of native platform APIs. It is designed to be used directly by low-level apps, and also as a foundation for higher-level frameworks such as `foxapp`, editors, tools, or engine UI layers.

The current implementation targets Windows through a Win32/GDI backend with double-buffered drawing.

## Status

This library is early-stage but usable for basic native drawing.

Feature support:

| Feature | Status | Notes |
| --- | --- | --- |
| Color | Supported | `UIColor` and `ui_rgb(...)` are available. |
| Rect | Supported | `UIRect` is available. |
| Canvas | Supported | `UiCanvas` is available through the GDI backend. |
| Text | Supported | `canvas.text(...)`, `canvas.text_bg(...)`, `ui_text(...)`, and text draw commands are available. |
| Mouse state | Supported | `UIMouseState` tracks client position and mouse button transitions. |
| Draw commands | Supported | `UIDrawList` records and replays clear, rect, line, and text commands. |
| Native backend | Supported on Windows | Windows GDI is available; Linux and engine backends are planned. |

Ready today:

- `UIColor` and `ui_rgb(...)`
- `UIRect`
- `UiCanvas`
- Filled rectangles and squares
- Outlined rectangles
- Lines
- Text drawing
- Opaque text backgrounds for stable GDI text rendering
- Mouse state helpers
- Hit testing helpers
- Basic draw command list
- Windows native GDI backend
- Double buffering to reduce flicker
- Works with `import window` for native Dolet windows

Planned next:

- Clipping
- Linux backend
- Engine backends such as Kobic or Vulkan
- Higher-level widget libraries built on top of `ui`

## Design

`ui` is intentionally not an application framework.

It does not own:

- Window creation
- Event loops
- Frame timing
- Application lifecycle
- Widgets such as buttons, labels, panels, or layouts

Those responsibilities belong to either the app, the `window` module, or a higher-level framework such as `foxapp`.

The intended layering is:

```text
foxapp / app framework
  uses ui
  uses window

ui
  drawing primitives
  colors
  rectangles
  canvas
  text
  mouse state
  draw commands
  native rendering backends

window
  native window creation
  event handling
  HWND / platform handles
```

This keeps `ui` reusable. A framework can use it internally, while advanced users can still draw directly with it.

## Current Layout

```text
mod.dlt
module.meta
types.dlt
input.dlt
commands.dlt

bindings/
  win32.dlt

backends/
  gdi.dlt
```

`bindings/` contains low-level FFI declarations.

`backends/` contains rendering implementations built on top of those bindings.

The current backend is `gdi`, which is Windows-only.

## Example

```dolet
import window
import ui

win: Window = Window.create("Dolet Native UI Test", 800, 600)
if win.get_handle() == 0:
    print("failed to create window")
else:
    background: UIColor = ui_rgb(18, 22, 28)
    square_color: UIColor = ui_rgb(64, 160, 255)
    border_color: UIColor = ui_rgb(240, 245, 255)
    accent_color: UIColor = ui_rgb(255, 88, 96)
    hover_color: UIColor = ui_rgb(255, 130, 136)

    mouse: UIMouseState = ui_mouse_create()
    commands: UIDrawList = ui_draw_list_create(64)

    while win.should_close() == 0:
        win.wait_events()
        mouse.update(win.get_handle())

        client_width: i32 = win.get_width()
        client_height: i32 = win.get_height()

        commands.clear()
        commands.push_clear(client_width, client_height, background)
        commands.push_text(32, 28, "Dolet UI: text, mouse state, and draw commands", border_color)
        commands.push_fill_rect(120, 100, 180, 180, square_color)
        commands.push_outline_rect(120, 100, 180, 180, border_color, 3)

        if mouse.is_over_xy(360, 140, 220, 120) == 1:
            commands.push_fill_rect(360, 140, 220, 120, hover_color)
            commands.push_text(386, 188, "hover", background)
        else:
            commands.push_fill_rect(360, 140, 220, 120, accent_color)
            commands.push_text(388, 188, "move mouse here", background)

        canvas: UiCanvas = ui_begin_gdi(win.get_handle(), client_width, client_height)
        if canvas.is_valid() == 1:
            commands.render(canvas)
            canvas.end()

    commands.destroy()
    win.destroy()
```

## Building The Example

Place this package where the Dolet compiler can resolve it, for example:

```text
dolet-compiler/packages/ui
```

Then build a consumer file that imports `window` and `ui`:

```powershell
doletc app.dlt -o app --platform windows
```

Run:

```powershell
.\app.exe
```

## Platform Notes

The current backend is Windows-only:

```text
bindings/win32.dlt
backends/gdi.dlt
```

Linux support should be added as a separate backend rather than forcing GDI abstractions onto Linux. Possible future backends:

- X11
- Cairo
- Vulkan
- Kobic renderer

## Future Frameworks

A higher-level framework can depend on `ui` and hide all low-level drawing from users:

```dolet
import foxapp

app: FoxApp = FoxApp.create("My App", 800, 600)
app.button("Save", 20, 20, 120, 40)
app.run()
```

Internally, that framework can use:

```dolet
import window
import ui
```

This keeps `ui` small and reusable while allowing richer toolkits to grow on top of it.
