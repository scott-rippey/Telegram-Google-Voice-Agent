# Telegram Voice/Photo → Google Tasks, Calendar, Contacts & Email Assistant — setup project

This project installs a ready-made **n8n** workflow: a Telegram bot that turns voice, text, **or
photos** into Google Tasks, Calendar events (invites, Google Meet links, recurring events,
availability checks), Google Contacts (full create/edit/delete), and email — driven by
**Claude + Whisper + GPT‑4o vision**, behind an Approve/Cancel confirmation gate, and locked to
one Telegram account.

The **sanitized workflow JSONs live in `docs/`** (all credentials stripped; personal data
replaced with placeholders). **Your job, Claude:** help me import them into my n8n instance
and wire everything up. Use the **n8n-cli skill** for the mechanical parts; guide me through
the parts only I can do (creating OAuth credentials in the browser).

## Files in `docs/`
- `voice-assistant-MAIN.json` — the bot: Telegram trigger → (voice→Whisper / photo→GPT‑4o vision) → Claude agent → tools. **Import this LAST.**
- `calendar-create-helper.json` — creates events (invites + Meet + recurrence) via the Calendar API
- `calendar-list-helper.json` — lists events over a wide date window
- `availability-check-helper.json` — checks a proposed slot for conflicts
- `contacts-search-helper.json` — name → email from saved Google Contacts + recent Gmail (fuzzy); returns each saved contact's resourceName (id) for editing/deleting
- `contact-create-helper.json` — creates a NEW Google Contact via the People API
- `contact-update-helper.json` — edits a saved contact (fetches the live etag, then patches only the changed fields)
- `contact-delete-helper.json` — deletes a saved contact

The MAIN workflow calls the **seven helpers** as **sub-workflow tools**.

## Setup procedure (please walk me through this)

1. **Check the CLI, then connect to my n8n.** This uses n8n's official CLI. If
   `n8n-cli --version` fails, tell me to install it: `npm install -g @n8n/cli` (and
   `n8n-cli skill install` pulls this very skill — the official n8n one, self-contained).
   Then connect: if there's no `.env` with `N8N_URL` + `N8N_API_KEY`, run `n8n-cli login`
   (or ask me for my instance URL + an API key from n8n → Settings → n8n API), then verify
   with `n8n-cli workflow list`.

2. **Import the seven helpers FIRST.** Run `n8n-cli workflow create --file=docs/<helper>.json`
   for each helper file (every `*-helper.json` in `docs/`). Capture the new id each one returns
   into a map keyed by the placeholder it corresponds to (the placeholders are the literal
   strings in MAIN's `workflowId.value` — see the mapping below). Heads-up: `--jq '.id'` returns
   the id wrapped in quotes — strip them before reusing the id, or a later `workflow activate`
   404s.

   | Helper file                      | Placeholder in MAIN to replace             |
   |----------------------------------|--------------------------------------------|
   | `calendar-create-helper.json`    | `REPLACE_WITH_CALENDAR_CREATE_HELPER_ID`   |
   | `calendar-list-helper.json`      | `REPLACE_WITH_CALENDAR_LIST_HELPER_ID`     |
   | `availability-check-helper.json` | `REPLACE_WITH_AVAILABILITY_CHECK_HELPER_ID`|
   | `contacts-search-helper.json`    | `REPLACE_WITH_CONTACTS_SEARCH_HELPER_ID`   |
   | `contact-create-helper.json`     | `REPLACE_WITH_CONTACT_CREATE_HELPER_ID`    |
   | `contact-update-helper.json`     | `REPLACE_WITH_CONTACT_UPDATE_HELPER_ID`    |
   | `contact-delete-helper.json`     | `REPLACE_WITH_CONTACT_DELETE_HELPER_ID`    |

3. **Rewrite MAIN to point at the new helper ids, THEN import MAIN.** MAIN's seven
   `@n8n/n8n-nodes-langchain.toolWorkflow` nodes ship with the `REPLACE_WITH_*` placeholders
   above in `parameters.workflowId.value`. Before importing, edit `docs/voice-assistant-MAIN.json`
   and replace each placeholder with the matching new helper id you captured in step 2 (a plain
   string-replace — the placeholders are unique). Then `n8n-cli workflow create --file=docs/voice-assistant-MAIN.json`.
   No manual UI re-linking needed.

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
   | Google OAuth2 API *(for contacts)* | saved-contacts search + create/edit/delete — see step 7 |

5. **Replace the placeholders** (edit via n8n-cli, or point me to them):
   - **`YOUR_TELEGRAM_USER_ID` in MAIN's `Authorized?` node** → my numeric Telegram user id. This
     gate is what locks the bot to ME — until it's set correctly the bot ignores **everyone**
     (it fails closed). To find my id: message **@userinfobot** on Telegram, or read `message.from.id`
     from any execution. It's a number like `123456789` (a plain number, not quoted).
   - `YOUR_GOOGLE_TASKS_LIST_ID` in MAIN's four Google Tasks nodes → my real task-list id
   - the `SELF` email array in Contacts Search Helper → **Rank Contacts** code node → MY email
     addresses (so the bot never suggests me from random mail)
   - *(optional)* MAIN's **AI Agent** system message addresses the user generically as "the user" —
     personalize it to my name if I want
   - timezone `America/New_York` (workflow settings + a few nodes + the prompt) → mine, if different

6. **Activate MAIN.** Flipping it Active makes n8n automatically register the Telegram webhook —
   pointing the @BotFather bot's messages at this n8n instance (no manual webhook setup), and it
   must stay Active to keep receiving. Then have me message the bot to test (and try sending a photo
   of a sticky note or a business card).

7. **Google People API** for contacts. The contacts helpers all use one **Google OAuth2 API** cred:
   - `List Contacts` (search helper) reads my address book.
   - `Create Contact` (create helper) adds a new contact.
   - `Get Contact` + `Patch Contact` (update helper) edit a contact (Google requires the live etag,
     so it GETs the contact first, then PATCHes only the changed fields).
   - `Delete Contact` (delete helper) removes a contact.

   **Scope:** search alone works with `contacts.readonly`, but create/edit/delete need the
   read-WRITE scope **`https://www.googleapis.com/auth/contacts`** (one scope, covers read + write —
   use it for all of them). n8n's one-click Google sign-in can't grant it, so walk me through
   creating my own Google Cloud OAuth client: enable the **People API** → configure the OAuth
   consent screen + add the `…/auth/contacts` scope → create a **Web** OAuth client (on n8n Cloud
   its redirect URI is `https://oauth.n8n.cloud/oauth2/callback`) → make a Google OAuth2 API
   credential with that client id/secret + scope, turn on **"Allow use in HTTP Request node,"** and
   attach it to every People API node above.

   ⚠️ **Editing the scope text is not enough** — I must COMPLETE the OAuth **reconnect** (on a
   desktop browser, popups allowed) so Google mints a fresh token carrying the new scope; otherwise
   create/edit/delete return **403 Forbidden** even though the scope looks right. If I skip the
   People API entirely, contact *search* still works from Gmail, but saved-contact features won't.

## What the assistant does (context)
- **Tasks:** create/list/edit/complete/delete Google Tasks from plain language.
- **Calendar:** create timed events; invite people (emails them) + attach Google Meet; recurring
  events from natural language; availability/conflict checks before booking; move/delete events
  incl. whole recurring series; time-blocking = a guest-less event.
- **Contacts:** full CRUD — resolve a name → email from saved Google Contacts + recent Gmail,
  fuzzy-matched (handles voice slips like "James"→"Jaymes", nicknames, spelling variants); and
  **create, edit, and delete** saved contacts.
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
- **Contacts helper:** the Gmail node uses `simple:false` with a low `limit` (full message bodies
  are heavy — too many OOM a cloud worker); `alwaysOutputData:true` keeps the chain alive when a
  Gmail search returns 0 hits; saved contacts come from `connections.list` (NOT `searchContacts`,
  which needs a flaky cache warmup and misses just-added contacts); a Levenshtein token matcher
  does the fuzzy matching. **Saved contacts are returned as distinct entities keyed by their
  resourceName and are NOT removed by the SELF filter** — so a contact you deliberately saved is
  findable (and editable/deletable) even if it uses one of your own email addresses; the SELF
  filter only suppresses *Gmail-only* correspondents.
- **Contact edit needs the live etag:** People `updateContact` requires it, so the update helper
  GETs the contact first, then PATCHes only the named fields (unmasked fields are preserved).
- **Inline buttons:** each tap is a separate execution; `Edit Draft` strips the buttons up front
  and uses an error-output "lock" (re-editing to identical text → Telegram "not modified") so a
  fast double-tap can't double-act.
- **Calendar list/create/availability and all contact writes** go through HTTP sub-workflows
  because the native tool nodes ignore date windows and drop options (invites / Meet / recurrence /
  `conferenceData`), and the People API has no create/update/delete tool node.
- Agent model is set in the **Anthropic Chat Model** node — swap it for whatever Claude model I
  have access to.
