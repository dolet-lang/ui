# Dolet UI

Native UI foundation library for Dolet.

`ui` provides a small, reusable layer for drawing user interfaces on top of native platform APIs. It is designed to be used directly by low-level apps, and also as a foundation for higher-level frameworks such as `foxapp`, editors, tools, or engine UI layers.

The current implementation targets Windows through a Win32/GDI backend with double-buffered drawing.

## Status

This library is early-stage but usable for basic native drawing.

Ready today:

- `UIColor` and `ui_rgb(...)`
- `UIRect`
- `UiCanvas`
- Filled rectangles and squares
- Outlined rectangles
- Lines
- Windows native GDI backend
- Double buffering to reduce flicker
- Works with `import window` for native Dolet windows

Planned next:

- Text drawing
- Mouse state helpers
- Hit testing helpers
- Clipping
- Basic draw command layer
- Linux backend
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

    while win.should_close() == 0:
        win.wait_events()

        client_width: i32 = win.get_width()
        client_height: i32 = win.get_height()

        canvas: UiCanvas = ui_begin_gdi(win.get_handle(), client_width, client_height)
        if canvas.is_valid() == 1:
            canvas.clear(client_width, client_height, background)
            canvas.fill_square(120, 100, 180, square_color)
            canvas.outline_rect(120, 100, 180, 180, border_color, 3)
            canvas.end()

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
