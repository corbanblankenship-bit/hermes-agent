# TUI gateway

Python owns sessions, tools, models and slash execution; Ink, Desktop and dashboard render them.
Transport and contracts:
- Stdio uses newline JSON-RPC. Register methods in `methods_*.py`; publish events through the replay publisher.
- `server_requests.send` waits for a response; `request.cancel` cancels work. Resume/events.since replay open requests.
- Declare `client.capabilities {server_requests: true}` on gateway.ready; unsupported WebSocket clients fail promptly.
- Desktop uses `apps/shared` JsonRpcGatewayClient/onRequest; Ink uses its server-request handler.
- Every method/request has Params and Result contracts; every event has a Payload contract.
- Default to extra-forbid; use OpenModel only for intentional producer dictionaries.
- Registration rejects undeclared contracts; bad params return 4000. Invalid outputs/events fail tests and log in production.
- Regenerate with `scripts/gen_gateway_contracts.py`: `apps/shared/src/gateway-contract.generated.ts` and `gateway-contract.openrpc.json`.
- `gateway-events.ts` adds the client-local envelope. Keep generated-file freshness tests.
- Native `hermes-gateway-v1` has separate `contracts/canonical*.py` declarations.
- AuthorityConnection validates canonical parameters/results (4001); reuse models only for identical wire shapes.
- Generated CanonicalRpcMethods/x-canonical-methods and handler coverage must agree.
- A new question needs its request contract, emitter, Desktop request routing and Ink handler; new events need contracts.

## Profile scope
- `default` means the launch home, never a literal `~/.hermes` fallback.
- The first hosted secondary activates multiplex guards through `launch_profile_policy.py`.
- Scope handlers with `_profile_scoped` and tool construction with `_profile_scoped_rpc`.
- Bind home, secret and terminal scopes together through model_switch runtime tokens; never change home alone.
- Scope finalize/teardown hooks, memory and close operations to the owning profile.
- Use `spawn_context_thread` for background work and `served_profile_child_env` for children, not raw os.environ.
- Test two homes, a secondary-only secret and unchanged process environment.

Session and tool behavior:
- Prompt submission, tools, approvals, sessions and slash execution stay behind the backend.
- `complete.path` uses the session's execution backend, never the UI host filesystem.
- Connection operations use the asynchronous, nonblocking `tools/connectors/live.py` path.
- Subagent lists are read-only conversation lineage, including the compression tip.
- Control requires the exact live session, transport and generation at the owning live transport slot.
- Foreign or retired agents are uncontrollable; `last_tool` does not prove an inflight call.
- Keep the empty legacy delegations shape; never expose dispatch/results/callback/routing secrets.
- Selected subagent detail reads at most 16 KiB from its existing transcript; accept no caller-supplied path.
- Missing, finished or foreign detail is unavailable; invalid authority returns 4001.
- Steering acceptance means queued, not delivered; preserve missed-steer race handling.
- Interrupt requires the same exact authority, including descendant control.

Slash and canonical discovery:
- Client-owned builtins stay local; otherwise `slash.exec` uses SlashWorker, then `command.dispatch` for skill aliases.
- Catalogs combine builtins, quick commands, plugins and skills; completion follows `hermes_cli/commands.py`.
- Discovery builders live in `command_discovery.py`; catalog visibility does not authorize execution.
- `gateway/session_discovery.py` requires authenticated session:read and the exact profile.
- An optional named profile must match; return profile_mismatch for a foreign profile.
- Resolve discovery off-loop in scope; preserve response shapes without creating legacy runtimes/sessions.
- Canonical `session_commands` dispatch uses registered actor-local sources, never legacy workers.
- Reject caller-supplied routing credentials or platform context; do not add a legacy fallback.
- Reviewed local reads: help, commands, status, context, version, whoami; title remains session control, even for queries.
- Preserve command aliases and busy policy. Skills require submit and return a directive only.
- Normal durable admission owns skill execution, queueing, retries and approvals; discovery cannot mutate prompt caches.
- Quick/plugin commands require session control and `run_inbound_hooks` slash access policy.
- Enforce the 30-second bound, sanitized environment and redacted output.
- Unsupported bundles, configuration mutation, approval shortcuts and lifecycle commands remain unsupported.

Canonical configuration:
- `session_config` exposes display preferences without credentials; model options use shared inventory.
- Model/reasoning resolve from the agent or frozen policy; preserve the pinned mcp_rev.
- Cosmetic changes do not reload runtime state; busy overrides stay session-local.
- Composer changes use session.mutate, not global config.set; discovery must never invoke inference.

Validation and pointers:
- Python: `scripts/run_tests.sh tests/tui_gateway/`; exercise the real registered contracts and profile boundaries.
- Ink: `ui-tui/package.json` defines dev/start/build/typecheck/lint/fmt/test commands.
- Keep JS source assertions in Vitest so JS-only CI executes them.
- Frontend contracts: `apps/shared/src`; native Desktop: `apps/desktop`; web frontend: `web`.
