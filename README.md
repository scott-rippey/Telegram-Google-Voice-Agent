# Voice → Google Tasks, Calendar & Email Assistant (n8n)

A Telegram bot that turns your **voice notes or typed messages** into action. Talk to it
in plain English ("schedule a call with Dana next Tuesday at 2 and add a Meet link") and
it drafts what it's about to do, shows you **Approve / Cancel buttons**, and only acts
once you tap Approve.

Under the hood it's an n8n workflow driving **Claude** (the agent's brain) + **Whisper**
(voice transcription), wired to your Google Tasks, Google Calendar, Gmail, and Google
Contacts.

---

## What it can do

**Tasks (Google Tasks)**
- Create tasks with due dates from a ramble ("remind me to renew the registration by Friday")
- List, edit, complete, and delete tasks by description ("mark the bank task done")

**Calendar (Google Calendar)**
- Create timed events; **invite people** (it emails them) and attach a **Google Meet** link
- **Recurring events** from natural language ("every Monday at 9", "weekdays at 8 for 2 weeks")
- **Availability check** — warns you before double-booking and offers a free alternative
- List / move / delete events, including **whole recurring series** in one shot
- **Time-blocking** is just an event with no guests ("block 9–11 Friday for deep work")

**Contacts (Gmail + Google Contacts)**
- Resolve a **name → email** so you can invite/email by name. Searches both your **saved
  Google Contacts** and recent email, with **fuzzy matching** (handles voice slips like
  "James" → "Jaymes", nicknames, and Catherine/Katherine-style spellings)

**Email (Gmail)**
- Compose and **send email** — give it the gist and it writes the subject + body, shows you
  the draft, and sends on approval. Resolves recipients by name like invites do.

**How it behaves**
- **Confirmation gate on everything**: it always drafts first and waits for your approval
  (tap a button or type "yes") before creating, changing, deleting, or sending anything.
- Voice **and** text both work. Replies are short and Telegram-friendly.

### Example things to say
- "Add a task to call the dentist Monday."
- "Schedule a planning call with Dana Thursday 2–3pm, invite her, and add a Meet link."
- "Block 8–9am every weekday for focus time."
- "Am I free Friday at 10?"
- "Move the dentist appointment to 3pm." / "Delete the whole standup series."
- "Email Dana a quick thank-you for today's call and say I'll send the proposal Friday."

---

## How it's built (5 workflows)

The main workflow calls four small **sub-workflows** ("helpers") as tools. They exist to
work around two n8n quirks: the Calendar `getAll` node ignores date windows, and a few
create-time options get dropped by the tool node — so those go through direct API calls.

| Workflow | Role |
|---|---|
| `voice-assistant-MAIN.json` | The bot: Telegram trigger → Whisper → Claude agent → all the tools |
| `calendar-create-helper.json` | Creates events (invites + Meet + recurrence) via the Calendar API |
| `calendar-list-helper.json` | Lists events over a wide date window |
| `availability-check-helper.json` | Checks a proposed slot for conflicts |
| `contacts-search-helper.json` | Name → email from saved Contacts + recent Gmail |

---

## Prerequisites

- An **n8n** instance (n8n Cloud or self-hosted).
- A **Telegram bot** — create one with [@BotFather](https://t.me/BotFather), copy the token.
- **Anthropic API key** (the agent's model) — https://console.anthropic.com
- **OpenAI API key** (Whisper voice transcription) — https://platform.openai.com
- A **Google account** with Calendar, Tasks, and Gmail.

---

## Fastest path — let an AI agent install it for you

This repo ships a `CLAUDE.md` so an AI coding agent (Claude Code, Cursor, Windsurf) can import
the workflows and walk you through setup. The tooling comes straight from n8n:

```bash
npm install -g @n8n/cli         # 1. the official n8n CLI (gives you the `n8n-cli` command)
n8n-cli login                   # 2. connect it to your n8n instance (saves URL + API key)
n8n-cli skill install --global  # 3. install the n8n skill into your agent
#    (default target is Claude Code; add --target=cursor or --target=windsurf for those)
```

Then open this folder in your agent and say **"set this up."** It reads `CLAUDE.md` and uses
the CLI to import the five workflows, re-link them, and guide you through credentials.

Prefer to do it by hand? The manual n8n-UI steps below need no CLI or agent.

---

## Setup

### 1. Import the workflows (helpers first)
In n8n: **Workflows → Import from File**, and import all five. **Do the four helpers
first, then the main one** — the main workflow links to the helpers, and it's easier to
re-link once they already exist.

### 2. Create the credentials
In n8n → **Credentials → Add credential**, create one of each below. (For the Google
OAuth ones on n8n Cloud, you can usually just click **"Sign in with Google."**)

| Credential type | What it's for |
|---|---|
| **Telegram API** | Your bot token from BotFather |
| **Anthropic API** | The Claude agent model |
| **OpenAI API** | Whisper voice transcription |
| **Google Tasks OAuth2 API** | Reading/writing tasks |
| **Google Calendar OAuth2 API** | Reading/writing events (+ invites/Meet) |
| **Gmail OAuth2** | Contact search + sending email |
| **Google OAuth2 API** *(optional)* | Saved-contacts search — see step 6 |

### 3. Attach credentials to the nodes
Open each workflow; any node with a red triangle needs its credential picked from the
dropdown:
- **MAIN**: Telegram (Trigger, Answer Callback, Edit Draft, Send Reply, Send Reply + Buttons),
  Anthropic (Anthropic Chat Model), OpenAI (Transcribe), Google Tasks (the 4 Tasks nodes),
  Google Calendar (Update / Delete / Get), Gmail (Send Email).
- **Helpers**: Calendar helpers → Google Calendar; Contacts helper → Gmail (Search Gmail) +
  Google OAuth2 (List Contacts, optional).

### 4. Re-link the sub-workflows in MAIN ⚠️ *(easy to miss)*
Imported sub-workflow links point at the original instance, so open **MAIN** and re-select
the helper in each of these four tool nodes (click the node → Workflow dropdown → pick the
matching helper you imported):
- **Google Calendar · Create** → Calendar Create Helper
- **Google Calendar · List** → Calendar List Helper
- **Check Availability** → Availability Check Helper
- **Search Contacts** → Contacts Search Helper

### 5. Replace the placeholders
- **Task list**: in MAIN's four `Google Tasks` nodes, the task list shows
  `YOUR_GOOGLE_TASKS_LIST_ID` — pick your real list (e.g. "My Tasks") from the dropdown.
- **Your own emails**: in the **Contacts Search Helper → Rank Contacts** node, the code has a
  `SELF` list of placeholder emails (`you@example.com`, …). Replace them with *your* email
  addresses so the bot never suggests you as a contact.
- **Your name** *(optional)*: MAIN's **AI Agent** system message addresses you generically as
  "the user" — personalize it to your name if you like.
- **Timezone**: it's set to `America/New_York` in the workflow settings, several nodes, and
  the prompt. Change it if you're elsewhere.

### 6. *(Optional)* Saved-contacts search via the People API
Contact lookup works out of the box from **recent Gmail**. To *also* search your **saved
Google Contacts**, the `List Contacts` node needs a **Google OAuth2 API** credential with
the `contacts.readonly` scope. n8n's one-click Google sign-in can't grant that scope, so you
must create your own Google Cloud OAuth client:
1. Google Cloud Console → new project → enable the **People API**.
2. Configure the **OAuth consent screen**; add scope `…/auth/contacts.readonly`.
3. Create an **OAuth client (Web application)**; add your n8n OAuth redirect URL to its
   *Authorized redirect URIs* (n8n shows the exact URL in the credential editor — on n8n
   **Cloud** it's `https://oauth.n8n.cloud/oauth2/callback`).
4. In n8n, create a **Google OAuth2 API** credential with that client ID/secret + the scope,
   connect it, and turn on **"Allow use in HTTP Request node."** Attach it to `List Contacts`.

If you skip this, the helper safely falls back to Gmail-only contact search (the People node
is set to continue-on-error).

### 7. Activate it (this connects your bot)
In n8n, open the main workflow (**voice-assistant-MAIN**) and flip it to **Active** (toggle,
top-right). On activation, n8n automatically registers a webhook with Telegram — pointing your
BotFather bot's incoming messages at this workflow. You never set up a webhook by hand. It must
**stay Active** to receive messages (deactivate it, or re-import without re-activating, and the
bot goes silent). Then just message your bot — anything from **Example things to say** above. 🎉

---

## Notes
- Everything is gated behind your approval, but **email and calendar invites are real and go
  out** once approved — they can't be un-sent.
- The agent model is set to a current **Claude** model in the *Anthropic Chat Model* node;
  swap it for whatever you have access to.
- Built with n8n + Claude + Whisper. Have fun. 🤖
