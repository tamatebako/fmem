# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

fmem is a small cross-platform C library that opens memory-backed `FILE *` streams on top of libc. It was originally written for [Criterion](https://github.com/Snaipe/Criterion) to stringify user-defined types. This is the `tamatebako/fmem` fork of `Snaipe/fmem`.

## Build, test, install

CMake-based build. Tests require [Criterion](https://github.com/Snaipe/Criterion) (v2.3.2+).

```bash
# Configure + build
cmake -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build

# Run tests (Criterion-based, via CTest)
ctest --test-dir build --output-on-failure

# Install
cmake --install build --prefix /usr/local
```

Run a single test by name (Criterion filter):
```bash
./build/test/unit_tests --filter fmem.cursor
```

List tests:
```bash
./build/test/unit_tests --list
```

### Forcing a specific backend

CMake auto-detects the best available backend, but for testing fallbacks you can override the detection cache variables (the original Travis matrix does this):

```bash
cmake -B build -DHAVE_OPEN_MEMSTREAM=FALSE -DHAVE_FOPENCOOKIE=FALSE
```

## Architecture

### Backend selection happens at CMake configure time

There is no runtime dispatch. `CMakeLists.txt` runs `check_symbol_exists` for every candidate libc/WinAPI symbol, then compiles **exactly one** of `src/fmem-*.c` into the library. The priority chain (first match wins):

1. `open_memstream` (glibc, BSDs) → `src/fmem-open_memstream.c`
2. `fopencookie` (glibc) → `src/fmem-fopencookie.c` + `src/alloc.c`
3. `funopen` (BSDs, macOS) → `src/fmem-funopen.c` + `src/alloc.c`
4. WinAPI temp file + memory mapping → `src/fmem-winapi-tmpfile.c`
5. `tmpfile()` fallback → `src/fmem-tmpfile.c`

Each `fmem-*.c` file independently defines the same four public symbols (`fmem_init`, `fmem_term`, `fmem_open`, `fmem_mem`). Adding a new backend = adding a new `src/fmem-<name>.c` implementing those four functions and a new `elseif` branch in `CMakeLists.txt`.

### The `fmem` ABI: an opaque 32-byte reserved region

`include/fmem.h.in` exposes:
```c
struct fmem_reserved { char reserved[32]; };
typedef struct fmem_reserved fmem;
```

The public type is intentionally a 32-byte blob. Each backend reinterprets those bytes through its own private struct via a `union fmem_conv { fmem *fm; struct <backend_state> *impl; }` cast at the top of every public function. This is how backends keep per-handle state (buffer pointers, cursors, Win32 `HANDLE`s, etc.) without leaking implementation details through the header. **Do not change the size of `fmem_reserved`** without bumping it on every backend — backends assume their state fits in 32 bytes.

### Header is generated, not committed

`include/fmem.h.in` is a CMake template. `configure_file()` produces `${BUILD_DIR}/gen/fmem.h`, substituting `@EXPORT_MACROS@` based on whether the library is built shared (Windows `__declspec(dllexport)` / GCC `__attribute__((visibility("default")))`) or static. The build also sets `FMEM_BUILD_LIBRARY` only on Windows for the dllexport side. Source files include `"fmem.h"` via the `gen/` include path — there is no committed `fmem.h`.

### Growing buffer strategy (`src/alloc.c`)

The `fopencookie` and `funopen` backends share `alloc.c`, which grows the backing buffer by φ (golden ratio, approximated as `207/128`) using integer math only. `fmemi_grow`, `fmemi_cursor`, `fmemi_copy` are the only shared helpers. Don't pull in floating point — the integer approximation is intentional.

## Testing

Tests live in `test/tests.c` and use Criterion's `Test(fmem, ...)` macros with a shared `TestSuite(fmem, .init=setup, .fini=teardown)` that calls `fmem_init`/`fmem_term` around each case. The custom `assert_written` macro flushes, calls `fmem_mem`, and compares the resulting buffer against an expected string. The `large` test writes 4 MiB to exercise buffer growth.

## CI / packaging

- **GitHub Actions** under `.github/workflows/`:
  - `ci.yml` — matrix build (Linux gcc/clang x64+ARM64, macOS Intel+ARM64, Windows MSVC+MinGW) plus a sanitizer build on Linux. Linux/macOS jobs build with tests (Criterion built from source — the old `ppa:snaipewastaken` is gone); Windows jobs build without tests.
  - `checks.yml` — trailing whitespace, CRLF line endings, and a CMake configure/install sanity check.
  - `release.yml` — manually-triggered release flow: bumps `VERSION` + `ChangeLog` via `.github/scripts/bump-version.sh`, opens a release PR, then on merge tags `v<x.y.z>` and publishes a GitHub release.
- `.travis.yml` and `.appveyor.yml` are the legacy CI configs (pre-GHA). They are kept for historical reference but are superseded by the GHA workflows; flag for deletion decision.
- **Packaging**: pkg-config (`fmem.pc.in`), CMake package config (`cmake/fmemConfig.cmake.in` — downstream uses `find_package(fmem)` and links `fmem::fmem`), and vcpkg (`vcpkg.json` manifest + `ports/fmem/` overlay).
- The library installs via CMake's `GNUInstallDirs`. The `fmem` target has `VERSION`/`SOVERSION` set from `VERSION` and exports an `fmem::fmem` namespaced alias.

## License

MIT — see `LICENSE`. Original author: Franklin "Snaipe" Mathieu.
