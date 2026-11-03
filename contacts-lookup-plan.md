# Contacts lookup for email drafts (later step, plan only)

Status: not started. Pick up after the current batch; nothing here is built yet.

## Problem

"Draft an email to Samar" opens a draft with an empty To. The interpreter only knows the
words spoken, so it either leaves `to` empty or guesses an address. Guessing is now blocked:
`spokenRecipient` in `agent/src/core/commands/emailRules.ts` keeps an address only when the
whole of it (local part and domain) was said, and otherwise clears `to` but keeps the draft.

Goal: turn a spoken name into an address from the person's own macOS Contacts, on the Mac.

## Flow

1. Interpreter returns `draftEmail` with a new `toName` (the name as spoken, e.g. "Samar")
   and `to: null`. It never returns an address it was not told. The validator keeps `toName`
   only if those words appear in the transcript, like `click` labels.
2. Agent asks the Swift helper to look the name up in Contacts.
3. Core decides the outcome from the helper's matches:
   - **One match with one email** → fill To, open the draft.
   - **Several matches, or one contact with several emails** → `needsChoice` with the
     existing "Which one?" sheet on the phone. Each choice label is `Name · address`
     (fits `MAX_CHOICE_LABEL_CHARS` = 80; truncate the name, never the address). Picking one
     opens the draft with that address.
   - **None** → open the draft with an empty To and speak "I couldn't find Samar in your
     contacts".
   - **Permission not granted** → draft with empty To, spoken line points to Settings.

## Pieces

| Where | Change |
|---|---|
| `shared/protocol` | `toName: string \| null` on the `draftEmail` interpret action. Additive. Ask before touching `shared`. |
| `web` interpret prompt | "Put a name in `toName`; `to` only for an address spelled out. Never invent an address." |
| `agent/src/core/commands` | `contactOutcome(matches)` → one / many / none. `emailRules` checks `toName` was spoken. Pure, tested. |
| Swift helper | New verb `contacts`: `{ name }` → `[{ name, emails[] }]` via `CNContactStore` with `predicateForContacts(matchingName:)`, keys: given name, family name, nickname, emails. No ranking in Swift. |
| Helper protocol (`core/helper/helperProtocol.ts`) | Zod schema for the request and reply. |
| `shell/commands` | Draft runner calls the lookup before `runWebIntent`. Choice goes through `ChoiceBook` with no overlay to clear. |
| Permissions page | New row "Contacts", status from `CNContactStore.authorizationStatus(for: .contacts)`, button requests access. Optional: commands still work without it. |
| Packaging | `NSContactsUsageDescription` in the agent's Info.plist (`forge.config.mts` `extendInfo`), plus the `com.apple.security.personal-information.addressbook` entitlement for the hardened runtime. The helper runs inside the agent bundle, so the prompt names Ahoy. |

## Rules this touches

- **Six helper verbs.** CLAUDE.md fixes the helper at six verbs. `contacts` would be the
  seventh. Needs your yes and a CLAUDE.md edit before building. The helper still decides
  nothing: it returns matches, and core picks the outcome.
- **Privacy.** Contacts never leave the Mac. They are not sent to `web`, the interpreter or
  ElevenLabs, and never logged. The only thing that leaves is the choice labels for the
  phone's "Which one?" sheet, inside the encrypted phone link (encryption sits above
  transport, so a hosted relay only sees ciphertext). Spoken lines never read an address
  aloud. "I couldn't find Samar" uses only the spoken name, which the server already
  transcribed.
- **Ambiguous → ask, never guess.** Two contacts called Samar always ask.

## Limitations

- Contacts that exist only in Gmail's web contacts won't match. Google contacts synced into
  macOS Contacts through Internet Accounts do match.
- Name matching is Apple's (`matchingName`), not fuzzy. A misheard name ("Samaar") misses
  and gets the "couldn't find" line.
- One recipient only. "Email Samar and Jake" stays out of scope.

## Effort

About 2–3 days.

- Protocol field + prompt + validator: 0.5 day
- Helper verb, permission check, packaging: 1 day (including signing and a fresh-permission run)
- Core outcome + runner + choice wiring: 0.5 day
- Permissions page row + copy: 0.5 day
- Prompt eval and manual pass: 0.5 day

## Test plan

Automated (`node:test`, `agent/src/core`):

- `contactOutcome`: no matches → none; one contact with one email → fill; one contact with
  two emails → choice; two contacts → choice; choice labels stay under 80 characters.
- Validator: `toName` not in the transcript → cleared; interpreter `to` not fully spoken →
  null (already covered by `spokenRecipient`).
- Protocol: the interpret response schema accepts `toName` and still parses old replies
  without it.

Prompt eval (scratchpad script against the real prompt, 3 runs each): "Draft an email to
Samar", "Email Jake saying I'm late", "Email jake at gmail dot com" → name in `toName`,
`to` null unless spelled out, never an invented address.

Manual on the Mac and phone:

1. Fresh install, Contacts not granted → draft opens with empty To, spoken hint, Permissions
   page shows the Contacts row.
2. Grant → one unique contact → To filled.
3. Two contacts with the same first name → "Which one?" on the phone shows name and address;
   picking one fills To; letting it expire opens nothing.
4. Unknown name → empty To plus "I couldn't find … in your contacts".
5. A contact that only exists in Gmail web → no match (expected).
6. Check Mac logs: no names or addresses anywhere.
