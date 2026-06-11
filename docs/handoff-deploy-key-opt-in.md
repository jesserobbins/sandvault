# Handoff: make sv-clone deploy keys opt-in with explicit write grant

## Background

- [Issue webcoyote/sandvault#129](https://github.com/webcoyote/sandvault/issues/129)
  proposed an **opt-in** `--deploy-key` flag for `sv-clone`. When engaged, the
  key was uploaded with write access so the sandbox could push its work back.
- [PR webcoyote/sandvault#139](https://github.com/webcoyote/sandvault/pull/139)
  (merged) implemented it, but dropped the flag: a deploy key is generated,
  uploaded with `--allow-write`, and the origin rewritten to SSH for **every**
  GitHub clone, automatically.
- [PR webcoyote/sandvault#170](https://github.com/webcoyote/sandvault/pull/170)
  (upstream, merged 2026-06-10, not yet in this fork) removed `--allow-write`
  from the upload: sandvault runs agents in YOLO/bypass-permissions mode, so
  write-by-default is unsafe. In that thread, SMillerDev argued the key
  "shouldn't upload to GitHub at all unless asked".

This change restores the original opt-in design and makes the write grant a
separate, explicit flag — converging the merged behavior, the Homebrew
feedback, and the original proposal.

## Target behavior

```
Usage: sv-clone [-v|-vv|-vvv] [options] <URL|PATH> [-- sv-args ...]

Options:
  -k, --repo-deploy-key    Create a deploy key for this repo and upload it to GitHub (read-only)
  -w, --allow-repo-write   Allow the deploy key to push to this repo (implies --repo-deploy-key)
  -v, --verbose            Enable verbose output
  -h, --help               Show this help message
```

| Invocation        | Key generated | Uploaded            | Origin rewritten to SSH |
| ----------------- | ------------- | ------------------- | ----------------------- |
| (no flag)         | no            | no                  | no                      |
| `-k`              | yes           | read-only           | yes                     |
| `-w`              | yes           | write (`--allow-write`) | yes                 |

Re-running `sv-clone` against an existing clone reconfigures it in place
(same spirit as `sv --rebuild`). In particular, re-running with `-w` upgrades
a previously read-only key — this is the recovery path for a rejected push,
so no re-clone is ever required.

## Implementation map (all in `sv-clone`)

1. **Arg parsing** — the `case` loop near the top (`--`, `-v|--verbose`,
   `-h|--help`, around line 67). Add `-k|--repo-deploy-key` and
   `-w|--allow-repo-write` setting `DEPLOY_KEY=true` / `DEPLOY_KEY_WRITE=true`
   (`-w` sets both). Update `show usage` and help text.
2. **Gate the deploy-key section** — the block under
   `# Deploy key — auto-generated for GitHub repositories` (around lines
   191–275) currently runs whenever the origin parses as a GitHub repo. Gate
   the whole block on `DEPLOY_KEY == true`. Update the section comment: it is
   no longer "auto-generated".
3. **Make the upload permission conditional** — the
   `gh repo deploy-key add` call (around line 258) currently passes
   `--allow-write` unconditionally. Pass it only when
   `DEPLOY_KEY_WRITE == true`, and say which level was granted in the `info`
   message, e.g. `Deploy key added to $GITHUB_REPO_PATH (read-only)` vs
   `(write access enabled)`.
4. **Only rewrite origin when the key is usable** — today the
   `remote set-url origin git@github.com:...` rewrite happens before the
   upload. If the upload fails (or `gh` is absent), a public repo that cloned
   fine over HTTPS can no longer fetch. Reorder so the rewrite (and the
   `core.sshCommand` config) happen only after the key is uploaded
   successfully or the user has been shown the public key for manual setup.
5. **Upgrade path (read-only → write)** — GitHub does not allow editing a
   deploy key's permission. When `-w` is given and a key with the same title
   already exists read-only, delete and re-add it:
   `gh repo deploy-key list -R "$GITHUB_REPO_PATH"` to find the id, then
   `gh repo deploy-key delete`, then `add --allow-write`. The local keypair
   in `$SV_PRIVATE_DIR/.ssh/` is reused, not regenerated.
6. **Capability summary** — at the end of the deploy-key section, print one
   line via the existing `info` helper (`helpers/sv-logging.sh`):
   `Sandbox access to org/repo: pull ✓  push ✗  (enable: sv-clone --allow-repo-write <url>)`.

## Out of scope

- Revocation tooling — keys are visible/deletable at the repo's
  Settings → Deploy keys page, which the manual-setup fallback already links.
- Any new script or subcommand (an earlier `sv-deploy-key` idea was rejected
  as sprawl).

## Tests / acceptance

`tests/tests` (bash) currently has **no** deploy-key or sv-clone-origin
coverage; PR #139 previously had to fix CI assertions broken by the
HTTPS→SSH rewrite. Add cases asserting:

1. Default clone of a GitHub HTTPS URL leaves the origin URL untouched and
   creates no key under `$SV_PRIVATE_DIR/.ssh/`.
2. `-k` generates the key, configures `core.sshCommand`, rewrites origin to
   SSH; the `gh` invocation (mock/stub) does **not** include `--allow-write`.
3. `-w` includes `--allow-write` and implies key generation without `-k`.
4. Re-running with `-w` over an existing `-k` clone re-adds the key with
   write access and does not regenerate the private key file.
5. With `gh` unavailable, `-k` prints the public key and does not rewrite
   the origin to SSH... unless we decide a printed key counts as "manual
   setup pending" — current code rewrites anyway; pick one and assert it.
   Recommendation: do not rewrite until the user confirms (simplest: never
   rewrite in the no-`gh` path; document it).

## Upstream follow-up

Once implemented and tested here, this is intended as a PR to
`webcoyote/sandvault` referencing #129/#139/#170. A draft comment proposing
the design (addressed to @webcoyote and @SMillerDev) already exists in the
session notes; the flag names `--repo-deploy-key` / `--allow-repo-write`
were chosen to make both the verb and the target explicit. Note the
trade-off discussed: `--allow-write` would match `gh repo deploy-key add`
verbatim; the explicit-target names were preferred deliberately.
