# Supervised worker dispatch over sandvault + tmux

You have a bounded task. You have a sandboxed worker. You want the worker to do the work while you do something else — and you want to pay almost nothing in your own context watching it happen.

This pattern makes that work. It composes primitives you already have: `sv`, `sv-clone`, `git clone --local`, and `tmux`. Nothing new in the sandvault binary. Just discipline about how you wire them together.

## When to use this

- The task is self-contained: a clear deliverable, not open-ended exploration.
- You can write a complete briefing. The worker will have no conversation context.
- The work is sandbox-appropriate: file writes, build steps, running untrusted code.
- You want the worker to spend *its* quota, not yours. `sv codex` from a Claude session (or vice versa) uses an independent quota pool. This matters more than you think.

## 1. Pick the worker

Default to the worker whose budget isn't yours:

| Supervisor | Default worker | Why |
| --- | --- | --- |
| Claude (weekly limit pool) | `sv codex` | Independent quota |
| Codex | `sv claude` | Independent quota |
| Either, ample budget | Either | Pick by capability |

**First-time codex auth:** a fresh sandbox isn't signed in. The first launch shows an OAuth URL in the tmux pane. Capture it once, hand it to whoever owns the browser, and stop. After sign-in, future dispatches are seamless.

## 2. Get the repo into the sandbox

The sandbox user only sees `/Users/Shared/sv-$USER/`. The repo must be inside the shared workspace.

**If the repo has a reachable `origin` remote:**

```bash
sv-clone <repo-url-or-path> -- <claude|codex> -- "<initial prompt>"
```

Clone, sandbox launch, and prompt in one call. Use this when you can.

**If the repo has no `origin`** (private working repo, never pushed):

```bash
git clone --local <host-repo-path> /Users/Shared/sv-$USER/repos/<name>
```

`--local` hardlinks the object database. Fast, cheap, preserves all branches and the current working-tree branch. Then launch against the cloned path:

```bash
sv <claude|codex> /Users/Shared/sv-$USER/repos/<name> -- "<initial prompt>"
```

After the worker commits, fetch back:

```bash
git -C <host-repo-path> fetch /Users/Shared/sv-$USER/repos/<name> <branch>
git -C <host-repo-path> merge --ff-only FETCH_HEAD   # or cherry-pick
```

**Do not skip this warning:** with a local clone, the sandbox repo's `origin` is your working repo on the host. A `git push` from inside the sandbox writes directly into your working tree. Your briefing must forbid `git push` and opening PRs. Say it explicitly. Workers follow instructions, not implications.

## 3. Write a self-contained briefing

The worker has zero context from your conversation. The briefing is the entire transmission.

Write it to:

```text
/Users/Shared/sv-$USER/handoff-<repo>.md
```

This path is reachable from inside the sandbox via the shared mount. The worker reads it as its first action.

A good briefing covers, in order:

1. **Heading + one-paragraph context.** What the project is and why this task matters. The worker has no project memory. If you wouldn't hand this to a new engineer on their first day and expect them to succeed, add more.
2. **Working tree and branch.** Exact sandbox path (`/Users/Shared/sv-$USER/repos/<name>`), branch name, and the rules: commit yes, push no, PR no.
3. **Setup.** Toolchain, build commands, what's gitignored and won't be in the clone (`node_modules/`, `.venv/`, build dirs). Tell the worker to install before touching anything.
4. **What to do.** The actual goal, in detail. Algorithms, exact field names, file paths. Write at the level of detail you'd give a capable junior engineer, not a vague request you'd give a senior one.
5. **Verification before commit.** Which `make`/`go test`/`pytest` target should pass. Whether to add unit tests.
6. **Constraints.** Style rules, commit-title format, attribution. By default the worker's commits attribute to the host user; add `Co-Authored-By` trailers only if explicitly wanted.
7. **What NOT to do.** Push, open a PR, change the schema, run destructive operations, write planning docs. Say it flat. Don't imply it.
8. **Issue-tracker requirement.** If the project uses a ledger (GitHub Issues, Linear, etc.), the worker identifies itself with a distinct actor name and posts start/decision/completion comments. It does not impersonate you.
9. **Reference reading.** Files to read first, in priority order.
10. **When stuck.** Make the conservative call. Note it as `Deferred:` in the commit body. Ship. Don't escalate mid-task.

The briefing is your product. A bad briefing produces bad work or a confused worker waiting for input that never comes.

## 4. Launch in a detached tmux session

```bash
tmux new-session -d -s sv-<task-name> -c $HOME
tmux send-keys -t sv-<task-name> \
  "sv <claude|codex> /Users/Shared/sv-$USER/repos/<repo> -- \
   'Read /Users/Shared/sv-$USER/handoff-<repo>.md and continue the task described there.'" Enter
```

Two-step — create the empty session, then `send-keys` — is more reliable on macOS than the one-shot `tmux new-session -d -s NAME COMMAND` form, which sometimes exits silently when the inline command finishes. Don't fight it, just use the two-step.

Tell whoever is watching the session name so they can `tmux attach -t sv-<name>` if they want to see what's happening.

**Re-dispatching to an already-running codex session:** the inline `Enter` in `send-keys` does NOT submit to codex's TUI. It gets buffered as text. Use a three-step sequence instead:

```bash
tmux send-keys -t sv-<name> "Read /Users/Shared/sv-$USER/handoff-<repo>.md and continue the task described there."
sleep 1
tmux send-keys -t sv-<name> Enter
sleep 5
tmux capture-pane -t sv-<name> -p | grep -E "Working \(|esc to interrupt"
```

If the grep returns nothing, codex is still idle at the input line. Send `Enter` again. The failure signature: prompt text on the input line, no "Working..." indicator, bottom of pane still shows the previous task's "Worked for X" banner. Don't trust the banner. Verify with the sentinel grep.

This is codex-specific. Claude Code's TUI handles the inline Enter correctly.

## 5. Supervise without burning your context

This is the part that matters most and the part people get wrong. The naive approach — `tmux capture-pane` every few seconds to watch progress — burns more of your context budget than the worker spends doing the actual work. Over a 20-minute task the difference is an order of magnitude.

The discipline: **silent waiting, then targeted reads.**

**Long sleep, then sentinel-grep:**

```bash
sleep 1200    # 20 minutes; adjust to the expected task duration
tmux capture-pane -t sv-<name> -p \
  | grep -E "shipped|FAILED|Press enter to continue" \
  | tail -5
```

A sentinel grep pulls one line. Capturing the whole pane pulls 30-50 lines of stale screen content. Grep for a signal, not a picture.

**Background wait loop:**

```bash
( until tmux capture-pane -t sv-<name> -p \
       | grep -qE "shipped|FAILED|Press enter to continue"; do
    sleep 30
  done ) &
```

One notification when the worker stops. You're not sitting in the loop.

**Targeted greps worth keeping handy:**

```bash
# Is the worker still active?
tmux capture-pane -t sv-<name> -p | grep -E "esc to interrupt" | tail -1

# Did a commit land?
tmux capture-pane -t sv-<name> -p | grep -E "^\[.*[a-f0-9]{7,}\]" | tail -3

# Is codex waiting for OAuth?
tmux capture-pane -t sv-<name> -p | grep -A2 "auth.openai.com" | tail -3
```

**What not to do:**

- `sleep 3 && tmux capture-pane -p | tail -40` repeated every cycle. This is the exact failure mode this pattern exists to prevent.
- Capturing the pane "to see how it's going" with no specific question in mind.
- Polling an interactive sign-in flow character by character. Capture the URL once, hand it off, and stop watching until you hear back.

## 6. Bring the result back

When a sentinel fires:

1. Read the worker's issue-tracker comment for the SHA and summary.
2. Fetch the commit into the host repo (see local-clone section above).
3. Report back: commit SHA, one-line summary, any deferred work the worker noted, issue number.
4. `tmux kill-session -t sv-<name>`.
5. Ask before deleting the sandbox clone. Leaving it makes follow-up dispatches faster.

Do not auto-merge. Do not auto-push. The human reviews and integrates.

## Things that go wrong

**`sv-clone` fails with "no 'origin' remote"** — the repo is local-only. Fall back to `git clone --local` and use `sv claude|codex <path>` directly.

**Sandbox path mismatch** — `sv claude /some/path` drops the agent into the sandbox home if that path isn't reachable from the sandbox user. Always use paths under `/Users/Shared/sv-$USER/`.

**codex needs sign-in on first run** — OAuth URL appears in the pane. Capture it once, hand it off, pause. Don't poll.

**Worker tries to push** — your briefing wasn't explicit enough. With a local clone the sandbox `origin` is your host working repo; a push corrupts your worktree. Say "do NOT git push, do NOT open a PR" in the briefing. Write it out. Don't rely on the worker inferring it.

**Worker can't reach the issue tracker** — include a fallback in the briefing: "if the issue tracker isn't reachable, leave a `HANDOFF.md` at the repo root with start/decision/completion notes."

## Quick reference

```bash
# 1. Clone into shared workspace (skip if using sv-clone)
git clone --local ~/src/myrepo /Users/Shared/sv-$USER/repos/myrepo

# 2. Write briefing to /Users/Shared/sv-$USER/handoff-myrepo.md

# 3. Launch
tmux new-session -d -s sv-myrepo -c $HOME
tmux send-keys -t sv-myrepo \
  "sv codex /Users/Shared/sv-$USER/repos/myrepo -- \
   'Read /Users/Shared/sv-$USER/handoff-myrepo.md and continue the task described there.'" Enter

# 4. Wait, then check for a sentinel
sleep 1200
tmux capture-pane -t sv-myrepo -p | grep -E "shipped|FAILED|Press enter" | tail -5

# 5. Fetch commit back
git -C ~/src/myrepo fetch /Users/Shared/sv-$USER/repos/myrepo <branch>
git -C ~/src/myrepo log --oneline FETCH_HEAD ^<branch> | head -5

# 6. Clean up
tmux kill-session -t sv-myrepo
```
