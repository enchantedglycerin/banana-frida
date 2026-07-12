# banana-frida — stealth Frida 17.9.1 (Android arm64)

A patched fork of [Frida](https://github.com/frida/frida) **17.9.1**, built to run on Android
**without leaving the `rwxp` anonymous-memory footprint** that map-scanning anti-cheats key on.
Everything else behaves like stock Frida 17.9.1.

> This is a flattened snapshot fork (the upstream git submodules were vendored into one tree so
> the whole thing clones and builds without `git submodule` juggling). It is tagged `17.9.1` so
> the build reports the correct version.

## What's patched (source-level)
1. **Force W^X in frida-gum** (`gum_query_rwx_support()` → `GUM_RWX_NONE` on Android): gum
   allocates `RW` then `mprotect`s to `RX`, so it never creates a writable+executable page —
   the primary tell. (`gum_ensure_code_readable()` also softens execute-only pages to `RX`, not
   `RWX`.)
2. **Name gum's anonymous regions** as `[anon:dalvik-LinearAlloc]` / `[anon:dalvik-jit-code-cache]`
   via `PR_SET_VMA_ANON_NAME`, so they blend with ART instead of showing as unnamed anon maps.
3. **Rename runtime-visible threads** (`gum-js-loop` → `gc-worker`, pool workers,
   `frida-eternal-agent` → `gc-daemon`, `frida-agent-container` → `gc-container`).
4. **Rename the injected agent memfd** (`frida-agent-<arch>.so` → `art-jit-cache-<arch>.so`) —
   its `/proc/self/maps` name inside the target.

Full write-up, rationale, and on-device verification steps: **[STEALTH_FRIDA_BUILD_RESULT.md](STEALTH_FRIDA_BUILD_RESULT.md)**.
The exact diffs are also in `stealth-frida-gum.patch` / `stealth-frida-core.patch`
(base commits in `stealth-patches-BASE-COMMITS.txt`) — they apply cleanly onto a stock
frida 17.9.1 checkout if you'd rather patch upstream yourself.

## Build (Android arm64)
Needs **NDK r29** (exactly — frida pins major==29), Node 20, and meson/ninja:
```sh
export ANDROID_NDK_ROOT=/path/to/android-ndk-r29
./configure --host=android-arm64
ninja -C build subprojects/frida-core/server/frida-server
# output (already stripped): build/subprojects/frida-core/server/frida-server
```
The binary is **not** committed here — build it, or verify a handed-off binary against the
sha256 in the build write-up (the build is deterministic / byte-identical).

## Using it
- Host `frida-tools` must be **17.9.1** to match.
- **Run scripts with `--runtime=qjs`** (the default). Do **not** use `--runtime=v8` — V8's JIT
  allocates its own executable pages outside frida-gum and reintroduces an `rwx` region.
- Capture/verify helpers: `presence_only.js` (rwx-baseline probe) and `capture_hwbp.js`
  (hardware-breakpoint keystream capture, no `.text` patching).

## Credits
See [CONTRIBUTORS.md](CONTRIBUTORS.md). Upstream Frida © its authors, wxWindows Library
Licence 3.1 — this fork keeps that license.
