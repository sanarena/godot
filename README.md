# Apple TV (tvOS) port — agent quickstart (`tvos-port` branch)

> Fork note: this block only exists on the `tvos-port` branch of
> `https://github.com/sanarena/godot` (tracking upstream
> `godotengine/godot`, PR #123885). Everything below the "Upstream README"
> marker is stock Godot. Full feature/support matrix:
> [`platform/tvos/README.md`](platform/tvos/README.md) — read that next.

You are looking at a Godot source branch that adds an **Apple TV (tvOS)**
export platform. What it is: tvOS 26.0+, arm64 only, device + simulator,
Compatibility renderer to ship, Siri Remote as keys + its own stick.
End-to-end loop is: build editor → build 4 template libs → pack `tvos.zip` →
install template → export game (produces an Xcode project) → build in Xcode →
run on simulator or Apple TV.

## 0. Prerequisites (all commands run on a Mac)

- Apple Silicon Mac, Xcode with the tvOS SDK, SCons + Godot build
  dependencies (same as the
  [iOS](https://docs.godotengine.org/en/latest/engine_details/development/compiling/compiling_for_ios.html)
  and macOS compile docs).
- No `timeout` command on macOS; use bounded waits instead.
- Clone: `git clone -b tvos-port https://github.com/sanarena/godot.git && cd godot`

## 1. Build the editor (has the tvOS exporter built in)

```
scons platform=macos arch=arm64
# -> bin/godot.macos.editor.arm64
```

## 2. Build the four tvOS template libraries

`opengl3=yes` is mandatory — without it the Compatibility renderer is
missing and every game stops at the Metal capability gate:

```
scons platform=tvos target=template_debug arch=arm64 opengl3=yes
scons platform=tvos target=template_release arch=arm64 opengl3=yes
scons platform=tvos target=template_debug arch=arm64 simulator=yes opengl3=yes
scons platform=tvos target=template_release arch=arm64 simulator=yes opengl3=yes
# -> bin/libgodot.tvos.template_{debug,release}.arm64{,.simulator}.a
```

## 3. Pack the export template (`tvos.zip`)

Bundle device + simulator libs into xcframeworks:

```
xcodebuild -create-xcframework \
  -library bin/libgodot.tvos.template_debug.arm64.a \
  -library bin/libgodot.tvos.template_debug.arm64.simulator.a \
  -output libgodot.tvos.debug.xcframework
xcodebuild -create-xcframework \
  -library bin/libgodot.tvos.template_release.arm64.a \
  -library bin/libgodot.tvos.template_release.arm64.simulator.a \
  -output libgodot.tvos.release.xcframework
```

Stage a folder with exactly these six entries — the first four copied from
`misc/dist/apple_embedded_xcode/`, the last two fresh from above:

```
godot_apple_embedded/  godot_apple_embedded.xcodeproj/  data.pck
PrivacyInfo.xcprivacy  libgodot.tvos.debug.xcframework/
libgodot.tvos.release.xcframework/
```

then `zip -r tvos.zip <those six>` from inside the staging folder.

## 4. Install the template where the exporter reads it

```
mkdir -p ~/Library/Application\ Support/Godot/export_templates/4.8.dev
cp tvos.zip ~/Library/Application\ Support/Godot/export_templates/4.8.dev/tvos.zip
```

The version folder must match the editor **exactly** (editor Help → About;
currently `4.8.dev`). The exporter reads **the zip**, not any expanded
folder — after every rebuild, refresh this zip.

## 5. Set up a game project

- Renderer must be Compatibility:
  `rendering/renderer/rendering_method="gl_compatibility"`
  (project settings → rendering → renderer). Forward+/Mobile needs an
  Apple4+ GPU (Apple TV 4K 2nd gen, 2021+) and never runs in the simulator.
- Project → Export… → Add… → **tvOS** (`platform="tvOS"` in
  `export_presets.cfg`): bundle identifier, team, min tvOS 26.0.

## 6. Export (output is an Xcode project, not an app)

GUI: Export Project… with the tvOS preset. Headless:

```
bin/godot.macos.editor.arm64 --headless --path /path/to/game \
  --export-release "tvOS" /path/to/Game.xcodeproj
```

(`--export-debug` for the debug template.) "Export template missing" means
step 4 went to the wrong version folder.

## 7. Run on the tvOS simulator

Open the `.xcodeproj` in Xcode, pick an Apple TV simulator destination,
Run — or headless:

```
xcodebuild -project Game.xcodeproj -scheme Game -sdk appletvsimulator \
  -destination 'platform=tvOS Simulator,name=Apple TV 4K' build
xcrun simctl install booted <path to .app> && xcrun simctl launch booted <bundle.id>
```

Simulator apps must use the Compatibility renderer (the Metal gate always
stops `MTLSimDevice`, by design — it reports what is missing).

## 8. Run on a real Apple TV

1. Preset: uncheck Simulator, set team + provisioning; export, install via
   Xcode Devices, Apple Configurator, or
   `devicectl device install app --device <udid> <file.ipa>`.
2. **Launch by hand on the TV.** `devicectl device process launch` reports
   success/exit-0 but does not foreground the app — exit 0 there is not
   evidence of anything.
3. Observe via `devicectl device capture screenshot --device <udid>
   --destination shot.png` (4K PNG, downscale with `sips` before viewing).
   `idevice*` tools cannot see network-paired Apple TVs; devicectl only.

## 9. Iterate loop (engine change → retest)

After ANY engine-side change: rebuild the affected lib(s) (step 2) →
re-create the xcframework(s) → re-zip → re-copy `tvos.zip` (steps 3–4) →
re-export → rebuild the app. Then **prove the fix is in the binary before
testing**, an `Ld` line alone does not:

```
strings /path/to/Game.app/Game | grep <newSymbol>
```

## 10. Input + platform facts your game code needs

| Siri Remote | Godot event |
|---|---|
| Clickpad edge / D-pad | `KEY_UP/DOWN/LEFT/RIGHT` (plain `ui_*` nav just works) |
| Click (Select) | `KEY_ENTER` (`ui_accept`) |
| Play/Pause | `KEY_MEDIAPLAY` (use for in-game pause) |
| Menu | `KEY_MENU` down/up + go-back request (see below) |
| Touch drag | Left stick on the remote's own device (`-3`, `Pads.REMOTE`) |

- Menu with `quit_on_go_back` on (default): press returns to the system
  (Menu-to-Home flow preserved). With it off: `KEY_MENU` down, `KEY_MENU`
  up, then `NOTIFICATION_WM_GO_BACK_REQUEST` — taps and holds tell apart.
- `SceneTree.quit()` is a warned no-op (Apple forbids it); first scene
  should be a menu, games leave via the Menu flow, never `quit()`.
- `user://` maps to the app's `Library/Caches` (Documents is not writable).
- No touchscreen, clipboard, orientation changes, `OS.execute`
  (`ERR_UNAVAILABLE`), camera/mic/photo-library/Files. MFi pads work via
  the standard joypad API.

## 11. tvOS feature status (same table as `platform/tvos/README.md`, kept in sync)

Legend: ✅ tested on hardware/simulator or headless · ⚠️ implemented, suspected to work, not yet verified · ❌ not supported by design.
Full details live in [`platform/tvos/README.md`](platform/tvos/README.md); that file is canonical if the two ever disagree.

| Feature | Status | Details |
|---|---|---|
| Compatibility renderer (OpenGL ES 3.0), device + simulator | ✅ | The renderer to ship; runs on every Apple TV incl. A10X. Templates must be built with `opengl3=yes` (see above). Verified on Apple TV 4K 1st gen and simulator; all four templates compile. |
| Siri Remote buttons as keys | ✅ | D-pad/clickpad edge → arrows, click (Select) → `KEY_ENTER`, Play/Pause → `KEY_MEDIAPLAY`, Menu → `KEY_MENU` down/up. Press + release verified on device with the exact keycodes in §10 above; default UI navigation and arrow/Enter bindings just work. |
| Remote touch as its own left stick | ⚠️ | Finger position on the pad is the deflection (absolute, corners included), on device `-3` (`Pads.REMOTE` in games); zeroed on lift, disconnect, and cancelled presses. Plain actions hear it, no pad's scoped copy does. Covered by simulator runs and the headless remote suite — but the absolute-dpad rework landed after the Sep on-TV run, so on-TV confirmation of the current behavior is still pending. |
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
| Audio (CoreAudio) | ⚠️ | Compiled in (`COREAUDIO_ENABLED`); gameplay runs with it, but no isolated audio probe on TV hardware was recorded. |
| Networking / multiplayer sockets | ⚠️ | Standard UNIX socket path; online play was not exercised on TV hardware per the verification notes. |
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

---
*End of `tvos-port` fork notes. Upstream README follows unchanged.*
---

# Godot Engine

<p align="center">
  <a href="https://godotengine.org">
    <img src="misc/logo/logo_outlined.svg" width="400" alt="Godot Engine logo">
  </a>
</p>

## 2D and 3D cross-platform game engine

**[Godot Engine](https://godotengine.org) is a feature-packed, cross-platform
game engine to create 2D and 3D games from a unified interface.** It provides a
comprehensive set of [common tools](https://godotengine.org/features), so that
users can focus on making games without having to reinvent the wheel. Games can
be exported with one click to a number of platforms, including the major desktop
platforms (Linux, macOS, Windows), mobile platforms (Android, iOS), as well as
Web-based platforms and [consoles](https://godotengine.org/consoles).

## Free, open source and community-driven

Godot is completely free and open source under the very permissive [MIT license](https://godotengine.org/license).
No strings attached, no royalties, nothing. The users' games are theirs, down
to the last line of engine code. Godot's development is fully independent and
community-driven, empowering users to help shape their engine to match their
expectations. It is supported by the [Godot Foundation](https://godot.foundation/)
not-for-profit.

Before being open sourced in [February 2014](https://github.com/godotengine/godot/commit/0b806ee0fc9097fa7bda7ac0109191c9c5e0a1ac),
Godot had been developed by [Juan Linietsky](https://github.com/reduz) and
[Ariel Manzur](https://github.com/punto-) for several years as an in-house
engine, used to publish several work-for-hire titles.

![Screenshot of a 3D scene in the Godot Engine editor](https://raw.githubusercontent.com/godotengine/godot-design/master/screenshots/editor_tps_demo_1920x1080.jpg)

## Getting the engine

### Binary downloads

Official binaries for the Godot editor and the export templates can be found
[on the Godot website](https://godotengine.org/download).

### Compiling from source

[See the official docs](https://docs.godotengine.org/en/latest/engine_details/development/compiling)
for compilation instructions for every supported platform.

## Community and contributing

Godot is not only an engine but an ever-growing community of users and engine
developers. The main community channels are listed [on the homepage](https://godotengine.org/community).

The best way to get in touch with the core engine developers is to join the
[Godot Contributors Chat](https://chat.godotengine.org).

To get started contributing to the project, see the [contributing guide](CONTRIBUTING.md).
This document also includes guidelines for reporting bugs.

## Documentation and demos

The official documentation is hosted on [Read the Docs](https://docs.godotengine.org).
It is maintained by the Godot community in its own [GitHub repository](https://github.com/godotengine/godot-docs).

The [class reference](https://docs.godotengine.org/en/latest/classes/)
is also accessible from the Godot editor.

We also maintain official demos in their own [GitHub repository](https://github.com/godotengine/godot-demo-projects)
as well as the [Asset Store](https://store.godotengine.org/).

There are also a number of other
[learning resources](https://docs.godotengine.org/en/latest/community/tutorials.html)
provided by the community, such as text and video tutorials, demos, etc.
Consult the [community channels](https://godotengine.org/community)
for more information.

[![Code Triagers Badge](https://www.codetriage.com/godotengine/godot/badges/users.svg)](https://www.codetriage.com/godotengine/godot)
[![Translate on Weblate](https://hosted.weblate.org/widgets/godot-engine/-/godot/svg-badge.svg)](https://hosted.weblate.org/engage/godot-engine/?utm_source=widget)
