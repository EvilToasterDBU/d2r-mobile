# Native ARM Linux port

This directory is the home for a native Linux shell for ARM handhelds such as
the Retroid Pocket 6 running Armada OS. The existing Android build is a useful
reference for the host and Vulkan engine, but it is not a Linux executable:
its app shell, WSI surface, audio output, and guest thread-pointer patch target
all assume Android.

## Target

- ARM64 Linux userspace (glibc), Vulkan 1.1 or newer.
- Retroid Pocket 6 / Snapdragon SM8550 / Adreno 740, running Armada OS.
- SDL2 for the window, Vulkan surface, gamepad and optional touch events.
- SDL2 audio, using the output backend configured by the Linux system
  (normally PipeWire or PulseAudio on Armada).
- User-owned game files and generated import assets stay outside the source
  tree and are never distributed by this project.

Armada lists the Retroid Pocket 6 (SM8550) as tested. The Linux port still
needs runtime validation on the actual handheld; Android device results do not
count as Linux results.

## Port plan

1. Add a Linux ARM64 rewrite target and verify its thread-local storage access
   against glibc. Android's `DM_TARGET=android` output reads the Bionic thread
   pointer layout and cannot be assumed to work with glibc.
2. Add an SDL2 front end and Vulkan WSI presenter, reusing the existing Vulkan
   NVN replay engine while removing `ANativeWindow` and JNI dependencies.
3. Add native gamepad input, then implement launch, save and resource paths
   under the user's data directory.
4. Build an AArch64 Linux package and validate startup, rendering, controls,
   sound, suspend/resume and saves on the Retroid Pocket 6.

## Current status

The native Linux application is still incomplete. The shared host now has an
SDL audio backend, and the rewriter/loader have an initial `DM_TARGET=linux`
path: glibc thread-specific-data slot offsets are probed at startup, with the
inline TSD layout required for fast guest TLS accesses. The Linux compatibility
header also enables the common POSIX/Linux shims.

Game-derived headers and native-art source can be prepared outside the clone:

```sh
ROMFS="$HOME/dm-work/data/switch/3.3.94012/Update/RomFS"
GEN="$HOME/dm-work/build/linux/generated"
mkdir -p "$GEN"
python3 dm/ios/tools/gen_vault_table.py "$ROMFS" > "$GEN/dm_vault_tab.h"
python3 dm/ios/tools/gen_hud_exp.py "$ROMFS" > "$GEN/dm_hud_exp.h"
python3 dm/tools/gen_bright_levels.py "$ROMFS" > "$GEN/dm_bright_levels.inc"
python3 dm/ios/tools/native_art/generate.py "$ROMFS" --output-dir "$GEN"
```

These generated inputs and Linux runtime files are prepared under `dm-work`
for this workspace. On a new checkout, generate the private Linux game pack
with `d2rm_android.py --target linux`; it rewrites modules for glibc, translates
Vulkan shaders and prepares the resource overlay. Keep that output outside the
repository. The AppImage can either use that pack from disk or bundle its `out`,
`shaders` and `overlay` directories with `DM_BUNDLE_RUNTIME=1`.

The container needs `cmake`, `ninja-build`, `libsdl2-dev`, `libvulkan-dev`,
`pkg-config`, `curl`, `mksquashfs` and Python 3. `build_appimage.sh` builds the
ARM64 executable, stages the AppDir and downloads the official aarch64
`appimagetool` and AppImage runtime into `dm-work/build/linux/tools/`.

The AppImage contains the Linux launcher and bundled SDL/audio libraries. A
personal AppImage built with `DM_BUNDLE_RUNTIME=1` also carries the rewritten
modules, Vulkan shader bundles and resource overlay. It never bundles the
original RomFS or keys: keep the user's extracted RomFS at
`~/dm-work/data/switch/3.3.94012/Update/RomFS`. The launcher uses the bundled
runtime when present and otherwise reads it from
`~/dm-work/build/linux/linux/{out,shaders,overlay}`. `DM_ROOT`, `DM_LINUX_OUT`,
`DM_SHADER_DIR`, `DM_ROMFS`, `DM_ROMFS_OVERLAY` and `DM_SAVE` can override these
paths. Armada's system Vulkan loader, GPU driver and display stack remain host
dependencies. The launcher also reads `DISPLAY` from the active Gamescope
environment when a terminal did not inherit it. Never publish a personally
bundled AppImage or its runtime pack; these files are generated from the user's
game copy. The public GitHub release contains source and build instructions,
not game-derived runtime files.
Actual rendering, input and audio still need validation on Armada OS.
The Armada launcher defaults to Russian (`DM_LANG=ru-RU`) and a 720-pixel
internal short side; override the latter with `DM_LINUX_RENDER_HEIGHT=900` for
more detail or `540` for more speed. Presentation stays synchronized at 60 FPS
by default. On the Retroid's 120 Hz Gamescope output, `DM_LINUX_FPS=120` opts
into a synchronized 120 FPS cap for testing; this is still a bounded mode and
does not use the known game-logic-breaking uncapped mode (`DM_PRESENT_HZ=0`).
Linux now defaults to the original Switch controller layouts and gameplay
rules. `DM_LINUX_TOUCH_UI=1` explicitly selects the touch layouts. The left
stick has a 5% radial dead zone to prevent idle drift.

Because this Distrobox runs AArch64 userspace through QEMU on the x86_64 host,
stage the AppDir inside the container, then run the AArch64 packager through the
host's QEMU binary:

```sh
distrobox enter ubuntu-arm64 -- env DM_ROOT="$HOME/dm-work" DM_BUNDLE_RUNTIME=1 \
  DM_APPIMAGE_STAGE_ONLY=1 \
  sh "$HOME/GitHub/d2r-mobile-armada/dm/linux/build_appimage.sh"
APPIMAGE_EXTRACT_AND_RUN=1 ARCH=aarch64 qemu-aarch64-static \
  "$HOME/dm-work/build/linux/tools/appimagetool-aarch64.AppImage" \
  --runtime-file "$HOME/dm-work/build/linux/tools/runtime-aarch64" \
  "$HOME/dm-work/build/linux/AppDir" \
  "$HOME/dm-work/build/linux/dist/d2r-mobile-aarch64.AppImage"
```
