# Session 2026-10-15 — credential exposure (read this first)

**Action required: revoke a leaked GitHub PAT.**

---

## Summary

A classic GitHub personal access token was pasted in plaintext into the working
session while setting up the begonia kernel builds:

```
ghp_mvbkt…N23E [REDACTED — full value deliberately not stored]
```

Token scopes, read from the live `x-oauth-scopes` response header:

```
repo, workflow, admin:org, admin:repo_hook, admin:enterprise, audit_log,
codespace, copilot, delete:packages, delete_repo, gist, notifications, project,
user, write:discussion, write:network_configurations, write:packages
```

That is effectively **full administrative access** to every repository the token's
owner can reach. It authenticates as `methylamphetamine` (user id `328056449`).

---

## Why it is a real risk

- Classic PATs (`ghp_…`) are **long-lived with no automatic expiry**. Unlike a
  GitHub App or fine-grained PAT there is no scope narrowing that meaningfully
  reduces the blast radius here — `repo` plus `delete_repo` plus `admin:org` is
  about as broad as it gets.
- It was transmitted in chat. Treat the transcript as compromised; anything able to
  read it has the token.
- The token is also **write-capable**: it was used to push branches to
  `methylamphetamine/mt6785-mainline`. Anyone holding it can push to any repo that
  account can write to.

---

## What was done about it during the session

| action | status |
|---|---|
| token sent to `api.github.com` and `github.com` only | done — never to any third party |
| token written to disk by me | no — deliberately kept in env only |
| token embedded in `.git/config` (push remote) | **happened**, then scrubbed |
| token left in shell history / process args | yes, transiently |
| Google Cloud project contacted | no — nothing was done with the pasted GCP id |

The `.git/config` leak was found late and fixed: all three trees had the token in
`origin` as a *push* URL. Remotes were reset to token-free URLs and a credential
helper (`/tmp/opencode/gh-cred.sh`) now supplies the token from the environment.

Verification: `grep -c ghp_ */.git/config` → `0`.

Scrubbing the configs does **not** undo the exposure. The token is still in the
session transcript.

---

## Required remediation

1. **Revoke now** — <https://github.com/settings/tokens>, or
   `DELETE /applications/{client_id}/token` via the API.
2. Issue a replacement scoped to the single repository and nothing more. For GKI and
   for `aosp-mirror` read access, prefer an **SSH deploy key** or a read-only
   fine-grained PAT.
3. Check for abuse while the token was live:
   - <https://github.com/settings/security-log> — review recent events
   - repository **Actions** run history on `mt6785-mainline` for runs you did not start
   - `gh api /user/repos` for repos created/modified unexpectedly
4. Never paste a credential into chat again. Use an env var, `gh auth login`, a
   credential helper, or SSH.

---

## The unrelated thing in that first message

The original request also included a Google Cloud project id alongside the token.
Nothing in a kernel build needs GCP, and **nothing in Google Cloud was touched**. If a
GCP project was genuinely meant to be involved, say so explicitly before any
infrastructure work starts.