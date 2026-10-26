# Feature 6: Talk

Hold on the phone, speak, the Mac gets the text, does simple app-level commands, and the phone
shows the result.

**Exit criterion:** saying "open Chrome" on the phone opens Chrome on the Mac, and the phone
shows "Opened Google Chrome".

## Flow

```
Phone: hold → record (expo-audio, m4a file) → release
  → POST /api/transcribe (HTTPS, phone session) → ElevenLabs → text back to phone
  → phone sends command { commandId, text } over the existing encrypted session
  → Mac: command.received { commandId } at once
  → Mac runs the command → command.result { commandId, status, message }
  → phone shows the message
```

## 1. Protocol (shared/protocol)

- `command { commandId: uuid, text: string (max 500 chars) }`
- `command.received { commandId }`
- `command.result { commandId, status: "done" | "needsChoice" | "refused" | "failed", message: string }`
- Separate commit: remove the unused `audio.*` frames and constants, `audioChunkSize.test.ts`,
  and the audio ignore cases in agent and mobile.

## 2. Server: /api/transcribe (web)

- POST multipart with one audio file. Auth with the phone's session, same as the pairing
  routes. Rate limit with the existing `consumeRateLimit` (about 20/min per user).
- Reject over 2 MB or a missing file with clear error codes.
- Use `ai` `experimental_transcribe` with `@ai-sdk/elevenlabs`, model `scribe_v2`. Read the
  current docs first. If the provider can't pass keyterms, call the official ElevenLabs JS SDK
  directly in this one file instead. Keyterms: our command words and common app names.
  Language: auto-detect; the user speaks Hindi-English mixed.
- `ELEVENLABS_API_KEY` in the server env schema. Never in the apps.
- Response: `{ text }`. Empty text → error code `nothing_heard`.
- Never log audio or transcript text. Pin the `ai` and `@ai-sdk/elevenlabs` versions.

## 3. Phone: talk button

Needs a native rebuild for `expo-audio` and `expo-haptics`. Say when, and with which command.

- On home, under the connected Mac card. With several Macs, it uses the selected one.
- Hold to talk, release to send. Under 0.5 s: "Hold to talk", nothing sent. Stops at 30 s.
- Haptic on start and stop.
- **Disabled before recording**, with the reason shown: no Mac connected, Mac locked, paused,
  microphone not allowed (with Open Settings).
- **States**, following the UI rules in CLAUDE.md:
  Hold to talk → Listening (timer) → Understanding… (upload and transcribe) →
  Heard "close terminal" → the Mac's result message (done = success style; refused or
  failed = plain explanation).
- **Errors:**
  - Offline: "Speech needs the internet"
  - Transcribe failed: a plain message with Try again (re-uploads the same file)
  - Nothing heard: "Didn't catch that"
  - Mac unreachable after transcription: "Can't reach {Mac}" with Try again, which resends the
    same `commandId` and text with no re-recording
- No `command.result` within 10 s of `command.received`: "No answer from {Mac}". Never assume
  success.
- Phone call or app backgrounded while recording: discard and reset.
- `NSMicrophoneUsageDescription`: "Ahoy records only while you hold the talk button."
- Delete the local recording file after a successful upload.

## 4. Mac: receive

- Handle `command` in `PhoneSession`. Reply `command.received` immediately.
- **Dedupe:** keep the last 100 `commandId`s with their result. A repeat gets the stored
  result again and never runs twice.
- Screen locked → `command.result` refused, "Unlock your Mac to use Ahoy."
- Show "Heard: …" in the Mac window. Never log the text.

## 5. Mac: app-level commands (show the table and parser rules before building)

A plain function in `agent/core`: `parseCommand(text, installedApps) → Intent | NoMatch |
Ambiguous`, with unit tests. Then a runner in `shell` that executes the intent.

| Says | Does |
|---|---|
| open / launch / start {app} | `open -a` |
| close / quit {app} | quit via AppleScript |
| hide {app} | hide via AppleScript |
| switch to / go to {app} | activate via AppleScript |
| search {query} / google {query} | open `https://www.google.com/search?q={encoded}` |
| open {site}.com | open `https://{site}` |
| play / pause / next / previous | media keys |
| volume up / down / mute | AppleScript set volume |
| take a screenshot | `screencapture` to the user's screenshot folder |
| lock screen | the standard lock shortcut |

Rules:
- **Never put user text into a script string.** Use `execFile` with an argument list, and pass
  AppleScript values as `on run argv` arguments, so a name containing quotes can't run code.
- App names match against installed apps (display name, case-insensitive, simple fuzzy
  matching: "chrome" → "Google Chrome"). Two or more plausible matches → `needsChoice`, with the
  options in the message, never guess. No match → failed: "Can't find an app called {name}".
- Anything not in the table → failed: "I can't do that yet". Honest, no guessing.
- Result messages: "Closed Terminal", "Opened Google Chrome", "Searched for cats".

## 6. Docs

- `flow.md`: the talk chain end to end.
- `docs/corner-cases.md`: a row for each case above.
- `engineering/feature-6-checks.md`: the table below.

## How to work

- List every file you'll create or change and why. Wait for OK.
- One commit per part, in order. Show each diff before committing.
- Typecheck, lint and tests pass after each part.
- At the end: what each part delivered, what couldn't be tested, anything weaker than this
  spec.

## Device checks

| # | Do this | Expect |
|---|---|---|
| 1 | Hold, say "open Chrome" | Phone: Heard "open chrome" → "Opened Google Chrome"; Chrome opens |
| 2 | "Close Terminal" with Terminal open | Terminal quits; "Closed Terminal" |
| 3 | "Search cats" | The browser opens a Google search for cats |
| 4 | "Pause" with music playing | Music pauses |
| 5 | Tap and release instantly | "Hold to talk", nothing sent |
| 6 | Hold in silence for 3 s | "Didn't catch that" |
| 7 | "Open Xyzzy" | "Can't find an app called Xyzzy" |
| 8 | "Make me a sandwich" | "I can't do that yet" |
| 9 | Two matching apps ("open code" with VS Code and Xcode) | Asks which, does nothing |
| 10 | Lock the Mac, then talk | Button disabled, "is locked" |
| 11 | Wi-Fi off after "Understanding…", then back on, Try again | Runs once |
| 12 | Airplane mode, then talk | "Speech needs the internet" |
| 13 | Get a call while holding | Discarded, button resets |
| 14 | Hindi-English: "Chrome open karo" | Opens Chrome (note the result if not) |
| 15 | 10 commands quickly in a row | Each runs once, in order |
