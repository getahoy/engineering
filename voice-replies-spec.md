# Voice replies, spoken choices and threads

Ahoy talks back. Replies are spoken on the phone (or Mac), choices can be answered by voice,
and a command plus its follow-ups is grouped as one thread.

**Exit criterion:** say "click save" with two Save buttons on screen → the phone says "I see two
Save buttons. One in the toolbar, two at the bottom. Say one or two." → hold and say "two" →
it clicks the second one, the phone says "Done. Clicked Save." and shows one thread with the
follow-up folded under it.

**Core rule: the voice is a bonus, never required.** The text always shows. If speech fails,
nothing else breaks and nothing waits on audio.

## Decisions (already made)

- **Engine:** ElevenLabs is the main voice (through our server). The device's built-in voice is
  the automatic fallback. Both sit behind a `VoiceEngine` interface so more engines can be added.
- **Where it plays:** the phone by default. A phone setting picks This iPhone / Mac / Both / Off.
- **How much it talks:** default "Questions and problems". Successes get a soft sound plus a
  haptic. Other options: "Everything" and "Off".
- **Destructive actions are confirmed by tap only.** Never by voice.
- **Ahoy never opens the microphone on its own.** The user always holds the button to answer.

## Flow

```
Mac finishes → command.result { …, threadId, message, speech }
  → phone shows message (always)
  → phone decides whether to speak (policy, silent mode, VoiceOver, staleness)
  → speakOn includes phone: POST /api/speak → audio → play (fallback: device voice)
  → speakOn includes mac: the Mac speaks the same `speech` itself
User presses talk while speaking → playback stops at once (and speech.stop to the Mac)
```

## 1. Protocol (shared/protocol)

- `command { commandId, text, speakOn: "phone" | "mac" | "both" | "off" }`. `speakOn` is sent with
  every command, so the Mac never stores the setting and can't get out of sync.
- `command.result` gains:
  - `threadId`: the root `commandId` of the thread (equal to `commandId` for a new command)
  - `speech?`: a short spoken version, max 140 characters. Omitted when there is nothing worth
    saying.
- New `speech.stop {}` (phone → Mac): stop any Mac speech now.
- New `choice.extend { choiceId }` (phone → Mac): sent when the user starts holding the talk
  button while a choice is open. The Mac pushes that choice's expiry to now + 30 s (at most twice
  per choice), so it can't expire while the user is answering.
- Schema tests for all of it.

## 2. Mac: what to say (agent/core, plain functions, unit tests)

### `buildSpeech(result) → string | undefined`

Templates per status, written to be heard, not read:

| Case | speech |
|---|---|
| done | "Done. Closed the Grok tab." |
| needsChoice, 2–3 options | "I see two Save buttons. One in the toolbar, two at the bottom. Say one or two." |
| needsChoice, 4–9 options | "I see five matches. They're numbered on your Mac. Say a number." |
| not found, with near-misses | "I can't find Sav. Did you mean one of these?" |
| not found, nothing close | "I can't find Sav on screen. Try saying it differently." |
| destructive confirm | "This will delete Q3 draft. Tap Delete on your phone to confirm." |
| draftEmail | "Drafted an email to Samar. Check it and press Send." |
| AI unavailable | "Smart commands aren't working right now." |
| refused (locked etc.) | the same as the message, shortened |

Rules:
- Only speak what the user said or what Ahoy did. **Never read screen contents** (document text,
  email bodies, tab titles the user didn't say). For choices, use the element's area or role only
  ("toolbar", "bottom", "sidebar").
- **Email:** say the person's name (the display name, or the local part made readable), never
  the full address, never the subject or body.
- Max 140 characters. If it's longer, cut it at a sentence boundary.
- No user text is ever put into a script. Speech text is data only.

### `parseChoiceAnswer(text, optionCount) → Pick | Cancel | Delegate | null`

Used only when this phone has an open choice (ChoiceBook, within TTL) **and** the transcript is
short (4 words or fewer). Otherwise the text is a new command and the open choice is closed as
`replaced`.

- Pick: digits 1–9; one…nine; first…ninth; "the last one"; "number two"; "option two"; Hindi:
  ek, do, teen, chaar, paanch, chhe, saat, aath, nau; pehla, doosra, teesra, chautha, aakhri.
- Cancel: cancel, none, neither, never mind, stop, chhodo, rehne do, kuch nahi.
- Delegate: any, anything, either, whichever, you pick, you choose, I don't know, koi bhi,
  tum chuno.
- A number outside the range → speech "There are only two. Say one or two." The choice stays open.
- Tests for each word list, plus "do" (Hindi two) only counting inside a short answer, and
  out-of-range numbers.

### Delegate rules

- Non-destructive: pick option 1 (the top-ranked one) and **say which**: "Picked one, the Save
  in the toolbar." Then run it through the normal safe-click path.
- Destructive: never. "I won't pick for a delete. Say one or two, or cancel." The choice stays
  open.

### Thread limits

- A thread is the root command plus its follow-up answers. Follow-ups attach **only** when Ahoy
  asked something back (choice or confirm). Never by time.
- At most 3 follow-up rounds. On the 4th: "Let's start over." The thread closes as `cancelled`.
- A thread ends as one of: `done`, `cancelled`, `expired` (TTL), `replaced` (a new command), or
  `screenChanged` (the existing two-snapshot check: "That screen changed. Ask again.").

## 3. Server: POST /api/speak (web)

- Auth: the phone or Mac session, the same as `/api/transcribe`. Rate limit with
  `consumeRateLimit`, about 60 a minute per user.
- Body `{ text, voice: "default" | "india" }`. Reject empty text or text over 200 characters with
  clear error codes.
- Use the official `@elevenlabs/elevenlabs-js` SDK (already in use). **Read the current docs
  first**, then use their lowest-latency model (the Flash family at the time of writing) and the
  streaming text-to-speech call. Output a small MP3 (for example 22 kHz, low bitrate).
- Voice IDs come from server env (`ELEVENLABS_VOICE_DEFAULT`, `ELEVENLABS_VOICE_INDIA`), never
  from the apps. Use a voice we're licensed to use commercially.
- Response: `audio/mpeg`, streamed. **Never log the text.** Pin the SDK version.

## 4. Phone: voice out (mobile)

### Settings (MMKV, per the storage rule)

- Voice replies play on: This iPhone (default) / Mac / Both / Off
- Speak: Questions and problems (default) / Everything / Off
- Voice: Default / Indian English (default to Indian English if the device region is India)
- Speak in silent mode: off by default

### `VoiceEngine` interface

`speak(text, { signal }) → Promise<void>` and `stop()`. Two implementations:
- `elevenlabs`: fetch `/api/speak` with the session, save to the cache dir, play with
  expo-audio. If expo-audio supports request headers for a stream URL, stream instead (check the
  docs).
- `device`: `expo-speech` using the best installed voice (prefer Enhanced or Premium quality,
  en-IN when the voice setting says so).

**Fallback:** if ElevenLabs fails, or the first audio isn't ready within 1.5 s, speak the same
text with `device`. Log only the reason (`timeout`, `http`, `offline`), never the text.

**Cache:** files keyed by sha256(voice + text), max 50, least-recently-used eviction. Repeated
phrases ("Done.", "Which one?") then play instantly.

### When to speak (one small pure function with tests)

Speak only if all of these are true:
1. `speakOn` includes the phone, and the Speak policy allows this status (Questions and problems
   = needsChoice, confirm, refused, failed; Everything = all; Off = none).
2. The result belongs to the **latest** command or the open thread. Results older than 10 s, or for
   an earlier thread, are shown but not spoken.
3. The user isn't recording right now.

Otherwise, for a `done` result under the default policy: play the soft "done" sound plus a light
haptic.

### Interruption and audio session

- Pressing the talk button **stops playback at once** (an 80 ms fade), cancels any pending fetch,
  sends `speech.stop` if the Mac might be speaking, and sends `choice.extend` if a choice is open.
- Nothing plays while recording. Only one utterance at a time: new speech replaces old.
- After every recording, reset the audio mode to playback (`allowsRecording: false`) so sound
  comes from the **speaker, not the earpiece**. Test with AirPods (they drop to a low-quality
  mode while the mic is on).
- Silent mode: iOS has no way for an app to read the switch. Set `playsInSilentMode` from the
  "Speak in silent mode" setting and let iOS mute it otherwise.
- VoiceOver on (`AccessibilityInfo.isScreenReaderEnabled`): don't use our voice. Call
  `announceForAccessibility(speech ?? message)` instead.
- App backgrounded: stop speaking.

### The "done" sound

`assets/sounds/done.m4a`, under 300 ms, soft. I'll supply the file (generated with ElevenLabs
Sound Effects or taken from a CC0 library). Use a placeholder until then.

## 5. Mac: voice out (agent)

- If `speakOn` includes `mac`: fetch `/api/speak` with the Mac's session, write it to the app's
  temp dir, and play it with `afplay` through `execFile` (an argument list). Delete the file
  afterwards.
- Fallback: `say` through `execFile` with the text as an argument (`["-v", voice, "--", text]`).
  Never a shell string.
- `speech.stop` or a new command kills the playing process.
- Never speak while the screen is locked. Only one utterance at a time.

## 6. Phone: threads (logic now, final UI after Claude Design)

- The store groups results by `threadId`. Each thread holds the root prompt, the follow-up turns
  (Ahoy line / user line), the final status, and the round count.
- **The home screen shows only the current thread**, expanded while it's open and folded to one
  line ("DONE · 4 steps ⌄") when finished. Tapping ⌄ toggles it.
- For now, use a plain functional list styled with existing components. The final visuals come
  from the Claude Design screens. Keep the layout in one component so restyling it is easy.
- Persisting threads to History is step 9, out of scope here.

## 7. Docs

- `flow.md`: the voice-out chain and the spoken-choice chain.
- `docs/corner-cases.md`: one row for each case in this file.
- `decisions.md`: ElevenLabs main plus device fallback; phone default; tap-only destructive
  confirm; no auto-listening; the delegate rules.
- `engineering/voice-replies-checks.md`: the table below.

## How to work

- List every file you'll create or change and why. Wait for OK.
- Order: protocol → Mac core (speech, parser, delegate, limits) → /api/speak → phone voice out →
  Mac voice out → threads → docs. One commit per part. Show each diff before committing.
- Typecheck, lint and tests pass after each part.
- At the end: what each part delivered, what couldn't be tested, anything weaker than this spec,
  and which native rebuild is needed (expo-speech), with the command.

## Device checks

| # | Do this | Expect |
|---|---|---|
| 1 | "Open Calculator" (default policy) | Opens; soft sound + haptic; no speech |
| 2 | Policy Everything, "open Calculator" | Speaks "Done. Opened Calculator." through the speaker, not the earpiece |
| 3 | "Click save" with two Save buttons | Speaks the two-option line; tags on Mac; sheet on phone |
| 4 | …then hold and say "two" | Clicks the second; "Done. Clicked Save."; one thread, folded after |
| 5 | …same, say "doosra" | Same as 4 |
| 6 | …same, say "five" | "There are only two. Say one or two."; choice stays open |
| 7 | …same, say "you pick" | "Picked one, the Save in the toolbar." and clicks it |
| 8 | "Delete" with two Delete buttons, say "you pick" | "I won't pick for a delete…"; nothing clicked |
| 9 | Confirm sheet open, say "yes" | Not confirmed; "Tap Delete on your phone to confirm." |
| 10 | Press talk while Ahoy is speaking | Speech stops instantly; recording starts |
| 11 | Hold for 25 s answering an open choice | Choice doesn't expire mid-answer |
| 12 | Answer a choice, then say "open Spotify" instead | Choice closes as replaced; Spotify opens |
| 13 | Four rounds of non-answers | "Let's start over."; thread closed |
| 14 | Airplane mode after the result arrives | Text shows; device voice speaks (fallback) |
| 15 | Slow network (Link Conditioner) | Device voice within about 1.5 s, never both voices |
| 16 | Silent switch on, default settings | No sound; text shows |
| 17 | Silent switch on, "Speak in silent mode" on | Speaks |
| 18 | VoiceOver on | VoiceOver reads it once; our voice stays quiet |
| 19 | AirPods connected, a full talk then a reply | Reply plays clearly in AirPods |
| 20 | Speak on: Mac | The Mac speaks; the phone is quiet |
| 21 | Speak on: Mac, press talk while the Mac is speaking | The Mac stops |
| 22 | "Draft a mail to samar at gmail dot com saying hi" | "Drafted an email to Samar. Check it and press Send." No address read out |
| 23 | Same phrase twice | Second plays instantly (cache) |
| 24 | Two commands quickly; the first's result arrives late | Only the latest is spoken |
| 25 | Background the app mid-speech | Speech stops |
