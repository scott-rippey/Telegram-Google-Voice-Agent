# Telegram Voice → Google Tasks & Calendar Assistant — setup project

This project installs a ready-made **n8n** workflow: a Telegram bot that turns voice or text
into Google Tasks, Calendar events (invites, Google Meet links, recurring events, availability
checks), contact lookups, and email — driven by **Claude + Whisper**, behind an
Approve/Cancel confirmation gate.

The **sanitized workflow JSONs live in `docs/`** (all credentials stripped; personal data
replaced with placeholders). **Your job, Claude:** help me import them into my n8n instance
and wire everything up. Use the **n8n-cli skill** for the mechanical parts; guide me through
the parts only I can do (creating OAuth credentials in the browser).

## Files in `docs/`
- `voice-assistant-MAIN.json` — the bot: Telegram trigger → Whisper → Claude agent → tools. **Import this LAST.**
- `calendar-create-helper.json` — creates events (invites + Meet + recurrence) via the Calendar API
- `calendar-list-helper.json` — lists events over a wide date window
- `availability-check-helper.json` — checks a proposed slot for conflicts
- `contacts-search-helper.json` — name → email from saved Google Contacts + recent Gmail (fuzzy)

The MAIN workflow calls the four helpers as **sub-workflow tools**.

## Setup procedure (please walk me through this)

1. **Connect to my n8n.** If there's no `.env` with `N8N_URL` + `N8N_API_KEY`, ask me for my
   instance URL and an API key (n8n → Settings → n8n API), save them, and verify with
   `n8n-cli workflow list`.

2. **Import the four helpers first, then MAIN** — `n8n-cli workflow create --file=docs/<file>.json`
   for each. Record the **new workflow id** each one returns.

3. **Re-link the sub-workflows in MAIN** (⚠️ import does NOT fix this for me). MAIN has four
   `@n8n/n8n-nodes-langchain.toolWorkflow` nodes whose `parameters.workflowId.value` still point
   at the original instance's helper ids. Rewrite each to the NEW id of the helper I just
   imported, then push MAIN back (`n8n-cli workflow update <mainId> --file=...`). You can
   automate this. The mapping:
   - `Google Calendar · Create` → Calendar Create Helper
   - `Google Calendar · List`   → Calendar List Helper
   - `Check Availability`       → Availability Check Helper
   - `Search Contacts`          → Contacts Search Helper

4. **Credentials** — I'll create these in the n8n UI (Credentials → Add). Tell me which to make,
   then which nodes to attach each to (any node with a red triangle). Types needed:
   | Credential | For |
   |---|---|
   | Telegram API | my bot token from @BotFather |
   | Anthropic API | the agent's Claude model |
   | OpenAI API | Whisper voice transcription |
   | Google Tasks OAuth2 | tasks |
   | Google Calendar OAuth2 | events / invites / Meet (MAIN + the calendar/availability helpers) |
   | Gmail OAuth2 | contact search + sending email |
   | Google OAuth2 API *(optional)* | saved-contacts search — see step 7 |

5. **Replace the placeholders** (edit via n8n-cli, or point me to them):
   - `YOUR_GOOGLE_TASKS_LIST_ID` in MAIN's four Google Tasks nodes → my real task-list id
   - the `SELF` email array in Contacts Search Helper → **Rank Contacts** code node → MY email addresses (so the bot never suggests me)
   - *(optional)* MAIN's **AI Agent** system message addresses the user generically as "the user" — personalize it to my name if I want
   - timezone `America/New_York` (workflow settings + a few nodes + the prompt) → mine, if different

6. **Activate MAIN** (this registers the Telegram webhook) and have me message the bot to test —
   e.g. "Add a task to test this bot tomorrow." It should reply with a draft + Approve/Cancel buttons.

7. **(Optional) People API** for saved-contacts search. The `List Contacts` node in the contacts
   helper needs a **Google OAuth2 API** cred with the `contacts.readonly` scope. n8n's one-click
   Google sign-in can't grant that scope, so walk me through creating my own Google Cloud OAuth
   client: enable the **People API** → configure the OAuth consent screen + add the scope →
   create a **Web** OAuth client (on n8n Cloud its redirect URI is
   `https://oauth.n8n.cloud/oauth2/callback`) → make a Google OAuth2 API credential with that
   client id/secret + scope, turn on **"Allow use in HTTP Request node,"** and attach it to
   `List Contacts`. If I skip this, contact search still works from Gmail (the node is set to
   continue-on-error).

## What the assistant does (context)
- **Tasks:** create/list/edit/complete/delete Google Tasks from plain language.
- **Calendar:** create timed events; invite people (emails them) + attach Google Meet; recurring
  events from natural language; availability/conflict checks before booking; move/delete events
  incl. whole recurring series; time-blocking = a guest-less event.
- **Contacts:** resolve a name → email from saved Google Contacts + recent Gmail, fuzzy-matched
  (handles voice slips like "James"→"Jaymes", nicknames, spelling variants).
- **Email:** compose + send via Gmail — give it the gist, it writes subject/body, you approve.
- **Everywhere:** it drafts first and waits for explicit approval (tap a button or type "yes")
  before creating/changing/deleting/sending. Voice and text both work.

## Design notes / gotchas (these are intentional — don't "fix" them)
- **n8n Cloud OAuth redirect** is the central `https://oauth.n8n.cloud/oauth2/callback`, NOT the
  instance `/rest/oauth2-credential/callback` path — the wrong one causes `redirect_uri_mismatch`.
- **Contacts helper:** the Gmail node uses `simple:false` with a low `limit` (full message bodies
  are heavy — too many OOM a cloud worker); `alwaysOutputData:true` keeps the chain alive when a
  Gmail search returns 0 hits; saved contacts come from `connections.list` (NOT `searchContacts`,
  which needs a flaky cache warmup and misses just-added contacts); a Levenshtein token matcher
  does the fuzzy matching.
- **Inline buttons:** each tap is a separate execution; `Edit Draft` strips the buttons up front
  and uses an error-output "lock" (re-editing to identical text → Telegram "not modified") so a
  fast double-tap can't double-act.
- **Calendar list/create/availability** go through HTTP sub-workflows because the Calendar tool
  node ignores date windows and drops some create-time options (invites / Meet / recurrence).
- Agent model is set in the **Anthropic Chat Model** node — swap it for whatever Claude model I
  have access to.
