# Hermes Agent — development map

Hermes shares one agent core across CLI, messaging, TUI and Electron surfaces; plugins and skills own extensions.
Long-form guidance: `website/docs/developer-guide/`; contributor workflow: the `hermes-agent-dev` skill.

## Architecture and routing

| Area / entry point | Focused guidance |
|---|---|
| `run_agent.py`, `agent/turn_*.py` | `agent/AGENTS.md`: turns, caching, compression, providers |
| `model_tools.py`, `toolsets.py`, `tools/registry.py` | `tools/AGENTS.md`: discovery, dispatch, delegation, session toolsets |
| `cli.py`, `hermes_cli/cli_*_mixin.py` | `hermes_cli/AGENTS.md`: commands, config, profiles, updater |
| `gateway/run.py`, `gateway/run_*.py`, `gateway/session*.py` | `gateway/AGENTS.md`: adapters, isolation, locks, streaming |
| New messaging adapter | `gateway/platforms/ADDING_A_PLATFORM.md` |
| `plugins/`, `hermes_cli/plugins*.py` | `plugins/AGENTS.md`: compatibility, discovery, host isolation |
| `plugin-catalog/` | `plugin-catalog/README.md`; keep mirrored admission policy identical |
| `tui_gateway/`, `ui-tui/` | `tui_gateway/AGENTS.md`: JSON-RPC and session lifecycle |
| `apps/desktop/`, `apps/shared/` | `apps/desktop/AGENTS.md`, `apps/desktop/src/AGENTS.md` |
| `web/`, `hermes_cli/web_routers/` | `web/AGENTS.md`: dashboard and embedded TUI |
| `skills/`, `optional-skills/`, `agent/curator*.py` | `skills/AGENTS.md` |
| `cron/`, kanban scheduling | `cron/AGENTS.md` |
| `pm/`, `pyproject.toml` | `pm/AGENTS.md`: dependency/environment management |
| `hermes_platform/` | `hermes_platform/AGENTS.md`: machine facts and executable lookup |
| `tests/`, `scripts/` | `tests/AGENTS.md`: isolated runner and behavior contracts |

`hermes_state.py` + siblings own SessionDB; `hermes_constants.py` owns profile-aware paths.
User settings, secrets and logs live under the active Hermes home; `.env` contains secrets only.

## Rules that apply everywhere

- Preserve the cached conversation prefix: no past-context, toolset, memory or system-prompt mutation mid-session except compression. Prompt-changing commands defer until the next session unless `--now` is explicit.
- Prefer extending code → CLI + skill → service-gated tool → plugin → catalog MCP → core tool. Core tools need broad use and no existing route; hooks need a concrete consumer.
- Reproduce bugs on `main`, identify the failing line and inspect intent/history (`git log -p -S`). Fix the behavior class and sibling paths without destroying the feature.
- Security disclosure follows `SECURITY.md`: §3.1 findings stay private; §3.2 hardening is public. Identify the §2 boundary and reproduction; never publish private findings in issues, commits or PRs.
- New capabilities use existing setup UX; third-party products ship standalone plugins. Plugins never edit core. Outbound telemetry requires opt-in.
- Use `get_hermes_home()` for runtime paths and `display_hermes_home()` for text; `_get_profiles_root()` intentionally stays HOME-anchored.
- A process may serve many profiles. Bind the owner's scope at boot probes, callbacks, RPCs, eviction, tickers, threads and child spawns; environment/module globals represent the launch profile. See `gateway/AGENTS.md` for binding seams.
- Use `hermes_platform` for host facts/executable lookup and canonical process matchers (`gateway.status.looks_like_gateway_command_line`, `hermes_cli.update_cmd._hermes_holder_subcommand`), never argv substrings or bare PID existence.
- Extend before duplicating; repeated extension categories use an ABC and orchestrator.
- Keep behavior in topical siblings and public entry points/state in facades. Late-import facades inside functions; avoid module cycles. Patch the binding production actually reads.
- Internal moves get updated callers/docs, not re-export shims. Search developer docs, skills and area guides when moving symbols.
- No dead flags/code, needless wrappers or swallowed exceptions. Comments explain why. Replace dispatch ladders with tables.
- Code-health ratchets: new functions CC≤20, ≤300 lines, nesting≤6; files≤2,000 lines. Existing oversized units may only shrink. Configuration: `scripts/code_health/config.py`.
- Dependencies need upper bounds; git dependencies/Actions need SHA pins. After `pyproject.toml`, run `hermes pm lock` and commit `uv.lock`; never raw pip/uv-mutate Hermes environments.
- TypeScript uses feature-owned nanostores, thin routes, focused hooks/actions, interfaces for shared shapes, table dispatch; `src/app` routes, `src/store` state, `src/lib` pure helpers.
- Rebase onto current `main` before merge; preserve authorship when salvaging. Triage closes only implemented-on-main, unreproducible or incoherent work; taste decisions remain with maintainers.

## Development and checks

- Pick isolated `HERMES_HOME`/`HERMES_RUNTIME_DIR`, then `source ./activate` (or platform equivalent). Setup details: `website/docs/reference/package-management.md#developer-workflow`; separate test environment: `CONTRIBUTING.md`.
- Run `python scripts/check` for pinned CI lint; refresh its pre-push hook after pulling. Ratchet waivers require `# health: allow <RULE> -- <why>`; `noqa` is insufficient.
- Run tests through `scripts/run_tests.sh`; never write tests against the real user home. Host-specific tests use their native host and `@pytest.mark.platforms(...)`.
- Prefer 1–2 regression contracts proven red on base; resolution/config/security/I/O changes require real-import E2E under an isolated home. Avoid counts, source-reading and change-detector assertions.
- `scripts/ci/check_agents_md_size.py` enforces root/chain character budgets; keep area adapters concise and detail in existing developer guides.
