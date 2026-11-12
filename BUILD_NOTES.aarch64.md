# CTFFIND5 (cisTEM `ctffind5` branch) — Linux aarch64

Upstream: https://github.com/timothygrant80/cisTEM/tree/ctffind5 (CTFFIND 5.0.0)

## Result: 🟢 PASS — 2 patches required

The table's earlier CTFFIND-4.1 failure ("x86 assembly macros") does not recur:
the only genuine x86 dependency in the CTFFIND5 tree is an MKL/CPUID helper that
is dead code when FFTW is used.

## Do not use `ctffind_standalone/`

That subtree is stale (CTFFIND 4.x era). Its `configure.ac` never emits
`cistem_config.h`, but the shared `src/core/core_headers.h` now `#include`s it,
so every object fails with `fatal error: cistem_config.h: No such file or
directory`. **Build from the repository root instead** — the top-level
`configure` generates `cistem_config.in` via `autoheader` and `ctffind` is one of
its `bin_PROGRAMS`.

## Dependencies (no root required)

`libwxgtk3.2-dev` is not installed and there is no passwordless sudo on this
host, so wxWidgets and FFTW come from conda-forge:

```bash
mamba create -y -p ./env -c conda-forge 'wxwidgets=3.2.*' fftw \
      gxx_linux-aarch64 make automake autoconf libtool pkg-config
```

Resolved: wxWidgets 3.2.9, FFTW3 (system `libfftw3-dev` also works), libtiff from
the system.

## Patch 1 — `src/core/functions.cpp`: x86-only CPUID / MKL block

`ActivateMKLDebugForNonIntelCPU()` and its helpers use `<cpuid.h>`, raw `cpuid`
and `xgetbv` inline assembly, and `<immintrin.h>`. None of that exists on
aarch64, and the function body is already `#ifdef MKL`-guarded (MKL is x86-only
anyway). Wrap the whole section:

```c
#if defined(__x86_64__) || defined(__i386__)
#if defined(__INTEL_COMPILER) && (__INTEL_COMPILER >= 1300)
...                                  // unchanged CPUID machinery
#include <cpuid.h>
#endif /* x86 only */

void ActivateMKLDebugForNonIntelCPU( )
{
#if defined(MKL) && (defined(__x86_64__) || defined(__i386__))
...
```

## Patch 2 — `src/core/cistem_parameters.cpp`: wxWidgets 3.1.6+ API

Not aarch64-specific, but it blocks any build against a modern wxWidgets.
`wxStringSortAscending` changed from `wxString*` to `const wxString&`:

```c
#if wxCHECK_VERSION(3, 1, 6)
    return wxStringSortAscending((*a)->reference_3d_filename, (*b)->reference_3d_filename);
#else
    return wxStringSortAscending(&(*a)->reference_3d_filename, &(*b)->reference_3d_filename);
#endif
```

## Build

```bash
E=$PWD/env
export PATH="$E/bin:$PATH" CPPFLAGS="-I$E/include" LDFLAGS="-L$E/lib" \
       PKG_CONFIG_PATH="$E/lib/pkgconfig"
autoreconf -fi                      # generates cistem_config.in
mkdir build_top && cd build_top
../configure --with-wx-config="$E/bin/wx-config"
cd src && make -j"$(nproc)" ctffind
```

Notes:
- `AX_CUDA: command not found` during configure is harmless — CTFFIND has no
  CUDA path; only the GPU-accelerated cisTEM programs use it.
- `fatal: Invalid revision range ...` is the shallow clone confusing the version
  stamp; also harmless.
- The binary needs conda's wx at run time:
  `export LD_LIBRARY_PATH=$E/lib:$LD_LIBRARY_PATH`.

Result: `build_top/src/ctffind`, `ELF 64-bit LSB pie executable, ARM aarch64`,
reporting `Version : 5.0.0 / From Branch : ctffind5`.

## Smoke test — defocus recovery

1024x1024 micrograph at 1.0 A/px: white noise multiplied in Fourier space by an
analytic CTF (300 kV, Cs 2.7 mm, amplitude contrast 0.07, **defocus 20000 A**).

```bash
printf 'mic.mrc\ndiag.mrc\n1.0\n300\n2.7\n0.07\n512\n30\n5\n5000\n50000\n500\nno\nno\nno\nno\nno\nno\nno\n' | ctffind
```

| quantity | value |
|---|---|
| defocus 1 / defocus 2 | **20001.76 / 19998.81 A** (ground truth 20000) |
| relative error | < 0.01 % |
| astigmatism azimuth | 7.66 deg (input was astigmatism-free) |
| score | 0.694 |
| Thon rings fitted to | 2.0 A |
| total run time | 0.22 s |

CTFFIND5 runs in "Scripted" mode when fed on stdin and requires **19** answers
(one more than CTFFIND4: sample tilt and sample thickness questions were added).
A missing final answer aborts with `Error: Blank answer in scripted mode`.
