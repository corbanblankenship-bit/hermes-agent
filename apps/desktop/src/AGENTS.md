# Desktop renderer contracts

Architecture: Electron + React + nanostores/@assistant-ui/react over the existing @hermes/shared JSON-RPC client.
See `../AGENTS.md`, `../DESIGN.md` and `../ENGINEERING.md` for application boundaries.

Canonical gateway ownership:
- Local primary and pooled launches run `hermes gateway ensure --json` and consume its credential-free endpoint.
- Authenticate WebSockets with fresh private control tickets; the gateway PID never becomes a Desktop-owned child.
- Closing, switching or evicting connections disconnects the viewer; explicit gateway lifecycle commands stop owners.
- Tickets cross private preload IPC transiently, are removed from URLs before socket construction and use subprotocols.
- Electron removes Origin only for an unexpired, one-use dial bound to the requesting window.
- Never scrape dashboard tokens or add credentials to public connection descriptors.
- SSH/URL intent keeps remote resolution/exposure; remote failures never start a local owner.
- One multiplexing owner serves each home; do not spawn per-profile `hermes serve --port 0` children.
- HERMES_DESKTOP is not gateway discovery. Secondary homes use GatewayEndpoint.multiplex_home/control sockets.
- Local, SSH, URL and Cloud backends can serve multiple profiles.
- Lifecycle/status/settings REST calls carry profile; every new-session tile records its owner route.
- Test profile scoping as both the launch home and a secondary served by the same process.
- Require canonical GUI creation policy and private HTTP credentials from the runtime.
- Never bypass missing capabilities by claiming CLI origin, dropping options or creating an independent serve owner.

Slash commands:
- Backend commands.catalog and complete.slash already include builtins, quick commands and skills.
- `lib/desktop-slash-commands.ts` owns DESKTOP_COMMAND_SPECS and client curation.
- Define each builtin's desktop disposition once on CommandDef in `hermes_cli/commands.py`.
- Live catalogs carry that disposition; `lib/desktop-slash-registry.json` is the generated offline fallback.
- Regenerate with `scripts/dump_desktop_slash_registry.py`; Python and Vitest drift tests must agree.
- TS_ONLY_NO_DESKTOP_SURFACE contains only commands unknown to Python (Ink-local commands and pets).
- isDesktopSlashCommand gates execution, admitting builtins and nonbuiltin extension commands.
- isDesktopSlashSuggestion gates BOTH composer completion paths and filterDesktopCommandsCatalog.
- Preserve isDesktopSlashExtensionCommand in suggestions and catalogs: curation never hides activated extensions.
- `app/session/hooks/use-prompt-actions/slash.ts` runSlash handles Desktop-owned builtins locally/catalog-based.
- Other commands use slash.exec then command.dispatch; skill results submit their message as a normal prompt.
- From apps/desktop: `npx vitest run src/lib/desktop-slash-commands.test.ts` (dependencies install at repo root).

Bot Mode (`plugins/hermes-bots`):
- One bot is one profile with one canonical forever-chat identified by (profile, exact title "Bot Chat").
- The state DB's unique title registry, not a saved session ID, is canonical identity.
- On every bot click, query session.list with title and include_hidden:true; resolve compression to the live tip.
- Existing row: open it. Missing row: re-query before minting, then create titled, hidden, with the bot intro.
- Adopt on creation conflicts; set_session_title can return zero for duplicate titles and must not cause forks.
- Ignore/drop legacy ui_meta chat pointers; never use stored IDs as fallback identity or verification.
- Recency, visibility and last_session never select the bot row's chat.
- Canonical Bot Chats stay hidden from Sessions; the bot row is their entry point.
- Side-chats stay visible, have no plumbing title and never become the bot row target.
- Do not restore a per-bot session browser; fix sidebar visibility through its hide-sweep.
- profiles.list canonical_session is resolved by title on the backend.
- Roster preview, activity, click identity and /new→/compact guard all use that same registry row.
- Tests in plugins/hermes-bots: canonical-chat-registry, canonical-chat-creation,
  canonical-chat-adopt-on-conflict, bot-row-opens-canonical-chat and hide-bot-chats.
- Python contract: `tests/tui_gateway/test_profiles_list_canonical_session.py`.

Free tier (`store/free-tier*.ts`, Billing, statusbar and ready screen):
- $freeTierStatus mirrors free_tier.status, refreshed with status snapshots and after sign-in.
- deriveBillingView checks billing.free_tier_account before logged_in.
- The free_tier view has a notice, one Sign in and Plan/Model/Connectors summary, no payment/usage rows.
- Logged-out Sign in opens the same dialog, never a portal link that cannot save credentials.
- The sign-in dialog has one claimed owner (first mount); states match poll status/reason exactly.
- Preserve approved copy: do not call users guest/anonymous or use claim/Nous Portal wording.
