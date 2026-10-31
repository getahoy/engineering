# Next batch: bug fixes, browser tabs, email drafts

Build all of it in one go, one commit per item, in this order. Typecheck, lint and tests pass
after each item. Don't commit until I've seen the summary.

## 0. Bug log (set up first)

Create `engineering/bugs.md`: a table with **#, date, what I did, what I expected, what
happened, status (open / fixed in <commit>)**. When I report a bug, add a row. Every fix adds a
row to the relevant checks file, so it can't come back.

## 1. Talk button misses "Connected" (blocking)

After a reload, the card says "Connected" but the talk button says "Connect to your Mac to
talk". `useTalkTarget` subscribes to `liveMacLinks` after the card's connection has already
reported "connected" (child effects run before parent effects), so it misses the event, and
the `queueMicrotask` workaround doesn't replay it.

Fix: `liveMacLinks` is the single source of each Mac's state. Both the card and
`useTalkTarget` read it with `useSyncExternalStore` (subscribe plus a getSnapshot returning
the current state). Remove the microtask workaround.
Check row: reload the app → the button is active as soon as the card says Connected.

## 2. Honest message when the AI fails

"Open a new tab in Chrome" → parser unknownApp → the AI call failed → the phone showed "Can't
find an app called a new tab in chrome". When the AI is unreachable, times out or returns
invalid output, reply **"Smart commands aren't working right now"**. Use the parser's message
only when the AI answered with no actions. Log the reason on the Mac (unavailable, timeout,
invalid, none), never the text.

## 3. Server: a bad cookie is a 401, never a 500

A malformed or unreadable cookie header must give `unauthorized`, never a server error. Find
where it throws and handle it.

## 4. Phone: no sign-out on a single unauthorized

Before signing out because a request said unauthorized, confirm with `getSession()`. Only
sign out if the session is really gone.

## 5. Account change wipes the pairing (security)

Store the owning `userId` with the paired Macs. On any sign-in, if the user differs from the
owner, wipe paired Macs, pending unlinks and device keys, the same as the sign-out button.
Check row: sign in as a different account on the same phone → no Macs listed, and the old Mac
can't be controlled.

## 6. Keys: denylist instead of the 25-key allowlist

Allow any combination with at least one modifier, plus the named keys we have, **except** a
denylist the Mac enforces: cmd+Q, cmd+option+Q, cmd+shift+Q, cmd+option+shift+Q,
cmd+option+esc, cmd+delete, cmd+option+delete, shift+cmd+delete, cmd+option+shift+delete, and
any combination with the power or eject key. Return stays gated on the spoken words. The
prompt tells the AI to use app shortcuts it knows (Notion cmd+P search, cmd+N new page, cmd+M
minimize, ctrl+cmd+F full screen, cmd+= zoom in). Validator tests for the denylist.

## 7. Browser tabs (AppleScript, no screen reading)

New intents `closeTab { browser, match }` and `switchTab { browser, match }` for Google
Chrome, Safari, Arc, Brave and Microsoft Edge (Chromium browsers share Chrome's dictionary;
check Arc's).

- Match case-insensitively against tab titles and URLs across all windows.
- Exactly one match → act ("Closed the Grok tab").
- Several → "3 tabs match: Grok – Chat, Grok docs, … Say which one." and do nothing. Choices
  arrive with step 7.
- None → "No tab matching grok in Chrome".
- No browser named → the frontmost browser, else the default browser.
- `match` reaches AppleScript only through `on run argv`.
- Parser phrases: "close the X tab", "go to / switch to the X tab". Also in the AI's action set.
- Tests for the parser and the tab matcher.

## 8. Email drafts (never sends)

New action `draftEmail { to?, subject, body, via: "mail" | "gmail" }`, AI only.

- `via: "gmail"` only when the user says Gmail; otherwise `mail`.
- `mail` opens a `mailto:` URL; `gmail` opens
  `https://mail.google.com/mail/?view=cm&to=…&su=…&body=…`. Everything URL-encoded.
- **It never sends.** No Return or cmd+Return after it, ever. Reply: "Drafted an email to
  jake@… Check it and press Send."
- Validation: `to` must be a valid address. Spoken addresses ("jake bauer at gmail dot com")
  may be normalised by the AI; the letters of the local part must appear in the transcript.
  Subject ≤ 150 characters, body ≤ 1,500.
- The body is AI-written (unlike `type`, which must be verbatim). That's allowed because it's a
  draft the user reviews. Record this in decisions.md.
- "Regarding this" (screen content) isn't supported until step 11. With no clear topic, the AI
  returns none.
- Validator tests.

## Eval additions

Add to `eval:interpret`:

- "close the grok tab" → closeTab grok
- "go to my gmail tab" → switchTab gmail
- "minimize this window" → keys cmd+M
- "force quit" → none
- "draft a mail to jake bauer at gmail dot com about the hackathon credits" → draftEmail to
  jakebauer@gmail.com, sensible subject and body, via mail
- "write a gmail to rahul at example dot com saying I'll be late" → draftEmail via gmail
- "email this to my boss" → none (no address, and "this" needs the screen)
- "send an email to x at y dot com saying hi" → draftEmail only, never sends

## At the end

Give me: what each item delivered, the commit list, what couldn't be tested, anything weaker
than this file, and new check rows to run on my phone and Mac.
