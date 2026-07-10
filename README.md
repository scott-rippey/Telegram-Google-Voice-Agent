# Voice / Photo → Google Tasks, Calendar, Contacts & Email Assistant (n8n)

**Version 1.2.0** · [Changelog](CHANGELOG.md)

A Telegram bot that turns your **voice notes, typed messages, or photos** into action. Talk to it
in plain English ("schedule a call with Dana next Tuesday at 2 and add a Meet link"), or **snap a
photo** of a whiteboard or a business card — it drafts what it's about to do, shows you **Approve /
Cancel buttons**, and only acts once you tap Approve.

Under the hood it's an n8n workflow driving **Claude** (the agent's brain), **Whisper** (voice
transcription), and **GPT‑4o vision** (reading photos), wired to your Google Tasks, Google
Calendar, Gmail, and Google Contacts. It's locked to your own Telegram account.

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

**Contacts (Gmail + Google Contacts) — full CRUD**
- Resolve a **name → email** so you can invite/email by name. Searches both your **saved
  Google Contacts** and recent email, with **fuzzy matching** (handles voice slips like
  "James" → "Jaymes", nicknames, and Catherine/Katherine-style spellings)
- **Create, edit, and delete** saved contacts ("add a cell number to Jane", "delete the old vendor")

**Email (Gmail)**
- Compose and **send email** — give it the gist and it writes the subject + body, shows you
  the draft, and sends on approval. Resolves recipients by name like invites do.

**Photos (GPT‑4o vision)**
- Send a **photo** and it reads it for you:
  - a **whiteboard / sticky note / handwritten or printed to-do list** → becomes **Tasks** (or
    timed Calendar events for anything with a time)
  - a **business card** → becomes a new **Google Contact**. It even fills in the company from the
    email or website domain when the card doesn't spell it out — and leaves it blank rather than
    guessing for personal/generic addresses.
- No caption needed — it figures out whether the photo is notes or a card on its own.

**How it behaves**
- **Locked to you:** the bot only responds to your own Telegram account; anyone else who messages
  it is silently ignored.
- **Confirmation gate on everything**: it always drafts first and waits for your approval
  (tap a button or type "yes") before creating, changing, deleting, sending, or saving anything.
- Voice, text, **and** photos all work. Replies are short and Telegram-friendly.

### Example things to say (or send)
- "Add a task to call the dentist Monday."
- "Schedule a planning call with Dana Thursday 2–3pm, invite her, and add a Meet link."
- "Block 8–9am every weekday for focus time."
- "Am I free Friday at 10?"
- "Move the dentist appointment to 3pm." / "Delete the whole standup series."
- "Email Dana a quick thank-you for today's call and say I'll send the proposal Friday."
- "Change the title on my Dana Reed contact to VP." / "Delete the old vendor contact."
- *(send a photo of a whiteboard)* → it drafts the tasks
- *(send a photo of a business card)* → it drafts the contact to save

---

## How it's built (8 workflows)

The main workflow calls seven small **sub-workflows** ("helpers") as tools. They exist to work
around a few n8n quirks: the Calendar `getAll` node ignores date windows, some create-time options
get dropped by the tool node, and the People API has no contact create/update/delete tool node — so
those go through direct API calls.

| Workflow | Role |
|---|---|
| `voice-assistant-MAIN.json` | The bot: Telegram trigger → Whisper / GPT‑4o vision → Claude agent → all the tools |
| `calendar-create-helper.json` | Creates events (invites + Meet + recurrence) via the Calendar API |
| `calendar-list-helper.json` | Lists events over a wide date window |
| `availability-check-helper.json` | Checks a proposed slot for conflicts |
| `contacts-search-helper.json` | Name → email from saved Contacts + recent Gmail (returns each contact's id) |
| `contact-create-helper.json` | Creates a new Google Contact via the People API |
| `contact-update-helper.json` | Edits a saved contact (fetches the etag, then patches changed fields) |
| `contact-delete-helper.json` | Deletes a saved contact |

---

## Prerequisites

- An **n8n** instance (n8n Cloud or self-hosted).
- A **Telegram bot** — create one with [@BotFather](https://t.me/BotFather), copy the token.
- **Anthropic API key** (the agent's model) — https://console.anthropic.com
- **OpenAI API key** (Whisper voice transcription + GPT‑4o photo vision) — https://platform.openai.com
- A **Google account** with Calendar, Tasks, Gmail, and Contacts.

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
the CLI to import the eight workflows, re-link them, and guide you through credentials.

Prefer to do it by hand? The manual n8n-UI steps below need no CLI or agent.

---

## Setup

### 1. Import the workflows (helpers first)
In n8n: **Workflows → Import from File**, and import all eight. **Do the seven helpers
first, then the main one** — the main workflow links to the helpers, and it's easier to
re-link once they already exist.

### 2. Create the credentials
In n8n → **Credentials → Add credential**, create one of each below. (For the Google
OAuth ones on n8n Cloud, you can usually just click **"Sign in with Google"** — except the
contacts one, see step 6.)

| Credential type | What it's for |
|---|---|
| **Telegram API** | Your bot token from BotFather |
| **Anthropic API** | The Claude agent model |
| **OpenAI API** | Whisper voice transcription **and** GPT‑4o photo vision |
| **Google Tasks OAuth2 API** | Reading/writing tasks |
| **Google Calendar OAuth2 API** | Reading/writing events (+ invites/Meet) |
| **Gmail OAuth2** | Contact search + sending email |
| **Google OAuth2 API** *(for contacts)* | Saved-contacts search + create/edit/delete — see step 6 |

### 3. Attach credentials to the nodes
Open each workflow; any node with a red triangle needs its credential picked from the
dropdown:
- **MAIN**: Telegram (Trigger, Answer Callback, Edit Draft, Send Reply, Send Reply + Buttons,
  Get Photo File), Anthropic (Anthropic Chat Model), OpenAI (Transcribe **and** Analyze Image),
  Google Tasks (the 4 Tasks nodes), Google Calendar (Update / Delete / Get), Gmail (Send Email).
- **Helpers**: Calendar helpers → Google Calendar; Contacts Search → Gmail (Search Gmail) +
  Google OAuth2 (List Contacts); Contact Create/Update/Delete → Google OAuth2 (Create Contact /
  Get Contact + Patch Contact / Delete Contact).

### 4. Re-link the sub-workflows in MAIN ⚠️ *(easy to miss)*
MAIN's seven sub-workflow tool nodes ship with placeholder ids like
`REPLACE_WITH_CALENDAR_CREATE_HELPER_ID` (so nothing accidentally calls a foreign instance).
You need to point each one at the helper you just imported. Open **MAIN** and re-select the
helper in each of these seven tool nodes (click the node → Workflow dropdown → pick the
matching helper you imported):
- **Google Calendar · Create** → Calendar Create Helper
- **Google Calendar · List** → Calendar List Helper
- **Check Availability** → Availability Check Helper
- **Search Contacts** → Contacts Search Helper
- **Create Contact** → Contact Create Helper
- **Update Contact** → Contact Update Helper
- **Delete Contact** → Contact Delete Helper

*(The agent-driven flow above does this automatically — it captures each helper's new id on
import and string-replaces the placeholders in MAIN before importing MAIN.)*

### 5. Replace the placeholders
- **Your Telegram id (required — this locks the bot to you):** MAIN's **`Authorized?`** node has
  `YOUR_TELEGRAM_USER_ID`. Set it to your numeric Telegram user id (message
  [@userinfobot](https://t.me/userinfobot) to get it — a number like `123456789`). Until it's set,
  the bot ignores everyone, including you.
- **Task list**: in MAIN's four `Google Tasks` nodes, the task list shows
  `YOUR_GOOGLE_TASKS_LIST_ID` — pick your real list (e.g. "My Tasks") from the dropdown.
- **Your own emails**: in the **Contacts Search Helper → Rank Contacts** node, the code has a
  `SELF` list of placeholder emails (`you@example.com`, …). Replace them with *your* email
  addresses so the bot never suggests you from random mail. *(Saved contacts you create are still
  fully findable even if they use one of these addresses — the SELF list only filters Gmail-only
  matches.)*
- **Your name** *(optional)*: MAIN's **AI Agent** system message addresses you generically as
  "the user" — personalize it to your name if you like.
- **Timezone**: it's set to `America/New_York` in the workflow settings, several nodes, and
  the prompt. Change it if you're elsewhere.

### 6. Google Contacts (search + create/edit/delete) via the People API
All the contacts nodes use a **Google OAuth2 API** credential. n8n's one-click Google sign-in
can't grant the contacts scope, so create your own Google Cloud OAuth client:
1. Google Cloud Console → new project → enable the **People API**.
2. Configure the **OAuth consent screen**; add the scope. Search-only works with
   `…/auth/contacts.readonly`, but to also **create/edit/delete** contacts use the read-write scope
   **`https://www.googleapis.com/auth/contacts`** — it covers both, so just use it.
3. Create an **OAuth client (Web application)**; add your n8n OAuth redirect URL to its
   *Authorized redirect URIs* (n8n shows the exact URL in the credential editor — on n8n
   **Cloud** it's `https://oauth.n8n.cloud/oauth2/callback`).
4. In n8n, create a **Google OAuth2 API** credential with that client ID/secret + the scope,
   **connect it (complete the Google sign-in)**, and turn on **"Allow use in HTTP Request node."**
   Attach it to every People API node: `List Contacts`, `Create Contact`, `Get Contact` +
   `Patch Contact` (update), and `Delete Contact`.

> ⚠️ **Editing the scope text isn't enough** — you must *complete the OAuth reconnect* (on a
> desktop browser, with popups allowed) so Google issues a fresh token carrying the scope.
> Otherwise create/edit/delete fail with **403 Forbidden** even though the scope looks correct.

If you skip the People API, contact *search* still falls back to Gmail-only (that node is set to
continue-on-error), but the saved-contact create/edit/delete features won't work.

### 7. Activate it (this connects your bot)
In n8n, open the main workflow (**voice-assistant-MAIN**) and flip it to **Active** (toggle,
top-right). On activation, n8n automatically registers a webhook with Telegram — pointing your
BotFather bot's incoming messages at this workflow. You never set up a webhook by hand. It must
**stay Active** to receive messages (deactivate it, or re-import without re-activating, and the
bot goes silent). Then just message your bot — anything from **Example things to say** above, or
send it a photo. 🎉

---

## Notes
- The bot is **locked to your Telegram account** by the `Authorized?` gate (step 5) — set it, or
  the bot won't respond to you either.
- Everything is gated behind your approval, but **email and calendar invites are real and go
  out** once approved — they can't be un-sent.
- **Photos** are read by GPT‑4o (the OpenAI key); voice is transcribed by Whisper (same key). The
  Claude agent makes all the decisions — vision just turns the image into text.
- The agent model is set to a current **Claude** model in the *Anthropic Chat Model* node;
  swap it for whatever you have access to.
- Built with n8n + Claude + Whisper + GPT‑4o vision. Have fun. 🤖
