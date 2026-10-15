# Session 2026-10-14/15 — mt6785-mainline begonia builds, branches 7.1 and 6.16

Cloned both branches of `methylamphetamine/mt6785-mainline`, added build scripts, set
up per-branch GitHub Actions, and packaged AnyKernel3 flashable zips. **Both branches
build and CI is green.**

---

## What the trees actually are

Plain **upstream Linux mainline**, not Android GKI trees:

| branch | `Makefile` VERSION/PATCHLEVEL/SUBLEVEL | head commit |
|---|---|---|
| `7.1` | 7.1.0 | `811c3c80d` |
| `6.16` | 6.16.4 | `203a993f8` |

Shallow clones (`--depth 1 --single-branch`) into `/content/mt6785-<branch>`, 2.0 GB
each.

### begonia device support

`Xiaomi Redmi Note 8 Pro (begonia)`, MediaTek MT6785. The two branches differ and this
caused several real bugs (see §Bugs):

| branch | device trees |
|---|---|
| `7.1` | `mt6785-xiaomi-begonia-csot.dts`, `mt6785-xiaomi-begonia-tianma.dts` — **two panels** (NovaTek NT36672A vs Tianma) |
| `6.16` | `mt6785-xiaomi-begonia.dts` — one, un-suffixed |

`7.1` needs both DTBs because the display panel IC genuinely differs. They are not
duplicate files. Any glob written as `begonia-*.dtb` silently misses `6.16`.

### Android support is minimal

```
arch/arm64/configs/   defconfig, hardening.config, virt.config   (no gki_defconfig)
build.config.*       none
tools/bazel          absent
```

Verified present in `drivers/android/Kconfig`:

- `CONFIG_ANDROID_BINDER_IPC` — **`bool`, not tristate**
- `CONFIG_ANDROID_BINDERFS`

Verified **absent** (dropped from mainline; cannot be enabled):

- `ASHMEM` — removed upstream; modern Android uses `memfd`
- `CONFIG_ANDROID_PARANOID_NETWORK`
- any `android/` config layer

So there is no Android userspace glue, init rc, or GKI split here. Binder is the only
Android facility present. Expect init/rc work before anything boots on Android 16.

---

## What was added to each branch

```
build-begonia.sh        core build: arm64 defconfig + Android fragment
build-<branch>.sh       thin branch-pinned wrapper
pack-anykernel3.sh      AnyKernel3 flashable zip packaging
anykernel3/             vendored assets (see below)
.github/workflows/build-<branch>.yml
```

Per-branch wrappers exist because a single script handling both branches hid a real
argument-parsing bug (§Bugs #3).

### Verified build config

```
CONFIG_ARCH_MEDIATEK=y
CONFIG_ANDROID_BINDER_IPC=y
CONFIG_ANDROID_BINDERFS=y
CONFIG_ANDROID_BINDER_DEVICES="binder,hwbinder,vndbinder"
```

---

## AnyKernel3 packaging

Adopted the layout, tool set and device config from **`POX-project/pox-begonia-kernel`**
(the project the user builds on) rather than stock osm0sis AnyKernel3, as requested.

Their release zip (`Pox-1.0.0-Beta-Onyx-3888146-begonia.zip`, release `v1.0.0-Beta`)
was downloaded and reverse-engineered for two facts that matter:

**1. begonia is non-A/B.** Their `anykernel.sh`:

```
BLOCK=/dev/block/by-name/boot;
IS_SLOT_DEVICE=0;      # banner says "(A-only)"
```

I had initially guessed `IS_SLOT_DEVICE=1` with
`BLOCK=/dev/block/bootdevice/by-name/boot`. **That was wrong** — on a non-A/B device
the slot-suffixed path does not exist. Corrected.

**2. `Image.gz-dtb` means `gzip(Image)` followed by raw DTBs appended *after* the gzip
stream.** Verified by decompressing their payload:

```
total 21370368 | gzip stream 20926110 | inner ARM64 magic 0x644d5241 OK
trailing 444258 bytes = 2 DTBs (222129 each)  -> both panels in one payload
```

So the packer emits `gzip(Image) || dtb1 || dtb2`. Shipping both panels and letting the
bootloader select by `compatible` is better than forcing one.

`ak3-core.sh` and `update-binary` from their zip are **byte-identical** to osm0sis
(AK_BASE_VERSION=20260904); they additionally ship 5 tools that stock does not —
`snapshotupdater_static`, `httools_static`, `lptools_static`, `magiskpolicy`, `fec`.
All vendored.

Their zip entry set differs from ours only by `LICENSE`, `README.md`, `CHANGELOG.txt`
— their project docs, deliberately **not** copied.

### Verified packaged output

```
META-INF/com/google/android/{update-binary, updater-script}
tools/  (8 binaries incl. ak3-core.sh)
anykernel.sh                      <- at zip root, as Pox does
Image.gz-dtb                      <- gzip(Image) + 2 begonia DTBs
ramdisk/  modules/system/lib/modules/  patch/
```

Verified on the real CI artifact: ARM64 magic `0x644d5241` OK, 2 DTBs parsed and
trailing bytes fully consumed, `BLOCK=/dev/block/by-name/boot`, `IS_SLOT_DEVICE=0`.

---

## CI

One workflow per branch, each triggered only by its own branch push:

- `build-7.1.yml` → `refs/heads/7.1`, calls `./build-7.1.sh`
- `build-6.16.yml` → `refs/heads/6.16`, calls `./build-6.16.sh`

Both **green**. Artifacts: `mt6785-begonia-7.1` (52.0 MB), `mt6785-begonia-6.16`
(48.6 MB).

### Why builds take ~3 minutes

ccache is enabled (`CROSS_COMPILE` prefixed with `ccache`, hit rate reported each run):

```
Cacheable calls: 23435
Hits:           18830 / 23435 (80.35%)
```

Cold build ≈ 25 min; warm ≈ 3–4 min. Confirmed *correct*, not stale — diffing the
cold-build `Image` against a cached one gave **57 differing bytes out of 52,644,352**
(0.0001%), identical size, and the only deltas were the runner hostname and the
`#1 SMP PREEMPT <timestamp>` string. DTBs were byte-identical.

---

## Bugs found and fixed during the session

1. **`ANDROID_BINDER_IPC` is `bool`, not tristate.** `--module` produced
   `symbol value 'm' invalid for ANDROID_BINDER_IPC`.
2. **Single-tree layout unsupported.** The script only resolved `$ROOT/mt6785-$branch`,
   so on a runner where the checkout *is* the tree it searched a nonexistent
   `/content`. Added `resolve_tree()` plus a version guard (asking for `6.16` from a
   7.1 tree now errors instead of building the wrong thing).
3. **Positional branch args appended instead of replacing.** `BRANCHES=("7.1" "6.16")`
   plus `BRANCHES+=("$1")` meant CI asking for `7.1` also built `6.16`. Defaults are
   now applied only when no branch was passed.
4. **DTB glob assumed the 7.1 layout.** `begonia-*.dtb` misses 6.16's
   `begonia.dtb` — failed CI twice ("built no begonia DTBs").
5. **`PANEL=csot` pinned in the 6.16 workflow**, where no such DTB exists.
6. **Cache-save race marked jobs failed.** Two jobs saving `ccache-<branch>-<sha>`
   → `Unable to reserve cache` → non-zero job. Made `continue-on-error: true`.
7. **`actions/cache/save` + `hendrikmuhs/ccache-action` conflicted**; replaced with
   explicit `actions/cache` restore/save.
8. **GKI workflow passed `TREE` implicitly.** Cloned into `gki-src/` but
   `build-gki.sh` defaults to `gki-android16-6.12` → `no kernel tree at …/gki-android16-6.12`.
   Fixed by passing `TREE=gki-src` (commit `5753436e3`).

actionlint caught #6-class and expression defects that YAML parsing missed — notably
`${{ PANEL }}`, which is invalid (job-level `env` is already a shell variable; it must
be `"$PANEL"`).

---

## A note on "Repository not found"

Early in the session `methylamphetamine/mt6785-mainline` returned `404` on **every**
access path — REST, `git ls-remote`, and `codeload` — while `POX-project/pox-begonia-kernel`
cloned fine. The repo did not exist under the *misspelled* login either, so the
documented spelling theory in `AGENTS.md` does not explain it. It later became
reachable and stayed reachable. Genuinely intermittent; retries succeeded. Do not
assume the first `404` is a spelling problem — re-check the spelling, then retry.