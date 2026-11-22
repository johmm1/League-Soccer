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


## Detailed build-system and renderer audit (2026-11-22)

The following points were verified directly in the current source:

- The top-level CMake configuration calls `find_package(OpenGL REQUIRED)` before creating the game targets. This must be conditional: Android should link the NDK-provided GLESv3/EGL libraries and must not search for desktop OpenGL.
- The current common include helper adds desktop OpenGL include directories to every object library. The Android path needs platform-specific include/link configuration rather than inheriting those variables.
- The executable is currently named `gameplayfootball`; runtime data is copied beside that desktop executable by a post-build command. Android must package these files as application assets instead.
- `OpenGLRenderer3D::CreateContext` creates an SDL desktop-style window and GL context, then loads the desktop function list from `sdl_glfuncs.h`. Android needs an SDL Android Activity-backed window and an ES context request (ES 3.x), with a GLES-only loader.
- The renderer currently logs and checks desktop OpenGL major/minor versions. That check is not a valid GLES capability test and must be replaced with an ES version/extension check.
- Frame capture currently uses desktop framebuffer/read-buffer constants and synchronous pixel readback. It should be disabled or implemented separately for Android rather than carried into the first mobile build.
- SDL initialization currently requests video, joystick and game-controller subsystems together. Android lifecycle and input availability need explicit handling; the app must tolerate pause/resume and context loss.

## Safe implementation decisions

1. Keep the desktop renderer and desktop CMake path working. Do not globally replace OpenGL headers or function names.
2. Introduce a separate Android/GLES renderer implementation behind the existing renderer interface, or perform a deliberate renderer refactor with compile-time platform separation. The choice must follow inspection of all renderer virtual methods and shader assets.
3. First Android milestone is a native arm64 build that creates an ES context and displays a diagnostic scene. Only after that should the full 3D scene, menus, assets, and match loop be enabled.
4. Do not report Android support until a real APK is produced and launched on an Android device.

## Remaining source audit

- [ ] Map all active fixed-function GL calls (not just commented examples) and identify which are reachable during startup/gameplay.
- [ ] Inspect shader files for desktop GLSL versions and unsupported syntax.
- [ ] Trace resource-manager path resolution and database writes before selecting Android asset/storage integration.
- [ ] Trace renderer thread ownership and SDL event loop before adding pause/resume handling.
- [ ] Audit touch/HID dispatch before designing on-screen controls.


## Shader and input audit (2026-11-22)

- The shipped shader files use desktop GLSL `#version 150`. Android OpenGL ES 3 requires GLSL ES source (normally `#version 300 es`) and precision declarations where required.
- The shaders use desktop-style fragment output declarations and a mixture of `texture2D` and newer `texture` calls. They need a deliberate ES shader variant/conversion and validation on a real GLES compiler; changing only the version line is not sufficient.
- The renderer has several render passes (ambient/SSAO, lighting/shadows, overlay and postprocess). A basic context test will not validate these; the mobile renderer milestone should bring up the simplest scene first, then enable passes incrementally.
- The keyboard HID implementation reads state through the engine's user-event manager. It is not a touch-control implementation. Android touch controls should inject virtual button/axis state through the same HID/gameplay input abstraction so gameplay logic remains shared.
- The resource manager requests resources by filename and delegates to loaders; the actual asset-root/path resolution is managed elsewhere. Android asset loading and writable database/config paths must be handled separately rather than assuming desktop relative paths work inside an APK.

## Updated risk order

1. GLES renderer and shader compatibility are the largest technical blockers.
2. Android build/dependency packaging and SDL Activity lifecycle are next.
3. Asset/database path redirection and app-private persistence follow.
4. Touch controls and low-end device performance tuning come after the game can render and run natively.


## Active renderer call-site inventory (2026-11-22)

A scan of the complete 2,682-line `opengl_renderer3d.cpp` found these Android-relevant call sites:

- Immediate-mode drawing is still present in active code, not just comments: `glBegin/glEnd`, `glVertex3f`, and `glColor3f/glColor4f` appear in sphere generation and both AABB debug-render paths. These must be converted to VBO/VAO-backed draws or excluded behind a deliberate debug-rendering option on Android.
- The renderer calls `glClientActiveTexture`, which is not part of OpenGL ES 3. It must be replaced by explicit vertex attribute arrays.
- The renderer uses `glMapBufferRange/glUnmapBuffer`; GLES 3 supports buffer mapping, but availability and flags must be checked against the actual NDK GLES headers and target API.
- Multiple render targets are used through `glDrawBuffers`, and fragment output locations are assigned with `glBindFragDataLocation`. These require GLES 3-compatible usage and shader output binding rules.
- The current pipeline requests floating-point render targets (including `GL_RGBA16F`), depth formats, depth comparison textures and framebuffer attachments. Each format/feature must be capability-checked on the Android GPU; unsupported optional effects need a fallback path.
- The source calls `glClearDepth`; GLES uses `glClearDepthf`. The GL mapping layer needs a platform-specific wrapper rather than assuming desktop function signatures.
- Frame capture/readback uses `glReadBuffer(GL_BACK)`, which is not a portable GLES path. Disable capture for Android's first milestone or use a GLES-safe framebuffer readback path.
- The renderer binds fragment outputs by name before linking. GLES 3 supports explicit fragment output locations in shader source; the desktop binding call should not be assumed available in the same form.

## Rendering strategy confirmed

The engine already has a modern VBO/VAO path for its main geometry, so the mobile port should preserve that shared geometry pipeline. The legacy immediate-mode calls are concentrated in procedural helper/debug drawing, while the larger remaining work is the desktop GL mapping and render-target/shader pipeline. The next code change should be a narrow platform abstraction and GLES compile path, not a wholesale replacement of the match renderer.

No C++ renderer changes have been committed yet. This inventory is a verified source audit, not a claim that the renderer is Android-ready.
