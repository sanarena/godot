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

## Feature status

Legend: ✅ tested on hardware/simulator or headless · ⚠️ implemented, suspected to work, not yet verified · ❌ not supported by design.

| Feature | Status | Details |
|---|---|---|
| Compatibility renderer (OpenGL ES 3.0), device + simulator | ✅ | The renderer to ship; runs on every Apple TV incl. A10X. Templates must be built with `opengl3=yes` (see above). Verified on Apple TV 4K 1st gen and simulator; all four templates compile. |
| Siri Remote buttons as keys | ✅ | D-pad/clickpad edge → arrows, click (Select) → `KEY_ENTER`, Play/Pause → `KEY_MEDIAPLAY`, Menu → `KEY_MENU` down/up. Press + release verified on device with the exact keycodes below; default UI navigation and arrow/Enter bindings just work. |
| Remote touch as its own left stick | ✅ | Finger position on the pad is the deflection (absolute, corners included), on device `-3` (`Pads.REMOTE` in games); zeroed on lift, disconnect, and cancelled presses. Plain actions hear it, no pad's scoped copy does. Covered by simulator runs and the headless remote suite; on-TV steering confirmed working on hardware. |
| No doubled input | ✅ | `SDL_HINT_TV_REMOTE_AS_JOYSTICK=0` (`drivers/sdl/joypad_sdl.cpp`): the remote never arrives as a joystick, clicks stay keys, so nothing reports twice. |
| Menu behavior both ways | ✅ | `quit_on_go_back` on (default): press goes back to UIKit, preserving the system Menu-to-Home flow. Off: the app gets `KEY_MENU` down, `KEY_MENU` up, then `NOTIFICATION_WM_GO_BACK_REQUEST`, so taps and holds tell apart. Menu never doubles as a hardware key; hardware Escape reports as Menu and acts as one. Verified on device in both modes. |
| Stuck-key release | ⚠️ | `pressesCancelled` releases every synthesized remote key and zeroes the stick (`drivers/apple_embedded/godot_view_controller.mm`), but no deterministic trigger for the path exists, so it is implemented yet untested. |
| Quit hardening | ✅ | `SceneTree.quit()` is a warned no-op (`scene/main/scene_tree.cpp`): Apple forbids programmatic termination, so the main loop keeps running and the system suspends the app. Verified: frame counter advances after `quit()`. |
| Persistent storage | ✅ | `user://` maps to the app's `Library/Caches`; the tvOS sandbox denies file creation in `Documents`. Verified: writes succeed and round-trip to the Mac. |
| Text entry | ✅ | Standard virtual-keyboard API backed by a hidden text field (no always-on keyboard view like iOS); Menu-dismissable. Exercised via login navigation in end-to-end gameplay. |
| Export packaging | ✅ | Parallax `.imagestack` app icons (400x240 home + 1280x768 App Store marketing rendition), Top Shelf images (1920x720 + 2320x720 wide), Apple-TV-retargeted launch storyboard, tvOS-cleaned `Info.plist` (iOS-only keys dropped). Options with no tvOS meaning (camera, microphone, photo library, Files access) are hidden. Verified: export + sim build, icon renders on the home screen. |
| Honest display features | ✅ | Reports no touchscreen, no clipboard, no orientation changes (`platform/tvos/display_server_tvos.mm`); refresh rate from the screen (60 in Low Power mode), content scale from the window scene. Process spawning (`OS.execute`, `OS.create_process`) fails with `ERR_UNAVAILABLE`. |
| Oversize GLES3 textures | ✅ | Textures larger than `GL_MAX_TEXTURE_SIZE` are downscaled with a logged warning instead of sampling black (`drivers/gles3/storage/texture_storage.cpp`). Found via a 6801px sprite sheet showing as a black box on tvOS while fine on desktop GPUs. |
| End-to-end gameplay + regression | ✅ | BubblyField demo on device and simulator (login, Practice round, settings, bomb, blast, bricks, pickups, player damage, Defeat panel) plus `tests/tvos_remote_regression.gd` (127 checks) headless. |
| Forward+/Mobile rendering on Apple4+ GPUs | ⚠️ | tvOS GPU-family/MSL profiles, the `appletvos` shader toolchain, and device capability detection are implemented, and the gate names the detected family instead of failing silently. But pixels were never exercised — no Apple4+ hardware (Apple TV 4K 2nd gen, 2021, or newer) was available. |
| Physical MFi controllers | ⚠️ | Standard Godot joypad API path is kept (the SDL hint change is remote-specific); no physical MFi controller was available to test with. |
| Audio (CoreAudio) | ✅ | Tested and verified working on hardware — audio plays perfectly. Compiled in (`COREAUDIO_ENABLED`). |
| Networking / multiplayer sockets | ✅ | Tested and verified working on hardware — networking/multiplayer plays perfectly. Standard UNIX socket path. |
| HDR/EDR reporting | ⚠️ | Potential/current headroom reporting is implemented (`display_server_tvos.mm`); never exercised against an HDR display. |
| C# / .NET templates | ⚠️ | Build flags list `mono` as supported, but no C# template was ever built or tested here. |
| App Store submission | ⚠️ | Export output has not been through App Store validation. |
| Other tvOS versions | ⚠️ | Tested on tvOS 26.x (device) and 26.5–27 (simulator) only. |
| Forward+/Mobile on Apple1–3 GPUs | ❌ | Apple TV HD and Apple TV 4K 1st gen lack the required features; only Compatibility runs there. The gate stops with an alert naming the detected family. |
| Metal rendering in the simulator | ❌ | `MTLSimDevice` reports only Apple1/Apple2 with tier-1 argument buffers, no image cube arrays, and aborting shared placement heaps, so the gate always stops it. Simulator apps must use Compatibility. |
| Vulkan | ❌ | Forced off in `platform/tvos/detect.py`. |
| Touchscreen, clipboard, orientation | ❌ | Reported as unsupported by `has_feature`; there is no touch screen, clipboard, or orientation change on tvOS. |
| Camera, mic recording, photo library, Files app | ❌ | No tvOS meaning and no template libraries; export options hidden, `Info.plist` keys dropped. |
| Process spawning | ❌ | Prohibited by the sandbox (`drivers/unix/os_unix.cpp`); fails with `ERR_UNAVAILABLE`. |

### Remote input map

| Remote button | Godot event |
|---|---|
| Clickpad edge / D-pad | `KEY_UP` / `KEY_DOWN` / `KEY_LEFT` / `KEY_RIGHT` |
| Click (Select) | `KEY_ENTER` |
| Play/Pause | `KEY_MEDIAPLAY` |
| Menu | `KEY_MENU` down/up, plus the go-back request (see above) |
| Touch drag | Left stick (`JOY_AXIS_LEFT_X`/`JOY_AXIS_LEFT_Y`) on device `-3` |

On a pad, X is a bubble button; on the remote, the same physical press is
Play/Pause and opens the case/settings instead (see `Scripts/pads.gd` in
games). Menu is back and nothing else — never a bubble.

### Last verified

Sep–Oct 2026 against this branch:

- **Apple TV 4K 1st generation (A10X, tvOS 26.x), device builds:** 6-button
  remote matrix with press + release and the exact keycodes above; Menu in
  both `quit_on_go_back` modes with `GO_BACK_REQUEST` delivered to the scene
  tree (log-verified); `quit()` leaves the main loop running; Forward+ and
  Mobile probes show the Apple3 gate alert; `user://` round-trip to the Mac;
  end-to-end BubblyField gameplay.
- **tvOS 26.5–27 simulator (Mac):** BubblyField practice round, quit probe,
  Forward+/Mobile Apple2 gate alerts, and a native `MTLSimDevice` capability
  probe.
- **Headless:** BubblyField `tests/tvos_remote_regression.gd` (127 checks:
  input map, remote buttons, Menu tap/hold case use, request-fallback Menu,
  remote stick routing, pause/back routing, menu navigation, walking,
  bubbles, suspend/resume) passes.
- **Builds:** all four tvOS templates (release/debug × device/simulator,
  GLES3 + Metal) and the iOS release template compile cleanly.
- **Hardware re-test (Oct 2026):** audio playback, remote-stick steering
  on-TV, and networking/multiplayer all verified working perfectly on
  hardware.
