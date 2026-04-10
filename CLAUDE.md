# Breeze -- AI Executive Assistant

## What This Is

Breeze is a personal AI executive assistant that lives in Slack. It manages your Outlook calendar, drafts emails, summarises Slack channels, and learns your preferences over time.

**You are helping the user build Breeze.** This file is the full spec. Read it, then help the user set up and build each component interactively.

---

## Architecture

```
breeze/
  main.py              # Entry point -- Slack Socket Mode listener + scheduler
  agent.py             # Claude agent with tools registered
  scheduler.py         # Periodic jobs (conflict check, morning digest)
  auth/
    microsoft.py       # MSAL OAuth2 -- device code flow, token caching
  tools/
    calendar.py        # Microsoft Graph: read events, detect conflicts
    email.py           # Microsoft Graph: draft + send emails as user
    slack_tools.py     # Slack API: read channels, summarise, post
    memory.py          # Knowledge CRUD + decision logging
```

Supporting files (root):
- `identity.md` -- Breeze's personality + user's communication style
- `knowledge.md` -- Learned rules, priorities, contacts (managed via Slack chat)
- `.env` -- API keys and config (never committed)

---

## Stack

| Component | Package | Purpose |
|-----------|---------|---------|
| LLM | `anthropic` | Claude Sonnet for reasoning, drafting, summarisation |
| Agent | `claude-agent-sdk` | Tool orchestration and conversation handling |
| Slack | `slack-bolt` | Socket Mode bot -- DMs, channels, interactive buttons |
| Calendar/Email | `msal` + `httpx` | Microsoft Graph API via OAuth2 delegated permissions |
| Scheduler | `apscheduler` | Cron-style periodic jobs (conflict checks, digests) |
| Memory | `sqlite3` (stdlib) | Decision log for implicit learning |
| Config | `python-dotenv` | Load .env at startup |

---

## How Breeze Works

### Interface: Slack Only

Breeze is a Slack bot using Socket Mode (no public URL needed). All interaction happens via DM.

**Proactive (scheduled):**
- Hourly: check calendar for conflicts, DM user if found
- Morning (configurable): daily briefing -- today's meetings, flagged Slack threads, action items

**Reactive (user DMs Breeze):**
- "What's on my calendar tomorrow?"
- "Summarise #product-updates from this week"
- "Reschedule my 2pm with Sarah to Thursday"

**Approval flow (interactive buttons):**
When Breeze wants to take action (send email, move meeting), it posts a message with Block Kit buttons:
- [Approve & Send] -- executes the action
- [Edit] -- opens modal for user to tweak
- [Dismiss] -- cancels

**Breeze never sends emails or modifies calendar without explicit user approval.**

### Calendar Management (Microsoft Graph API)

**Auth:** MSAL with device code flow. User signs in once, refresh token is cached in `msal_token_cache.json`. Breeze acts as the user (delegated permissions), not as a separate account.

**Required delegated permissions:**
- `Calendars.ReadWrite` -- read/write calendar
- `Mail.Send` -- send emails as user
- `User.Read` -- basic profile

**Conflict detection:**
1. Fetch events for the next 7 days via `GET /me/calendarView?startDateTime=...&endDateTime=...`
2. Sort by start time
3. Any two events where `event_a.end > event_b.start` = conflict
4. For each conflict, use Claude to determine priority (consider: attendee count, external vs internal, recurrence, user's rules from knowledge.md)
5. Draft reschedule email in user's voice (guided by identity.md)
6. Post to Slack with approval buttons

**Email sending:**
- `POST /me/sendMail` with the approved draft
- Always include the original meeting context in the email body

**Graph API base URL:** `https://graph.microsoft.com/v1.0`

### Slack Integration

**Connection:** Socket Mode via `slack-bolt` AsyncApp. Requires Bot Token (`xoxb-...`) and App-Level Token (`xapp-...`).

**Bot token scopes needed:**
- `channels:history`, `channels:read` -- read channel messages
- `chat:write` -- post messages and DMs
- `im:history`, `im:read`, `im:write` -- DM conversations
- `users:read` -- look up user names

**Event subscriptions (bot events):**
- `message.im` -- triggers when user DMs Breeze

**Interactivity:** Enable for Block Kit buttons (approve/edit/dismiss).

**Channel summarisation:**
1. Pull messages from channel via `conversations.history` (last N hours)
2. Format as context for Claude
3. Claude summarises: key decisions, action items, things needing user's attention

### Agent Setup (Claude Agent SDK)

Register tools with the agent:

```python
from claude_agent_sdk import Agent, tool

@tool
def get_calendar_events(days_ahead: int = 1) -> list[dict]:
    """Get calendar events for the next N days."""

@tool
def get_calendar_conflicts(days_ahead: int = 7) -> list[dict]:
    """Check calendar for scheduling conflicts in the next N days."""

@tool
def draft_reschedule_email(event_id: str, reason: str) -> dict:
    """Draft a reschedule email for a meeting. Returns draft for approval."""

@tool
def send_approved_email(draft_id: str) -> dict:
    """Send a previously approved email draft. Only call after user approves."""

@tool
def summarise_slack_channel(channel: str, hours: int = 24) -> str:
    """Summarise recent messages in a Slack channel."""

@tool
def get_knowledge() -> str:
    """Read current knowledge base (rules, priorities, contacts)."""

@tool
def update_knowledge(action: str, content: str) -> str:
    """Add, update, or remove a rule/fact from the knowledge base.
    action: 'add', 'update', or 'remove'
    content: the rule or fact in plain English"""

@tool
def log_decision(suggestion: str, action: str, context: dict) -> None:
    """Log a user decision (approved/edited/dismissed) for implicit learning."""

@tool
def get_recent_decisions(limit: int = 20) -> list[dict]:
    """Get recent decision history for pattern detection."""
```

**System prompt** should include:
- Contents of `identity.md` (personality + user's style)
- Contents of `knowledge.md` (current rules)
- Recent decisions summary (from decision log)
- Current date/time and timezone
- Safety rules (never act without approval)

### Teaching System

Three layers:

**1. Identity (`identity.md`)**
User fills in once, tweaks over time. Defines:
- How they write emails (tone, formality, patterns)
- How they want Breeze to communicate
- Their role, team, context

**2. Knowledge (`knowledge.md`)**
Rules and facts, managed primarily via Slack chat:

Teach commands (detect these in user messages):
- `learn: ...` or `remember: ...` -- add a new rule/fact
- `rule: ...` -- add a priority/scheduling rule
- `forget: ...` -- remove a rule
- `what do you know?` -- list all current knowledge

When user teaches Breeze:
1. Parse the instruction
2. Use `update_knowledge` tool to write to knowledge.md
3. Confirm back to user what was saved

Knowledge.md format -- simple markdown sections:
```markdown
## Priority Rules
- Client meetings beat internal 1:1s
- Investor meetings are highest priority

## Protected People
- Sarah Chen (boss) -- never reschedule

## Time Rules
- No meetings before 9am
- Thursday afternoons protected for deep work

## Contacts
- James = direct report
- Meridian Corp = top client
```

**3. Decision Memory (`decisions.db`)**
SQLite table logging every suggestion + outcome:

```sql
CREATE TABLE decisions (
    id INTEGER PRIMARY KEY,
    timestamp TEXT,
    suggestion TEXT,      -- what Breeze suggested
    action TEXT,          -- 'approved', 'edited', 'dismissed'
    edit_details TEXT,    -- if edited, what changed
    context TEXT          -- JSON: meeting type, attendees, etc.
);
```

Include recent decisions in the agent's system prompt so Claude picks up patterns naturally.

**Pattern graduation:** Periodically (e.g. weekly), review decision history. If a pattern appears 5+ times with the same outcome, Breeze asks:

> "I've noticed you always keep investor meetings over team syncs (5/5 times). Want me to make that a rule?"
> [Yes, add rule] [No, keep asking]

**Auto-act (future, opt-in):** Once a rule is well-established, user can promote it to auto-act. Breeze executes and notifies with [Undo] instead of asking first.

### Scheduler

Use `apscheduler` for periodic jobs:

```python
# Conflict check -- every hour during work hours
scheduler.add_job(check_conflicts, 'cron', hour='9-17', minute=0)

# Morning digest -- once daily
scheduler.add_job(morning_digest, 'cron', hour=8, minute=30)

# Pattern review -- weekly
scheduler.add_job(review_patterns, 'cron', day_of_week='mon', hour=9)
```

Times should be configurable via .env (`BREEZE_TIMEZONE`, `BREEZE_DIGEST_TIME`, etc.).

---

## Safety Rules (Non-Negotiable)

1. **Never send emails without user approval** -- always show draft + approval buttons first
2. **Never modify calendar without user approval** -- propose changes, wait for OK
3. **Auto-act is opt-in per rule** -- user explicitly promotes rules
4. **Auto-acted items always show [Undo]** -- reversible within 5 minutes
5. **All credentials stay local** -- .env is gitignored, MSAL token cache is gitignored
6. **Breeze acts as the user** -- delegated permissions, not a separate identity

---

## Coding Conventions

- **Async throughout** -- use `asyncio`, `httpx.AsyncClient` for Graph API calls
- **Type hints** on all functions
- **Pydantic models** for structured data (calendar events, email drafts, decisions)
- **python-dotenv** at entry point for config
- **Logging** via stdlib `logging` module
- **No hardcoded values** -- everything via .env or knowledge.md

---

## Build Order

When the user asks you to build Breeze, follow this sequence:

### Phase 1: Foundation
1. Help user create Slack app (walk through api.slack.com setup)
2. Help user register Azure AD app (walk through portal.azure.com)
3. Help user fill in .env
4. Build `auth/microsoft.py` -- MSAL device code flow + token caching
5. Build `main.py` -- Slack Socket Mode connection
6. Test: Breeze connects to Slack, responds to DMs with "Hello!"

### Phase 2: Calendar
1. Build `tools/calendar.py` -- fetch events, detect conflicts
2. Register calendar tools with agent
3. Build `scheduler.py` -- hourly conflict check
4. Test: DM Breeze "What's on my calendar today?" and get a real answer

### Phase 3: Email + Approval
1. Build `tools/email.py` -- draft + send via Graph API
2. Add Block Kit approval buttons to Slack messages
3. Wire up button handlers (approve/edit/dismiss)
4. Test: conflict found -> draft shown -> approve -> email sent

### Phase 4: Teaching
1. Build `tools/memory.py` -- knowledge CRUD + decision logging
2. Detect teach commands in user messages (learn/remember/rule/forget)
3. Include knowledge + recent decisions in system prompt
4. Test: teach Breeze a rule, verify it influences next conflict suggestion

### Phase 5: Slack Intelligence
1. Build `tools/slack_tools.py` -- channel history + summarisation
2. Add morning digest scheduled job
3. Test: "Summarise #general from today" returns useful summary

### Phase 6: Learning Loop
1. Add pattern detection over decision history
2. "Want me to make this a rule?" prompts
3. Optional: auto-act promotion for trusted rules

---

## Setup Checklist (for guiding the user)

Before writing any code, the user needs:

- [ ] **Anthropic API key** -- from console.anthropic.com
- [ ] **Slack app created** with Socket Mode enabled and correct scopes
- [ ] **Azure AD app registered** with delegated permissions (Calendars.ReadWrite, Mail.Send, User.Read)
- [ ] **IT admin consent** (if corporate M365) -- this is the #1 blocker, check early
- [ ] **`.env` filled in** with all keys
- [ ] **`identity.md` filled in** with their voice and preferences

Walk the user through each of these before starting to code.
