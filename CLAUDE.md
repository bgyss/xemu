# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

xemu is an original Xbox (and Chihiro) emulator built as a fork of QEMU (`qemu-system-i386`). Read `AGENTS.md` and `CONTRIBUTING.md` before making commits or PRs; their rules are mandatory and summarized below.

## Build

```bash
./build.sh                # release build -> build/qemu-system-i386, packaged into dist/
./build.sh --debug        # -DXEMU_DEBUG_BUILD=1, --enable-debug, log trace backend
./build.sh -j8            # job count
./build.sh -p win64-cross # cross-compile for Windows (also: -a <arch> on macOS)
```

`build.sh` runs `configure --target-list=i386-softmmu --extra-cflags=-DXBOX=1 ...` and then `make qemu-system-i386`; extra args are forwarded to `configure`. On macOS it first downloads prebuilt dependencies into `macos-libs/<arch>/` via `scripts/download-macos-libs.py` and requires a CLT SDK (arm64 min 14.0). After the first configure, incremental rebuilds can be done with `make -C build qemu-system-i386` (or `ninja -C build`).

Dependencies such as imgui, implot, SDL3, glslang, volk, VMA, SPIRV-Reflect, xxhash, tomlplusplus, `genconfig`, `nv2a_vsh_cpu` and `dsp56300` are Meson subprojects (`subprojects/*.wrap`).

## Tests

Xbox-specific unit tests live in `tests/xbox/` (Meson suite `xbox`, `build_by_default: false`). CI runs `./configure --target-list=i386-softmmu --disable-werror && make check-unit`.

```bash
meson test -C build --suite xbox             # all xbox tests
meson test -C build test-nv2a-vsh-carryover  # single test (also: test-nv2a-ptimer)
make -C tests/xbox/swizzle                   # standalone swizzle test (own Makefile)
```

These tests compile the real source file under test (e.g. `hw/xbox/nv2a/pgraph/vertex.c`) together with a mock (`mock-pgraph.c`, `mock-nv2a-ptimer.c`), so new tests follow the same pattern: add a subdir with a `meson.build`, list it in `tests/xbox/meson.build`.

Hardware-behavior changes are expected to come with (or suggest) a test XBE that can run on both real Xbox hardware and xemu.

## Architecture

QEMU core is largely untouched; Xbox-specific code is isolated under `hw/xbox/` and `ui/xemu*` / `ui/xui/`. Avoid modifying QEMU core outside these areas.

- **Machine**: `hw/xbox/xbox.c` (`TYPE_XBOX_MACHINE`) and `hw/xbox/chihiro.c` assemble the system: `xbox_pci.c` (MCPX/nForce host bridge, LPC, SMBus), the SMBus devices (`smbus_*.c` for the SMC, video encoders conexant/focus/xcalibur, EEPROM, temp sensor), `acpi_xbox.c`, and USB XID gamepads (`xid*.c`).
- **NV2A GPU** (`hw/xbox/nv2a/`): `nv2a.c` is the PCI device; each MMIO block has its own file (`pfifo.c`, `pgraph`, `pcrtc.c`, `pramdac.c`, `ptimer.c`, `pvideo.c`, ...) and register definitions are in `nv2a_regs.h`. PFIFO pulls pushbuffer commands and hands methods to PGRAPH.
- **PGRAPH** (`hw/xbox/nv2a/pgraph/`): `pgraph.c` decodes methods; the method list is the X-macro `methods.h.inc` (`DEF_METHOD(NV097, ...)`), included several times in `pgraph.c` to generate handlers and tables. Rendering is behind a backend vtable `PGRAPHRenderer` (`pgraph.h`), registered via `pgraph_renderer_register()` and switchable at runtime:
  - `gl/` — OpenGL backend
  - `vk/` — Vulkan backend
  - `null/` — no-op backend
  - `glsl/` — shared GLSL generation used by both GL and VK: fixed-function vertex shaders (`vsh-ff.c`), vertex programs (`vsh-prog.c`), pixel combiners (`psh.c`), and geometry shaders (`geom.c`)
  - Backend-independent helpers: `texture.c`, `swizzle.c`, `s3tc.c`, `vertex.c`, `surface.h`, `vsh_regs.h` / `psh_regs.h`

  Changes to emulated GPU behavior generally need to be made consistently in both the GL and VK backends.
- **MCPX** (`hw/xbox/mcpx/`): `apu/` is the audio processing unit — `vp/` is the voice processor (ADPCM, HRTF, SVF filters) and `dsp/` emulates the DSP56300 GP/EP cores (interpreter plus `dsp_jit.c`); `aci.c` is AC'97; `nvnet/` is the NIC.
- **Frontend/UI**: `ui/xemu.c` is the SDL-based main loop replacing QEMU's display frontends; `ui/xui/` is the Dear ImGui interface (menus, settings, snapshots, compatibility reporting). Settings are declared in `config_spec.yml` and turned into C structs at build time by the `genconfig` subproject; they are accessed through `g_config` (`ui/xemu-settings.h`). New user-facing settings belong in `config_spec.yml`.
- **Tracing**: subsystems have `trace-events` files (e.g. `hw/xbox/nv2a/trace-events`) using QEMU's tracetool; debug builds use the `log` backend.

## Conventions

- Commits: `<subsystem>: <Imperative short description>` (e.g. `nv2a/vk: ...`, `nv2a: ...`, `mcpx/apu: ...`) plus an explanatory body. Use `git log -n 5 <file>` to find the right prefix.
- Style: QEMU style (`docs/devel/style.rst`), except that declarations may appear at point of use and `//` comments are allowed. Run `clang-format` (repo `.clang-format`) on **new** files only; match local style in existing files. Never mix unrelated reformatting into functional changes.
- Comment only non-obvious code.
- One fix or feature per PR, with minimal changes, rebased on upstream `master`; check open PRs for duplicates first.
- No per-game hacks or title checks; model the hardware correctly. Enhancements must be optional and configurable.
- Do not invent register definitions or hardware behavior; cross-reference existing code in `hw/xbox/` or verified documentation.
- PR descriptions must include an agent/model declaration, e.g. `> **Agent Declaration**: This pull request was created with assistance from <Agent / Model>.`
