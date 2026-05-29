# Telegram Voice/Photo → Google Tasks, Calendar, Contacts & Email Assistant — setup project

This project installs a ready-made **n8n** workflow: a Telegram bot that turns voice, text, **or
photos** into Google Tasks, Calendar events (invites, Google Meet links, recurring events,
availability checks), Google Contacts, and email — driven by **Claude + Whisper + GPT‑4o vision**,
behind an Approve/Cancel confirmation gate, and locked to one Telegram account.

The **sanitized workflow JSONs live in `docs/`** (all credentials stripped; personal data
replaced with placeholders). **Your job, Claude:** help me import them into my n8n instance
and wire everything up. Use the **n8n-cli skill** for the mechanical parts; guide me through
the parts only I can do (creating OAuth credentials in the browser).

## Files in `docs/`
- `voice-assistant-MAIN.json` — the bot: Telegram trigger → (voice→Whisper / photo→GPT‑4o vision) → Claude agent → tools. **Import this LAST.**
- `calendar-create-helper.json` — creates events (invites + Meet + recurrence) via the Calendar API
- `calendar-list-helper.json` — lists events over a wide date window
- `availability-check-helper.json` — checks a proposed slot for conflicts
- `contacts-search-helper.json` — name → email from saved Google Contacts + recent Gmail (fuzzy)
- `contact-create-helper.json` — creates a NEW Google Contact via the People API

The MAIN workflow calls the **five helpers** as **sub-workflow tools**.

## Setup procedure (please walk me through this)

1. **Check the CLI, then connect to my n8n.** This uses n8n's official CLI. If
   `n8n-cli --version` fails, tell me to install it: `npm install -g @n8n/cli` (and
   `n8n-cli skill install` pulls this very skill — the official n8n one, self-contained).
   Then connect: if there's no `.env` with `N8N_URL` + `N8N_API_KEY`, run `n8n-cli login`
   (or ask me for my instance URL + an API key from n8n → Settings → n8n API), then verify
   with `n8n-cli workflow list`.

2. **Import the five helpers first, then MAIN** — `n8n-cli workflow create --file=docs/<file>.json`
   for each. Record the **new workflow id** each one returns. (Heads-up: `--jq '.id'` returns the
   id wrapped in quotes — strip them before reusing the id, or a later `workflow activate` 404s.)

3. **Re-link the sub-workflows in MAIN** (⚠️ import does NOT fix this for me). MAIN has five
   `@n8n/n8n-nodes-langchain.toolWorkflow` nodes whose `parameters.workflowId.value` still point
   at the original instance's helper ids. Rewrite each to the NEW id of the helper I just
   imported, then push MAIN back (`n8n-cli workflow update <mainId> --file=...`). You can
   automate this. The mapping:
   - `Google Calendar · Create` → Calendar Create Helper
   - `Google Calendar · List`   → Calendar List Helper
   - `Check Availability`       → Availability Check Helper
   - `Search Contacts`          → Contacts Search Helper
   - `Create Contact`           → Contact Create Helper

4. **Credentials** — I'll create these in the n8n UI (Credentials → Add). Tell me which to make,
   then which nodes to attach each to (any node with a red triangle). Types needed:
   | Credential | For |
   |---|---|
   | Telegram API | my bot token from @BotFather |
   | Anthropic API | the agent's Claude model |
   | OpenAI API | Whisper voice transcription **and** GPT‑4o photo vision (the `Transcribe` and `Analyze Image` nodes) |
   | Google Tasks OAuth2 | tasks |
   | Google Calendar OAuth2 | events / invites / Meet (MAIN + the calendar/availability helpers) |
   | Gmail OAuth2 | contact search + sending email |
   | Google OAuth2 API *(for contacts)* | saved-contacts search **and** creating contacts — see step 7 |

5. **Replace the placeholders** (edit via n8n-cli, or point me to them):
   - **`YOUR_TELEGRAM_USER_ID` in MAIN's `Authorized?` node** → my numeric Telegram user id. This
     gate is what locks the bot to ME — until it's set correctly the bot ignores **everyone**
     (it fails closed). To find my id: message **@userinfobot** on Telegram, or read `message.from.id`
     from any execution. It's a number like `123456789` (a plain number, not quoted).
   - `YOUR_GOOGLE_TASKS_LIST_ID` in MAIN's four Google Tasks nodes → my real task-list id
   - the `SELF` email array in Contacts Search Helper → **Rank Contacts** code node → MY email
     addresses (so the bot never suggests me)
   - *(optional)* MAIN's **AI Agent** system message addresses the user generically as "the user" —
     personalize it to my name if I want
   - timezone `America/New_York` (workflow settings + a few nodes + the prompt) → mine, if different

6. **Activate MAIN.** Flipping it Active makes n8n automatically register the Telegram webhook —
   pointing the @BotFather bot's messages at this n8n instance (no manual webhook setup), and it
   must stay Active to keep receiving. Then have me message the bot to test (and try sending a photo
   of a sticky note or a business card).

7. **Google People API** for contacts. The contacts helpers use a **Google OAuth2 API** cred:
   - `List Contacts` (in the search helper) reads my address book.
   - `Create Contact` (in the contact-create helper) writes a new contact.

   **Scope:** search alone works with `contacts.readonly`, but **creating** contacts needs the
   read-WRITE scope **`https://www.googleapis.com/auth/contacts`** (one scope, covers read + write —
   use it for both). n8n's one-click Google sign-in can't grant it, so walk me through creating my
   own Google Cloud OAuth client: enable the **People API** → configure the OAuth consent screen +
   add the `…/auth/contacts` scope → create a **Web** OAuth client (on n8n Cloud its redirect URI is
   `https://oauth.n8n.cloud/oauth2/callback`) → make a Google OAuth2 API credential with that
   client id/secret + scope, turn on **"Allow use in HTTP Request node,"** and attach it to both
   `List Contacts` and `Create Contact`.

   ⚠️ **Editing the scope text is not enough** — I must COMPLETE the OAuth **reconnect** (on a
   desktop browser, popups allowed) so Google mints a fresh token carrying the new scope; otherwise
   contact creation returns **403 Forbidden** even though the scope looks right. If I skip the
   People API entirely, contact *search* still works from Gmail, but saving contacts (and saved-
   contact lookups) won't.

## What the assistant does (context)
- **Tasks:** create/list/edit/complete/delete Google Tasks from plain language.
- **Calendar:** create timed events; invite people (emails them) + attach Google Meet; recurring
  events from natural language; availability/conflict checks before booking; move/delete events
  incl. whole recurring series; time-blocking = a guest-less event.
- **Contacts:** resolve a name → email from saved Google Contacts + recent Gmail, fuzzy-matched
  (handles voice slips like "James"→"Jaymes", nicknames, spelling variants); and **save new
  contacts**.
- **Email:** compose + send via Gmail — give it the gist, it writes subject/body, you approve.
- **Photos:** send a photo and GPT‑4o reads it. A whiteboard / handwritten list / to-dos becomes
  **Tasks** (or timed Calendar events); a **business card** becomes a new **Google Contact** — it
  even infers the company from the email or website domain when the card doesn't spell it out
  (and leaves it blank rather than guessing if the domain is a personal/generic one). Same draft-
  then-approve gate.
- **Access control:** the bot responds ONLY to my Telegram user id (the `Authorized?` gate);
  everyone else is silently ignored.
- **Everywhere:** it drafts first and waits for explicit approval (tap a button or type "yes")
  before creating/changing/deleting/sending. Voice, text, and photos all work.

## Design notes / gotchas (these are intentional — don't "fix" them)
- **Access gate:** an `Authorized?` IF node sits right after the Telegram trigger and checks
  `message.from.id` / `callback_query.from.id` == my Telegram id, dead-ending everyone else
  (fail-closed). The trigger fires for ANYONE who messages the bot, so this is what stops a
  stranger from creating/deleting/emailing as me.
- **Vision** is the OpenAI **"Analyze Image"** node (`resource:image, operation:analyze`, model
  `gpt-4o`, `inputType:base64`, `binaryPropertyName:data`, `simplify:true` → output `{content}`).
  It mirrors the Whisper voice step: image → text, then the Claude agent does all the routing
  (a self-labeled `CONTACT` vs `NOTES` marker tells it card-vs-notes). GPT‑4o only turns pixels
  into text; Claude makes every decision.
- **Contacts create** needs the read-WRITE contacts scope; changing the scope text without
  completing the OAuth reconnect leaves a read-only token → 403 on create.
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
- **Calendar list/create/availability and contact create** go through HTTP sub-workflows because
  the native tool nodes ignore date windows and drop options (invites / Meet / recurrence /
  `conferenceData`), and the People API has no create-contact tool node.
- Agent model is set in the **Anthropic Chat Model** node — swap it for whatever Claude model I
  have access to.
