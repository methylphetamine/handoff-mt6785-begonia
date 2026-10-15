# handoff-mt6785-mainline-begonia

Handoff notes and working log for the **`mt6785-mainline` begonia kernel** work:
branches `7.1` and `6.16` of `methylamphetamine/mt6785-mainline`, AnyKernel3 flashable
packaging, and an Android 16 GKI build.

Companion to [`handoff-poxksun`](https://github.com/methylamphetamine/handoff-poxksun),
which covers the `POX-project/pox-begonia-kernel` kernel work. **Read
[`session-20261015-credential-exposure.md`](session-20261015-credential-exposure.md)
before doing anything else** — a GitHub PAT was exposed during the session that
triggered this work.

---

## Status

| track | state |
|---|---|
| begonia `7.1` build + CI | **green**, artifact `mt6785-begonia-7.1` |
| begonia `6.16` build + CI | **green**, artifact `mt6785-begonia-6.16` |
| AnyKernel3 packaging | **verified** on real artifacts |
| Android 16 GKI tree | local build running; CI re-run pending |

Verified in the built config:

```
CONFIG_ARCH_MEDIATEK=y
CONFIG_ANDROID_BINDER_IPC=y
CONFIG_ANDROID_BINDERFS=y
CONFIG_ANDROID_BINDER_DEVICES="binder,hwbinder,vndbinder"
```

---

## The short version

`methylamphetamine/mt6785-mainline` holds **upstream Linux mainline** trees —
7.1.0 on branch `7.1`, 6.16.4 on `6.16` — with MediaTek MT6785 support and Xiaomi
Redmi Note 8 Pro (**begonia**) device trees. They are **not** Android GKI trees: no
`gki_defconfig`, no `build.config.*`, no `tools/bazel`, no `android/` config layer.
Binder is the only Android facility present.

Device trees differ between the branches, which caused several real build failures:

| branch | DTBs |
|---|---|
| `7.1` | `begonia-csot`, `begonia-tianma` (two panels, genuinely different display ICs) |
| `6.16` | `begonia` (single, un-suffixed) |

Flashable zips follow `POX-project/pox-begonia-kernel`'s AnyKernel3 layout and device
config — notably **non-A/B**: `BLOCK=/dev/block/by-name/boot`, `IS_SLOT_DEVICE=0`.
Payload is `Image.gz-dtb` = `gzip(Image)` with the begonia DTBs appended raw after the
gzip stream, both panels included so the bootloader selects by `compatible`.

---

## Sessions

| file | topic |
|---|---|
| [`session-20261015-credential-exposure.md`](session-20261015-credential-exposure.md) | **Leaked PAT — revoke it** |
| [`session-20261014-begonia-builds-and-ci.md`](session-20261014-begonia-builds-and-ci.md) | begonia trees, build scripts, CI, AnyKernel3, 8 bugs |
| [`session-20261015-gki-tree.md`](session-20261015-gki-tree.md) | Android 16 GKI tree + 3 host-toolchain fixes |
| [`session-20261015-repo-state.md`](session-20261015-repo-state.md) | repo state + outstanding actions |

---

## Using the build scripts

```bash
export GH_TOKEN=...            # keep out of chat; use SSH or a credential helper

# begonia (locally, per branch)
./build-7.1.sh   -j "$(nproc)"
./build-6.16.sh  -j "$(nproc)"
./pack-anykernel3.sh 7.1 all     # all -> gzip(Image) + both panels

# Android 16 GKI
./build-gki.sh -j "$(nproc)" --targets Image --out out-gki
```

Scripts support `--deps` to install host packages, `-j N`, `--out DIR`, and `-h`.
`build-begonia.sh` takes a branch argument and defaults to building both.

See [`AGENTS.md`](AGENTS.md) for the continuation protocol, credential hygiene rules,
environment layout, and the `git add -f` requirement for `.github` (the trees'
`.gitignore` has a `.*` rule).