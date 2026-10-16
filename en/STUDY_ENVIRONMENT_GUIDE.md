# Study Environment Guide — Enterprise Guide to Best Practices for AI Agents

This guide sets up an **isolated, independent** environment just for
running the workflows in Appendix B of the book. It neither uses nor
touches any production database. It is a schema of its own, with sample
data, designed so that you can test the book's concepts without needing a
real system behind them.

---

## 1. Start a dedicated Postgres database

If you already run Postgres locally (Docker, a native installation or a
managed service), create a new database just for this:

```sql
CREATE DATABASE ai_agents_guide_study;
```

If you do not have Postgres running yet, the fastest route is Docker:

```bash
docker run -d --name postgres-study \
  -e POSTGRES_PASSWORD=study123 \
  -e POSTGRES_DB=ai_agents_guide_study \
  -p 5432:5432 \
  postgres:16
```

## 2. Run study-schema.sql

Connect to the `ai_agents_guide_study` database and run the whole
`study-schema.sql` file. It creates 8 tables (`business`, `faq`,
`execution_logs`, `conversation_messages`, `conversation_evals`,
`appointments`, `document_extractions`, `leads`) and inserts a sample
business, an FAQ, an appointment and a test conversation.

```bash
psql "postgresql://postgres:study123@localhost:5432/ai_agents_guide_study" \
  -f study-schema.sql
```

Or, if you prefer, open the file in a graphical client (DBeaver,
TablePlus, pgAdmin) and run its entire contents at once.

**Confirm that the data was loaded** by running:
```sql
SELECT * FROM business;
```
You should see one row: "Sample Clinic (Study)".

## 3. Set up the Postgres credential in n8n

In n8n: **Credentials → New → Postgres**. Point it to the
`ai_agents_guide_study` database you have just created. This is the
credential you will select in every Postgres node of the imported
workflows.

## 4. Set the environment variables

Define these variables in your n8n instance's environment (the `.env` file or `docker-compose.yml`) and restart n8n. The workflows read the values with `$env`. Do not use **Settings → Variables**: the variables created there are read with `$vars`, and the workflows cannot see them.

**If you use n8n 2.x:** version 2 blocks access to environment variables by default. Also add `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` to the instance's environment. This requires your own (self-hosted) installation. On n8n Cloud, environment variables cannot be read.

| Variable | Example value | Required for |
|---|---|---|
| `WAHA_BASE_URL` | `https://your-waha-instance.com` | All WhatsApp flows |
| `WAHA_API_KEY` | (your WAHA API key) | All WhatsApp flows |
| `WAHA_DEFAULT_SESSION` | `default` | 04, 06, 07, 08 |
| `OPERATOR_WHATSAPP` | your number, in the format `5511999999999@c.us` | 04, 06, 07 |
| `ANTHROPIC_API_KEY` | (your Anthropic API key) | 07, 10 |
| `API_BASE_URL` | your n8n base URL + `/webhook` (e.g. `https://your-n8n.com/webhook`) | 01, 02, scheduling sub-agent |
| `API_INTERNAL_KEY` | any secret string of your choice | 01, 02, scheduling sub-agent |
| `N8N_BASE_URL` | your n8n base URL + `/webhook` (the same value as `API_BASE_URL` in the study environment) | 04, multi-agent |
| `N8N_API_BASE_URL` / `N8N_API_KEY` | n8n's own API (Settings → API) | 06 |
| `FAQ_SHEET_ID` | the ID of one of your Google Sheets spreadsheets | 03 (optional, see the note below) |
| `DEFAULT_BUSINESS_ID` | `1` (the sample business created in step 2) | 03, study API |
| `DEFAULT_CALENDAR_ID` | the email of a test Google Calendar | scheduling sub-agent, 09 |

In the study environment, **`API_BASE_URL` and `N8N_BASE_URL` point to
n8n itself**. It is the `study-environment-api.json` workflow (step 6)
that receives these calls, not a separate backend.

The FAQ spreadsheet read by workflow 03 must have the columns `question`,
`answer` and `category` in its first row (see Template A.3 in the book).

## 5. Set up the Anthropic credential in n8n

**Credentials → New → Anthropic**. Paste your API key. This is the
credential used in every `Anthropic Chat Model` node.

## 6. Import the workflows, in this order

The order matters because some workflows call other workflows' webhooks.

1. **`study-environment-api.json`**: import it and **activate** it (in n8n 2.x, activating is the **Publish** button). Without it, workflows 01 and 02 and the scheduling sub-agent have nothing to call.
2. **`dynamic-faq-sheets.json`** (03): import and activate it, even before setting up Google Sheets (the webhook lookup already works with the sample data inserted directly into the database).
3. **`human-handoff-redis.json`** (04): it also needs a Redis credential (see step 7).
4. **`freebusy-cache.json`** (09): it needs a Google Calendar OAuth2 credential (optional if you only want to test the other flows).
5. The 4 sub-agents (`sub-agent-triage-example.json`, `sub-agent-faq-example.json`, `sub-agent-scheduling-example.json`, `sub-agent-support-example.json`): import all of them. **They do not need to be activated**, because they are called as sub-workflows and have no webhook of their own.
6. **`multi-agent-manager.json`** (05): after importing the 4 sub-agents above, copy the ID of each one (it appears in the workflow's URL in n8n) and paste it into the `workflowId` fields of the corresponding `Tool` nodes in this workflow.
7. The others (`smb-customer-service-base.json`, `lead-qualification.json`, `error-handler-backoff.json`, `evaluation-pipeline.json`, `appointment-reminder.json`, `document-analysis.json`): import them in whatever order you prefer.

## 7. Redis credential (only needed for workflows 04 and 09)

If you do not have Redis running:
```bash
docker run -d --name redis-study -p 6379:6379 redis:7
```
In n8n: **Credentials → New → Redis**, host `localhost`, port `6379`.

## 8. Test each workflow on its own

Do not try to run all 14 at once. Suggested test order:

1. **`study-environment-api.json`**: manually run a call to
   `GET {{API_BASE_URL}}/business-info/1`. It should return the data for
   "Sample Clinic".
2. **`dynamic-faq-sheets.json`**: run `POST {{API_BASE_URL}}/faq-search`
   with `{"search_term": "hours"}`. It should return the FAQ entry about
   opening hours.
3. **`evaluation-pipeline.json`**: run it manually (Execute Workflow in
   n8n). There is already a sample conversation in the database to
   evaluate, so you do not need real WhatsApp.
4. **`smb-customer-service-base.json`**: this one needs WAHA really set up,
   with an active session. It is the first flow to test by actually
   having a conversation.

## 9. Environment isolation

If at some point you want to use these workflows as the basis for a real
project (no longer study), **do not point them at the
`ai_agents_guide_study` database**. Create a new schema and adapt the data
model to your project's real needs. The schema in this guide is
intentionally simplified (a flattened `business` table, with no separation
between configuration and information, as a production system would
normally have) to keep the teaching focus on the book's concepts.
