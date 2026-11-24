# Android Port Plan (Retroid Pocket 5)

Status: **planning, nothing implemented yet**. Written 2026-11-24 from a read of the code at `2ab0207`.
Repo: [mathias-ir/KIU-Android](https://github.com/mathias-ir/KIU-Android), an independent fork of UprisingNX.
There is no upstream remote; don't add one or merge from the original project.
This file is the hand-off document for any agent or contributor picking up the Android port.
Update the checklists and the "Decisions" / "Findings" sections as work lands.

**This is a replacement, not an addition.** The project becomes Android-only. Every piece of Nintendo
Switch code (libnx C shell, Horizon `cfg` gates, patched std/libc, mesa-switch, NRO packaging, the
updater, Kickstand tooling) is deleted or rewritten for Android. No Switch build is kept.
`2ab0207` is the last commit with a working Switch build. To look at old code, use
`git show 2ab0207:<path>`. Don't keep dead copies in the tree.

## Goal

Run the game on 64-bit Android devices in general (`arm64-v8a`). The **Retroid Pocket 5** is the
reference device and the only one we can test on, so it sets the performance bar. Don't hardcode
anything to it: core layout, screen size, controllers and GPU driver are all detected at runtime.

| | Switch (being removed) | Retroid Pocket 5 (reference device) |
|---|---|---|
| SoC | Tegra X1, 4× Cortex-A57 @ ~1 GHz | Snapdragon 865: 1× A77 @ 2.84, 3× A77 @ 2.42, 4× A55 |
| GPU | Maxwell, Mesa NVK (`mesa-switch`, statically linked) | Adreno 650, **Turnip v30** (Mesa) loaded via libadrenotools; system driver as fallback |
| OS | Horizon, libnx | Android 13 (verify), NDK |
| Exec memory | `CodeMemory`, 10 regions per process | `mmap` + `mprotect`, no region cap |
| Rust `std` | patched libc + `std` built from source | `aarch64-linux-android`, official tier-2 target |
| Display | 1280×720 handheld / 1080p docked | 1920×1080 panel, 16:9 (plus USB-C DisplayPort out) |
| Package | `.nro` via devkitA64 Makefile | `.apk` via Gradle + `cargo-ndk` |

The user model stays the same: **the APK ships zero game code or assets.** The user picks their decrypted
US 1.0 ROM (`0004000000030100`). On first launch, `zakuro-aot` compiles it on the device to AArch64.

## Why start from UprisingNX and not kiu-pc

- `zakuro-aot` already emits AArch64, which is the SD865's instruction set. The A77 is ARMv8.2, a superset
  of what the Switch runs. The generated code never touches `x18`, which Android reserves as the platform
  register (`zakuro/crates/zakuro-aot/src/compiler.rs:7`).
- The AOT runtime already has a non-Horizon Unix/AArch64 path with `mmap` exec buffers and icache flush
  (`zakuro/crates/zakuro-aot/src/runtime.rs:547-660, 810`). `compile_aot` in
  `zakuro/crates/zakuro-core/src/loader.rs:212` has the same path.
- kiu-pc (`Clefox9820/kiu-pc`) uses 3dsrecomp to translate the game to C on a PC, which takes about 75 min
  and 6 GB. On Android, that means either an APK with game code inside, which can't be distributed, or
  users who own a PC toolchain.
- The 16:9 single-screen UI, gamepad/gyro aim and lock-on, and the Rust gameplay overrides already exist
  here. kiu-pc uses keyboard/mouse and lists single-screen as WIP.
- kiu-pc is still a useful **reference** for the FSR 1 / Anime4K filters, 21:9, EU ROM support and its
  OpenGL fallback.

## Target architecture

`nx/` is replaced by `android/`. There is only one frontend, so there's no shared frontend crate to
extract. The libnx C shell (`nx/c`) disappears. Its jobs move into Rust, plus a thin Kotlin/Java layer
for things that only exist in the Android SDK (SAF picker, file paths).

```text
android/
  rust/     cdylib libuprising.so: today's nx/rust moved here (aim, menu, saves, settings,
            bootstrap, frame loop) plus android-activity glue, input, audio, storage
  app/      Gradle project: manifest, GameActivity (or NativeActivity), SAF ROM picker,
            icon/launcher assets
zakuro/     emulator crates, with every Horizon branch removed
```

Today the boundary between C and Rust is narrow. Once `main.c` is gone, it becomes internal Rust calls:

- `unx_create(nwindow, rom, data_dir, recomp)` → a Rust constructor that takes an `ANativeWindow`
  (from `android-activity`) and `data_dir` from `AndroidApp::external_data_path()`
- `unx_frame(handle, &UnxInput)` → called from the `android-activity` loop. `UnxInput`
  (`nx/rust/src/lib.rs:319`) is already platform-neutral: 3DS buttons, sticks, gyro, touch, host buttons
- `unx_destroy`, `unx_set_font`, `unx_minus_opens_menu` → plain Rust functions. Drop the `extern "C"`.

All Rust → C callbacks (`grep -rn 'unx_\|nx_audio' nx/rust/src`) get Android implementations in Rust:

| Callback | Switch implementation (delete) | Android replacement |
|---|---|---|
| `unx_aot_alloc` / `unx_aot_finish` / `unx_aot_free` (`bootstrap.rs:22`) | `nx/c/source/jit.c`, CodeMemory | `ExecBuffer::mmap_stream` (already in `runtime.rs`) |
| `unx_jit_make` (`zakuro-gpu/src/shader/jit.rs:719`) | CodeMemory for shader JIT | `mmap` RW → copy → `mprotect` RX, or the cranelift-jit path |
| `nx_audio_init/play/pause/exit` (`lib.rs:651`) | `nx/c/source/audio.c` (audren + WSOLA) | AAudio/Oboe sink, with WSOLA ported to Rust |
| `unx_display_resolution`, `appletGetOperationMode` (`lib.rs:657-658`) | applet queries, docked detection | `ANativeWindow` size. "Docked" means an external display, or drop the split |
| `unx_is_applet_mode` | applet vs title override | delete, along with every applet-mode branch |
| `unx_memory`, `unx_kernel_memory_mib` | `svcGetInfo` | `/proc/self/status`, `/proc/meminfo` |
| `unx_thread_role` (`lib.rs:655`) | core masks/priorities via svc | `sched_setaffinity` to the big cores, found at runtime from `cpuinfo_max_freq` (CPU 4-7 on SD865), `setpriority` |
| `unx_write_crash`, `unx_note_panic_logged` | SD card crash log | file in `data_dir` + logcat (`android_logger`) |
| `unx_updater_*` (`updater.rs:29`) | libcurl + nifm, NRO swap | delete `updater.rs` entirely. Updates come through GitHub Releases / Obtainium |
| `unx_bootstrap_step` | progress bar render step | same idea, rendered through the Android presenter |

### Touchscreen input (keep it)

The single-screen layout keeps full touchscreen support, as on the Switch. Right-stick and gyro aiming are
optional extras that sit alongside touch. They never replace it, and touch stays usable whether or not
they're turned on.

What exists today and has to survive the port:

- `touch_on_bottom` (`nx/rust/src/lib.rs:1305`) maps a touch on the screen onto the 3DS bottom screen
  (320x240) for each layout. That includes the reworked Chapter Select, Mode Select, Solo Hub and Fiend's
  Cauldron screens, and turns the touch into reticle aim when the layout is top-screen only.
- `aim.rs` already gives a real touch priority. A real touch turns off fallback stick/gyro touch aim, and
  the `MOUSE_AIM` path only runs while nothing is touching. Keep that behaviour.
- `menu.rs` `filter` keeps touches from reaching the game while the menu is open and just after it closes.

Android work:

- Read the first pointer from GameActivity motion events (`ACTION_DOWN` / `MOVE` / `UP` / `CANCEL`) into
  `UnxInput.touching/touch_x/touch_y`. Ignore any other pointers.
- Every touch position is still written for a 1280x720 screen. That covers `touch_on_bottom`, the Fiend's
  Cauldron Back button and the cauldron slider hit box near `lib.rs:2040`. Android screens have other sizes
  and aspect ratios, and the image is letterboxed by the output filter. Convert the touch from surface
  pixels to the presented game viewport first, using the same rect the FSR/blit pass uses, then scale it
  to 1280x720. Drop touches that land in the letterbox bars. The existing mapping code then works
  unchanged. Recompute the rect when the swapchain is recreated (rotation, resize, external display).
- Turn off Android's default gesture handling over the game surface so a drag reaches the game unchanged:
  immersive mode, with no system back gesture while a touch is held.
- On an external display with no touch, nothing changes. Stick and gyro aim still work.

## Full inventory of Switch code and its replacement

### A. `cfg(target_os = "horizon")` gates in `zakuro/`

Collapse every gate to a single code path and delete the Horizon branch. Some Horizon branches carry
values tuned for a weak handheld CPU that are also right for Android. For those, keep the value and drop
the `cfg`. Gate on `target_os = "android"` only where a Linux host build used for tests needs something
different (see Decisions).

| File:line | What | Action |
|---|---|---|
| `zakuro-aot/Cargo.toml:13` | `libc` for unix non-horizon | drop the horizon condition. `libc` is unconditional on unix |
| `zakuro-aot/src/runtime.rs:547,583,591,810` | `mmap` exec buffers, streaming cache load | keep the unix path and delete the CodeMemory path. **Verify** it reads into anonymous memory and then calls `mprotect` RX, because Android 10+ forbids exec from app-writable files |
| `zakuro-aot/src/runtime.rs:658` | icache flush for aarch64 | keep the unix path |
| `zakuro-aot/src/compiler.rs:195` | compile threads: 3 on Horizon | replace with the number of big cores detected at runtime (4 on SD865). Little cores are slow |
| `zakuro-core/src/loader.rs:212,235` | `compile_aot` only on unix aarch64 | remove the horizon half of the condition |
| `zakuro-core/src/recompiled.rs:46` | `FOUND` table 8192 vs 65536 | keep 65536 (the RP5 has 8 GB RAM) |
| `zakuro-core/src/recompiled.rs:209` | file-based recomp lookup | keep the non-Horizon path |
| `zakuro-core/src/hints.rs:74` | remove before rename (Horizon rename can't overwrite) | delete the workaround |
| `zakuro-core/src/services/shared_font.rs:410` | serial vs rayon glyph build | keep rayon |
| `zakuro-core/src/services/fs.rs:1184` | default for a "fast" fs flag | probably keep the Horizon value. Measure |
| `zakuro-core/src/gpu_thread.rs:397,404` | hooks into Switch profiler | delete (later: ATrace / simpleperf markers if needed) |
| `zakuro-fs/Cargo.toml:18`, `src/lib.rs:22-194` | ROM via `memmap2` vs Horizon read+cache | keep `memmap2` and delete the Horizon reader. A SAF `content://` URI gives an fd: `mmap` the fd, or copy the ROM to app storage once |
| `zakuro-gpu/Cargo.toml:41,48` | `ash` "loaded" vs "linked"; cranelift-jit needs `mmap` | "loaded" only (system `libvulkan.so`), cranelift-jit available. Delete the `linked` / mesa-switch feature |
| `zakuro-gpu/src/lib.rs:247` | shader warmup at boot | make it unconditional |
| `zakuro-gpu/src/shader/jit.rs:19-863` (~22 gates) | Horizon: custom install into CodeMemory, compile queue, on-disk shader cache, preload. Desktop: cranelift-jit module, one thread per compile | Start with the cranelift path. Port the Horizon queue and disk cache onto `mmap`/`mprotect` later, because they cut stutter. `cache_path` at `:827` hardcodes `sdmc:`. Then delete the CodeMemory code |
| `zakuro-gpu/src/backend/vulkan.rs:44,181,200,215` | `SurfaceSource::Vi`, `VK_NN_vi_surface` | **replace** with `SurfaceSource::Android(*mut ANativeWindow)` + `VK_KHR_android_surface` (`ash::khr::android_surface`) |
| `zakuro-gpu/src/backend/vulkan.rs:523` | swapchain extent from window size on Horizon | Android reports `current_extent`. Also handle `preTransform` / rotation |
| `zakuro-gpu/src/backend/vulkan.rs:1217` | present wait timeout 250 ms vs infinite | keep 250 ms (avoids hangs on driver bugs) |
| `zakuro-gpu/src/raster/hardware.rs:547` | readback row path | keep the non-Horizon path |
| `zakuro-gpu/src/raster/hardware.rs:696-709,807` | pipeline cache path (hardcoded `sdmc:`), cold-cache mode, remove-before-rename | path from `data_dir`. Delete the rename workaround |
| `zakuro-gpu/src/raster/hardware.rs:1515` | 2 vs N of something (frames in flight / workers) | try the Horizon value first |
| `zakuro-gpu/src/raster/hardware.rs:4580,5842,6988` | surface `checked` flag | the Horizon behaviour is likely right for tile-based GPUs. Measure, then keep one |
| `zakuro-gpu/src/raster/hardware.rs:7125,7156` | fence wait in 500 ms slices vs plain | keep the sliced wait (safer with driver hangs) |
| `zakuro-gpu/src/raster/hardware.rs:7887` | `Entry::linked()` vs `Entry::load()` | `load()` only |
| `zakuro-gpu/src/raster.rs:996,1013,1102,1238,7803` | vertex parallelism, subpixel snap, serial vs rayon fill | Start with the non-Horizon branches. At `:1238`, Horizon skips 1/16 px snapping; keep the snapping unless profiling says otherwise |

When done, `grep -rn 'horizon' zakuro/ android/` should only match unrelated words (egui `horizontal`).

### B. Hardcoded `sdmc:` paths

Replace all of them with a `data_dir` passed down from the frontend. On Android that is
`Context.getExternalFilesDir()` (`/sdcard/Android/data/<pkg>/files`), so the user can still drop saves
into it with a file manager.

- `zakuro-gpu/src/shader/jit.rs:833`
- `zakuro-gpu/src/raster/hardware.rs:702, 709`
- `zakuro-core/src/services/apt.rs:70-72`
- `zakuro-core/src/services/shared_font.rs:447-450`
- `nx/rust/src/{lib,settings,saves,bootstrap,menu,updater}.rs`: 53 occurrences in total across `nx/rust` and `zakuro`
  (`grep -rn 'sdmc:' nx/rust/src zakuro/crates`)

### C. `nx/c` (libnx C shell): delete all of it

| File | Did | Replacement |
|---|---|---|
| `nx/c/source/main.c` | init, applet hooks, `padGetButtons`/`padGetStickPos`, `hidGetSixAxisSensorStates`, touch, the frame loop | `android-activity` event loop in `android/rust`: GameActivity input events (gamepad keys + axes), `ASensorManager` for gyro, touch for the bottom-screen overlay |
| `nx/c/source/jit.c` | CodeMemory allocator | `mmap`/`mprotect` (already in `runtime.rs`) |
| `nx/c/source/audio.c` | audio thread, WSOLA → audren | Oboe/AAudio callback with WSOLA ported to Rust |
| `nx/c/source/updater.c` | libcurl, nifm, NRO swap | nothing |
| `nx/c/source/prof.c` | stack-sampling profiler via svc | nothing (use `simpleperf` / Android GPU Inspector) |
| `nx/c/source/nx_shims.c` | libc gaps on Horizon | nothing (bionic has them) |
| `nx/c/Makefile`, `nx/c/icon.jpg` | NRO packaging + icon | Gradle + `cargo-ndk`, launcher icon in `android/app/src/main/res` |

### D. `nx/rust/src` (~7k lines) → `android/rust/src`

Move the crate with `git mv`, done by the maintainer: agents leave renames to them. Then change the
crate type from `staticlib` to `cdylib`.

| File | Action |
|---|---|
| `aim.rs` | move as-is. It only uses `UnxInput` |
| `menu.rs` | move. Fix font loading and paths, and remove updater/applet entries |
| `saves.rs` | move after the `sdmc:` removal. Import/export goes to a user-reachable folder |
| `settings.rs` | move. Simplify the handheld/docked split to a single profile, or "external display" |
| `bootstrap.rs` | move. Replace `alloc_nx_jit` (`:229`) with `ExecBuffer::mmap_stream`. ROM discovery reads the SAF-picked file |
| `lib.rs` | move the frame loop. Replace `unx_create`'s `nwindow` + `VulkanPresenter::new_vi` (`:1084`) with the Android surface. Delete `query_window_size` and the externs at `:650`, implementing their jobs in Rust (table above) |
| `updater.rs` | delete. This also removes the `Angle-Brackets/UprisingNX` release URL at `:21` |

### E. Build system, scripts, CI: delete or replace

| Path | Action |
|---|---|
| `scripts/cargo-nx.sh`, `scripts/horizon-std-setup.sh` | delete |
| `patches/libc-0.2.191-horizon-aarch64.patch`, `patches/target-lexicon-switch.patch`, `patches/mesa-switch-linux-rust-target.patch` | delete |
| `patches/3dsrecomp-label-hints.patch`, `patches/recomp-h-aarch64-vfp.patch`, `scripts/build-recomp-nx.sh`, `scripts/o2-hot-shards.sh`, `scripts/a1-codesize.sh`, `scripts/desktop-port.sh` | delete: leftovers of the devkitA64/3dsrecomp path. First check that nothing in `zakuro/` still reads their outputs |
| `scripts/prof.py` | delete (it parses `prof.c` output) |
| `scripts/release.sh` | rewrite for APK: VERSION bump check, `./gradlew assembleRelease`, signing |
| `scripts/build-raster-shaders.sh` | keep. Remove NVK-specific notes and re-check whether `-O` still matters on Adreno |
| `probes/zakuro-nx` | delete |
| `probes/std-probe` | turn into the M0 Android probe, then delete once M0 is done |
| `nx/rust/Cargo.toml` `[patch.crates-io]` (libc/target-lexicon → `build/vendor/`) | delete |
| `.github/workflows/tag-build.yml` | rewrite: NDK + `cargo-ndk` + Gradle, upload a signed APK to Releases |
| `README.md` | rewrite for Android. Remove the Switch install steps, Kickstand workflow and releases links. **Keep the credits section.** |
| `LICENSE`, `zakuro/LICENSE`, `zakuro/UPSTREAM.md` | keep (the MIT notices are required) |
| `tools/kiu_romfs.py`, `VERSION` | keep |

New toolchain: `rustup target add aarch64-linux-android`, NDK r26+, `cargo-ndk`, Gradle, JDK 17.

## Risks and open questions

1. **GPU drivers.** The primary driver is **Turnip v30** (Mesa's Adreno driver), loaded through
   [libadrenotools](https://github.com/bylaws/libadrenotools) as Azahar and the Yuzu forks do. Qualcomm's
   stock SD865 driver is buggy and may only report Vulkan 1.1, so it's only a fallback. Develop and test
   against Turnip, but keep `zakuro-gpu` within what it needs
   (`grep -n 'extension\|features' zakuro/crates/zakuro-gpu/src/backend/vulkan.rs zakuro/crates/zakuro-gpu/src/raster/hardware.rs`)
   and fail with a clear message when a feature is missing.
   libadrenotools only works on Adreno. Mali/PowerVR/Xclipse devices always use their system driver,
   and we can't test those. Treat them as best effort.
2. **Exec memory.** The AOT cache must be loaded into anonymous memory; a file in app storage can't be
   mapped `PROT_EXEC`. Confirm that `ExecBuffer::mmap_stream` loads it that way.
3. **Does the RP5 have a gyroscope?** Unverified. If it doesn't, gyro aim is disabled.
4. **Tile-based GPU.** Adreno is tile-based, so render-pass load/store ops and readbacks
   (`raster/hardware.rs`) may cost more than on Maxwell. Profile early.
5. **Thermal / big.LITTLE.** Pin the game, GPU and compile threads to the big cores. Detect them by
   reading `/sys/devices/system/cpu/cpu*/cpufreq/cpuinfo_max_freq` (CPU 4-7 on SD865); other SoCs differ.
6. **Android lifecycle.** The surface is destroyed on pause and when the app is backgrounded. The presenter
   must rebuild its swapchain, and the core must pause cleanly.
7. **Baseline.** Azahar already runs this game on SD865. Compare performance against it at each milestone.
8. **Only one test device.** Anything non-RP5 (other SoCs, aspect ratios, Android versions) is untested
   until someone else reports on it. Keep the logs useful (logcat + crash file with device and driver info).
9. **No runnable build between M1 and M3.** Deleting the Switch shell before the Android frontend renders
   leaves the repo without a playable target for a while. Accepted. M2's headless CLI is the safety net.

## Milestones

Each milestone should end with something running on the device and a short note under "Findings".

### M0: Toolchain and host check (no game)
- [ ] `aarch64-linux-android` builds for `zakuro-common`, `zakuro-fs`, `zakuro-cpu`, `zakuro-aot`
- [ ] Port `probes/std-probe` to an Android executable and run it over `adb shell` (threads, TLS, rayon, fs, big allocs)
- [ ] Small exec-memory test: `ExecBuffer::mmap_stream` + a call into a generated stub on the device
- [ ] Record the Android version and whether there is a gyro
- [ ] Record the Vulkan version and extensions (`vulkaninfo` via adb) for both the stock driver and Turnip v30

### M1: Strip the Switch
- [ ] Delete `nx/c`, the Switch scripts, patches, `probes/zakuro-nx` and `updater.rs` (section E/C/D)
- [ ] Collapse every `cfg(target_os = "horizon")` in `zakuro/` (section A). `SurfaceSource::Vi` is gone
- [ ] Move `nx/rust` → `android/rust` (maintainer does the `git mv`), as a `cdylib` with the externs replaced by Rust stubs
- [ ] `sdmc:` paths replaced by a `data_dir` parameter (section B)
- [ ] `cargo ndk -t arm64-v8a build` succeeds for the whole workspace

### M2: Headless AOT on device
- [ ] `adb push` a ROM to `/data/local/tmp` and run a CLI binary that does the `zakuro-aot` compile + cache write
- [ ] Load the cache back and run the core headless for N frames (no GPU present, or offscreen)
- [ ] Measure the compile time against the Switch (30-60 s) and the cache size (~87 MiB)

### M3: Title screen in an APK
- [ ] `android/app` Gradle skeleton + `cargo-ndk`, `android-activity` with GameActivity or NativeActivity
- [ ] `SurfaceSource::Android` + `VK_KHR_android_surface` in `zakuro-gpu`
- [ ] Load Turnip v30 through libadrenotools from the start, from a driver package extracted into app-private storage. Fall back to the system `libvulkan.so` if loading fails or the GPU isn't Adreno
- [ ] Hardcoded ROM path (app files dir, pushed via adb). The title screen renders on Turnip

### M4: Playable with controls and audio
- [ ] Gamepad → `UnxInput`, gyro if present
- [ ] Touchscreen → `UnxInput` mapped through the presented viewport (see "Touchscreen input"). Every touch-driven screen works on the RP5 panel, including when it's letterboxed
- [ ] Oboe/AAudio audio with the WSOLA stretcher ported to Rust
- [ ] Thread roles: `sched_setaffinity` / `setpriority` replacing `unx_thread_role`
- [ ] Lifecycle: pause/resume, surface lost/recreated
- [ ] Play through Chapter 1 (air + land sections)

### M5: First-run experience
- [ ] SAF ROM picker. Validation reuses the `discover_rom` notices (encrypted, wrong title)
- [ ] First-launch AOT progress screen (the `unx_bootstrap_step` equivalent)
- [ ] Save import/export in a user-visible folder
- [ ] **GPU driver picker**: import any Turnip/Adreno driver `.zip` (the usual adrenotools package format with `meta.json`), list the installed ones, pick one, or pick "system driver". The choice takes effect on the next launch. Hide the picker on non-Adreno GPUs
- [ ] Settings presets for the 1080p panel (maybe `2x`/`3x` defaults). Remove the leftover docked/handheld wording

### M6: Performance, polish, release
- [ ] Measure each tunable kept from section A
- [ ] Port the shader-JIT disk cache + preload (formerly Horizon-only) onto `mmap`. Baked pipeline warmup on boot
- [ ] Profile the Chapter 14 flight, the old reference stress scene, with simpleperf + AGI
- [ ] Compare against Azahar on the same device
- [ ] **FSR 1 window-adaptive scaling filter**: the final pass that scales the game image to the window. FSR 1 runs as EASU (upscale) and then RCAS (sharpen), and is the default
  - Input is the internal render size (the 3DS resolution × the scale setting). Output is the current swapchain extent, so the same pass covers the 1080p panel, external displays, other devices' screen sizes, and a window resized by split-screen or freeform mode
  - Recompute the scale factors and EASU constants whenever the swapchain is recreated. Keep the aspect ratio, letterboxed
  - When input and output are the same size, skip EASU and run only RCAS (or nothing)
  - Settings: filter (FSR 1, bilinear, nearest/integer) and an RCAS sharpness slider
  - Shaders are compiled to SPIR-V at build time. kiu-pc's FSR 1 is the reference implementation
- [ ] Later, optional: extra built-in filters (Anime4K, CRT/LCD) or user-supplied `.spv` files from `data_dir/shaders/`
- [ ] Release: signed APK on GitHub Releases from the rewritten `tag-build.yml`. Rewritten README

## Rules for whoever works on this

- Never commit ROMs, decrypted game code, AOT caches, or anything under `build/`. The APK must contain
  zero copyrighted game code or assets.
- Don't keep Switch compatibility. Delete Horizon code rather than gating it, and don't add
  `cfg(target_os = "horizon")` anywhere. The old code lives at `2ab0207`.
- Keep the MIT notices: `LICENSE`, `zakuro/LICENSE`, and the README credits.
- Don't stage or commit, including `git mv` / `git rm`. Delete with plain `rm` and leave renames to the
  maintainer, who does all staging and commits.
- Code comments only where the code isn't self-explanatory, kept short.

## Decisions

- 2026-11-24: Android-only. All Switch code is removed, not kept behind `cfg` gates.
- 2026-11-24: Target all `arm64-v8a` Android devices; the RP5 is the reference device and the only tested one.
- 2026-11-24: Turnip v30 via libadrenotools is the primary driver; the stock driver is a fallback. Users can swap drivers in-app (M5).
- 2026-11-24: Touchscreen support stays in the single-screen layout. Right-stick and gyro aim are optional extras next to it, not replacements.
- 2026-11-24: Output scaling uses FSR 1 (EASU + RCAS) as a filter that adapts to the window size: render at the internal resolution, then scale to whatever the swapchain extent is. Bilinear and nearest are the alternatives; other filters come later (M6).
- _(open)_ Keep `zakuro/` compiling on a Linux host for unit tests and the M2 CLI? Default: yes, if it's
  free (the non-Horizon branches are already Linux-compatible). Don't add host-only code to keep it.
- _(record further choices here with the date: e.g. GameActivity vs NativeActivity, Oboe vs AAudio, minimum Android version)_

## Findings

_(record device measurements and surprises here with the date)_

- 2026-11-24: `ExecBuffer::mmap_stream` (`zakuro-aot/src/runtime.rs:592`) already does what Android needs. It maps anonymous `MAP_PRIVATE | MAP_ANONYMOUS` RW memory, streams the cache into it, flushes the icache, then calls `mprotect` RX. No file is ever mapped executable, so risk 2 is resolved in code. Still to confirm on the device (M0).
