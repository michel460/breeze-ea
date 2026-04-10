# Breeze

**Your AI Executive Assistant -- built with Claude, lives in Slack.**

Breeze manages your Outlook calendar and Slack so you don't have to. It detects scheduling conflicts, drafts reschedule emails in your voice, summarises Slack channels, and gives you a morning briefing.

**Breeze learns from you.** Teach it rules and preferences by chatting in Slack. It gets smarter over time.

---

## What Breeze Does

- Checks your Outlook calendar for double-bookings and suggests which to move
- Drafts reschedule emails in your voice (you approve before anything sends)
- Summarises Slack channels on demand
- Morning digest with today's schedule + Slack highlights
- Learns your priorities from explicit rules and your approval patterns

## How You Interact

All via Slack DM:

```
You:     What's on my calendar tomorrow?
Breeze:  You have 6 meetings. Conflict at 10am -- your 1:1 with Tom
         overlaps with the Meridian client call...
```

Teach it naturally:
```
You:     learn: client meetings always beat internal 1:1s
Breeze:  Got it. I'll prioritise client meetings when resolving conflicts.
```

---

## Quick Start

### Prerequisites

- Python 3.11+
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
- Slack workspace (admin access to install apps)
- Microsoft 365 account
- [Anthropic API key](https://console.anthropic.com)

### 1. Clone

```bash
git clone https://github.com/YOUR_ORG/breeze-ea.git
cd breeze-ea
```

### 2. Let Claude Code Guide You

```bash
claude
```

Then say:

> "Set up Breeze for me"

Claude Code will:
1. Walk you through creating a Slack app
2. Walk you through registering an Azure AD app (for Outlook access)
3. Help you fill in `.env`
4. Help you fill in `identity.md` (your voice and preferences)
5. Build out the code with you, phase by phase

### 3. Run

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # fill in your keys
python -m breeze.main
```

DM Breeze in Slack to test.

---

## Important: Microsoft 365 Admin Consent

If you're on a **corporate M365 account**, your IT admin may need to approve the Azure AD app registration. Check with IT before starting -- this is the most common blocker.

---

## Teaching Breeze

Via Slack DM:

| Command | What it does |
|---------|-------------|
| `learn: ...` | Add a rule or fact |
| `remember: ...` | Add a rule or fact |
| `rule: ...` | Add a scheduling/priority rule |
| `forget: ...` | Remove a rule |
| `what do you know?` | List all current knowledge |

Breeze also learns implicitly from your approve/edit/dismiss choices over time.

---

## Project Structure

```
breeze-ea/
  CLAUDE.md            # Full spec -- Claude Code reads this to build Breeze
  identity.md          # Your voice and communication style
  knowledge.md         # Rules, priorities, contacts (managed via Slack)
  .env.example         # API keys template
  requirements.txt     # Python dependencies
  breeze/
    main.py            # Entry point
    agent.py           # Claude agent + tools
    scheduler.py       # Periodic jobs
    auth/
      microsoft.py     # MSAL OAuth2 for Outlook
    tools/
      calendar.py      # Calendar read/write
      email.py         # Email drafting + sending
      slack_tools.py   # Channel summarisation
      memory.py        # Knowledge + decision logging
```

---

## Safety

- Breeze **never** sends emails or modifies your calendar without your approval
- All actions require explicit approve/dismiss via Slack buttons
- Credentials stay local (`.env` and token cache are gitignored)
- Breeze uses your account (delegated permissions), not a separate identity

---

## Cost

~$10-25/month (Claude API + optional hosting). Slack and Microsoft Graph APIs are free.

---

## License

MIT
