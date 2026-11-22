# FloodlightFC Android port status

This document tracks the Android-only port of the existing Gameplay Football engine. The original desktop sources are being retained while the Android target is brought up.

## Current baseline

- Native C++17 game built around SDL2.
- Existing 3D match engine, player/ball simulation, menus, career/league systems and SQLite persistence.
- Desktop build currently discovers desktop OpenGL and SDL2 packages directly in the top-level CMake configuration.
- The current renderer is not an OpenGL ES renderer.

## Verified Android blockers

1. **Desktop OpenGL API usage** — `src/systems/graphics/rendering/opengl_renderer3d.cpp` includes desktop OpenGL headers and uses compatibility-era calls such as `glBegin`, `glEnd`, `glMatrixMode` and `glLoadIdentity`. Android exposes OpenGL ES, so this renderer cannot simply be compiled unchanged for Android.
2. **GL function loader** — `src/systems/graphics/rendering/sdl_glfuncs.h` contains desktop-only entry points and needs an ES-compatible function set / loading strategy.
3. **Desktop CMake dependencies** — the game target currently requires `find_package(OpenGL REQUIRED)` and desktop SDL2 package discovery. Android needs an NDK toolchain configuration and Android-compatible SDL2/native dependencies instead.
4. **Android lifecycle and packaging** — the project currently has no verified Android application wrapper, SDL Android Activity integration, manifest, APK packaging or Android CI build.
5. **Input and persistence** — touch controls and Android app-private save/config paths still need to be integrated and tested.

## Porting order

- [ ] Add an Android NDK / Gradle application wrapper and SDL Android Activity integration.
- [ ] Split platform dependency discovery in CMake so Android uses GLES and Android-compatible SDL libraries while desktop configuration remains intact.
- [ ] Port the renderer to OpenGL ES. Replace immediate-mode drawing with buffer-based geometry and adapt shader sources and GL function loading. Do not hide unsupported calls behind macros and call the renderer Android-ready.
- [ ] Integrate Android lifecycle events (pause, resume, surface/context recreation) with the engine's render thread.
- [ ] Package media, locale, configuration and database assets; move writable data to app-private storage.
- [ ] Add landscape touch controls and scalable quality settings.
- [ ] Build arm64-v8a APK in CI, install it on a device, and verify launch, menus and a complete match.

## Acceptance checks

An APK is not considered complete merely because it installs. The Android build must launch without a blank screen, load the game's assets and database, render the match, accept touch input, survive pause/resume, and complete a match without crashing.

## Current state

This is an engineering audit, not a claim that an Android build is already working. No Android APK has yet been produced or device-tested.
