# Workflow Library — Enterprise Guide to Best Practices for Artificial Intelligence Agents

Workflows referenced in Appendix B of the book. They all use native n8n
nodes (HTTP Request, Postgres, Redis) and the AI nodes from the
`@n8n/n8n-nodes-langchain` package (AI Agent, Chat Model, Memory, Tools)
for the flows that involve the agent's reasoning, consistent with what
section 9.3 teaches.

**This is an isolated study environment, independent of any production
system.** The tables (`business`, `faq`, `appointments`, etc.) and the
support API (`study-environment-api.json`) were created only so that you
can run the book's examples from scratch, with test data. They do not
represent the schema of any real product. **Start with
`STUDY_ENVIRONMENT_GUIDE.md`**, which has the complete step-by-step setup
to follow before importing any workflow.

**These are starting templates, not plug-and-play solutions** (the same
warning as in Appendix B). Review the credentials, webhook URLs and
system prompts before any production use.

The node names, fields, tables and identifiers in these files match the
English edition of the book exactly, so you can follow each chapter with
the corresponding workflow open in n8n.

## n8n version

The workflows were tested on n8n 1.123.82, the same version cited in the
book's Legal notice. On n8n 2.x, two precautions are needed before testing:

- **Environment variables.** Version 2 blocks access to `$env` by default. Set `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` in the instance's environment (see step 4 of `STUDY_ENVIRONMENT_GUIDE.md`).
- **Publishing.** In version 2, saving and publishing are separate steps. After importing, use **Publish** so that the workflow receives production calls.

The **HTTP Request Tool** nodes (legacy) and the 1.x versions of the **AI
Agent** used here work on 2.x, but n8n has announced their removal in the
future version 3.0. When it is released, an updated version of these files
will be published in this repository.

## Differences from the flows in the book

The YAML flows in the chapters are teaching material. These files are
starting implementations and, in a few details, make different choices:

- `human-handoff-redis.json`: human-mode TTL of 1 hour (30 minutes in the book).
- `appointment-reminder.json`: checks every 30 minutes (every hour in the book).
- `dynamic-faq-sheets.json`: the synchronized FAQ lives in Postgres (cached in Redis in the book).
- `evaluation-pipeline.json`: daily evaluation at 6 a.m. (weekly in the book).
- `error-handler-backoff.json`: up to 3 new attempts per hour, waiting 30 s, 60 s and 120 s plus a random amount, and an alert to the operator on WhatsApp.

## How to import

In the n8n panel: side menu → **Workflows** → **Import from File** →
select the corresponding `.json` file.

## Reference table

| File | Description | Chapter | Level |
|---|---|---|---|
| `study-environment-api.json` | Support API (companion), study endpoints backed by Postgres | — | Setup |
| `smb-customer-service-base.json` | Base customer service agent with WhatsApp + Claude | Ch. 9 | Beginner |
| `lead-qualification.json` | Lead qualification with 5 fields and CRM | Ch. 9 | Beginner |
| `dynamic-faq-sheets.json` | FAQ synchronized with Google Sheets (15 min) | Ch. 10 | Intermediate |
| `human-handoff-redis.json` | Handoff with TTL auto-return and the `!close` command | Ch. 10 | Intermediate |
| `multi-agent-manager.json` | Manager pattern + specialized workers | Ch. 7 | Advanced |
| `sub-agent-triage-example.json` | Triage sub-agent (bonus, not listed in the book) | Ch. 7 | Advanced |
| `sub-agent-faq-example.json` | FAQ sub-agent (bonus, not listed in the book) | Ch. 7 | Advanced |
| `sub-agent-scheduling-example.json` | Scheduling sub-agent (bonus, not listed in the book) | Ch. 7 | Advanced |
| `sub-agent-support-example.json` | Technical support sub-agent (bonus, not listed in the book) | Ch. 7 | Advanced |
| `error-handler-backoff.json` | Error handling with retry and alerts | Ch. 14 | Intermediate |
| `evaluation-pipeline.json` | Automated quality evaluation pipeline | Ch. 15 | Advanced |
| `appointment-reminder.json` | Reminder agent with Google Calendar | Ch. 10 | Beginner |
| `freebusy-cache.json` | Calendar availability cache with Redis | Ch. 10 | Intermediate |
| `document-analysis.json` | Structured extraction from contracts and PDFs | App. A | Advanced |

## Complete setup (schema, study API, variables, import order)

Everything is in **`STUDY_ENVIRONMENT_GUIDE.md`**: the SQL schema with
sample data (`study-schema.sql`), the support API, the complete table of
environment variables, the credentials you need, and the correct order
for importing and testing. This section of the README only summarizes the
essentials. The guide is the source of truth.

## Credentials to set up in n8n

- **Anthropic** (in the `Anthropic Chat Model` nodes, type `@n8n/n8n-nodes-langchain.lmChatAnthropic`)
- **Postgres** (in all Postgres nodes), pointing to the database created with `study-schema.sql`
- **Redis** (workflows 04 and 09)
- **Google Sheets OAuth2** (workflow 03, optional)
- **Google Calendar OAuth2** (workflow 09, optional)

## About the Multi-Agent pattern (workflow 05)

`multi-agent-manager.json` references 4 sub-workflows by ID
(`REPLACE_WITH_..._SUB_AGENT_ID`). All 4 are already in this library,
ready to import (`sub-agent-triage-example.json`, `sub-agent-faq-example.json`,
`sub-agent-scheduling-example.json`, `sub-agent-support-example.json`).
You only need to import each one, copy the workflow ID that n8n generates,
and paste it into the `workflowId` field of the corresponding `Tool` node
in the Manager.

Dependencies between them:
- The **FAQ sub-agent** needs workflow 03 (`dynamic-faq-sheets.json`) imported and active, because it calls that workflow's `/faq-search` webhook.
- The **Scheduling sub-agent** needs workflow 09 (`freebusy-cache.json`) imported and active, plus the `/appointments` and `/appointments/cancel` routes in your internal API.
- The **Technical Support sub-agent** needs workflow 04 (`human-handoff-redis.json`) imported and active, because it calls that workflow's `/handoff-activate` webhook.
- The **Triage sub-agent** has no external dependency, only the call to the model.

Recommended import order: 03, 04, 09, then the 4 sub-agents, then
`multi-agent-manager.json` last (it is the only one that references the
other workflows' IDs).

## Questions or problems with a specific flow

Describe the expected versus observed behavior and the node where it got
stuck. That speeds up the diagnosis a great deal.
