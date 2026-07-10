# Design: Add pi as a first-class agent in sandvault

**Date:** 2026-07-10
**Branch:** `feature/pi-support`

## Summary

Add the `pi` coding agent (`@earendil-works/pi-coding-agent`) as a first-class
agent alongside claude, codex, opencode, and gemini. The addition follows the
existing wrapper + `sv` plumbing pattern exactly; no new mechanisms are
introduced.

## Background: how an agent is wired today

Each supported agent is wired across four locations:

1. **`guest/home/bin/<agent>`** — a bash wrapper that:
   - sources `sv-tool-prompts.sh`
   - resolves the agent binary (native install under `~/.local` → Homebrew → PATH)
   - errors if not found
   - injects the shared tool prompt via the agent's own prompt-injection flag
   - execs the agent with a sandbox-bypass / autonomy flag
2. **`sv` help text** — a line `  <alias>, <name> [PATH]    Open <Name> in sandvault`
3. **`sv` dispatch** — a `case` arm `<alias>|<name>) COMMAND=<name> ...`
4. **`sv` `install_deps`** — a `case` arm per agent in both the
   `NATIVE_INSTALL` branch and the Homebrew branch

The README documents each agent in three places: Quick Start, Native Install
bullet list, and the `sv -N <agent>` examples. The CHANGELOG records the
addition.

## Decisions

### Aliases
Both `p` and `pi` launch pi. Matches the `cl`/claude, `co`/codex,
`o`/opencode, `g`/gemini convention. No third alias.

### Autonomy flag
The wrapper passes `--approve` (`-a`). pi's `--approve` ("trust project-local
files for this run") is the closest analog to the other agents' skip-permissions
flags (`--dangerously-skip-permissions`, `--dangerously-bypass-approvals-and-sandbox`,
`--yolo`, `OPENCODE_PERMISSION='{"*":"allow"}'`). The whole point of these
wrappers is autonomous sandboxed use, so `-a` matches intent.

### System-prompt injection
pi supports `--append-system-prompt <text>` natively — the same mechanism the
claude wrapper uses. When `sv_tool_prompt` returns non-empty, the wrapper appends
it via `--append-system-prompt`.

### Binary resolution order
The wrapper mirrors the opencode/claude pattern with pi's install methods:

1. **`SV_NATIVE_INSTALL=true`** → install into `~/.local/bin/pi` via
   `npm install -g @earendil-works/pi-coding-agent`. Uses `~/.local` as the npm
   prefix, identical to the codex/gemini native-install wrappers, to avoid the
   Bun `node_modules`-at-`$HOME` ancestor-walk trap (webcoyote/sandvault#169).
2. **Homebrew** → `$HOMEBREW_PREFIX/bin/pi` if present (pi has no official
   Homebrew formula today, but this branch costs nothing and picks up any
   user-installed formula).
3. **Known PATH locations** → check explicit paths `~/.local/bin/pi` (a prior
   native install) and `~/.bun/bin/pi` (the common Bun-global install). We check
   explicit paths rather than `command -v pi` because the wrapper itself is
   installed at `~/bin/pi` in the sandbox and `$HOME/bin` precedes these on
   PATH, so a generic `command -v pi` would resolve to the wrapper and re-exec
   it in an infinite loop.

If none resolve, error with guidance: `ERROR: pi is not installed` plus a hint
about `sv -N pi` and the `~/.bun/bin/pi` location.

This A/B/D resolution (Bun-global via the explicit `~/.bun/bin/pi` check;
npm-global as the explicit native-install path; `~/.local/bin/pi` covering a
prior native install) matches the user's confirmed install landscape.

## Files changed

### New file: `guest/home/bin/pi`

Structurally identical to `guest/home/bin/opencode`. Outline:

```bash
#!/bin/bash
set -Eeuo pipefail
trap '...' ERR

source "$(dirname "${BASH_SOURCE[0]}")/sv-tool-prompts.sh"

unset PI
if [[ "${SV_NATIVE_INSTALL:-}" == "true" ]]; then
    NPM_PREFIX="${HOME}/.local"
    NPM_BIN="$NPM_PREFIX/bin"
    if [[ ! -x "$NPM_BIN/pi" ]]; then
        echo >&2 "Installing pi natively with npm..."
        npm install --prefix "$NPM_PREFIX" -g @earendil-works/pi-coding-agent
    fi
    PI="$NPM_BIN/pi"
elif command -v brew &>/dev/null ; then
    HOMEBREW_PREFIX="${HOMEBREW_PREFIX:-"$(brew --prefix)"}"
    if [[ -x "$HOMEBREW_PREFIX/bin/pi" ]]; then
        PI="$HOMEBREW_PREFIX/bin/pi"
    fi
fi

# Fallback: explicit known locations (NOT `command -v pi`, which would resolve
# to this wrapper at ~/bin/pi in the sandbox and loop forever). Covers a prior
# native install (~/.local/bin/pi) and the common Bun-global install
# (~/.bun/bin/pi).
if [[ -z "${PI:-}" || ! -x "$PI" ]]; then
    if [[ -x "$HOME/.local/bin/pi" ]]; then
        PI="$HOME/.local/bin/pi"
    elif [[ -x "$HOME/.bun/bin/pi" ]]; then
        PI="$HOME/.bun/bin/pi"
    fi
fi

if [[ -z "${PI:-}" || ! -x "$PI" ]]; then
    echo >&2 "ERROR: pi is not installed"
    echo >&2 "       Install it with 'sv -N pi' (native npm install) or ensure pi is"
    echo >&2 "       on PATH (e.g. a Bun-global install at ~/.bun/bin/pi)."
    exit 1
fi

TOOL_PROMPT=$(sv_tool_prompt)
EXTRA_ARGS=()
if [[ -n "$TOOL_PROMPT" ]]; then
    EXTRA_ARGS+=(--append-system-prompt "$TOOL_PROMPT")
fi

echo >&2 "$USER: running pi --approve"
exec "$PI" --approve ${EXTRA_ARGS[@]+"${EXTRA_ARGS[@]}"} "$@"
```

### `sv` — five edits

1. **Help text** — after the `g,  gemini [PATH]` line, add:
   ```
   echo "  p,  pi     [PATH]  Open pi in sandvault"
   ```
   And update the "Arguments after -- are passed to the command" line to
   include `pi`.

2. **Dispatch case** — after the `g|gemini)` arm, add:
   ```bash
   p|pi)
       COMMAND=pi
       INITIAL_DIR="${2:-}"
       ;;
   ```

3. **`install_deps` native branch** — add a `pi)` case:
   ```bash
   pi)
       ensure_brew_tool "node" "node"
       ;;
   ```
   (npm-based native install, same as codex/gemini — node needed for `npm`.)

4. **`install_deps` Homebrew branch** — add a `pi)` case that is an explicit
   no-op (pi has no Homebrew formula; users run it from PATH or native npm):
   ```bash
   pi)
       # No Homebrew formula; pi runs from PATH (e.g. ~/.bun/bin/pi) or native npm install.
       ;;
   ```

5. **`--fix-permissions` Homebrew symlink loop** — add `pi` to the
   `for tool_cli in claude codex opencode gemini ...` list so a user-installed
   brew pi symlink is repaired like the other agents.

### `README.md` — three edits

1. **Quick Start** — after the gemini block, add:
   ```bash
   # Run pi in the sandbox
   # shortcut: sv p
     sv pi
   ```
2. **Native Install bullet list** — add:
   - **pi** — installed via `npm install -g @earendil-works/pi-coding-agent`
3. **`sv -N` examples** — add `sv -N pi` after `sv -N gemini`.

### `CHANGELOG.md`

Add an Unreleased / new-version entry recording the addition.

## Out of scope (YAGNI)

- No new `prompts/*.md` tool-prompt file for pi; pi reuses the shared
  `sv_tool_prompt` mechanism.
- No `sv-clone` changes (it is agent-agnostic; the claude reference is a
  comment example only).
- No third alias.
- No tests beyond the existing project validation (`scripts/validate`); the
  `tests/` directory holds integration fixtures unrelated to agent wiring.

## Testing / verification

- `shellcheck guest/home/bin/pi`
- `sv --help` shows the `p, pi` line
- Dispatch resolves: trace `COMMAND=pi` without error
- `./scripts/validate` passes (per `AGENTS.md`)
- No regressions in existing behavior

## Open questions

None. All decisions confirmed with the user.
