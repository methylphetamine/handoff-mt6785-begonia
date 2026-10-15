# Session 2026-10-15 — Android 16 GKI tree (aosp-mirror/kernel_common)

Attempting a full Android 16 GKI build alongside the begonia work. **Local build is
still running; CI was fixed but not yet confirmed green.**

---

## What was set up

`/content/gki-android16-6.12` — shallow clone (`--depth 1 --single-branch`) of
`aosp-mirror/kernel_common`, branch `android16-6.12`, 1.9 GB, head `105b5745f`.

- Linux **6.12.52**
- `arch/arm64/configs/gki_defconfig` present
- fragments present: `amlogic_gki`, `db845c_gki`, `fips140_gki`, `rockpi4_gki`, `tv_gki`, `allmodconfig`
- **no `gki.fragment`** — that is device/board-specific and normally supplied per
  board, so plain `gki_defconfig` is the correct generic GKI build
- **no `tools/bazel`, no `build/config`** — the GitHub mirror omits Kleaf. Building
  via Bazel is not possible from this mirror; the classic `make gki_defconfig` path
  is used instead.

`build-gki.sh` + `.github/workflows/build-gki.yml`, on a new **`gki`** branch of
`methylamphetamine/mt6785-mainline` (commit `5753436e3`). The workflow clones
`kernel_common@android16-6.12` itself rather than the branch carrying kernel source.

> Deliberately **not** pushed to `aosp-mirror/kernel_common` — that is Google's public
> mirror. Read-only.

---

## Scope reality-check

GKI is architecturally the opposite of begonia. GKI is device-agnostic and boots on any
arm64 device, pushing device specifics into vendor modules. A begonia MT6785 kernel with
device DTBs is a **vendor** kernel. There is no such thing as a "GKI tree for begonia".

What this branch actually produces is a **stock Android 16 GKI kernel** (Linux 6.12,
no MT6785 support, no begonia DTB). It does not replace or replaceable-with the begonia
build. The two tracks are independent. If the goal is a working Android 16 begonia
kernel, the mainline begonia tree on `7.1`/`6.16` is the relevant track and the
GKI config work is infrastructure for it.

---

## Host-toolchain fixes needed to get it building

Three environment gaps, none of them code defects. All are now handled by the script.

### 1. `gendwarfksyms` needs libdwarf

```
scripts/gendwarfksyms/gendwarfksyms.h:6:10: fatal error: 'dwarf.h' file not found
CONFIG_GENDWARFKSYMS=y   (set by gki_defconfig)
```

Install `libdwarf-dev` + `libdw-dev`. Debian ships the header at
`/usr/include/libdwarf/dwarf.h` but the source includes `<dwarf.h>`, so the script adds
the include via `HOST_EXTRACFLAGS` — the kernel's supported hook for host-tool paths:

```bash
[ -f /usr/include/libdwarf/dwarf.h ] && [ ! -f /usr/include/dwarf.h ] \
  && EXTRAC="-I/usr/include/libdwarf"
```

### 2. LLVM binutils are version-only — use `LLVM=-18`, not `LLVM=1`

```
/bin/sh: 1: llvm-ar: not found
make[3]: *** [.../vdso.so] Error 127
```

Ubuntu ships `llvm-ar-18`, `llvm-objcopy-18`, … with **no unversioned alias**. Passing
`LLVM=1` makes the Makefile derive `llvm-ar`, which does not exist.

The kernel already supports the suffix form (`Makefile:441-446`, and
`tools/scripts/Makefile.include`):

```make
ifneq ($(filter -%,$(LLVM)),)
LLVM_SUFFIX := $(LLVM)
```

So `LLVM=-18` correctly yields `llvm-ar-18`, `clang-18`, `ld.lld-18` for **both** target
and `HOST*` tools. Enumerating `AR=`/`NM=`/`OBJCOPY=` by hand does **not** fix it —
`lib/vdso/Makefile` uses the target-level `$(READELF)`/`$(OBJCOPY)` while
`tools/bpf/resolve_btfids` uses `HOSTAR`, and they resolve independently. Verified:

```
LLVM=1   ->  READELF=llvm-readelf   HOSTAR=llvm-ar
LLVM=-18 ->  READELF=llvm-readelf-18  HOSTAR=llvm-ar-18
```

Also note `llvm-link`/`llvm-as` are referenced by the Makefile and come from the same
package.

### 3. A stray build starved it

The very first GKI attempt sat at ~40 objects for 13 minutes, which looked like a
toolchain fault. It was not: the **2-hour-old local begonia `make` from earlier in the
session was still running and holding both cores**. Killed it; throughput recovered.
With 2 cores, confirm nothing else is building before starting:

```bash
ps -eo pid,etimes,cmd | grep -E "make|clang" | grep -v grep
```

---

## Current state

| item | state |
|---|---|
| local GKI build | **still running**, ~213 objects, 0 errors |
| `/content/out-gki/Image` | not produced yet |
| `build-gki` CI | first run failed (`no kernel tree at …/gki-android16-6.12`) |
| fix | `TREE=gki-src` in the build step, commit `5753436e3` — **CI re-run not yet confirmed** |

---

## Next steps

1. Wait for the local build to finish and confirm `out-gki/Image` exists and carries
   ARM64 magic `0x644d5241` at offset `0x38`.
2. Confirm the `build-gki` CI run goes green after `5753436e3`.
3. If a full GKI is wanted, add `--targets "Image modules"` — the current run builds
   `Image` only. GKI modules are the vendor-facing half of a GKI release.
4. ccache is enabled and cached per branch, so re-runs should be fast.
5. `build-gki.sh` warns at `-j>=5`: this VM has 12 GB and 2 cores, and GKI is
   LTO/RAM-sensitive.