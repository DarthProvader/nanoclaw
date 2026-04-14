# NanoClaw Demo Integration Plan — Handoff Document

## Context

We're building a demo for a prospective customer (Martin) who runs a solar panel installation company. The demo should showcase NanoClaw as an AI project coordinator that integrates with his existing tools. This document contains all research findings and implementation details needed to build the integrations.

### Customer's Systems
- **Raynet CRM** — customer data, business cases/deals, quotes/offers, attachments
- **Freelo** — project management, task lists, deadlines, process templates
- **Gmail** — email monitoring (BCC/CC to track if communications were sent)
- **Google Drive** — document storage (future phase)
- **WhatsApp** — team communication (preferred channel)

### Three Demo Use Cases

#### 1. Project Coordinator (Priority)
- Agent checks Raynet + Freelo daily via cron job
- Monitors process status: are tasks on track? deadlines met?
- Sends smart notifications via WhatsApp — not just "deadline passed" but contextual ("why isn't this done? do you need help?")
- Monitors emails (BCC) to verify required communications were sent (e.g., applications to ČEZ power company)
- Read-only access initially, no direct external communication
- Can prepare email drafts for the assistant to review

#### 2. ČEZ Application Document Generation
- Pull customer data from Raynet (meter number, address, contacts...)
- Pre-fill application documents automatically
- Based on example successful applications provided by the customer

#### 3. Web-Based Price Quotes (Future Phase)
- Generate beautiful HTML pages from Raynet quote data instead of ugly PDFs
- Interactive elements, visit tracking, salesperson notifications

---

## Raynet CRM API

### Documentation
- **Czech docs:** https://app.raynet.cz/api/doc/
- **English docs:** https://app.raynetcrm.com/api/doc/index-en.html

### Authentication
HTTP Basic Auth + instance header:
```
Authorization: Basic base64(email:api_key)
X-Instance-Name: <instance-name>
Content-Type: application/json
```

### Base URL
```
https://app.raynet.cz/api/v2/
```

### Key Endpoints

| Resource | Endpoint | Notes |
|----------|----------|-------|
| Companies | `GET /company/` | Filtering, addresses, relationships |
| Contacts | `GET /person/` | firstName, lastName, email filters |
| Business Cases | `GET /businessCase/` | Lock/unlock, participants, status pipeline |
| Offers/Quotes | `GET /offer/` | Line items, PDF export via `/offer/{id}/export` |
| Attachments | Inline in entity responses | `attachments[]` array |
| Products | `GET /product/`, `/priceList/` | Product catalog |
| Activities | `/task/`, `/meeting/`, `/email/`, `/phoneCall/` | Various activity types |
| Webhooks | `PUT/GET/DELETE /webhook/` | Event-driven notifications |
| Custom Fields | Inline in entities | `customFields: { "fieldId": value }` |

### Filtering
Operators in brackets: `name[LIKE]=test%`, `id[IN]=1,2,3`, `date[GT]=2022-06-01`
Available: `EQ`, `NE`, `LT`, `LE`, `GT`, `GE`, `LIKE`, `LIKE_NOCASE`, `IN`, `NOT_IN`, `CUSTOM`

### Pagination
- `offset` and `limit` (max 1,000 per page)
- `sortColumn` and `sortDirection` (ASC/DESC)
- `fulltext` parameter for search

### Rate Limits
- **24,000 requests/day** (UTC midnight reset)
- Max 4 concurrent connections
- Response headers: `X-Ratelimit-Limit`, `X-Ratelimit-Remaining`, `X-Ratelimit-Reset`
- HTTP 429 when exceeded
- 20+ failed auth attempts = 60-minute IP block

### Existing MCP Server
**`AiPulseInc/raynet-mcp-server`** on GitHub:
- TypeScript, 91 tools, v0.80.0
- Covers: companies, contacts, deals, activities, leads, products, offers, orders, projects, enums
- Has "mobile mode" (25 tools) and "full mode" (91 tools)
- Supports stdio and HTTP transport
- Production-quality with Zod validation, rate limiting, retry logic
- Install: check the GitHub repo for setup instructions

---

## Freelo API

### Documentation
- **OpenAPI spec:** https://api.freelo.io/docs/v1/freelo-api.yaml
- **Swagger UI:** https://api.freelo.io/docs/v1/freelo-api#/
- **Legacy Apiary docs:** https://freelo.docs.apiary.io/
- **Help page:** https://www.freelo.io/en/help/api

### Authentication
HTTP Basic Auth:
```
Authorization: Basic base64(email:api_key)
User-Agent: NanoClaw (agent@example.com)   # Required!
```
API key from: https://app.freelo.io/profil/nastaveni

### Base URL
```
https://api.freelo.io/v1
```

### Key Endpoints

| Resource | Endpoint | Notes |
|----------|----------|-------|
| Verify credentials | `GET /users/me` | Test auth |
| Projects | `GET /projects` | Active projects |
| Archived projects | `GET /projects/archived` | |
| Template projects | `GET /projects/templates` | Process templates |
| Project details | `GET /project/{id}` | |
| Project tasklists | `GET /project/{id}/tasklists` | |
| Tasks in tasklist | `GET /project/{pid}/tasklist/{tid}/tasks` | |
| All tasks (filtered) | `GET /all-tasks` | 14 filter params |
| Task details | `GET /task/{id}` | |
| Create task | `POST /project/{pid}/tasklist/{tid}/tasks` | |
| Edit task | `PUT /task/{id}` | |
| Finish task | `POST /task/{id}/finish` | |
| Activate task | `POST /task/{id}/activate` | |
| Move task | `POST /task/{id}/move` | |
| Task comments | `POST /task/{id}/comments` | |
| All comments | `GET /all-comments` | |
| Subtasks | `GET /task/{id}/subtasks` | |
| Search | `POST /search` | Elasticsearch-powered |
| Time tracking | `GET /timetracking/status` | |
| Work reports | `GET /work-reports` | |
| Files | `GET /all-files`, `GET /file/{uuid}/download` | |
| Labels | Various | Task and project labels |
| Custom fields | Various | Per-project, premium feature |
| Notifications | `GET /all-notifications` | |

### `/all-tasks` Filter Parameters
`search_query`, `state_id`, `projects_ids`, `tasklists_ids`, `order_by`, `order`, `with_label`, `without_label`, `no_due_date`, `due_date_range`, `finished_overdue`, `finished_date_range`, `worker_id`, `p`

### Pagination
- Parameter: `?p=0` (page number, zero-indexed)
- Ordering: `order_by` and `order` (asc/desc)

### Rate Limits
- **25 requests per minute** (this is quite low!)
- HTTP 429 when exceeded, wait 60 seconds
- Agent should cache data aggressively and batch reads

### Currency Format
Amounts as strings multiplied by 100: `1,000.25` -> `"100025"`

### Existing MCP Server
**`freelo-mcp`** on npm:
- 104 tools, 100% Freelo API v1 coverage
- Install: `npx -y freelo-mcp`
- Config env vars: `FREELO_EMAIL`, `FREELO_API_KEY`, `FREELO_USER_AGENT`
- Stdio + HTTP transport
- 207 unit tests, 82 API tests

**Official SDK:** `@freeloapp/js-sdk` v2.1.0 — if you prefer building a custom tool.

---

## NanoClaw Integration Architecture

### How MCP Servers Work in NanoClaw

MCP servers are registered in `container/agent-runner/src/index.ts` in the `mcpServers` config object (around line 477). Example of existing setup:

```typescript
mcpServers: {
  nanoclaw: {
    command: 'node',
    args: [mcpServerPath],
    env: {
      NANOCLAW_CHAT_JID: containerInput.chatJid,
      NANOCLAW_GROUP_FOLDER: containerInput.groupFolder,
      NANOCLAW_IS_MAIN: containerInput.isMain ? '1' : '0',
    },
  },
}
```

To add Raynet and Freelo, add new entries to this object:

```typescript
mcpServers: {
  nanoclaw: { /* existing */ },
  raynet: {
    command: 'npx',
    args: ['-y', 'raynet-mcp-server'],  // or path to local install
    env: {
      RAYNET_EMAIL: process.env.RAYNET_EMAIL,
      RAYNET_API_KEY: process.env.RAYNET_API_KEY,
      RAYNET_INSTANCE_NAME: process.env.RAYNET_INSTANCE_NAME,
    },
  },
  freelo: {
    command: 'npx',
    args: ['-y', 'freelo-mcp'],
    env: {
      FREELO_EMAIL: process.env.FREELO_EMAIL,
      FREELO_API_KEY: process.env.FREELO_API_KEY,
      FREELO_USER_AGENT: 'NanoClaw (agent@nanoclaw.dev)',
    },
  },
}
```

Once registered, tools appear to the agent as `mcp__raynet__*` and `mcp__freelo__*`.

### Container Architecture Notes

- **Container runner:** `src/container-runner.ts` — spawns Docker containers for each group
- **Agent runner:** `container/agent-runner/src/index.ts` — runs inside the container, configures Claude SDK
- **MCP server (internal):** `container/agent-runner/src/ipc-mcp-stdio.ts` — handles messaging/tasks
- **Skills:** `container/skills/` — instruction-only SKILL.md files synced to agent's `.claude/skills/`
- **Groups:** `groups/{name}/CLAUDE.md` — per-group memory and instructions
- **IPC:** File-based JSON communication via `/workspace/ipc/`
- **Credentials:** Managed by OneCLI gateway (intercepts HTTPS, injects auth headers). If not using OneCLI, pass via env vars in `.env`.

### Volume Mounts
- Main group gets read-only project root + writable store/group/global
- Non-main groups get only their own folder (writable) + global (read-only)
- Additional mounts configurable via `containerConfig.additionalMounts`

---

## Implementation Steps

### Step 1: Install MCP Server Dependencies
Either install into the container image (preferred) or use npx at runtime:

```bash
# Option A: Add to container/Dockerfile
RUN npm install -g raynet-mcp-server freelo-mcp

# Option B: Add to container/agent-runner/package.json
npm install raynet-mcp-server freelo-mcp
```

Check the exact package names on npm/GitHub first — the Raynet MCP server may need to be cloned from `AiPulseInc/raynet-mcp-server` rather than installed from npm.

### Step 2: Configure Environment Variables
Add to `.env` (or OneCLI vault):

```env
RAYNET_EMAIL=test@example.com
RAYNET_API_KEY=your-raynet-api-key
RAYNET_INSTANCE_NAME=your-instance-name

FREELO_EMAIL=test@example.com
FREELO_API_KEY=your-freelo-api-key
```

### Step 3: Register MCP Servers
Edit `container/agent-runner/src/index.ts` — add raynet and freelo to the `mcpServers` object as shown above. Make sure env vars are passed through from the container runner.

Also check `src/container-runner.ts` to ensure env vars are forwarded to the container (look at how existing env vars like `JIRA_*` or others are handled).

### Step 4: Create Demo Group
Create `groups/renta_demo/CLAUDE.md` with the coordinator role description:

```markdown
# Solar Installation Project Coordinator

You are a project coordinator for a solar panel installation company. Your role is to monitor projects across Raynet CRM and Freelo, identify delays or risks, and proactively notify the team via WhatsApp.

## Your Responsibilities

1. **Daily Process Check** — Every morning, review all active business cases in Raynet and their corresponding Freelo tasks. Identify:
   - Tasks past their deadline
   - Business cases stuck in a pipeline stage too long
   - Missing follow-ups (e.g., ČEZ application not submitted)

2. **Smart Notifications** — Don't just report "task X is late." Ask the responsible person what's blocking them and offer help. Suggest deadline adjustments if appropriate.

3. **Email Monitoring** — Check if required emails were sent (you receive BCC copies). Verify that communications to ČEZ, customers, etc. happened on schedule.

4. **Document Preparation** — When a deal reaches the right stage, pre-fill ČEZ application documents from Raynet data (customer info, meter number, address, etc.).

## Process: Solar Panel Installation (Residential)
[To be filled with actual process steps from Freelo templates]

## Systems Access
- Raynet CRM: read-only (companies, business cases, offers, attachments)
- Freelo: read tasks, deadlines, statuses
- Gmail: read incoming/sent emails (BCC monitoring)

## Communication Rules
- Write in Czech
- Use WhatsApp for team notifications
- Never send external emails directly — prepare drafts only
- Be helpful, not annoying — batch notifications, prioritize by urgency
```

### Step 5: Set Up Cron Job
Register a scheduled task for daily morning check. The agent can do this itself via `mcp__nanoclaw__schedule_task`, or you can set it up manually. Example cron: every weekday at 7:00 AM.

### Step 6: Gmail Integration
Run `/add-gmail` skill to set up email monitoring. Configure with the test Gmail account. The agent should be able to read incoming emails (forwarded via BCC).

### Step 7: Rebuild Container
```bash
./container/build.sh
```
If using cached builds, prune first:
```bash
docker builder prune
./container/build.sh
```

---

## Test Data to Prepare (Martin's side)

### Raynet
1. Create a test account with read-only permissions + API key
2. Add 3-5 fake companies (residential customers)
3. Create business cases with pipeline stages: Start → Nabídka → Smlouva → Instalace → Revize → ČEZ žádost → Dotace → Dokončeno
4. Add sample offers/quotes with products (solar panels, inverters, batteries)
5. Fill in custom fields: číslo elektroměru, adresa instalace, etc.

### Freelo
1. Create a test account + API key
2. Set up a project template for "Instalace FVE — rodinný dům"
3. Create task lists matching the process stages
4. Add sample tasks with deadlines, assignees
5. Some tasks should be overdue (for demo of coordinator alerts)

### Gmail
1. Create a test Gmail account (e.g., renta-agent-test@gmail.com)
2. Set up forwarding rules from team emails (BCC)
3. Have a few sample emails: ČEZ application sent, customer communication, etc.

---

## Phased Rollout

### Phase 1 (Demo — this week)
- Raynet + Freelo read-only integration
- Daily status check via cron
- WhatsApp notifications
- Basic coordination ("task X is overdue, what's the status?")

### Phase 2 (After demo approval)
- Gmail monitoring (BCC email tracking)
- ČEZ document pre-filling from Raynet data
- Google Drive file organization
- Draft email preparation

### Phase 3 (Future)
- Interactive HTML price quotes
- Customer visit tracking
- Process improvement suggestions
- Freelo task auto-creation from Raynet deal stage changes

---

## Important Notes

- **Freelo rate limit is 25 req/min** — agent must cache aggressively, don't poll too frequently
- **Raynet rate limit is 24,000 req/day** — much more generous, but still avoid unnecessary calls
- **Read-only first** — customer explicitly wants no write access initially
- **Communication channel: WhatsApp** — customer's team already uses it, they rejected Slack/Discord for now
- **Next check-in meeting: April 16, 2025, 10:30** — demo progress should be ready by then
- **Czech language** — all agent communication must be in Czech
