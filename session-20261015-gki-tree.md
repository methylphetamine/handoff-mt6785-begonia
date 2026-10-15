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
| local GKI build | still running, ~680 objects, 0 compile errors |
| `/content/out-gki/Image` | not produced yet |
| `build-gki` CI | `TREE` fix confirmed — now fails much later, at the **vmlinux link** |

### CI failure: BTF at the vmlinux link — UNRESOLVED (two hypotheses disproven)

```
BTFIDS  vmlinux
FAILED: load BTF from vmlinux: Invalid argument
make[3]: *** [.../scripts/Makefile.vmlinux:45: vmlinux] Error 255
```

**What is established fact:**

1. `scripts/link-vmlinux.sh:283` runs `${RESOLVE_BTFIDS} vmlinux`.
2. -> `btf__parse_split()` -> `btf_parse_elf()` -> `btf_new()` -> `btf_parse_hdr()`.
3. `tools/lib/bpf/btf.c:228`: `if (btf->raw_size < sizeof(struct btf_header))
   return -EINVAL;`. `sizeof(struct btf_header)` is **24 bytes**. So the `.BTF`
   section in the final `vmlinux` is **present but under 24 bytes** — i.e. empty.
   (`btf_find_elf_sections` would have returned `-ENODATA` had the section been
   missing entirely, so it exists but is empty.)
4. `gen_btf()` at `link-vmlinux.sh:119` runs
   `LLVM_OBJCOPY="${OBJCOPY}" ${PAHOLE} -J ${PAHOLE_FLAGS} ${1}` with
   **no exit-status check**, and the following `${OBJCOPY} --only-section=.BTF ...`
   has `2>/dev/null`. Both can fail silently; only the final EINVAL surfaces.
5. The only thing pahole prints is
   `Reached the limit of per-CPU variables: 4096` (that string is **not** in the
   kernel source — it comes from pahole).

**Hypothesis A — DWARF5 vs pahole 1.25 — DISPROVEN.**
`gki_defconfig` selects `CONFIG_DEBUG_INFO_DWARF5=y`, and pahole 1.25's DWARF5
support is incomplete, so this looked like the cause. Forced
`CONFIG_DEBUG_INFO_DWARF4=y` (commit `4d64ed27`); CI confirmed
`CONFIG_DEBUG_INFO_DWARF4=y` was applied and the failure was **byte-for-byte
identical**. DWARF5 is not the cause. The override is kept only because DWARF4 is
the conventional pahole-1.x pairing — it is **not** a demonstrated fix.

**Hypothesis B — `llvm-objcopy` instead of GNU objcopy — DISPROVEN.**
Under `LLVM=1`, `${OBJCOPY}` is `llvm-objcopy`, which is used to build
`.tmp_vmlinux1.btf.o`; a failure there is swallowed by `2>/dev/null`. Tested
directly on a synthetic arm64 ELF carrying a 64-byte `.BTF`: both
`aarch64-linux-gnu-objcopy` and `llvm-objcopy-18` preserve the section exactly
(`000040` bytes, rc=0). The objcopy step is not inherently broken.

**Remaining lead: pahole itself is failing.** The per-CPU-variable limit of 4096 is
the only diagnostic pahole emits, and it is the one thing consistent with an
empty `.BTF`. GKI's config has an unusually large per-CPU variable count.
`scripts/Makefile.btf` also has an explicit branch for **pahole >= 1.26**
(`--btf_features=encode_force,var,float,enum64,...`) that pahole 1.25 never
reaches, which suggests the tree targets a newer pahole than 1.25.
Ubuntu noble only ships `dwarves` 1.25, and `github.com/matrosov/pahole` was not
reachable from this environment, so a newer pahole could not be built or fetched.

**How to confirm (needs one full build, ~32 min on CI):** keep the build alive past
the link by running `resolve_btfids` manually, or patch `gen_btf()` to check
`${PAHOLE}`'s exit status — that alone turns a 4-minute silent failure into an
immediate, explicit error.

**Fallback if it cannot be resolved:** `scripts/config --disable DEBUG_INFO_BTF`
makes the build complete and produces a working GKI `Image`, but without BTF the
kernel cannot serve eBPF/CO-RE — which is a large part of what GKI exists for. That
is a real trade-off, not a clean fix.

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