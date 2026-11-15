# Spoken email addresses (plan A1)

Status: built 2026-11-15 (agent 4457a02, d488652; web prompt line and address eval cases).
No schema change: "myself" and "my email" are filled by Mac rules, no AI marker. First item of
the plan: basics before anything new.

## Problem

Saying an email address fails in two places.

1. **Drafts.** `spokenRecipient` (agent `core/commands/emailRules.ts`) keeps `to` only if every
   letter and digit of the local part appears in the letters of the transcript. It fails when:
   - digits are said as words: "one four zero four", "fourteen oh four", "double four"
   - separators are said: "shahwaiz **dot** islam", "underscore"
   - speech-to-text spells an unusual name slightly differently ("Shahwaz")

   The draft then opens with an empty To: "Drafted an email. Add who it goes to".
2. **Typing.** `said(text, transcript)` in `validateActions.ts` compares plain words, so
   "type shahwaizislam1404@gmail.com" is rejected whenever the transcript says "at … dot com".
   Typing a spoken address only works if speech-to-text already wrote it as an address.

There's also no way to say "my email" or "to myself".

## Fix

### 1. One spoken-form normalizer (agent core, pure function)

A new file such as `core/commands/spokenForm.ts` that turns a transcript into its written form,
used only for comparisons:

- Number words to digits: zero / oh / o → 0, one–nine, ten–nineteen, twenty…ninety plus units,
  "double X" and "triple X". Neighbouring digits join: "one four zero four" → 1404.
- "at", "at the rate", "at sign" → `@`; "dot", "period", "point" → `.`; "underscore" → `_`;
  "dash", "hyphen", "minus" → `-`; "plus" → `+`.
- Spelled-out single letters join: "s h a h" → "shah".
- Everything else is unchanged.

### 2. Draft recipient

- The domain must still be spoken (now compared through the normalizer).
- The local part matches exactly after normalizing, with separators optional → keep it. The reply
  is unchanged.
- **Close but not exact** → keep it, and the reply says "Drafted an email to X. Check the
  address, then press Send." "Close" means:
  - the person clearly said an address: the transcript has `@` and the spoken domain after
    normalizing
  - the local part is within a small edit distance (about 2 characters, or at least 85% similar)

  Speech-to-text mishearing a name lands here instead of an empty To.
- **No address attempt** (a bare name, a guessed domain) → clear it, as today. Keep the existing
  "guessed domain" protection and its behaviour.

### 3. Typing

- When the text to type looks like an email address, a URL or a number, compare it against the
  normalized transcript.
- Every other text keeps today's verbatim rule. "type hello at noon" must type "hello at noon".

### 4. "My email", "myself", "me"

- "Draft an email to myself about…" and "type my email" use the account email the Mac already
  has (`appState.accountEmail`).
- **The account email is never sent to the AI.** The AI returns a marker and the Mac fills in the
  address. Propose the exact shape (for example an optional `toSelf` / `typeSelf` field). It must
  be additive and optional; ask before changing `shared`.

### 5. Interpret prompt (web)

- Add one line: in addresses, write spoken digits as digits and spoken separators as symbols, and
  use the self marker for "my email" / "myself".
- Don't add personal addresses to the speech keyterms (privacy).

### 6. Logging

Log why an address was kept, flagged or cleared on the Mac (`exact`, `close`, `none`). Never log
the address or the transcript.

## Bug row

Add the next row to `engineering/bugs.md`:
- I did: said my email address in a draft and in "type …"
- Expected: the address
- Got: an empty To, and typing refused

## Eval additions (`web/scripts/interpretCases.ts`)

Add the phrases from the device checks below, with the expected `to` / `text`.

## Device checks

| # | Say | Expected |
|---|---|---|
| 1 | "draft an email to shahwaiz islam 1404 at gmail dot com about the demo" | To: shahwaizislam1404@gmail.com |
| 2 | "… shahwaiz islam one four zero four at gmail dot com …" | Same |
| 3 | "… shahwaiz islam fourteen oh four at gmail dot com …" | Same |
| 4 | "… s h a h w a i z islam 1404 at the rate gmail dot com …" | Same |
| 5 | "… shahwaiz dot islam at gmail dot com …" | To: shahwaiz.islam@gmail.com |
| 6 | Name misheard (say it fast) | Address filled; reply says to check it |
| 7 | "email Samar about lunch" | No address (unchanged) |
| 8 | "draft an email to myself about groceries" | To: my account email |
| 9 | Form field focused: "type shahwaiz islam 1404 at gmail dot com" | Types the address |
| 10 | "type my email" | Types my account email |
| 11 | "type hello at noon" | Types "hello at noon" |

## Rules for this batch

- Follow `engineering/CLAUDE.md`.
- One commit per item, in this order: normalizer, draft recipient, typing, self address, prompt
  and eval, docs.
- Typecheck, lint and existing tests pass after each item.
- **Never commit a test file.** Throwaway tests stay in the scratchpad.
- Protocol changes are additive and optional; ask before changing `shared`.
- Don't commit until I've seen the summary.

At the end, give me:
- what each item delivered
- the commit list
- what couldn't be tested
- anything weaker than this file
