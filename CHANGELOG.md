# Changelog

## v1.2.0

Adds email reading — the assistant can now read your inbox, not just send.

- **Read Emails** — a new READ-ONLY tool: ask the bot "do I have any unread emails?",
  "summarize my latest 2 emails", or "did the title company email me?" The agent sets a
  Gmail search query (`is:unread`, `from:…`, `subject:…`, `newer_than:2d`, or empty for most
  recent) plus how many to fetch, and summarizes concisely. Because reading is safe and
  reversible, it runs with **no confirmation gate** (unlike Send Email).
- **No new credential and no new helper** — it's a native `gmailTool` inside MAIN and reuses
  the same Gmail OAuth2 connection your Send Email already uses. Still **8 workflows** (MAIN
  + 7 helpers); the agent now has 16 tools (was 15).

## v1.1.0

Adds photo input, a security gate, and full contact management.

- **Access gate** — the bot now responds only to your own Telegram account (the `Authorized?`
  node, fail-closed); everyone else is silently ignored. Set `YOUR_TELEGRAM_USER_ID` in MAIN.
- **Photo / vision capture** (GPT‑4o) — send a photo: a whiteboard / handwritten list becomes
  **Tasks** (or timed events), and a **business card** becomes a **Google Contact** (it infers the
  company from the email/website domain when the card doesn't print it). Mirrors the existing
  voice→Whisper path; the Claude agent still makes every decision.
- **Contacts: full CRUD** — added **create, edit, and delete** for saved Google Contacts (was
  search-only). New helpers: `contact-create-helper`, `contact-update-helper`,
  `contact-delete-helper`. Search now returns each saved contact's `resourceName` so edits/deletes
  can target it, and saved contacts are no longer hidden when they share one of your own email
  addresses (the SELF filter now only suppresses Gmail-only matches).
- The OpenAI key now powers **both** Whisper (voice) and GPT‑4o (photo vision).
- **8 workflows** total (MAIN + 7 helpers); 7 sub-workflow re-links on import.

## v1.0.0

Initial public release.

- Telegram **voice/text → Claude agent →** Google Tasks, Calendar, and Gmail.
- **Calendar:** timed events, invites, Google Meet links, recurring events, availability checks,
  move/delete (including whole recurring series).
- **Contacts:** name → email lookup from saved Google Contacts + recent Gmail (fuzzy-matched).
- **Email:** compose + send via Gmail.
- **Approve/Cancel confirmation gate** on every create/change/delete/send.
