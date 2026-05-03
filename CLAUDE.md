# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

`AGENTS.md` is the canonical, in-depth guide for AI coding assistants — read it for architecture details, plugin/skill systems, profile rules, skin engine, and the long list of known pitfalls. This file is the short orientation; the rules below are the load-bearing ones you'll trip over without warning.

## Commands

### Testing — always use the wrapper

```bash
scripts/run_tests.sh                                  # full suite, CI-parity
scripts/run_tests.sh tests/gateway/                   # one directory
scripts/run_tests.sh tests/agent/test_foo.py::test_x  # one test
scripts/run_tests.sh -v --tb=long                     # pass-through pytest flags
```

**Do not call `pytest` directly.** The wrapper unsets every credential-shaped env var, pins `TZ=UTC LANG=C.UTF-8 PYTHONHASHSEED=0`, and runs with `-n 4` xdist workers (matches CI's ubuntu-latest). On a 16+ core dev box, `-n auto` surfaces ordering flakes CI never sees, and your real `~/.hermes/.env` API keys leak into tests. The wrapper probes `.venv` → `venv` → `$HOME/.hermes/hermes-agent/venv` for the venv.

If you must invoke pytest directly (IDE integration, etc.), at minimum activate the venv and pass `-n 4`. The autouse fixture in `tests/conftest.py` enforces credential blanking + `HERMES_HOME` redirection, but the wrapper is belt-and-suspenders.

### Dev environment

```bash
./setup-hermes.sh                  # one-shot: installs uv, creates venv, installs .[all], symlinks ~/.local/bin/hermes
# or manually:
uv venv venv --python 3.11
source venv/bin/activate
uv pip install -e ".[all,dev]"
```

### TUI development

```bash
cd ui-tui
npm install            # first time
npm run dev            # watch mode (rebuilds hermes-ink + tsx --watch)
npm run type-check     # tsc --noEmit
npm run lint           # eslint
npm test               # vitest
```

### Entry points

- `hermes` — interactive CLI (`hermes_cli.main:main`)
- `hermes --tui` — Ink/React TUI (Node frontend, Python `tui_gateway` backend over stdio JSON-RPC)
- `hermes gateway start` — messaging gateway (Telegram, Discord, Slack, WhatsApp, Signal, Email, ...)
- `hermes-acp` — ACP server for VS Code / Zed / JetBrains
- `python run_agent.py` / `python batch_runner.py` — direct agent / batch trajectory generation

## Architecture map (the big picture)

```
tools/registry.py    (no deps — central registry)
       ↑
tools/*.py           (each calls registry.register() at import time)
       ↑
model_tools.py       (imports tools/registry + triggers tool discovery, dispatch, plugin hooks)
       ↑
run_agent.py         (AIAgent class — synchronous conversation loop, ~12k LOC)
       ↑
cli.py               (HermesCLI — interactive CLI orchestrator, ~11k LOC)
gateway/run.py       (GatewayRunner — multi-platform messaging)
batch_runner.py      (parallel trajectory generation)
tui_gateway/         (Python JSON-RPC backend for ui-tui Ink frontend)
```

**Self-registering tools.** Any `tools/*.py` file with a top-level `registry.register()` is auto-discovered when `model_tools.py` is imported. Adding a tool = create the file + add it to a toolset in `toolsets.py`. All handlers must return a JSON string. See AGENTS.md "Adding New Tools".

**Slash commands have a single source of truth.** `hermes_cli/commands.py` `COMMAND_REGISTRY` (list of `CommandDef`) drives CLI dispatch, gateway dispatch, Telegram bot menu, Slack subcommands, autocomplete, and `/help`. To add a command: add a `CommandDef`, add a handler branch in `cli.py:process_command()`, and (if gateway-available) a branch in `gateway/run.py`. Aliases require only a tuple update.

**Two plugin surfaces, both under `plugins/`.** General lifecycle/tool/CLI plugins (`hermes_cli/plugins.py`) and memory-provider plugins (`agent/memory_manager.py` + `agent/memory_provider.py`). **Plugins MUST NOT modify core files** (`run_agent.py`, `cli.py`, `gateway/run.py`, `hermes_cli/main.py`) — extend the generic plugin surface instead. Discovery only runs as a side effect of importing `model_tools.py`; if a code path reads plugin state without that import chain, it must call `discover_plugins()` explicitly.

**Skills live in two parallel surfaces.** `skills/` ships and is active by default; `optional-skills/` ships but installs explicitly via `hermes skills install official/<category>/<skill>`. Heavy-dep or niche skills belong in `optional-skills/`.

**TUI = Ink (Node) + tui_gateway (Python) over stdio JSON-RPC.** TypeScript owns the screen; Python owns sessions, tools, and slash logic. `hermes dashboard` embeds the same `hermes --tui` over a PTY/WebSocket — **do not re-implement the chat experience in React**; extend Ink and the dashboard picks it up.

## Non-negotiable rules

### Profiles: never hardcode `~/.hermes`

```python
# GOOD
from hermes_constants import get_hermes_home, display_hermes_home
config_path = get_hermes_home() / "config.yaml"     # for code paths
print(f"Saved to {display_hermes_home()}/config.yaml")  # for user-facing strings

# BAD — breaks profiles
config_path = Path.home() / ".hermes" / "config.yaml"
```

`_apply_profile_override()` in `hermes_cli/main.py` sets `HERMES_HOME` before any module imports; module-level constants are fine because they cache after that. Tests that mock `Path.home()` must also set the `HERMES_HOME` env var. Profile *operations* (e.g. `_get_profiles_root()`) are intentionally HOME-anchored, not HERMES_HOME-anchored, so `hermes -p coder profile list` works regardless of active profile.

### Prompt caching must not break

Do not alter past context, change toolsets, reload memories, or rebuild system prompts mid-conversation. The only sanctioned cache-breaker is context compression. Slash commands that mutate system-prompt state default to **deferred invalidation** (next session); add an opt-in `--now` flag for immediate effect (canonical pattern: `/skills install --now`).

### Skill slash commands inject as user messages

`agent/skill_commands.py` injects skill content as a user message, not a system prompt addition — this is intentional to preserve prompt caching. Don't "fix" it by moving to system prompt.

### Don't write change-detector tests

A test that asserts a snapshot of routinely-changing data (model catalog names, config version literals, enumeration counts) only guarantees that source updates break CI. Assert relationships/invariants instead — "every catalog entry has a context length", "no plan-only model leaks into the legacy list", "migration bumps to current latest." See AGENTS.md "Don't write change-detector tests" for examples.

### Tests must not write to `~/.hermes/`

`tests/conftest.py` has an autouse `_isolate_hermes_home` fixture that redirects `HERMES_HOME` to a temp dir. Profile tests must additionally `monkeypatch.setattr(Path, "home", lambda: tmp_path)` so `_get_profiles_root()` resolves into the temp dir.

### Config: secrets in `.env`, settings in `config.yaml`

Add API keys/tokens/passwords to `OPTIONAL_ENV_VARS` in `hermes_cli/config.py` (with `password: True` and a `category`). Add timeouts/thresholds/feature flags/paths/display preferences to `DEFAULT_CONFIG`. Bump `_config_version` *only* when migrating/transforming existing user config (renames, structure changes) — adding a new key is handled by the deep-merge automatically.

Three loaders exist (`load_cli_config()` in `cli.py`, `load_config()` in `hermes_cli/config.py`, raw YAML in `gateway/run.py` + `gateway/config.py`). If a key is visible to the CLI but not the gateway, you're on the wrong loader — check `DEFAULT_CONFIG` coverage.

`MESSAGING_CWD` and `TERMINAL_CWD` in `.env` are deprecated; canonical is `terminal.cwd` in `config.yaml` (the gateway bridges it to `TERMINAL_CWD` for child tools).

### Tool schema descriptions: no cross-toolset name references

Don't write things like "prefer `web_search`" in `browser_navigate`'s description — `web_search` may be unavailable (missing API keys, disabled toolset) and the model will hallucinate calls. If a cross-reference is genuinely needed, add it dynamically in `get_tool_definitions()` in `model_tools.py` (see the `browser_navigate` / `execute_code` post-processing blocks).

### Display gotchas

- Don't use `\033[K` (ANSI erase-to-EOL) in spinner/display code — leaks as literal `?[K` under prompt_toolkit's `patch_stdout`. Use space-padding: `f"\r{line}{' ' * pad}"`.
- Don't introduce new `simple_term_menu` usage. Use `hermes_cli/curses_ui.py` (canonical pattern in `hermes_cli/tools_config.py`). Existing `simple_term_menu` calls in `hermes_cli/main.py` are legacy fallback only — it has ghost-duplication bugs in tmux/iTerm2.

### Gateway has two message guards

When an agent is running, messages traverse (1) base adapter (`gateway/platforms/base.py`) which queues to `_pending_messages` and (2) gateway runner (`gateway/run.py`) which intercepts `/stop /new /queue /status /approve /deny` before they reach `running_agent.interrupt()`. Any new command that must reach the runner while the agent is blocked must bypass **both** guards and dispatch inline — not via `_process_message_background()` (races session lifecycle).

### Squash merges from stale branches silently revert recent fixes

Before squash-merging, ensure the branch is up to date with main (`git fetch origin main && git reset --hard origin/main` in the worktree, then re-apply the PR's commits). A stale branch's version of an unrelated file will overwrite recent fixes. Verify with `git diff HEAD~1..HEAD` after merge — unexpected deletions are a red flag.

## Code style

- PEP 8 with practical exceptions; we don't enforce strict line length.
- Comments only for non-obvious intent / API quirks / trade-offs. Don't narrate what the code does.
- Catch specific exceptions; log with `logger.warning()` / `logger.error(exc_info=True)` for unexpected errors.
- Cross-platform: `termios`/`fcntl` are Unix-only (catch both `ImportError` and `NotImplementedError`); `os.setsid`/`os.killpg` need a `platform.system() != "Windows"` guard; use `pathlib.Path`, not string `/` concat; `.env` files on Windows may be `cp1252` (fall back to `latin-1` on `UnicodeDecodeError`); if you change `scripts/install.sh`, mirror to `scripts/install.ps1`.
- Conventional Commits for messages: `<type>(<scope>): <description>`. Types: `fix feat docs test refactor chore`. Scopes: `cli gateway tools skills agent install whatsapp security` etc.

## When in doubt

- Architecture / class layout / loop internals → AGENTS.md and the source files it points to (`run_agent.py`, `model_tools.py`, `cli.py`, `gateway/run.py`).
- "Should this be a skill or a tool?" → `CONTRIBUTING.md` "Should it be a Skill or a Tool?" (answer is almost always *skill*).
- Adding a platform adapter → `gateway/platforms/ADDING_A_PLATFORM.md`.
- Full website docs → https://hermes-agent.nousresearch.com/docs/
