# Auretris: engineering notes

Auretris is a self-hosted WhatsApp automation stack running in production for
[Skeure Education](https://education.skeure.com), an education consulting business. It
handles inbound student enquiries on WhatsApp: answering programme questions, sending
brochures, and handing conversations to a CRM. **Kuanli** is the same application under a
different name, white-labeled for the CRM-facing side.

These are engineering notes, not the source. The application code is private. What is here
is the architecture and the decisions behind it, published because the decisions are the
part worth reading.

## The stack

| Piece | What it does |
|---|---|
| **Meta Cloud API** | The WhatsApp Business transport. Inbound webhooks, outbound sends, templates, media. |
| **CRM app** | Next.js / TypeScript / Supabase. Conversation UI, contacts, broadcasts, templates. A white-labeled deployment of an MIT-licensed open-source CRM. |
| **n8n** | Hosts the actual bot logic as a workflow, not as application code. LLM agent plus two tool sub-workflows: FAQ search and brochure sending. |
| **Postgres + Supabase services** | Data, auth, storage, realtime. Self-hosted alongside the app. |
| **Docker Compose** | One network, one stack. Every service above is a container. |

[Full architecture and the decisions →](docs/architecture.md)

## Three decisions worth explaining

**Fork and maintain, do not rebuild.** The CRM is a fork of a maintained MIT-licensed
project rather than something written from scratch. The build-versus-adopt call was made
explicitly, and the deciding factor was not initial development time. It was that a
maintained upstream keeps absorbing WhatsApp API changes that a bespoke CRM would have to
chase alone. The cost is living with someone else's schema and merge conflicts on every
upstream pull. That is a real cost, and it is smaller than the one avoided.

**Bot logic lives in a workflow, not in the app.** The conversational logic sits in n8n
rather than in the CRM's TypeScript. This is deliberate. Prompt and routing changes are the
highest-churn part of the whole system, and putting them in application code means a
deploy for every wording tweak. The tradeoff is honest: the workflows live in the database,
not in files, so they are outside git and need their own export discipline. That is a
worse story for version control and a much better one for iteration speed.

**A deprecated integration path is worth removing early.** The client-onboarding routes
were built against Meta's Embedded Signup v2. Meta set a deprecation date for v2, so the
routes were removed rather than left in place to rot until the cutoff, and a rebuild on
v4 was scoped as its own piece of work. Deleting working code because of a date on
someone else's roadmap feels wrong in the moment. Leaving it is worse: dead-on-arrival
code accumulates callers, and every caller makes the eventual migration bigger.

## A mistake worth recording

An automated session once built a product marketing site directly inside the production
CRM's source tree, taking over its root route in the process. Two applications that should
have been siblings became one, and a live service was serving marketing pages from the
path its own users needed.

Untangling it meant separating the two into genuinely independent applications with their
own containers and build configs, rather than adding a route guard and calling it fixed.

The useful lesson is not "be careful". It is that the mistake was only possible because
the two things shared a name. "Auretris" meant both the bot and the product being marketed,
so instructions that said "add a landing page to Auretris" were genuinely ambiguous, and
the agent resolved the ambiguity the wrong way. Naming two different things the same thing
is a latent bug, and it eventually gets executed.

## Where this sits

Part of a wider personal operating system: [operator-os](https://github.com/Prthm-G/operator-os).

MIT licensed. See [LICENSE](LICENSE).
