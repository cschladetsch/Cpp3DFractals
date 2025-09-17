# 3DFractals

This is a real-time Mandelbox and distance-estimator fractal explorer built with
SDL2, OpenGL, and GLSL. It extends Jan Kadlec's Boxplorer 1.02 with Catmull-Rom
keyframe splines, stereoscopic and panoramic capture pipelines, automated sequence
rendering, and a parameter editor powered by AntTweakBar. This README consolidates
the notes that were previously spread across `README`, `README-1.02`, `HISTORY`, and
the individual configuration folders.

### Demo

![Boxplorer2 screenshot](Demo1.jpg)

## Highlights

- Navigate Mandelbox, Menger, bulb, blend, and custom distance-estimator fractals in real time.
- Record keyframes, spline them into flight paths, and render deterministic TGA sequences.
- Output stereoscopic, cubemap, spherical, dome, and Oculus-ready imagery from the same scene.
- Live-edit shader parameters, camera state, and timing through the AntTweakBar overlay.
- Works on Windows, Linux, and macOS via CMake with bundled AntTweakBar and optional SDL2 fetch.
- Includes helper utilities for config editing, shader shrinking, and batch encoding.

## Repository Tour

- `main.cpp` and friends: main application sources.
- `cfgs/`: curated fractal presets (rrrola, knighty, syntopia, life) plus per-scene shader data.
- `textures/` and `resources/`: optional assets referenced by configs.
- `AntTweakBar/`: bundled AntTweakBar library.
- `utils/`: post-processing helpers, such as the x264 encoding scripts.
- Legacy docs (`README`, `README-1.02`, `HISTORY`) remain for reference.

## Prerequisites

- GPU with OpenGL 2.0+ and GLSL support.
- C++11-capable compiler (MSVC 2019+, clang, or GCC).
- CMake 3.10 or newer.
- SDL2 development headers and libraries.
  - CMake will fetch SDL2 2.28.5 automatically if not found (requires Git and internet access),
    otherwise point CMake at an existing installation via `-DSDL2_DIR=...`.
- Optional: Oculus SDK 0.4+ for `--oculus`, Sixense SDK for `HYDRA` builds.

## Building (CMake recommended)

```powershell
# From the repository root
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release
```

- Windows: run the commands in a Developer Command Prompt or open the generated solution
  in Visual Studio. CMake links against bundled AntTweakBar and auto-detects SDL2.
- Linux: install dependencies such as `sudo apt install build-essential cmake libsdl2-dev
  libgl1-mesa-dev`. Build with `cmake --build build -j`.
- macOS: install `cmake` and `sdl2` via Homebrew (`brew install cmake sdl2`) before configuring.

The default build places binaries under `build/` (or the generator-specific output directory).
CMake copies `cfgs/`, `fragment.glsl`, `vertex.glsl`, and `textures/` into the build tree so the
executables can be run from there.

### Legacy Makefiles

For historical parity the repository ships `Makefile.linux`, `Makefile.osx`, and
`Makefile.win32`. Copy the appropriate file to `Makefile`, adjust include and library paths,
and run `make` or `nmake`. These scripts expect SDL2 development headers to be present
system-wide.

## Launching Boxplorer2

```bash
./boxplorer2 [config.cfg | path/to/config_dir] [options...]
```

- If no argument is provided Boxplorer2 loads `cfgs/rrrola/default.cfg`. On Windows a file
  dialog is offered as a fallback.
- A configuration directory (for example `cfgs/rrrola/`) keeps the `.cfg` plus its `.cfg.data`
  folder, which stores per-scene shaders and rendered frames.
- Custom GLSL overrides: drop `vertex.glsl` and/or `fragment.glsl` next to the executable or
  inside the configuration directory to replace the embedded defaults. Run
  `prepare_default_shaders.bat` after editing the embedded shader headers.

### Key Runtime Files

- `*.cfg` - full camera state, render settings, parameter values.
- `*.cfg.data/` - derived content (fragment shader, cached textures, `keyframe-*.cfg`, TGA frames).
- `Demo1.jpg` - sample output.

## Command-Line Options

### Projection and Stereo Modes

- `--xeyed` - cross-eyed side-by-side layout (half horizontal resolution).
- `--sidebyside` - parallel side-by-side stereo (half width).
- `--overunder` - over-under stereo (half height).
- `--interlaced` - row-interlaced output (for example Zalman displays).
- `--anaglyph` - red-cyan anaglyph with shader defines.
- `--quadbuffer` - OpenGL quad-buffer stereo (requires suitable GPU and driver).
- `--spherical` - 360x180 panorama.
- `--dome` - 180-degree dome master.
- `--cubes` - render a six-face cubemap sequence (fixed 2048x2048 faces).
- `--oculus` - Oculus Rift warp (Windows plus Oculus SDK).

### Playback, Capture, and Automation

- `--render` - play back Catmull-Rom splines and dump `frame-XXXXX.tga` images into the scene data folder.
- `--speed` - when rendering, advance along the spline using constant distance steps.
- `--time` - when rendering, honor keyframe `delta_time` values (frame-time driven).
- `--loop` - force the spline to treat the first and last keyframes as connected.
- `--fixedfov` - keep field of view fixed during spline playback even if keyframes vary.
- `--fullscreen` - start in fullscreen (equivalent to pressing `Enter`).
- `--output=shot.tga` - take a single screenshot and exit.
- `--disable-dof` / `--enable-dof` - override depth of field enablement.
- `--disable-de` / `--enable-de` - toggle automatic distance-estimator navigation speed.
- `--disable-spline` / `--enable-spline` - force spline playback off or on regardless of config.
- `--kf=name` - change the keyframe filename prefix (default `keyframe`).

### Input Devices and Extras

- `--joystick=N` - pick a specific joystick index (defaults to the first Xbox 360-style pad).
- `--no360` - skip automatic Xbox 360 controller detection.
- `--lifeform=pattern.rle` - seed the backbuffer with a Conway's Game of Life pattern (plain text
  or RLE format) before rendering.

Options can be combined, for example `boxplorer2 cfgs/mymovie/ --render --speed --xeyed`.

## Controls

### Navigation

- `W`, `A`, `S`, `D` plus mouse - move and look.
- `Q` / `E` - roll the camera.
- `Z` / `C` or mouse wheel - adjust navigation speed.
- Hold `Shift` / `Ctrl` - temporarily double or halve speed.
- `ESC` - in fullscreen exits; in windowed mode releases the mouse (click to re-grab).
- `Enter` - toggle fullscreen and reload shaders.

### Keyframes and Playback

- `Space` / `Insert` - add a keyframe after the current one; `Ctrl+Space` overwrites the selected keyframe.
- `Home` - spline keyframes and start playback from the first keyframe.
- `End` - stop playback and jump to the last keyframe.
- `Tab` / `Backspace` - jump to next or previous keyframe.
- `Delete` - remove the last keyframe; `Ctrl+Delete` removes the selected keyframe.
- `Ctrl+Tab` - start playback from the selected keyframe.
- Hover over the blue timeline bars (with the mouse cursor released by `ESC`) to select keyframes; drag to retime.
- Numpad `7`/`9` or the mouse wheel while hovering the timeline - tweak per-keyframe speed.
- Numpad `2`/`4`/`6`/`8` - nudge the selected keyframe in the timeline.

### Parameter Editing Modes (AntTweakBar)

- Press `L`, `F`, `R`, `I`, `O`, `G`, or `0`-`9` to select the camera, FoV, raymarching, iteration,
  ambient occlusion, glow, or user-defined parameter channels respectively.
- Arrow keys adjust the active parameter (`Up`/`Down` change `.y`, `Left`/`Right` change `.x`).
- The AntTweakBar overlay mirrors changes and exposes additional toggles.
- `P` - flip stereo polarity; `Ctrl+P` saves the current config (`YYYYMMDD_HHMMSS.cfg`) and a TGA
  screenshot alongside it.

## Rendering a Movie

1. Explore a scene (for example `boxplorer2 cfgs/rrrola/`) and add keyframes with `Space`.
2. Exit, then restart with `--render` plus `--speed` or `--time`:
   ```bash
   ./boxplorer2 cfgs/mymovie/ --render --speed --sidebyside
   ```
3. Frames are written as TGA files into the scene's `.cfg.data` directory. Use the batch scripts
   in `utils/` (for example `encode-x264.bat`) or your tool of choice to assemble them into a video.

To loop perfectly, either add `loop 1` to the `.cfg` or pass `--loop`. For cubemap or 360-degree
content, include `--cubes`, `--spherical`, or `--dome`.

## Custom Fractals and Shaders

- Each preset keeps its shader overrides in `<scene>.cfg.data/fragment.glsl`.
- To explore a new shader, duplicate an existing configuration directory and replace its GLSL files.
- The CPU distance-estimator fallbacks read from shader-declared `DECLARE_DE` functions; ensure your
  shader exports the necessary symbols if you rely on DE-driven navigation.
- `DefaultShaders.h` and `ShaderProcs.h` contain the baked-in GLSL. Use
  `prepare_default_shaders.bat` to regenerate them after editing the source GLSL.

## Utilities

- `shadershrink` - cleans up GLSL for shipping and embeds it into headers.
- `edit-cfg` - command-line utility for editing `.cfg` files.
- `glsl` - quick GLSL experiment harness using the built-in fragment shader parser.
- `sdl2-test` - minimal SDL2 and OpenGL sanity check.
- `utils/encode*.bat` - Windows batch scripts for assembling renders into videos.

## Troubleshooting

- If SDL2 is not detected, set `SDL2_DIR`, `SDL2_INCLUDE_DIR`, or `SDL2_LIBRARY` when configuring CMake.
- Automatic SDL2 fetching requires network access; offline builds should pre-install SDL2.
- Depth-of-field, Oculus, or Sixense features compile only when the respective SDKs are available.
- When experimenting with embedded shaders, re-run `prepare_default_shaders.bat` to update header copies.

## History

See `HISTORY` for the full changelog. Highlights:

- **v2.00 (2011-01-23)** - stereo rendering, z-buffered splining, keyframe editing UI, AntTweakBar,
  animation renderer, new fractal presets.
- **v1.02 (2010-07-20)** - config files, screenshots and parameter snapshots, fullscreen switching,
  improved controls, smarter shader shrinking, Linux support.
- **v1.00 (2010-06-20)** - initial Mandelbox explorer release.

## License 

MIT
