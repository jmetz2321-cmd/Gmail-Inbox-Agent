# Gmail Inbox Agent

An automated Gmail assistant that monitors your inbox for emails from **MentorCruise** and **Leland**, classifies them by type, and uses **Claude AI** to generate draft replies — all without sending anything until you review and approve.

---

## How It Works

Every 15 minutes, a Google Apps Script trigger fires and:

1. Searches your Gmail for emails from `mentorcruise.com`, `leland.com`, or `lelandapp.com`
2. Classifies each new email into one of four categories
3. Calls the Claude API to write a context-aware draft reply
4. Saves the draft to your Gmail Drafts folder
5. Remembers which emails it has already processed so it never duplicates drafts

Nothing is ever sent automatically — every draft is yours to review, edit, and send.

---

## Email Categories

| Category | Icon | Triggered By |
|---|---|---|
| **Meetings** | 📅 | New applications, intro call scheduling, calendar invites, reschedules, call feedback requests |
| **Messages** | 💬 | Mentee chat messages forwarded from MentorCruise |
| **Reviews** | ⭐ | New review or rating notifications |
| **Stops** | 🛑 | Mentee pausing, cancelling, or ending a mentorship |

---

## Setup Instructions

### Prerequisites
- A Google account with Gmail
- An Anthropic API key — get one at [console.anthropic.com](https://console.anthropic.com)

### Step 1 — Open Google Apps Script
Go to [script.google.com](https://script.google.com) and click **New project**. Make sure you're signed in with the Gmail account that receives your MentorCruise/Leland emails.

### Step 2 — Paste the code
Select all the placeholder code in the editor (`Cmd+A` / `Ctrl+A`), delete it, and paste in the full contents of `MentorshipInboxAgent_Code.md` (the code block only, not the markdown wrapper).

### Step 3 — Add your Anthropic API key
1. Click the **⚙️ Project Settings** icon in the left sidebar
2. Scroll down to **Script Properties**
3. Click **Add property**
4. Set the key to `ANTHROPIC_API_KEY` and the value to your API key
5. Click **Save script properties**

### Step 4 — Run the setup function
1. Back in the editor, click the function dropdown (next to the ▶ Run button) and select `setupTrigger`
2. Click **▶ Run**
3. When prompted, click **Review permissions** → choose your account → click **Allow**

The script will install a recurring 15-minute trigger and run immediately for the first time. Check the **Execution log** at the bottom to see it working.

---

## Configuration

At the top of the script, the `CONFIG` object lets you customize behavior:

```javascript
var CONFIG = {
  SEARCH_QUERY: "from:mentorcruise.com OR from:leland.com OR from:lelandapp.com",
  CHECK_INTERVAL_MINUTES: 15,   // how often to check for new emails
  MAX_PER_RUN: 20,              // max drafts created per run
  MODEL: "claude-sonnet-4-20250514",
  SIGN_OFF: "Jon",              // name used in draft sign-offs
};
```

Change `SIGN_OFF` to your name, and adjust `CHECK_INTERVAL_MINUTES` if you want more or less frequent checks (minimum is 15 minutes for Apps Script time-based triggers).

---

## Utility Functions

These can be run manually from the Apps Script editor:

| Function | What it does |
|---|---|
| `setupTrigger()` | Installs the 15-min trigger and runs immediately |
| `checkNewEmails()` | Runs the agent manually right now |
| `checkTriggerStatus()` | Shows whether the trigger is active |
| `resetProcessedIds()` | Clears memory so all emails are re-processed from scratch |

---

## Viewing Execution Logs

To see what the script is doing:
1. Go to [script.google.com](https://script.google.com) → open the project
2. Click **Executions** in the left sidebar
3. Click any execution to see the full log output

Each run logs which emails were found, their category, and whether a draft was created.

---

## Stopping the Agent

To turn off the automatic trigger:
1. In the Apps Script editor, click **Triggers** (clock icon) in the left sidebar
2. Find the `checkNewEmails` trigger and click the three-dot menu → **Delete trigger**

---

## Files

| File | Description |
|---|---|
| `MentorshipInboxAgent.gs` | The raw Apps Script file |
| `MentorshipInboxAgent_Code.md` | The same code in a copyable markdown format |
| `README.md` | This file |
| `mentorship-inbox-agent.jsx` | The companion React UI (Claude.ai artifact) for manually browsing and drafting from within Claude |

---

## Architecture Notes

- **Classification** is done with regex pattern matching — no API call needed, instant and reliable
- **Draft generation** uses `claude-sonnet-4-20250514` with category-specific system prompts tailored to the tone of each email type
- **Deduplication** is handled via Script Properties, storing a JSON map of processed message IDs with timestamps, capped at 2,000 entries
- **Rate limiting** — a 1-second sleep between emails prevents hitting Anthropic API rate limits during large backfill runs
