# Microsoft gateways: implementation handoff

These documents describe proposed integrations. They do not add a working gateway
or configure a live tenant.

## Start here

1. Read the repository's `AGENTS.md` and `CONTRIBUTING.md`.
2. Read the [Teams specification](../integrations/microsoft-teams.md).
3. Implement the [Teams plan](teams-implementation-plan.md) in order when assigned
   an implementation task. Treat the specification as the product contract.
4. Keep [Outlook mail](../integrations/outlook-mail.md) separate until its intake
   trust model and authorization modes have been finalized. It is a researched
   proposal, not an instruction to add general mailbox access or sending.

The source baseline is `6ccbdd178ac7dcaf970acfc9a3836be2ca864c49`. Reinspect current
upstream before implementation, especially migrations, channel UI and runtime
delivery contracts. The fork's `origin` is the contribution/development target;
`upstream` is the original project. Do not open a PR or contact maintainers as an
implicit consequence of implementing the fork.

## Architecture boundaries

- **Teams:** bidirectional chat transport into the existing channel engine.
- **Outlook:** initially folder-to-issue intake using Graph; different conversation
  semantics and permissions, sharing issue/identity services where appropriate.
- **Agents:** existing Multica runtimes and tools; no transport-specific LLM loop.
- **Configuration:** per-installation tenant, agent and credentials; no assumptions
  about a particular organization, cloud host, engineering role or model vendor.

An email conversation and a Teams Chat can refer to the same issue without being
the same session. Correlate through explicit issue/source references. Do not
merge histories automatically.

## Implementation assignment

An implementation task can use this concise handoff:

> Implement the Microsoft Teams gateway described in
> `docs/integrations/microsoft-teams.md`, following
> `docs/development/teams-implementation-plan.md`. Follow repository instructions,
> use existing channel/identity/delivery patterns, keep provider tests local and
> finish the stated acceptance checks. Preserve generic configuration and shared
> web/desktop UI. Do not implement Outlook, provision cloud resources, send live
> messages, publish private configuration or open a PR unless the assigned task
> separately calls for those actions. Report actual checks and skipped live tests.

## Deployment sequence

Prepare the development toolchain and managed environment only when starting the
implementation task. Test protocol behavior using synthetic Activities and a fake
runtime. After the code and local checks are complete, separately configure a
test Azure Bot, reachable HTTPS endpoint and Teams app installation for a live
roundtrip. Real-agent smoke tests follow the repository's explicit opt-in rules.

Use a backend built from this fork after implementation. Installing the upstream
CLI or Desktop alone does not update a hosted backend with custom server code.
Keep local development profiles separate from any existing installation.

## Publication hygiene

Commit only product code, technical documentation and synthetic tests. Keep all
credentials, OAuth grants/caches, mailbox content, tenant-specific settings,
private addresses, local paths, conversation logs and assistant workflow artifacts
outside the repository and its Git history. Use example domains and placeholders
in instructions. Review commit metadata and screenshots as well as file contents.

Before proposing upstream integration, rebase the implementation, rerun checks,
review the complete diff, and follow the project's contribution/license and visual
evidence requirements. A fork does not change those requirements or guarantee
acceptance or deployment by the original project.
