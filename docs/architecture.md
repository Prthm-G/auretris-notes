# Architecture

A self-hosted WhatsApp automation stack. Everything runs as containers on one Docker
network, on hardware the business controls.

## Why self-hosted

The obvious alternative was a SaaS WhatsApp CRM. Self-hosting won on three counts, and
lost on one that matters:

- **Data residency.** Student enquiries include names, phone numbers, and academic
  history. Keeping them on infrastructure the business controls is simpler to reason about
  than a third party's retention policy.
- **Per-seat pricing does not apply.** The cost curve of a SaaS CRM bends the wrong way as
  conversation volume grows.
- **The bot logic is the product.** Anything that made the conversational layer someone
  else's black box was disqualifying.
- **What it costs:** every upgrade, backup, and outage is now the operator's problem.
  There is no support contract. That is the real price of this decision and it gets paid
  monthly.

## The layers

### Transport: Meta Cloud API

Inbound messages arrive as webhook POSTs. Every one carries an HMAC-SHA256 signature that
the receiver verifies before doing anything with the payload. An unsigned or wrongly
signed request is rejected outright, because a webhook endpoint is a publicly reachable
URL and anything less means accepting messages from anyone who finds it.

Outbound goes back through the Cloud API: session messages inside the 24-hour customer
service window, pre-approved templates outside it.

### Application: a forked CRM

A white-labeled deployment of an MIT-licensed open-source WhatsApp CRM. Next.js and
TypeScript on Supabase.

White-labeling is entirely environment variables: site URL, sender identity, branding.
There is no forked "Kuanli edition" of the codebase. This matters more than it sounds.
The moment branding lives in the source, every upstream merge conflicts on cosmetic
changes, and the fork drifts until pulling upstream stops being worth it. Keeping the
delta to config is what makes the fork maintainable rather than a snapshot.

### Logic: n8n workflows

The bot is an n8n workflow, not application code:

- A main conversational agent backed by an LLM
- A FAQ search tool over a Postgres-backed question set
- A brochure-sending tool

Tools are separate sub-workflows rather than branches inside one graph, so each can be
tested and changed on its own.

The honest cost: workflows are stored in the database, not as files in the repo. They are
outside version control by default and need deliberate export to be backed up or diffed.
Anyone copying this pattern should decide up front how workflow state gets into git,
because the default is that it never does.

### Data: self-hosted Supabase

Postgres plus the Supabase service set: auth, REST, realtime, storage, and an API gateway.
Running these as containers alongside the app rather than using hosted Supabase keeps the
whole system restorable from one compose file and one volume backup.

## Multi-tenancy: an unresolved tension

The application already supports multiple client accounts in a single deployment. Tenant
data is account-scoped with row-level security.

The deployment pattern does not use that. It clones the entire stack per client instead.

These two facts are in tension, and the cloned-stack approach is the one that was never
validated as necessary. It costs a full set of containers, a database, and a port
allocation per client, plus N stacks to upgrade whenever anything changes. The app-level
isolation that would make one shared deployment viable already exists and is already
enforced in the schema.

This is written down precisely because it is unresolved. The cost of the current pattern
grows linearly with clients while the cost of fixing it stays flat, which means the right
time to fix it is always "sooner than now feels".

## Operational posture

- **Secrets** live in environment files outside every git tree. Never in the repo, never
  in a prompt, never in a log.
- **Signature verification** on every inbound webhook, before parsing.
- **Token encryption at rest.** Stored WhatsApp credentials are encrypted with a key held
  outside the database. One consequence worth knowing before you copy this: rotating that
  key orphans everything encrypted under the old one. Key rotation is a migration, not a
  config change, and designing for it afterwards is painful.
- **Production changes ship as scripts.** Backup, patch, apply, verify, run by hand.
  Never a direct edit against a live service, and never without a rollback path already in
  place.

## What I would do differently

**Decide where workflow state lives on day one.** Logic in n8n was the right call, but
"the workflows are in the database" became true by default rather than by decision, and
the export discipline was retrofitted.

**Name things once.** Two applications sharing the name "Auretris" produced a genuine
production incident, because an instruction naming the shared word was ambiguous and got
resolved the wrong way. Ambiguity in a name is a latent bug.

**Treat a vendor deprecation date as a real deadline.** The Embedded Signup v2 to v4
migration was foreseeable from the moment Meta published the date. Integration code built
against a version with an announced end-of-life is already technical debt on the day it
ships.
