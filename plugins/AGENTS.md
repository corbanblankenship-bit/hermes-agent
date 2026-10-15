# Plugins — extension boundary and compatibility

Root invariants apply. Canonical guide: `website/docs/developer-guide/plugins/index.md`; per-kind guides live beside it.

## Extension policy

- Plugins own their directory and use ABCs/hooks/ctx; never patch core. A concrete missing consumer may justify a generic extension seam, never plugin-specific core behavior.
- Memory setup uses `hermes memory setup` → `provider.post_setup`, not parallel top-level commands. New memory providers and third-party products ship standalone repositories/catalog entries; no new in-tree providers or vendor integrations.
- Catalog provider modules resolve through `plugins.memory.import_provider_module`, not historical bundled imports. Reference plugins live in NousResearch/hermes-example-plugins.
- Departing core features register in `hermes_cli/left_core_migration.py::LEFT_CORE` with a read-only usage predicate. Core-era conversion and automatic install have separate once-per-home markers; write installed marker only after success so failures retry and deliberate removal sticks.
- Preserve names/config/env through generic PlatformEntry fields, not special name checks. Former core auth commands redirect to the plugin's own command.

## Catalog and discovery

- `plugin-catalog/` is the sole out-of-tree name index. Entries require a full SHA pin and review. Admission text in README and `plugins/catalog-submission.md` stays identical; CI validates the pinned plugin.
- `removed.yaml` gates every install/update/enable/load path by canonical repository identity. Only explicit CLI `--allow-removed`, persisted on the install record, grants an exception.
- Trusted provenance is installer-owned `.install-metadata.json` and checked-out SHA. In-tree `.hermes-catalog.json` is untrusted convenience data; Python never uses it as authority.
- `hermes_cli/plugin_catalog.py` owns live catalog/fallback loading; `hermes_cli/plugins_cmd_catalog.py` owns resolution, validation, payloads and re-pinning. Unknown bare names error; do not add a second index.

| Kind | Entry points and precedence |
|---|---|
| General | `hermes_cli/plugins.py::PluginManager`; bundled/user/project/entry-point discovery, later-wins; `register(ctx)` owns hooks, tools, CLI and locales |
| Memory | `plugins/memory/__init__.py`; bundled-first, then home, opted-in project, entry points; enumerate without importing; ABC `agent/memory_provider.py`, orchestrator `agent/memory_manager.py` |
| Model | `providers/__init__.py`; lazy first profile/list lookup, bundled then user then legacy; last-writer-wins; PluginManager records manifests without double-importing |
| Context/image | `agent/context_engine.py`, `agent/image_gen_provider.py`; ABC + orchestrator + per-plugin directory |
| Platform | `plugins/platforms/<name>/adapter.py`; gateway token-lock and scoped-secret contracts |

- Memory-provider CLI registers only for the active provider. Native language packs may be manifest-only; `provides_locales` registers catalogs and `hermes_cli/plugin_validate_locales.py` validates them.
- `model_tools` import normally triggers discovery; other readers must explicitly call idempotent `discover_plugins`. Mid-run install/enable/update/reload uses `force=True`.
- `on_plugin_loaded` originates inside discovery, never RPC; activation summaries distinguish live handlers, next-session tools/prompt, and MCP awaiting reload (`hermes_cli/plugins_activation.py`).
- Auxiliary LLM calls fire `pre_auxiliary_call`/`post_auxiliary_call` with aux_task, not turn-scoped API hooks. Default changes preserve explicitly configured existing users through migration guards.

## Profile and host boundaries

- Lifecycle callers bind the owning profile for start/end/sync/shutdown, including eviction, teardown and cron. Key provider state by the per-call home, never a home cached at initialization.
- Credentials use `agent.secret_scope.get_secret`, including check_fn. Background threads use `spawn_context_thread`; platform plugins never mutate os.environ. YAML uses PlatformConfig.extra and `_shared.apply_yaml_bridge`; gates use platform_gate_env.
- `plugins.isolation: host` runs non-bundled plugins in a per-profile process (`hermes_cli/plugin_host.py`, `hermes_cli/plugin_host_child.py`, `hermes_cli/plugin_host_wire.py`). Every user-code import routes there or refuses, including provider discovery and dashboard APIs.
- `hermes_cli/plugin_host_profiles.py` extracts credential-free model profiles by fingerprint; `plugins/plugin_loader.py` refuses unapproved synthetic namespace imports. No host starts while its module import lock is held.
- Unsupported host methods have one authority: `plugin_isolation.HOST_UNSUPPORTED_CTX_METHODS`, shared by runtime and audit. Do not leak live gateway/adapter objects across it.
- Hosts never spawn hosts. Operator-pinned isolation uses managed overlay readers, never raw user config. Children receive only their profile's secrets through `served_profile_child_env`; plugin APIs remain tenant-unaware.

## Compatibility and checks

- Documented native surfaces stay additive; no monolithic API-version matching or unrelated version literals. Add hook kwargs via signature filtering; narrow callbacks receive only declared fields.
- Never remove/rename PluginContext methods; new parameters are optional, preferably keyword-only. Ignore unknown manifest fields; provider methods get defaults and optional callback kwargs are inspected.
- Version only a real persisted/wire capability and migrate or preserve old state/replay. Deprecation needs one warning per process, a replacement/migration note and at least two later minor releases.
- Internal import paths are not API: fix callers instead of adding re-export shims. `hermes_cli/plugin_compat.py` retains only inert updater compatibility stubs.
- `tests/plugins/`: load frozen plugins through real discovery under an isolated home; assert tool/hook outcomes, not counts, source shape or version literals. Telemetry remains opt-in.
