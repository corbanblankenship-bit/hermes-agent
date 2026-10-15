# Tools — registry, execution and delegation

Scope: `tools/`, `toolsets.py`, `model_tools.py`; root invariants apply.
Details: `website/docs/developer-guide/adding-tools.md`, `tools-runtime.md`, `subagent-lifecycle-api.md`.

## Registry and toolsets

- `tools/registry.py` is dependency-free; tools register at import. `model_tools.discover_builtin_tools()` discovers top-level registrations automatically.
- A package tool registers in `tools/<package>/tool.py` and needs `__init__.py`; siblings are libraries. Discovery does not itself expose a tool: `toolsets.py` must name it.
- Local/custom tools belong in the active home's `plugins/<name>/` via `ctx.register_tool`, not core. Only explicitly contributed core tools enter `_HERMES_CORE_TOOLS` or a new toolset.
- Handlers return JSON strings; native-image results may use exactly `{_multimodal: true, content: [...], text_summary: str}` through `vision_tools._native_tool_result_images`.
- Schema text must not name tools from other toolsets. Conditional cross-references belong in `model_tools.get_tool_definitions()`; instructional tools do not paginate with offset/limit.
- Use `display_hermes_home()` in schemas and call-time `get_hermes_home()` for state. Never freeze a state path at import.
- Agent-local tools (`todo`, `memory`) dispatch through `agent/inline_tool_executors.py::INLINE_TOOL_EXECUTORS` before normal dispatch.
- `_last_resolved_tool_names` is process-global; delegation saves/restores it, so readers may see stale values during child execution.
- Register secrets in `OPTIONAL_ENV_VARS` and use existing tools/setup UX. Per-platform toggles are `tools.<platform>.enabled/disabled`; browser-use replaces browser tools with `browser_exec`.

## Surface capability is a property of the SESSION, never of the process env

- `check_fn` answers profile reachability/opt-in, never session surface. Its cache is keyed by `hermes_home_key()`; read credentials through `agent.secret_scope.get_secret`, not `os.getenv`. Unscoped boot probes log at DEBUG.
- GUI capabilities use named session toolsets (`desktop_ui`, `project`) folded in by `_load_enabled_toolsets(platform)`. `HERMES_DESKTOP` describes backend origin, not whether a GUI watches; remote GUI sessions must work without it.

## Runtime ownership

- Add provider/backend table entries or topical siblings, not name ladders. Terminal backends: `tools/environments/`, `terminal_tool_backends.py`, `tool_backend_helpers.py`; browser/MCP: `browser_tool_*.py`, `mcp_tool_*.py`; TTS: `tts_tool_providers.py`, `tts_command_provider.py`; optional skills: `skills_hub_official.py`.
- Fix remote file visibility at the mount. Native image embeds persist in history; all size/repeat policy belongs in `vision_tools_history_budget.py`, keyed by session and resolved source so crops share a budget. The default repeat cap applies to delegated child contexts only.
- Every process spawn uses `environments/local.py::build_subprocess_env`; profile-serving children use `served_profile_child_env(target_home=, inherit_credentials=)`. Strip launch-profile residue, pin target home and overlay only target secrets before spawning.
- Never use a copied process environment as a profile substitute or add credential fallthrough after `UnscopedSecretError`. Contextvars do not cross processes; threads use `agent.memory_provider.spawn_context_thread`.
- MCP trust is per home: `mcp_tool_registration._record_scope_trust` and `mcp_tool_handlers._trust_gate_check` use the caller's profile, never another profile's same-named server.
- Background teardown snapshots descendants, SIGTERMs the recorded parent first, waits its configured grace, then signals survivors and finally kills holdouts. In `process_registry_termination.py`, systemd scope teardown runs after the PID path or proven parent exit/recycling, never before a live parent can reap children.

## Delegation

- `delegate_tool.py` spawns isolated context/terminal sessions. Foreground waits; background returns an id and reports through the completion queue. Single and batch calls share the configured concurrency bound.
- `delegate_tool_dispatch._units_of` groups independent completions; units of one call share one pool slot. Size executors by live units and start stall clocks only when runners start.
- Leaf roles omit delegation, clarify, memory, messaging and cron but keep code execution. Orchestrators require the opt-in and depth bound; nested delegations are synchronous.
- Child background processes are reaped at teardown and notices suppressed in the parent. Explicit child-only `process_manage(action="handoff")` transfers ownership under the registry lock; report orphaned processes/unread completions before cleanup.
- Child code kernels use `<parent-owner>::child::<child-session-id>`, stay pinned while running, then are disposed by `shutdown_kernels_for_delegated_child` so finished children cannot consume the LRU capacity.
- Failed output-schema retry preserves the raw summary with completed status plus schema error metadata; never discard a child's result.
- Background delegation is process-local. Restart-surviving work uses cron or background terminal completion notification.
- Heartbeat events share the notification pipeline and single registry timer; every watch-match renderer also handles heartbeats, including gateway drain and TUI dedup fields.
- Every stop surface interrupts active children AND `async_delegation.interrupt_for_session`; detached background units otherwise survive. Interrupt summaries preserve the child's last real assistant text.

## Verification

`tests/tools/`: exercise real registry dispatch and contracts, not counts. Approval/security boundaries require real-import isolated-home E2E (example: `tests/tools/test_approval_config_readonly.py`). Test GUI capability with desktop-origin environment absent. Use the root's isolated test runner.
