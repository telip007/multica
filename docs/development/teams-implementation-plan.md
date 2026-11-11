# Teams implementation plan

Status: implementation handoff, no gateway code implemented.

**Goal:** Add a reusable Microsoft Teams text gateway using Multica's existing
channel engine.

**Architecture:** A native Go Activity/Connector REST adapter authenticates
Microsoft requests and normalizes them for existing routing. Task-bound response
delivery sends results to the original conversation. Shared TypeScript UI owns
setup and account binding.

**Specification:** [Microsoft Teams](../integrations/microsoft-teams.md).
**Source baseline:** `6ccbdd178ac7dcaf970acfc9a3836be2ca864c49`.

## Constraints

- Follow `AGENTS.md`, `CONTRIBUTING.md` and the specification; do not introduce a
  second orchestrator or a dependency on a particular model provider.
- Use Go 1.27.4 to match current CI. `go.mod` and CONTRIBUTING currently still
  declare 1.26.9; do not silently lower CI or rewrite unrelated toolchain files.
  Use Node 22 from `.nvmrc`, pnpm 10.28.2 and the managed Docker-backed environment.
- Azure Public Cloud, tenant-bound BYO bot per installation, final text only.
  Limits: 1 MiB incoming body; less than 24 KiB per outgoing serialized Activity;
  binding token TTL 15 minutes. Preserve case-sensitive provider identifiers.
- Keep credentials, real tenant/mailbox values, local paths, user content and
  raw provider transcripts out of repository files, fixtures and screenshots.
- Standard tests use local fixtures and HTTP stubs, never real agent CLIs,
  accounts, Teams or Graph. Provisioning and real-agent smokes are separate work.
- No Outlook implementation in this plan. Its proposal is separately documented.
- Match current agent-page channel UI. The old agent Integrations tab was removed.

## Review focus

These cases belong to the owning task's tests, rather than a final manual review:

1. A valid Microsoft token for another bot/tenant must not authorize this route
   or send a token to an attacker-controlled service URL (tasks 1 and 3).
2. Binding links in shared conversations can bind the wrong person; private
   onboarding, one-time redemption and membership checks are mandatory (task 4).
3. A `/new` or `/clear` during a run must not alter that run's response target or
   expose old context in the next generation (tasks 4 and 5).
4. A retry after an ambiguous outbound timeout can duplicate an accepted answer;
   persist uncertainty and do not blindly retry (task 5).
5. Older desktop clients, unavailable integrations and failed queries must not
   appear as successful installation or silently disconnected state (task 6).

## File map

| Responsibility | Files |
| --- | --- |
| Protocol, verification, REST client | New `server/internal/integrations/teams/{activity,auth,client}.go` with adjacent tests |
| Configuration and installation | New `teams/{config,install}.go`; existing `server/internal/util/secretbox` and shared channel queries |
| Transport adapter and resolver ports | New `teams/{webhook,channel,resolvers,binding,replier}.go`; existing `channel`/`engine` contracts |
| Task-bound responses | New `teams/{outbound,sender,delivery}.go`; existing generic delivery tables/queries |
| HTTP and startup | New `server/internal/handler/teams.go`; `handler.go`, `server/cmd/server/{router,main}.go`, `server/pkg/protocol/events.go` |
| Origin and context | New migration pair(s); `server/internal/daemon/execenv/channel_type.go`, issue-origin classification where applicable |
| Client types and API | New `packages/core/types/teams.ts`, `packages/core/teams/{index,queries}.ts`; `types/index.ts`, `api/{client,schemas}.ts`, realtime sync, package exports |
| Shared setup UI | `packages/views/settings/components/{settings-navigation,use-messaging-channels}.ts`, channel connect/disconnect dialogs, icon mapping; `packages/views/agents/components/agent-channels-section.tsx` |
| Account binding | New `packages/views/teams/{index,bind-page}.tsx` (use `.ts` for a barrel without JSX), `apps/web/app/teams/bind/page.tsx`; reserve/generate slug |
| Copy and operator documentation | Existing view locales, public channel docs, env/deployment examples and new Teams setup guide |

Names above define ownership, not a requirement to fill unused files. Keep
existing patterns and avoid generic refactors of other adapters.

## Development baseline

- [ ] Inspect local changes and the current branch. Use the assigned isolated
  checkout; do not overwrite a user's working changes.
- [ ] With an implementation assignment, prepare the toolchain versions above
  and run the repository-managed setup in `CONTRIBUTING.md`. Start API/web only
  using `make up C=api,web`; do not start a real agent daemon for baseline tests.
- [ ] Run `make test`, `pnpm typecheck`, `pnpm lint` and `pnpm test` once before
  feature changes. Report pre-existing failures separately. Pure protocol unit
  tests below may run without a database; DB-backed tests use this managed
  environment and the repository fixtures, not a production database.

## Task 1: Verify the Microsoft protocol boundary

**Files:** New `teams/activity.go`, `auth.go`, `client.go` and adjacent tests.
**Consumes:** Official Activity, Connector OAuth and validation requirements.
**Produces:** `TypeTeams channel.Type = "teams"`; provider `Activity` and `Claims`
types; `Verifier.Verify(ctx context.Context, token, appID string) (Claims, error)`;
`ValidateActivity(a Activity, c Claims, cfg PublicConfig) (VerifiedActivity, error)`;
`Client.Token(ctx context.Context, cfg Credentials) (string, error)`.
`PublicConfig` owns AppID, TenantID, DisplayName; `Credentials` embeds it plus
ClientSecret. `VerifiedActivity` wraps a validated Activity; its identity cannot
be constructed from an unchecked HTTP body by downstream public APIs.

- [ ] Write `TestVerifyConnectorToken` with table cases: correct RS256 signature;
  wrong audience/issuer/algorithm; expired and not-yet-valid tokens; unknown key;
  missing required Teams endorsement; successful JWK refresh after rotation.
  Sign tokens using test-only keys and serve metadata/JWKS with `httptest`.
- [ ] Write `TestValidateActivityBoundary`: wrong/conflicting tenant, wrong bot,
  missing sender object ID, wrong channel, HTTP/untrusted/redirected service URL,
  and service-URL claim mismatch all fail before any side effect.
- [ ] From `server/`, run
  `go test ./internal/integrations/teams -run 'TestVerifyConnectorToken|TestValidateActivityBoundary' -count=1`
  and confirm the new tests fail because the boundary is not implemented.
- [ ] Implement the boundary against fixed production metadata; only private
  test constructors can override endpoints. Use the existing JWT dependency and
  a maintained JWK helper only if necessary. Do not fetch untrusted `jku` URLs.
- [ ] Add `TestConnectorTokenCache`: per-installation credential cache,
  expiry/refresh, revocation invalidation, no cross-tenant reuse; no token in
  errors. Use the tenant-specific OAuth endpoint and documented Connector scope.
- [ ] Run those tests and `go test -race ./internal/integrations/teams`.
  Record the official endpoints/claim rules selected in code comments where
  non-obvious. If the wire protocol cannot be validated, stop expansion and
  document the actual blocker rather than bypassing verification.
- [ ] Commit `feat(teams): validate Connector activities and credentials`.

## Task 2: Store tenant-bound bot installations

**Files:** New `teams/{config,install}.go`, handler installation methods and tests;
shared DB queries only where existing ones cannot express the operation.
**Consumes:** PublicConfig, Credentials and token client from task 1; shared
secretbox and transactional installation-ownership queries.
**Produces:** `InstallService.Register(ctx context.Context, p RegisterParams)
(db.ChannelInstallation, error)`, `ListByWorkspace`, `Revoke`; RegisterParams
contains WorkspaceID, AgentID, InitiatorID, AppID, TenantID, ClientSecret,
DisplayName. Add handler `ListTeamsInstallations`, `RegisterTeamsBot` and
`RevokeTeamsInstallation` matching the specification routes.

- [ ] Write `TestRegisterTeamsBot`: valid install saves encrypted credentials;
  invalid credentials do not write; missing secretbox key disables installation;
  cross-workspace app ownership conflicts; same-owner credential replacement
  preserves bindings and invalidates cache. Include archived-owner behavior
  following the existing adapter's explicit reclaim rules.
- [ ] Write `TestTeamsInstallationPermissions`: members may list but cannot
  install/revoke; wrong-workspace agent IDs and installation IDs cannot mutate
  anything; responses never include encrypted or plaintext client secrets.
- [ ] Run the narrow tests, observe failure, implement installation/service and
  HTTP boundaries, then rerun them. Follow the current UUID loader rules.
- [ ] Return stable public installation metadata plus configuration flags;
  credential validation is not an inbound reachability success signal.
- [ ] If SQL changes are needed, add narrow queries, use current migration
  numbering and index rules, and regenerate using `make sqlc`.
- [ ] Commit `feat(teams): manage encrypted bot installations`.

## Task 3: Receive and normalize webhook messages

**Files:** New `teams/{webhook,channel}.go`, webhook HTTP wiring and tests.
**Consumes:** Verifier/VerifiedActivity, installation lookup and
`channel.InboundHandler`.
**Produces:** `NormalizeActivity(a VerifiedActivity) (channel.InboundMessage,
bool, error)` where `false` means an ignored event;
`Webhook.ServeHTTP(http.ResponseWriter, *http.Request)`;
`Register(reg *channel.Registry, deps ChannelDeps)` with a passive
`channel.Channel` implementation. ChannelDeps supplies the store, verifier,
client and inbound handler; it must not obtain them through global state.

- [ ] Write `TestTeamsWebhook` using a recording fake handler: authenticated text
  invokes it once; unsupported events, self/bot messages and edits never do;
  oversized JSON returns 413; invalid identities never invoke or persist input.
- [ ] Write `TestTeamsNormalization`: bot-mention stripping, preservation of
  other mentions, immutable Activity IDs and tenant/object sender identity;
  Unicode, empty messages and personal/group/channel types are explicit cases.
- [ ] Observe failures; implement HTTP validation, normalization and registration.
  Missing installation is not an invitation to auto-create one.
- [ ] Add `TestTeamsWebhookAcknowledgement`: success does not await a model;
  engine infrastructure failure returns a retryable failure; a nil engine
  classification result succeeds even when the product outcome is a drop.
  Repeat delivery is delegated to shared dedup, not a process-local cache.
- [ ] Verify token-bearing HTTP clients refuse redirects/untrusted hosts on
  actual send paths, not just in a standalone URL helper.
- [ ] Run `go test -race ./internal/integrations/teams` and handler webhook tests.
- [ ] Commit `feat(teams): route verified webhook input to channel engine`.

## Task 4: Bind accounts, isolate conversations and support commands

**Files:** New `teams/{resolvers,binding,replier}.go`; origin migration(s),
daemon channel type/context; adjacent unit and DB tests.
**Consumes:** Normalized messages; shared installation, identity, dedup, audit,
session and permission ports. Use existing Slack/Telegram resolver patterns.
**Produces:** `NewTeamsResolverSet(q *db.Queries, tx engine.TxStarter,
replier engine.OutboundReplier, typing engine.TypingNotifier) engine.ResolverSet`;
`teamsSessionRouting(msg channel.InboundMessage) (string, []byte, string)`;
BindingTokenService with `Mint`/`RedeemAndBind` matching the existing generic
channel token-service contracts; issue origin `teams_chat`.

- [ ] Write `TestTeamsSessionRouting`: same DM continues; separate conversations,
  installations, tenants and channel roots isolate; opaque IDs containing
  delimiters cannot collide. Base fixtures on documented Teams coordinates.
- [ ] Write DB tests `TestTeamsBindingPrivacy` and `TestTeamsAuthorization`:
  unbound group/channel users get no token; personal binding is one-time and
  expires after 15 minutes; wrong installation, nonmember, removed member,
  archived agent and invocation-denied user cannot execute. The identity used
  privately must match the same user's group identity.
- [ ] Write `TestTeamsChannelCommands`: without a mention, group input is ignored;
  duplicate Activities create no second run/issue; `/new` rotates the route;
  `/clear` keeps Chat identity and hides prior context; `/issue` assigns the
  existing agent and persists `teams_chat`. Exercise actual engine wiring.
- [ ] Observe failure, implement resolver/binding/replier ports using generic
  tables and `engine.NewChatSession`, then rerun those tests.
- [ ] Add migration(s) extending the current issue-origin constraint and its
  validation/rollback behavior. Fetch current source before numbering. Do not
  introduce foreign keys or hand-edit generated DB Go files.
- [ ] Wire `teams` into daemon display/capability context, without file-delivery
  claims. From `server/`, run
  `go test ./cmd/migrate ./internal/migrations ./internal/daemon/execenv`
  plus the targeted DB tests in the managed environment.
- [ ] Commit `feat(teams): bind members and isolate channel conversations`.

## Task 5: Deliver final results to the originating conversation

**Files:** New `teams/{sender,outbound,delivery}.go`; startup/shutdown wiring,
protocol constants if required; adjacent tests.
**Consumes:** `channel.OutboundMessage`, Connector credentials, original task's
`channel_task_delivery`, generic reply-delivery ownership and terminal events.
**Produces:** `Sender.Send(ctx context.Context, out channel.OutboundMessage)
(channel.SendResult, error)`; Outbound subscriber `Register(bus *events.Bus)`
and bounded worker lifecycle following current adapter patterns.

- [ ] Write `TestTeamsReplyTarget`: completion goes to the original bot/thread;
  a later `/new`, `/clear` or direct web chat cannot retarget/leak the answer;
  revoked or deleted installations cannot send.
- [ ] Write `TestTeamsTerminalDelivery`: success and terminal failure are posted
  once per known delivery; concurrent duplicate callbacks share ownership;
  chunks retain order and Unicode and stay below 24 KiB serialized size.
- [ ] Write `TestTeamsSendOutcomes`: explicit 429 respects Retry-After;
  pre-send authentication rejection can refresh safely; timeout after an
  uncertain send records uncertainty without automatic duplicate sending.
- [ ] Observe failures; implement final-only delivery, bounded HTTP calls and
  shutdown. Do not copy the whole Telegram streaming implementation or claim
  restart-safe event recovery from process-local subscriptions.
- [ ] Run targeted tests with `-race`; include a concurrent duplicate completion
  fixture and a revoked-installation fixture. Document one-backend support and
  the precise interrupted-delivery behavior in the operator guide.
- [ ] Commit `feat(teams): deliver task-bound final replies`.

## Task 6: Add shared setup, binding and app-package download

**Files:** New core Teams types/hooks, API schemas/client, realtime handling,
shared channel connect/disconnect and agent channel components; bind page/route;
server package-generation endpoint; locales and icon implementation.
**Consumes:** Task 2 installation routes and task 4 binding service.
**Produces:** TeamsInstallation, ListTeamsInstallationsResponse,
RegisterTeamsRequest and RedeemTeamsBindingTokenResponse, plus API methods
`listTeamsInstallations`, `registerTeamsBot`, `deleteTeamsInstallation`,
`redeemTeamsBindingToken` and `downloadTeamsAppPackage`; `teamsKeys` and
`teamsInstallationsOptions(wsId)` matching existing adapters.

- [ ] Write core schema tests for unknown/absent optional fields, invalid payloads
  and safe fallbacks. Add workspace-keyed query/realtime invalidation tests.
- [ ] Write view tests for connect from an agent page and Settings, wrong role,
  credential errors, pending/failed list queries, disconnect, reconnect and
  binding success/expired token. Require ordinary shared navigation adapters.
- [ ] Observe failures; extend the current generic dialogs and
  `useMessagingChannels`, `ChannelIntegration`, `ChannelConnection` and icon map;
  do not recreate per-provider tabs. Keep business state in core and UI in views.
- [ ] Add `/teams/bind`, reserve its slug and run
  `pnpm generate:reserved-slugs`. A normal web bind page completes the browser
  authentication flow even when the originating user was in Desktop.
- [ ] Write `TestTeamsAppPackage`: ZIP contains a valid manifest and required
  icons; bot ID/scopes/public operator URLs match configuration; credentials
  never appear in any file. Update schemas/routes/errors for the admin endpoint.
- [ ] Add a small provider-specific package metadata input if needed; website,
  privacy and terms URLs must be public operator values, never private defaults.
  Validate against the current official manifest schema and document its version.
- [ ] Add translation keys across supported view locales, following repository
  naming/icon rules. Run narrow suites, then `pnpm typecheck`, `pnpm lint`,
  `pnpm test` and relevant web/desktop Playwright flows.
- [ ] Commit `feat(teams): add channel setup and app package`.

## Task 7: Verify, document and prepare deployment

**Files:** User-facing Teams setup guide, channel overview, env/deployment
examples and tests when those files have corresponding guards.
**Consumes:** All previous tasks. **Produces:** An independently usable adapter,
operator instructions and a truthful verification report.

- [ ] Run `git diff --check`, `make test`, `pnpm typecheck`, `pnpm lint`,
  `pnpm test`, relevant Playwright flows and migration checks. Run
  `pnpm test:scripts` only if scripts/CI changed. Record commands and outcomes;
  distinguish skipped live checks from passed simulated ones.
- [ ] Document Azure Bot registration/channel, tenant/app/secret configuration,
  current Teams custom-app policy, manifest upload, binding, secret rotation,
  revocation, a development HTTPS tunnel, one-server deployment and rollback.
  Use placeholders and example domains; store credentials only at runtime.
- [ ] On a separately authorized live task, create/use a test bot, sideload its
  package and check personal chat, group mention and two channel threads. Use
  deterministic fake runtime first. Run a real coding-agent smoke only with
  the repository's explicit `agentintegration` authorization and gate.
- [ ] Capture UI evidence with synthetic content. Check the public diff for
  secrets/private metadata, not just `.env` files. Do not publish raw tokens,
  mailbox content, organization identifiers or local profile paths.
- [ ] Commit `docs(teams): document setup and verified support boundary`.

## Completion and continuation

Completion means the specification's acceptance matrix is demonstrated, not
merely that a bot answers once. A planning-only branch does not implement Teams.
Deploy this fork's backend after implementation; official hosted Multica and
upstream release assets remain unchanged. Outlook should be implemented as a
separate feature after its intake/authentication policy has been finalized.
