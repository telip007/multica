# Outlook mail gateway proposal

Status: researched proposal, separate from the Teams implementation plan. No
Graph connection, mailbox synchronization or automatic mail sending exists yet.

## Product boundary

An Outlook gateway is useful alongside Teams, but mail and chat have different
interaction models. Start with a dedicated Microsoft 365/Exchange Online user
mailbox as an issue-intake source. Keep Teams as the interactive conversation
surface. This is a Microsoft Graph integration, not automation of the Outlook
desktop app and not a general IMAP/SMTP connector.

The product should work for any workspace and configured mailbox. Tenant,
mailbox, folder, agent, project, sender policy and polling interval are
installation settings. No organization-specific values belong in source code or
defaults. A deployment can have installations for different tenants/mailboxes.
Outlook.com, shared/delegated mailboxes and app-only authorization can be added
as explicit supported modes later; do not accidentally treat them as equivalent
to a signed-in user's own work mailbox.

## Recommended first scope

1. An administrator connects one dedicated work mailbox to a configured agent
   and selects its intake folder and destination project.
2. A deterministic worker checks that folder every 60 seconds using Graph delta
   state. The interval is configurable; throttling can delay a cycle.
3. New authorized messages produce a traceable Multica issue assigned to the
   configured agent. Subsequent authorized messages in the same conversation
   append to that issue. Mailbox/conversation/sender boundaries and permissions
   remain explicit.
4. Existing assignment/run mechanisms start agent work. The transport worker
   does not prompt a model to decide whether a message is new or authorized.
5. Source references and the issue link connect mail intake and Teams discussion.
   They do not silently merge email threads into an existing Teams Chat or copy
   an entire mailbox into agent context.

Use an intake service reusing existing issue/comment/permission APIs. The current
channel engine is a chat engine: forcing every message into a hidden Chat merely
to create an issue adds unnecessary session and reply semantics. A later fully
conversational mail adapter may reuse its ports, but should have its own design.

## Authorization and credential model

For the first mode, sign in as the owner of the dedicated mailbox through
tenant-bound OAuth authorization-code flow with PKCE, validated state and an
explicit mailbox object ID. Use delegated `User.Read`, `Mail.Read` and
`offline_access` as needed for identity, body reading and renewable access.
Validate the returned tenant/user against the installation, encrypt renewable
credential material using the existing secretbox pattern, and support reconnect
and revoke. Do not obtain tokens by scraping an existing personal MCP connector,
CLI cache, browser profile or another agent's credentials.

Do not add Mail.ReadWrite merely to mark messages as processed: processing state
belongs in the application's database. `Mail.ReadBasic` does not supply the full
message body required for intake. Keep permissions and operator instructions
specific to each supported account mode.

Shared mailbox support would need separately tested permissions. Graph's
delegated change-notification subscriptions only cover the signed-in user's own
mailbox; sharing scopes do not enable subscriptions for another mailbox.
App-only access is an alternative for unattended deployments, but must be scoped
to configured mailboxes through Exchange authorization rather than silently
requesting tenant-wide access. These modes are not a prerequisite for Teams.

## Durable intake semantics

- Use a per-installation, per-folder delta cursor. Follow opaque `nextLink` pages
  until `deltaLink` is reached; persist completion state and restart progress.
  Validate URLs as trusted Graph origins before sending credentials.
- Request immutable IDs consistently. A dedup key includes installation,
  mailbox and immutable message ID; message IDs are case-sensitive. Folder
  moves, read-status changes and replayed pages must not create duplicate work.
- Initial connection establishes a baseline without executing historical mail.
  Show the installation as active only after that baseline completes. Importing
  older mail requires an explicit bounded import action.
- Persist a processing ledger before advancing the cursor. Reserve a message
  atomically and correlate issue creation using a durable idempotency key so a
  crash between creating an issue and marking the message cannot duplicate it.
  Use existing service transactions where possible. A failed operation remains
  retryable; a cursor update must not discard unprocessed messages.
- Recover expired/invalid delta state through a bounded resynchronization using
  the retained ledger and activation boundary. Reconnect does not reset dedup or
  silently re-import the mailbox. Token revocation pauses intake visibly.
- Treat message deletion/moves and updates as synchronization facts, not new
  requests. A conversation key includes the mailbox, Graph conversation ID and
  authorized participant boundary. Never correlate solely by subject line.
- Fetch only needed fields/body. Prefer plain text; normalize HTML safely,
  ignore remote images, treat quoted text/signatures as source material and do
  not execute embedded instructions outside normal agent task policy.
- V1 excludes attachment ingestion, calendar events, address-book import and
  automatic mailbox writes. Skip auto-replies, bounces and messages produced by
  the integration itself to avoid loops.

## Sender trust is a separate policy

Reading a message through Graph proves the mailbox source, not who authored an
arbitrary `From` header. A sender allowlist alone is not strong authentication.

The implementation must define and test an operator-configurable trust boundary
before automatic issue execution is enabled. An initial organization can route
authenticated allowed senders into a dedicated intake folder using Exchange mail
flow rules, with an explicit mapping to existing Multica members. The application
must document that prerequisite and cannot present an ordinary Inbox folder or
an untrusted header as equivalent proof. External, forwarded, delegated or
ambiguous identities require review rather than automatic privileged execution.

Default to review-only intake until that source policy is configured and
verified. Unknown senders must not auto-join a workspace, inherit the mailbox
owner's rights or trigger an agent as the installer. For trusted automatic mode,
resolve an existing member, check invocation/project permissions, and preserve
that initiator on the issue. This decision is independent of content triage.

## Replies and future push delivery

Incoming intake does not imply automatic outbound mail. Start with issue intake
and Teams discussion. Add mail replies as a separately enabled capability with
validated recipient policy; `Reply-To` can differ from `From` and must not become
an unchecked route for generated content. Do not send reply-all automatically.

Microsoft Graph's direct `reply` operation requires Mail.Send. Creating a reply
draft first requires Mail.ReadWrite, with sending as a separate action. Therefore
draft mode is a distinct permission/configuration choice, not a free consequence
of Mail.Read plus Mail.Send. Use the same source conversation and persist
delivery/uncertainty state before attempting retries.

Push can be added through Graph subscriptions. Validate endpoint challenges and
notification state, renew subscriptions before expiry, handle lifecycle gaps,
and use delta reconciliation because notifications can be duplicated or missed.
Polling is the simpler first transport and remains useful for reconciliation;
it is not an agent heartbeat and costs no model tokens by itself.

## Separate implementation outline

After finalizing the trust/auth modes, create an Outlook implementation plan:

| Work package | Result to verify |
| --- | --- |
| Connection | Tenant/mailbox-bound OAuth, encrypted renewable tokens, reconnect/revoke |
| Synchronization | 60-second delta worker, pagination, immutable IDs, no baseline import, recovery |
| Trust and policy | Review-only default, authenticated intake prerequisite, existing-member mapping |
| Issue intake | Durable dedup/idempotency, issue/conversation correlation, permission-preserving initiator |
| UI and operations | Configurable agent/project/folder, clear paused/review status, operator documentation |
| Optional reply/push | Independent capability and permissions, threading and lifecycle tests |

Keep provider code under `server/internal/integrations/outlook`; use the current
shared UI/core boundaries. Do not implement OAuth, mail parsing or sender trust
inside a role prompt. No personal account or deployment values belong in fixtures.

## References

- [Delta synchronization of a mail folder](https://learn.microsoft.com/en-us/graph/delta-query-messages)
- [Immutable message IDs](https://learn.microsoft.com/en-us/graph/outlook-immutable-id)
- [Outlook notification permissions and constraints](https://learn.microsoft.com/en-us/graph/outlook-change-notifications-overview)
- [Message reading and body format](https://learn.microsoft.com/en-us/graph/api/message-get?view=graph-rest-1.0)
- [Direct reply and permissions](https://learn.microsoft.com/en-us/graph/api/message-reply?view=graph-rest-1.0)
- [Reply draft and permissions](https://learn.microsoft.com/en-us/graph/api/message-createreply?view=graph-rest-1.0)
- [Exchange sender predicates](https://learn.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/conditions-and-exceptions)
