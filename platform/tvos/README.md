# Apple TV (tvOS) platform port

Run Godot games on Apple TV. This folder contains the C++, Objective-C and
Objective-C++ code for the tvOS platform port (minimum deployment target
tvOS 26.0, arm64 only, device and simulator).

This platform derives from the Apple Embedded abstract platform
([`drivers/apple_embedded`](/drivers/apple_embedded)) and uses shared Apple
code ([`drivers/apple`](/drivers/apple)). See also
[`misc/dist/apple_embedded_xcode`](/misc/dist/apple_embedded_xcode) for the
Xcode project template used for packaging the tvOS export templates.

## Documentation

The compiling and exporting process is the same as on iOS, but replacing the
`ios` parameter by `tvos`.

- [Compiling for iOS](https://docs.godotengine.org/en/latest/engine_details/development/compiling/compiling_for_ios.html)
  - Instructions on building this platform port from source.
- [Exporting for iOS](https://docs.godotengine.org/en/latest/tutorials/export/exporting_for_ios.html)
  - Instructions on using the compiled export templates to export a project.

Build the four template libraries with `opengl3=yes` (off by default on
this platform); without it the Compatibility renderer is missing and every
game falls back to the Metal capability gate:

```
scons platform=tvos target=template_debug arch=arm64 opengl3=yes
scons platform=tvos target=template_release arch=arm64 opengl3=yes
scons platform=tvos target=template_debug arch=arm64 simulator=yes opengl3=yes
scons platform=tvos target=template_release arch=arm64 simulator=yes opengl3=yes
```

## Supported

- **Compatibility renderer (OpenGL ES 3.0)** on device and simulator. This is
  the renderer to ship: it runs on every Apple TV including the A10X models.
- **Siri Remote as keys plus its own stick**, on device and in the simulator.
  Clicks arrive as ordinary `InputEventKey`, so the default UI navigation
  (`ui_up`/`ui_down`/`ui_left`/`ui_right`/`ui_accept`) and the arrow/Enter
  bindings just work:

  | Remote button | Godot event |
  |---|---|
  | Clickpad edge / D-pad | `KEY_UP` / `KEY_DOWN` / `KEY_LEFT` / `KEY_RIGHT` |
  | Click (Select) | `KEY_ENTER` |
  | Play/Pause | `KEY_MEDIAPLAY` |
  | Menu | `KEY_MENU` down/up, plus the go-back request (see below) |
  | Touch drag | Left stick (`JOY_AXIS_LEFT_X`/`JOY_AXIS_LEFT_Y`) on device `-3` |

  The touch steers like the stick the remote used to be: where the finger
  sits on the pad becomes the deflection, absolute - holding a corner
  walks it with no drag first, the middle rests - corners and all, zeroed
  on lift. So rounds walk the held way with a fine aim while menus step
  through the stick bindings of the ui actions. The stick rides the
  remote's own device (`-3`, games name it `Pads.REMOTE`), which plain
  actions hear and no pad's scoped copy does; the remote is *not* an SDL
  joystick, and clicks stay keys, so nothing arrives twice. MFi game
  controllers keep working through the standard Godot joypad API.
- **Menu button behavior.** With `quit_on_go_back` enabled (the default), the
  Menu press is handed back to UIKit, preserving the system Menu-to-Home flow.
  With it disabled, the app receives `KEY_MENU` going down, `KEY_MENU` going
  up, and then `NOTIFICATION_WM_GO_BACK_REQUEST`
  (the `WINDOW_EVENT_GO_BACK_REQUEST` window event) instead of suspending,
  so taps and holds tell apart. Menu never doubles as a hardware key: a
  hardware Escape reports as Menu too, and acts as one. Stuck keys are
  released if the system takes the press stream (`pressesCancelled`).
- **Metal renderer profiles (Forward+/Mobile).** The tvOS GPU-family/MSL
  profiles, the `appletvos` shader toolchain, and device capability detection
  are implemented. On GPUs older than Apple family 4 the engine shows an
  alert naming the detected family (e.g. "this GPU reports Apple3") instead
  of failing with an empty feature list. Simulator templates keep Metal
  enabled and fail gracefully at the same gate.
- **Quit hardening.** `SceneTree.quit()` is a warned no-op on tvOS (and the
  other Apple embedded platforms): Apple forbids programmatic termination,
  so the main loop keeps running and the system suspends the app. Games
  leave through the Menu flow above, never through `quit()`.
- **Persistent storage.** `user://` maps to the app's `Library/Caches`: the
  tvOS sandbox denies file creation in `Documents`, so a Documents-backed
  `user://` breaks on first write on device.
- **Text entry** through the standard virtual keyboard API (backed by a
  hidden text field; no always-on-screen keyboard view like iOS).
- **Export**: parallax `.imagestack` app icons, Top Shelf images, an
  Apple-TV-retargeted launch storyboard, and a tvOS-cleaned `Info.plist`
  (iOS-only keys dropped). Export options with no tvOS meaning (camera,
  microphone, photo library, Files app access) are hidden.
- Display features report honestly: no touchscreen, no clipboard, no
  orientation changes. Process spawning (`OS.execute`, `OS.create_process`)
  is prohibited by the sandbox and fails with `ERR_UNAVAILABLE`.

## Not supported

- **Forward+/Mobile rendering on Apple1-3 GPUs** (Apple TV HD and Apple TV
  4K 1st generation). The hardware lacks the required features; only the
  Compatibility renderer runs there. Forward+/Mobile needs an Apple4+ GPU
  (Apple TV 4K 2nd generation, 2021, and newer).
- **Metal rendering in the simulator.** `MTLSimDevice` ("Apple tvOS
  simulator GPU") only reports Apple1/Apple2 with tier-1 argument buffers
  and no image cube arrays, and shared placement heaps abort, so the gate
  always stops it. Simulator apps must use the Compatibility renderer.
- Vulkan, .NET, touchscreen, clipboard, orientation changes, camera,
  microphone recording, photo library, and Files app integration.

## Verified on hardware

All of the following was verified in Sep 2026 against this branch:

- **Apple TV 4K 1st generation (A10X, tvOS 26.6), device builds:**
  - Siri Remote 6-button matrix: every button arrives with press+release and
    the exact keycodes from the table above (Play/Pause = `MEDIAPLAY`),
    with zero joystick events.
  - Menu with `quit_on_go_back` true and false: app stays in the foreground
    in both modes; with it false, `GO_BACK_REQUEST` is delivered to the
    scene tree (log-verified, not just state-asserted).
  - `quit()` leaves the main loop running (frame counter advances).
  - Forward+ and Mobile probes show the Apple3 gate alert.
  - `user://` file writes succeed and round-trip back to the Mac.
  - End-to-end gameplay (BubblyField demo): login navigation, Practice
    round, Play/Pause and Menu settings open/close, Select bomb, blast
    breaking bricks with pickups, player damage, Defeat panel.
- **tvOS 27 simulator (Mac):** BubblyField practice round (login, settings
  toggle, bomb, blast, bricks, player caught), quit probe alive, FP/Mobile
  Apple2 gate alerts, and a native probe characterizing `MTLSimDevice`
  (families, argument-buffer tier, heap behavior).
- **Headless suite:** BubblyField `tests/tvos_remote_regression.gd`
  (127 checks: input map, remote buttons, Menu key tap/hold case use,
  request-alone Menu fallback, remote stick routing, pause/back routing,
  menu navigation, walking, bubbles, window suspend/resume) passes.
- **Builds:** all four tvOS templates (release/debug x device/simulator,
  GLES3 + Metal) and the iOS release template compile cleanly.

## Not yet verified

- Forward+/Mobile actual rendering on an Apple4+ Apple TV (no such hardware
  available; profiles, toolchain query, linking, and gating are verified,
  pixels are not).
- Physical MFi controllers (no hardware available; the SDL hint change is
  remote-specific by API).
- The `pressesCancelled` stuck-key path (no deterministic trigger exists).
- Touch-hold-until-lift and `KEY_MENU` tap/hold on a physical remote (both
  ship in this branch; headless-covered, on-TV confirmation pending).
- App Store submission and validation of an exported tvOS app.
- tvOS versions other than the tested 26.x device / 26.5–27 simulators.
