# Agent — turn phases, prompts and profile-scoped providers

Root caching/facade rules apply. Details: `website/docs/developer-guide/agent-loop.md`, `prompt-assembly.md`, `context-compression-and-caching.md`, `provider-runtime.md`, `session-storage.md`, `subagent-lifecycle-api.md`.

## Code map

- `run_agent.py` exposes AIAgent; `agent_init.init_agent` constructs it. Mixins own turn facade, client lifecycle, streams, persistence and compression; the constructor in the facade is the current parameter authority.
- `conversation_loop.run_conversation` runs synchronously after `turn_facade_lease.py` takes the session lease. `chat` returns text; run_conversation returns final_response and messages.
- `turn_*.py` siblings own preflight/preparation, request/response, validation/tool round, overflow/truncation, compaction/recovery, retries, liveness/budget, stopping and finalization. Change the responsible phase rather than growing the facade.
- `tool_executor.py` uses `inline_tool_executors.INLINE_TOOL_EXECUTORS` for agent-owned todo/memory before normal dispatch; add table entries, not name chains.
- Messages use OpenAI role shape; assistant reasoning stays in the reasoning field. Respect interrupt checks, shared iteration budgets and the single grace call.

## Message and context invariants

- Keep system prompt/past context/toolsets/memory byte-stable for the conversation; compression is the only cache-breaking mutation. New mid-session content uses user messages/tool results, never a rebuilt system prompt.
- Preserve role alternation. `/steer` is the sole mid-loop user injection, as a separate row after a tool result; never mutate persisted tool rows. Cron deliveries have separate sessions.
- Context loads at startup from the cwd chain with configured/dynamic caps. Never inject install-tree guides or outside-cwd files as project context.
- `subdirectory_hints.py` adds hints to tool results; its 32,000-character head/tail cap is independent of context_file_max_chars. Log truncation without queuing chat-status warnings.
- `_last_resolved_tool_names` is process-global; delegation saves/restores it and readers can see temporary stale state.

## Compression and model resolution

- Every manual /compress surface calls `conversation_compression_manual.compress_now`; keep one parser for here/count/focus/preview/aggressive. Surface adapters only parse argv, install after_messages and render.
- Gateway hygiene and ContextCompressor have distinct thresholds (85% / configurable 50%). Prune old tool results, choose boundaries, then summarize through the auxiliary compression model; preserve the stable session id.
- Overflow cooldown and provider-native compaction remain provider-specific. A stalled summary retries once on the auxiliary fallback chain; repeated stall uses deterministic fallback through the same leased/fenced pipeline, never an outside-lease prune.
- Provider profiles live in plugins; model metadata/capabilities in `model_metadata.py`. Follow provider-runtime.md precedence; resolution/credential fallback requires real-import isolated-home E2E.
- Auxiliary tasks resolve via `auxiliary_client._resolve_auto_route` with per-task overrides. All physical attempts pass _relay_sync_completion/_relay_async_completion/_relay_sync_stream and observer-only fail-open auxiliary hooks with aux_task; never fire main-loop API hooks for them.

## i18n, memory and isolation

- `i18n.py`, `i18n_layers.py`, `i18n_languages.py`: last-registered plugin pack → profile overlay → bundled chosen language → English → bare key. Partial catalogs use flat dotted keys; cache by home/language.
- Pack registration and language changes reset merged caches. supported_languages is live union, SUPPORTED_LANGUAGES bundled-only; bundled endonyms agree with `apps/shared/src/i18n.ts`.
- TUI/desktop catalog payloads are opaque beyond parse/flatten/serve; bundled TUI YAML is the backend bottom layer while TS ships English.
- Memory uses `memory_provider.py` ABC plus `memory_manager.py`; context/image engines use their respective ABCs. Curator lives in curator/curator_backup siblings. Cron defaults skip_memory=True intentionally.
- Session-end extraction/provider hooks also run on eviction, shutdown and gateway teardown. CALLERS bind the owner before flushing; the session DB parent identifies home, never the current process environment.
- Background provider/title work uses `memory_provider.spawn_context_thread` to copy scope. Never use bare threads or process-global cached homes.
- `secret_scope.get_secret` fails closed once multiplex is active; every new multi-home host must call set_multiplex_active(True). A child's UnscopedSecretError is a spawn bug, not permission to fall back to os.getenv.
- Delegated children inherit context and use DELEGATED_CHILD_ENV_MARKER containing the fenced Kanban board root, not a boolean flag.

## Checks

`tests/agent/` through the isolated runner. Patch the facade binding each phase reads; test role alternation, byte-stable prompts, scope ownership and compression outcomes, not whole prompt snapshots.
