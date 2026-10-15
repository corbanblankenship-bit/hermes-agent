# Desktop engineering

Architecture and detailed rationale: `ENGINEERING.md`; interaction contract: `DESIGN.md`.
Backend, slash and Bot Mode invariants are in `src/AGENTS.md`.

Authority and boundaries:
- Electron owns native machine capabilities, process lifecycle, filesystem/git, windows and installation/update.
- The renderer owns presentation, navigation and ephemeral interaction state.
- The backend owns shared sessions, tools, models and streaming; renderer copies are caches.
- Renderers never import Node/Electron directly or gain a generic native escape hatch.
- Expose narrow typed capabilities; keep agent behavior behind the gateway.
- Put shared feature state in small owned stores, invalidatable request data in the query layer,
  transient interaction state in components and nonpainting coordination in refs.
- Persisted keys declare their global/connection/profile/session/project/window scope.

Identity and consistency:
- Distinguish durable, runtime and lineage-root session IDs; translate at the boundary.
- `prompt.submit` persists the session/user rows before a turn adopts that durable message.
- New profile-keyed localStorage families join BOTH migrateTilesForProfile and dropTilesForProfile.
- A verifiably gone session (goneSessionVerdict → draft) becomes a fresh draft and adopts unsent text once.
- Do not toast, navigate or steal focus during that recovery.
- Merge server state; optimistic changes roll back visibly on failure.
- Generation/request tokens prevent stale results overwriting newer intent.
- Only the foreground surface publishes the shared view; coalesce cosmetics but flush terminal transitions.
- Preserve reference identity on no-ops.
- Connection/mode changes are soft re-homes: retain shell, explicitly clear gateway-bound stores, reconnect.
- Runtime HERMES_HOME changes are hard re-homes requiring reload.
- Live profile swaps merge lists while background profiles stream; socket/profile/connection atoms must agree.

Resolution and compatibility:
- Each policy has one ordered resolution ladder with data-driven precedence and probed candidates.
- Failed reads may advance to the next candidate; failed authoritative writes surface or roll back.
- Missing capabilities are not transient failures; retries are bounded with a recovery affordance.
- OAuth gateway connections mint a fresh ticket each dial; only confirmed 401/403 prompts reauthentication.
- Connection tests exercise the actual connection leg.
- `electron/oauth-partition.ts` partition names avoid Electron percent-escaping hazards.
- Desktop and runtime versions can differ: keep compatibility narrow, tied to a known runtime and tested.
- Preserve the current feature rather than silently dropping options or capabilities.

Untrusted content and lifecycle:
- Artifact iframes and preview webviews cannot open the OS browser autonomously.
- setWindowOpenHandler denies guest opens; no webview allowpopups.
- Only the persist:hermes-preview guest preload's trusted-click bridge may call hermes:openExternal.
- Only http/https are allowed; retain partition, trusted-click and scheme gates (GHSA-9f4c-93c8-jc8g).
- Agent-callable renderer tools are capabilities of the session's client (`source: desktop` on session.create).
- Never infer that capability from backend process environment; the gateway can be remote or multiplexed.
- Use existing seams and small feature-local extensions; internal registries are not a public plugin ABI.
- Share contracts after multiple real consumers establish their shape; plugin models are not interchangeable.

Interaction and performance:
- Background events must not navigate, move focus or open surfaces; offer a user action instead.
- Distinguish empty, loading, reconnecting, stale/degraded and exhausted recovery states with useful actions.
- Focus owns keyboard input; one cancel gesture performs one cancellation.
- Stateful terminals/live tools stay alive while hidden; visibility is not lifecycle.
- Keep drag/resize/typing/scroll/stream hot paths local and narrowly subscribed.
- Coalesce pointer work, avoid layout read-after-write and expensive mounts during gestures.
- Test realistic long transcripts; do not use motion to mask latency.

Free tier:
- free_tier.status/free_tier.ack_notice are authoritative; ready screen and own-key strip share notice_pending.
- Never latch notice state in localStorage; every entry point opens the single sign-in dialog.
- Branch on free_tier_row/free_tier, never provider display names.

Validation:
- Use the actual package scripts; test behavior and invariants rather than frozen lists/counts.
- Cover resolver precedence/failure, identity/scope, optimistic rollback and stale response ordering.
- Exercise both local and remote adapters with profile routing intact.
- Verify usable failure recovery, realistic interaction performance, DESIGN checklist and all locales.
