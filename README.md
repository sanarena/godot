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
