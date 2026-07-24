# fmem

[![CI](https://github.com/tamatebako/fmem/actions/workflows/ci.yml/badge.svg?branch=master)](https://github.com/tamatebako/fmem/actions/workflows/ci.yml)
[![Checks](https://github.com/tamatebako/fmem/actions/workflows/checks.yml/badge.svg?branch=master)](https://github.com/tamatebako/fmem/actions/workflows/checks.yml)
[![GitHub release](https://img.shields.io/github/v/release/tamatebako/fmem)](https://github.com/tamatebako/fmem/releases)

A cross-platform library for opening memory-backed libc streams.

This library was written for [Criterion][criterion] to implement stringification functions for user-defined types.

This is the `tamatebako/fmem` fork of `Snaipe/fmem`, with a modernized build
system, GitHub Actions CI, and pkg-config / CMake package config / vcpkg
packaging.

## Rationale

C doesn't define any way to open "virtual" streams that write to memory rather than a real file. A lot of libc implementations roll their own nonstandard mechanisms to achieve this, namely `open_memstream`, or `fmemopen`. Other implementations provide more generic functions to call users functions for various operations on the file, like `funopen` or `fopencookie`. Finally, some implementations support none of these nonstandard functions.

fmem tries in sequence the following implementations:

* `open_memstream`.
* `fopencookie`, with growing dynamic buffer.
* `funopen`, with growing dynamic buffer.
* WinAPI temporary memory-backed file.

When no other mean is available, fmem falls back to `tmpfile()`.

## Building

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
ctest --test-dir build --output-on-failure   # requires Criterion
cmake --install build --prefix /usr/local
```

## Using fmem in your project

### pkg-config

```bash
gcc myapp.c $(pkg-config --cflags --libs fmem)
```

### CMake `find_package`

```cmake
find_package(fmem CONFIG REQUIRED)
target_link_libraries(myapp PRIVATE fmem::fmem)
```

### vcpkg

Add the port overlay from `ports/fmem/` (or use the manifest from `vcpkg.json`),
then in your vcpkg-consuming project:

```json
{
  "dependencies": ["fmem"]
}
```

## Releasing

Releases are driven by the [Release workflow](.github/workflows/release.yml).
Trigger it manually from the Actions tab with a bump type (`major`, `minor`,
`patch`, or an explicit `x.y.z`). The workflow opens a release PR; merging that
PR tags `v<x.y.z>` and publishes the GitHub release.

[criterion]: https://github.com/Snaipe/Criterion
