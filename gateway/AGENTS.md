# Messaging gateway

Long form: `website/docs/developer-guide/gateway-internals.md`; adapter guide: `platforms/ADDING_A_PLATFORM.md`.
- `run.py` is the facade; run_*.py own lifecycle phases, session*.py sessions, slash_commands_*.py commands.
- `authz_mixin.py` owns authorization; platforms/<name>.py extend platforms/base.py; builtin_hooks is the hook seam.
- Gateway YAML is read raw via run.py/config.py, not DEFAULT_CONFIG; each adapter selects its base toolset.
- Dispatch slash handlers through _command_handler_table; run_busy.py IDLE/PLAIN classifications permit mid-run use.
- Approval/control commands must bypass BOTH adapter _pending_messages and runner busy guards.
- Dispatch those commands inline, never via _process_message_background, to avoid lifecycle races.

Streaming and notifications:
- For draft_stream_is_message is True, retain one cumulative native stream per turn.
- Drafts must remain prefix-stable: no per-tick fence closing, cursors, reset at tools or mrkdwn conversion.
- finish(final_text) carries the fully augmented final; do not mutate final_response after sealing.
- Every nonfinal consumer send carries metadata['_interim_send']=True.
- Both send and send_for_platform enforce interim/seal checks; new egress paths must too.
- Reconcile finals beside sealed streams by edit_message first; plain send only without an editable message.
- Test `draft_stream_is_message is True` explicitly so MagicMock attributes cannot enable the behavior.
- Slack streams use standard Markdown; stopStream text appends and stream endpoints have bounded API limits.
- Contract: `tests/gateway/test_stream_final_contract.py`, using real delivery paths with fake transports.
- Background watchers live in run_notifications.py; honor display.background_process_notifications.
- Require an admit_internal_event receipt; None is not acceptance. Refusal refunds all claimed siblings without attempts.
- Canonical CLI/TUI/Desktop notifications enter the durable human-input FIFO with private launch policy/internal_notification.
- Observers never own/consume turns; interrupted started turns remain unknown and pause followers.

Login and lifecycle:
- /login only runs in real paired DMs; ntfy/raft/a2a broadcast-like dm labels do not qualify.
- It binds install-wide providers.nous; shared gateways require allow_admin_from.
- Use the private single-worker login executor, one process-wide attempt, then evict nous/welcome agents.
- Never change model overrides to implement sign-in.
- Messaging gateway owners survive Desktop disconnect/quit; canonical Desktop attaches using gateway ensure.
- A legacy Desktop-owned serve child is distinct from the detached gateway owner.
- Do not reparent the gateway or widen tree-kills to solve update locks; preserve pause-for-update ownership gates.
- status.py stamps code_sha/code_version in gateway_state.json for fleet verification.

Routing identity and delivery:
- Resolve session_identity.resolve_identity ONCE before any ingress key derivation; pin frozen RoutingIdentity on source.
- Adapter canonicalization covers text/photo/albums, active-session paths and every source key.
- Runner canonicalization covers profile/default messages, busy/platform events, auth callbacks and inbound admission.
- Unserved multiplex identities warn and drop; never key them into agent:main.
- Existing resolvers read the identity; extend it rather than deriving a competing answer.
- transport_profile and runtime_profile may differ; source copies use session_identity.replace_source.
- _intake_adapter_for identifies the RECEIVING live transport and gates intake policy; absent provenance fails closed.
- _delivery_adapter_for answers through the receiving bot, else the unique (platform,runtime_profile) owner.
- Never resolve a source with self.adapters[platform] or a third resolver.
- Shared/routed events reply through the receiving bot even when the routed profile owns another bot.
- Restored sources have no intake adapter; disconnected secondary delivery returns None, never the default bot.
- SessionEntry.transport_profile persists the receiving bot separately from runtime key namespace.
- Restore through _restored_source/restore_identity with transport=None; keep pre-column shared-bot fallback only for old rows.
- Deferred callbacks capture identity/home; _run_in_executor_with_context preserves scope across thread hops.
- Relay outbound frames echo routed profile; follow_up derives it from key namespace for passthrough forwarding.
- Ambient active-profile reads are boot-only; identity-bearing paths use identity.runtime_profile.

## Profile scope
- Credential-owning adapters acquire_scoped_lock on connect/start and release on disconnect/stop (gateway.status).
- Example: `plugins/platforms/irc/adapter.py`; two profiles must not share one credential connection.
- Bind home, secret and terminal scopes for EVERY profile activity, not only turns.
- Multiplex scoped misses use the provided default, never launch-profile os.environ.
- Adapters import platforms._shared.get_scoped_secret; authz uses platform_gate_env.
- extra_or_secret, seed_extra_from_env and env_is_connected build on the same reader; never clone the fallback logic.
- Only genuine UnscopedSecretError/default single-profile paths may read process environment.
- Scope release/TTL/LRU/memory-pressure eviction, shutdown, media delivery, callbacks, notifiers and webhooks.
- Resolve ownership from session profile_home/key; never from live os.environ.
- api_server_memory_sessions.py exclusively checks MemoryManager out/in across requests by home + session ID.
- Adopt it without reinitializing providers; idle/LRU eviction shuts it down under its owning home.
- multiplex_profiles:false still needs scope after native hosting activates a secondary.
- Standalone paths use _standalone_launch_scope: .env over frozen activation environment, never .env-only/live environ.
- Hooks/shell hooks/observers register per served profile; idempotence keys include profile and hook paths resolve at call time.
- apply_yaml_bridge seeds PlatformConfig.extra and avoids secondary environ writes; gate reads stay scope-aware.
- Test two profiles with conflicting flags and secondary-only secrets; no launch-profile credential/allowlist leakage.
- Shared ingress enabled on unsupported secondary profiles warns once with remedy and runtime status.
- WhatsApp owns per-profile adapter/port; report unpaired state and require pid+start time+port before adopting/signalling bridges.
- _update_platform_runtime_status is the sole needs_attention writer on connect paths.
- Routing compares against hermes_constants.get_routing_process_hermes_home(), never live HERMES_HOME.
- Embedding hosts pin process home; get_process_hermes_home remains for process-level assets, not new routing decisions.

Validation:
- Run `scripts/run_tests.sh tests/gateway/`; test actual adapter delivery, session keys and guard behavior.
- Include test_multiplex_transport_matrix.py and test_shared_platform_boilerplate.py for profile changes.
- Avoid frozen platform/command counts; keep platform API quirks in connector comments and behavioral tests.
