# AGENTS.md — continuation protocol for this handoff repo

This repo is the **working log and handoff point** for the `mt6785-mainline` begonia
kernel task (branches `7.1` and `6.16` of `methylamphetamine/mt6785-mainline`, plus
the Android 16 GKI build).

Follow these instructions every time you work on this task.

---

## 1. Always commit as `methylamphetamine`

> ⚠️ Spelling: the login is **`methylamphetamine`** (15 chars — methyl + phetamine,
> user id `328056449`). Writing it as `methylamphetamine` (with an extra "a") breaks
> `git clone`/`push`, REST-by-name and GraphQL against this repo with
> *Repository not found*, which reads like a network or credential problem but is not
> one. If you hit that error, check the spelling before touching auth.

```
git -c user.name="methylamphetamine" -c user.email="rootrequired@proton.me" commit ...
```

`rootrequired@proton.me` is the verified primary email on the `methylamphetamine`
GitHub account (user id `328056449`), so commits attribute correctly.

**Do not** set this in `--global` config; keep it repo-scoped.

> **Known deviation in the kernel repo:** the commits pushed on 2026-10-14/15 for
> `build-*.sh`, `pack-anykernel3.sh` and the workflows are authored as
> `opencode <opencode@localhost>`, not `methylamphetamine`. They were written before
> this convention was read. Re-authoring them means a force-push to `7.1` and `6.16`;
> left for a human decision. See `session-20261015-repo-state.md`.

---

## 2. Credential hygiene — do not put the token in a remote URL

Cloning with the token inline (`https://x-access-token:$TOKEN@github.com/...`) writes
it into `.git/config` as a **push** remote. That happened here on 2026-10-14 for all
three trees and has been scrubbed.

Use the credential helper instead — it serves the token from the environment and never
touches disk:

```bash
# /tmp/opencode/gh-cred.sh  (mode 0755)
git config --unset-all credential.helper
git config --add    credential.helper '!/tmp/opencode/gh-cred.sh'
git remote set-url origin https://github.com/<owner>/<repo>.git   # no token
git ls-remote --heads origin     # authenticated by the helper
```

Verify with `grep -c ghp_ .git/config` → must be `0`.

> ⚠️ A classic PAT (`ghp_…`) was pasted in plaintext into the working session and
> **must be revoked**. It carried `repo, workflow, admin:org, delete_repo, audit_log`.
> Rotating it is the single highest-priority outstanding action. See
> `session-20261015-credential-exposure.md`.

---

## 3. Update this repo after every work session

When you finish a unit of work, leave the next agent a clear state. Add a file named
`session-YYYYMMDD-<short-topic>.md` at the repo root containing:

```markdown
# Session <date> — <what you worked on>

## What I did
- Concrete changes, with commit SHAs and file paths.
- Commands that were run and their outcomes.

## Outcome
- What works now, verified how.

## Open items / next steps
- What is unfinished or broken, with the exact error text.
```

---

## 4. Layout of the working environment

| path | what |
|---|---|
| `/content/mt6785-7.1` | kernel tree, branch `7.1` (Linux **7.1.0**) |
| `/content/mt6785-6.16` | kernel tree, branch `6.16` (Linux **6.16.4**) |
| `/content/gki-android16-6.12` | GKI tree, `aosp-mirror/kernel_common` @ `android16-6.12` (Linux **6.12.52**) |
| `/content/out/<branch>/` | begonia build output (`Image`, DTBs, AnyKernel3 zip) |
| `/content/out-gki/` | GKI build output |
| `/tmp/opencode/gh-env.sh` | `GH_USERNAME` + helpers (source it; needs `GH_TOKEN`) |
| `/tmp/opencode/gh-cred.sh` | git credential helper |
| `/tmp/opencode/actionlint` | workflow linter — **run it on every workflow change** |

The sandbox has **2 cores and 12 GB RAM**. Kill stray builds before starting another;
a forgotten 2-hour `make` starved a later build to ~40 objects and looked like a
toolchain fault.

---

## 5. Lint workflows before pushing

`.gitignore` in these trees has a `.*` rule that ignores `.github`, so workflows need
`git add -f`. Always validate first — actionlint caught two real defects that YAML
parsing alone missed:

```bash
/tmp/opencode/actionlint -shellcheck= -pyflakes= <tree>/.github/workflows/*.yml
```