# Scheduled jobs and Kanban

Architecture: `website/docs/developer-guide/cron-internals.md`; user docs: `website/docs/user-guide/features/cron.md` and `kanban.md`.
- `jobs.py` owns storage; scheduler.py and siblings execute ticks; cronjob tool, CLI cron and /cron are entry points.
- Schedules accept durations, every expressions, five-field cron and ISO one-shots.
- Jobs can select skills/model/provider, scripts, context_from, workdir and multiple delivery targets.
- A script is a FILE beneath HERMES_HOME/scripts, never a shell command; no_agent makes it the entire job.
- Idle watchdog defaults to 600s (HERMES_CRON_TIMEOUT=0 disables it); it is not total wall-clock time.
- Script execution has a separate 3600s timeout.

Occurrences and ownership:
- Catch-up is half the period bounded to 120s–2h; one-shots use 120s.
- Advance next_run_at and persist pending_slot atomically before dispatch.
- occurrences.py restores dead-owner pending slots once; scheduled_instant ledger deduplicates occurrences.
- catch_up_missed:false records skips with reason; do not silently discard occurrences.
- Locks and state resolve per home, never literal ~/.hermes.
- A headless external tick without a gateway refuses agent jobs without spawning; stamp failure/warning and leave due unchanged.
- In-process/manual runs are not that headless path.
- One sequential ticker visits profiles; scope the WHOLE tick (home, secrets, terminal, locks, counters and env flush).
- Do not create per-profile ticker threads over module globals.
- Supervision re-enumerates profile directories/tombstones and stands down for another gateway owner.
- Tick all profiles regardless of the multiplex-adapter flag.
- Inflight identity is (home,jobid), with per-home pools; host shutdown accessors can report the union.
- Liveness checks pass home to is_job_running; release claims using the original home path, not a reconstructed normcase key.
- Reclaim pools when a home leaves; another-owner gating compares PID with current PID, not bare liveness.

Worker boot and sessions:
- External worker PYTHONPATH includes this checkout and the committed package-manager generation/boot marker.
- worker_bootstrap activates generation, lease and .pth before the first third-party import; failure prevents acknowledgement.
- finish_worker_boot performs Hermes bootstrap/recovery before payload execution.
- Source-update -I re-exec repeats the complete bootstrap; do not rerun activation in the live gateway.
- Scheduled agents use skip_memory=True and separate execution sessions.
- Mirror only eligible continuable deliveries: origin, originless home, bare-platform and opted-in explicit targets.
- Expanded all-target delivery does not gain home eligibility.
- Mirroring uses a labeled user message at a turn boundary and preserves role semantics.
- Desktop-spawned backends still tick; HERMES_DESKTOP does not grant GUI capabilities.
- Process-local delegation is not durable scheduling; use cron or background terminal completion notifications.

Kanban:
- SQLite board CLI facade delegates to its sibling modules; it owns its connection, never a projects-DB alias.
- Dedicated worker toolsets exclude Kanban tools outside worker context unless the platform explicitly enables them.
- Use selected-toolset context, not the legacy top-level toolset field.
- Gateway dispatcher defaults to 60s, configurable; standalone plugin service uses the same ownership rules.
- Claims are atomic, assigned-profile aware and stale claims return to ready.
- The board is a hard boundary pinned in environment; tenant paths/memory are soft namespaces within it.
- Default failure limit is two before auto-block.
- Terminal provider exit 78 is a credential/model error: nonretryable and sticky on first failure.
- Never identify a worker by a command substring such as serve; require PID + start-time fingerprint before signalling.
- Notifications use the owning profile's adapter scope and fail closed with actionable warnings.
- Workers pin board and assignee home; scrub the child environment and remove launch-profile secrets.
- Prompt injection requires owned_kanban_task()/is_dispatcher_owned_worker_context, not tool availability or inherited env alone.
- Delegated children retain the pinned board ROOT fence.
- Deny access to that pinned DB/root only; unrelated scratch-home boards remain writable.

Validation:
- Run `scripts/run_tests.sh tests/cron/` plus relevant tests/hermes_cli/test_kanban*.py and tests/tools/test_kanban*.py.
- Test pure data and synchronization behavior; timing assertions allow at least two seconds.
- Assert scheduling/ownership invariants, not the number of CLI verbs.
