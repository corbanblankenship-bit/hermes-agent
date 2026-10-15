# CLI — commands, configuration, updates and profiles

Scope: `cli.py`, `hermes_cli/`; root invariants apply. Details: `website/docs/developer-guide/cli-internals.md`.

## Entry points

- `cli.py` owns HermesCLI and mutable facade state; `cli_*_mixin.py` owns behavior. `cli_config_load.py`, `cli_render.py`, `cli_terminal_input.py`, `cli_shutdown.py`, `cli_single_query.py`, `cli_auto_maintenance.py` own topical helpers.
- Siblings late-bind facade names, so patch `cli` or `hermes_cli.gateway` where runtime reads them. Gateway siblings use `_gw()`; facade globals retain mutable caches.
- Rich renders panels, prompt_toolkit owns input, `agent/display.py` owns activity display. Menu pickers use `curses_ui.py`; spinner output space-pads instead of ANSI erase-to-EOL, which leaks through patch_stdout.
- Wrapper CLIs use protected `cli_tui_mixin.py` hooks, not overridden `run()` (`extending-the-cli.md`).

## Commands

- `commands.py::COMMAND_REGISTRY` is the single source for aliases, completion, help and gateway/platform menus. Resolve canonical names, then `_SLASH_DISPATCH` or handler-name convention; no elif ladders.
- New commands register a `CommandDef` plus CLI mixin handler. Gateway support adds its slash-command mixin handler and `_IDLE_COMMANDS`/`_PLAIN_COMMANDS` entry in `gateway/run_busy.py`.
- Gated gateway commands remain dispatchable in the known-command set, while menus/help respect their config gate. Persist settings through `save_config_value`.
- Skills inject user messages, never rebuild the system prompt. Prompt-changing commands defer by default with explicit `--now` opt-in.
- `goal_command.dispatch_goal_command` owns goal parsing/mutations for every supported surface; adapters authorize and render. Off-loop callers preserve ContextVars; ACP has no goal loop.

## Config contracts

- Non-secret settings belong in `config_defaults.py::DEFAULT_CONFIG`; bump `_config_version` only for a migration, not additive defaults. New root keys also enter `_KNOWN_ROOT_KEYS`.
- `.env` holds secrets only, registered with `OPTIONAL_ENV_VARS` (description, prompt, URL, password and category). Bare registered/extra/platform secret keys route through `config_env_routing.py`, never YAML top-level keys.
- `terminal.cwd` is canonical; CLI uses current cwd, messaging bridges it to `TERMINAL_CWD`. `MESSAGING_CWD` is removed; `.env` TERMINAL_CWD is deprecated.
- Config writes use `atomic_config_write` (additive, refuses omission deletion) or explicit `atomic_config_replace` (unset/migration/full-state), preserving comments/order/quotes and failing closed on unreadable input. Never dump YAML directly to config paths.
- Choose the correct loader: CLI `load_cli_config`; setup/subcommands `config.load_config` with defaults; presence-sensitive runtime `config_effective.load_user_config_effective` with managed overlay/expansion/model canonicalization and no defaults. Raw reads are for round-trip writing only.
- Every registered key needs a runtime reader and vice versa. Test behavior through the consuming surface's loader under an isolated home; a launch environment never overrides a served profile's YAML.
- `skin_engine.py` owns pure-data skins, inheritance and cache. Extend YAML rather than code; detailed keys: `website/docs/user-guide/features/skins.md`.

## Update and service lifecycle

- Preserve the transactional pipeline: plan → snapshot → apply → restart-per-kind → verify → report. Every profile receives the same recovery file set; `--backup` owns rollback.
- Windows ZIP fallback requires a git failure and clean tree, preserving `_ZIP_PRESERVED_NESTED` outputs. Never run newly pulled code in the old interpreter.
- `update_handoff.py` and `update_serve_obligations.py` are frozen compatibility surfaces (`tests/compat/old_updater_surface.json`); do not remove names.
- Restart the owning install's whole fleet, draining first and respecting `update_fleet_scope.py`. Post-commit failure owes follow-up (exit0; leftover stash1; interrupt130); receipts live in root logs/update_receipts.
- `gateway.py` owns PM-aware systemd generation/currentness/refresh; `gateway_launchd.py` owns macOS services; Windows siblings own scheduled tasks. Validate all affected service kinds, including both user/system units.
- Service environments explicitly carry target HERMES_HOME and service-user HOME. Unresolved systemd User is a blocker, not a directory-owner fallback. Resolve launchd gui/user domains, not plist globs.
- Process identity uses `(pid,start_time)` and canonical matchers, not argv substrings/bare PID presence. Read gateway-owned control-socket design before adding process heuristics.
- Externally supervised gateways declare ownership via control-socket identify or argv marker; `gateway_supervised_restart.py` hands back via SIGUSR1 and waits a fresh supervised PID, never foreground replacement.

## Profile isolation and migration

- Single-profile CLI binds HERMES_HOME before imports. Multiplex/serve binds each activity through contextvars; module globals/environment retain the launch home.
- Profiles are independent, with no live inheritance. Clone copies at creation but strips messaging ownership using `profile_channels.py` under the SOURCE plugin scope; shared prefixes are stripped only for active owning adapters, including LEFT_CORE ownership.
- `--clone-channels` is explicit and `create_profile` enforces the live-multiplexer refusal across all surfaces. Build clones in hidden staging, materialize symlinked config/secrets, then publish by one rename; never write through source symlinks.
- `profiles_to_serve(multiplex=True)` enumerates default plus live, non-tombstoned identity-bearing profile directories. Dangling identity symlinks still count. Marker-less directories may contain user files: do not delete/recreate them unless the allowed tombstone path applies.
- Enumeration is read-only. `mkdir_under_hermes_home`/`_ensure_cron_dir` refuse missing/deleted named homes. Bot roster/relay checks share the identity predicate; discovery slots key by `hermes_home_key()`, not global flags.
- `gateway_multiplex_mode.resolve_multiplex_mode` settles an unset mode once at boot using live topology/preflight. UI readers use `default_gateway_multiplexes`, not merged defaults; there is no profile allowlist.
- `gateway migrate --multiplex` is the supported migration target; an unfinished gateway_migration.json resumes. Named-profile install/start/run refuses under a multiplexer without explicit force; dashboard shares the refusal.
- Auto migration runs only after successful update and honors `gateway_migrate_guards.py` plus `gateway.auto_multiplex_migration`. Cross-domain/user/outside-profile homes block automatic migration and require explicit-path notices.
- Reuse credential fingerprints, `platform_binds_port` and adapter `serves_profile_prefix`; never duplicate platform-name lists.

## Sign-in and tests

`anon_auth.run_sign_in` is the common free-tier sign-in state machine: persist only after promotion AND token grant; settle exactly once. CLI, gateway and desktop render its states; plain device-code login stays separate.
Use `tests/hermes_cli/` through `scripts/run_tests.sh`; preserve real-import config/profile/security tests and native-OS service coverage.
