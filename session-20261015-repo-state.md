# Session 2026-10-15 — repository state and outstanding actions

Snapshot of all repos touched this session, and what still needs a human decision.

---

## Repos

| repo | branch | head | CI |
|---|---|---|---|
| `methylamphetamine/mt6785-mainline` | `7.1` | `f0e3b425` | green |
| `methylamphetamine/mt6785-mainline` | `6.16` | `a999a6c5` | green |
| `methylamphetamine/mt6785-mainline` | `gki` | `ef4a87bf` | failing at vmlinux BTF link (unresolved) |
| `methylamphetamine/mt6785-mainline` | `battery-downstream` | `fad73078` | untouched |
| `methylamphetamine/handoff-mt6785-mainline-begonia` | `main` | this repo | n/a |
| `aosp-mirror/kernel_common` | `android16-6.12` | `105b5745f` | read-only, never pushed |
| `POX-project/pox-begonia-kernel` | `v1.0.0-Beta` | release assets | read-only, never pushed |

---

## Outstanding actions, in priority order

### 1. Revoke the leaked PAT — blocking, security

Full detail in `session-20261015-credential-exposure.md`.
`ghp_mvbkt…N23E [REDACTED — full value deliberately not stored]`, full admin scope, pasted in plaintext.

### 2. Decide on commit authorship (needs a human)

`AGENTS.md` in `handoff-poxksun` requires commits be authored as

```
methylamphetamine <rootrequired@proton.me>
```

but everything pushed on 2026-10-14/15 was authored as `opencode <opencode@localhost>`,
written before that convention was read. Only the final GKI workflow fix
(`5753436e3`) used the correct identity.

Correcting this means `git rebase` + `--force-with-lease` on `7.1` and `6.16`, which
invalidates SHAs already referenced in the session notes above. Left undone deliberately
— force-pushing shared branches is not an agent's call.

### 3. Confirm `begonia` panel choice

7.1 ships two DTBs and the packer includes **both** in `Image.gz-dtb`, letting the
bootloader select by `compatible`. That matches what `POX-project` does and should be
correct as-is. If you would rather ship a single panel:

```bash
./pack-anykernel3.sh 7.1 csot     # or tianma
```

Flashing the wrong panel gives a black screen. Worth confirming on-device.

### 4. `build-gki` CI still fails at the vmlinux BTF link

Fixed so far: `TREE` mismatch, and a real relative-`O=` path bug
(`ef4a87bf`) that had been silently skipping the config fragment under CI.
The remaining `FAILED: load BTF from vmlinux: Invalid argument` is **not solved**;
DWARF5 and llvm-objcopy were both tested and disproven. See the GKI session file for
what is established and the remaining pahole lead.

### 5. Decide whether GKI work continues

See `session-20261015-gki-tree.md` §Scope reality-check. As it stands the GKI branch
produces a stock Android 16 kernel with no begonia support, which does not by itself
advance a bootable begonia device.

---

## Things that are known-incomplete

- **No Android userspace glue on the begonia trees.** Binder is the only Android
  facility present; `ASHMEM` and `ANDROID_PARANOID_NETWORK` do not exist in mainline.
  Nothing here will boot Android 16 without init/rc work.
- **No `gki.fragment`** on either track — board-specific, must be written per device.
- **`build-begonia.sh` builds `Image` + `dtbs` only**, no modules. If vendor modules are
  needed, add `modules` to the target list.
- **Local GKI build had not finished** at the time of writing.

---

## Working notes that will save time later

- `.gitignore` in these trees contains `.*`, which ignores `.github` —
  `git add -f` for any workflow file.
- `/tmp/opencode/actionlint` must be run on workflow changes; it caught defects that
  YAML parsing did not.
- `/tmp/opencode/gh-env.sh` resolves `GH_USERNAME` from the token;
  `/tmp/opencode/gh-cred.sh` is the credential helper that keeps the token out of
  `.git/config`.
- 2 cores / 12 GB. Kill stray `make` processes before starting a build.
- Build outputs: `/content/out/<branch>/` (begonia), `/content/out-gki/` (GKI).