# Decisions

Append-only. Newest first. About four lines each: what, why, what was rejected, how to
reverse. A later decision that reverses one adds a new entry linking back; history is never
edited.

## 2026-11-05 — Speech is paced: speed 0.9 and a 0.3 s break between sentences

Flash v2.5 at speed 1.0 read short replies near 195 words a minute. `speechVoices.ts` sets
`voiceSettings.speed` 0.9 and `pacedText` adds `<break time="0.3s" />` after `.`, `!`, `?`
for `/api/speak` and the sample script alike, so phone, Mac, landing and previews match.
Rejected: switching to multilingual v2 (slower first audio). Phones keep cached audio
of a line until evicted, since the cache key has no pace. Reverse: drop both in one file.

## 2026-11-05 — Voice previews ship as bundled clips, checked by voice id and line

`mobile/assets/voice-samples/` holds one MP3 per voice plus `manifest.json`
(`voice, voiceId, text, generatedAt`), built by web `pnpm voice:samples` in the replies'
format (`mp3_22050_32`, not 64 kbps, so a preview sounds like a real reply). The phone plays
a clip only while its voice id and `voiceCopy.previewLine` match; an unknown id (never
fetched) trusts the clip. Re-run the script when a server voice id or `previewLine` changes,
and keep the script's `PREVIEW_LINE` equal to `previewLine`; until then previews fetch once
and cache. Rejected: fetching on first tap (silent offline).
Reverse: delete the folder and `bundledSamples.ts`; the chooser falls to fetch.

## 2026-11-04 — The launch wordmark is the splash PNG, held 1 s from JS start

`Wordmark` renders `splashWordmark(Dark)` at the splash `imageWidth`, centred on the full
screen, so the native splash is cut (no fade) to an identical frame and LaunchIntro, Welcome
and SigningIn all show one Ahoy. The reveal waits until 1 s after JS start
(`welcome/launchTiming.ts`), never longer for the server. Rejected: Fraunces text at 52 px
(a second, smaller Ahoy crossfaded under the splash). Reverse: render the text in `Wordmark`
and restore the splash fade. `imageWidth` in `app.config.ts` and `--spacing-splash-mark-x`
must stay equal.

## 2026-11-04 — Connecting never blocks talking; background keeps the session 20 s

Talk is blocked only by hard link states (paused, locked, off, asleep, quit, settled
unreachable, unrecognised, version mismatch); while connecting, recording starts and the
send waits up to 5 s for the link. Backgrounding keeps the Noise session 20 s and foreground
pings it (500 ms) instead of always redialling. Rejected: blocking on connecting (talk was
disabled for 1–2 s after every unlock behind a green chip). Reverse: map `connecting` to
`noMac` in `talkBlockOf` and call `stop()` in `MacConnection.background`.

## 2026-11-04 — The reinstall marker `ahoy.install` / `installSeen` is never renamed

The iOS Keychain survives an uninstall, so a missing MMKV flag means a new install and the
better-auth cookie and session cache are wiped before the session loads. The flag is set even
when the wipe fails. Renaming the MMKV instance or key would make every existing install look
new and sign everyone out once. Reverse: only by adding a second marker alongside, never by
renaming.

## 2026-11-04 — Cold launch trusts the cached session plus the stored cookie

The launch gate opens as soon as the expo plugin has hydrated a cached session and
`getCookie()` is non-empty, or at once when the cookie is empty; `/get-session` keeps running
in the background and a 401 signs out. A failed check without a session (offline, 5xx,
unreadable Keychain) is not treated as signed out; it waits for the 6 s give-up. Rejected:
waiting for `/get-session` (SigningIn flashed under the splash on every launch). Reverse:
make `sessionReady` require `!isPending`.

## 2026-11-03 — Hiding the window pauses the pairing code instead of discarding it

A hidden window or popover aborts polling but keeps a ready code; showing again reuses it
when it has 30 s left (no spinner, no fade), else makes a new one. An approval that lands
mid-pause still completes. Rejected: stop on hide (every reopen showed "Making a code…" and
asked the server again). Reverse: call `stop()` instead of `pause()` in `surfacePairing`.

## 2026-11-03 — Pairing code is a pure state machine; QR drawn from qrcode-generator

`core/pairing/pairingCode.ts` maps events (issued, missed, due, online, approved) to the next
status and effect; `PairingSession` only performs effects, so retry, refresh, offline and
scanned are unit-tested. The QR is our own SVG from `qrcode-generator` (react-qrcode-logo's
own engine) because the 9×9-module pennant square needs the module count, which
react-qrcode-logo hides. Rejected: react-qrcode-logo `logoImage` (logo size in pixels, not
modules). Reverse: put react-qrcode-logo back in `QrCode`.

## 2026-11-03 — The shell, not the pair screen, decides when a pairing code runs

`surfacePairing` is the only owner: it starts the session whenever a surface is visible and
`needsPairingCode` holds, and stops it when the window and popover are both hidden, so a
fresh code is requested the moment a phone ends the pairing and no key refreshes unseen.
Rejected: renderer mount/unmount as a second owner (navigating away aborted a slow first
request; a minimised window kept refreshing). Reverse: restore `usePairingSession` from git.

## 2026-11-02 — "Ended" eyebrow time is when this device found out

The server's links reply has no end time, so U1/U2 show the age since this device learned of
the end, plus "while Ahoy was closed" (phone, learned within 15 s of launch) or "while this
Mac was asleep" (link check right after wake). Rejected for now: an `endedAt` field on the
links response (needs `shared` + `web`). Reverse: add it and pass it as `at`.

## 2026-11-02 — An ended pairing removes the Mac and leaves a notice

The phone no longer keeps an ended Mac in `pairedMacs` with an Ended card; it removes it and
saves `{ reason, deviceName, at, whileClosed }` in MMKV, shown on Home for 24 h. Old records with `ended`
are migrated on load (`settleEndedMacs`). Rejected: keeping the tombstone on the phone (the
Mac already keeps one; a dead row blocked the empty state). Reverse: restore the `ended`
field on `PairedMac` and the Ended card from git history.

## 2026-11-02 — Local Network guide only for a Mac never reached over LAN

iOS has no API for the Local Network setting, and a denied one fails like an asleep Mac.
The guide opens itself only on iOS, on Wi-Fi, with no goodbye reason, for a Mac without the
MMKV `reachedOnLan` flag (set on each handshake, gone on reinstall like the permission).
Everywhere else the Can't reach card carries a "Check Local Network" link. Rejected: guide
on every unreachable (blames asleep Macs). Reverse: drop the flag, or use dial timing
(`lanDial` dev log) if denial proves to fail instantly.

## 2026-11-02 — Permission screens read Expo stage, restricted is not detectable

Stage comes from `PermissionResponse`: undetermined → explainer, denied + canAskAgain →
ask again (Android only in practice), denied → Settings guide. Expo maps iOS
`.restricted` to `denied` (`CameraPermissionsRequester.swift`), so the restricted screen is
built but nothing routes to it. Reverse: a native module reading `AVAuthorizationStatus`.

## 2026-11-02 — Settings return survives an iOS restart through an MMKV flag

iOS kills the app when a privacy toggle changes, so the gate writes `resumePairing` or
`resumeTalk` before `Linking.openSettings()`; Home takes it at launch, any AppState
`active` clears it. Rejected: route state (lost on restart). Reverse: delete
`settingsReturn.ts` and its two callers.

## 2026-11-01 — Scanner zoom is a 0–1 level mapped per camera curve, no × label

expo-camera never sends the camera max zoom to JS, and maps `zoom` exponentially on iOS
(`min × (max/min)^zoom`) but linearly on Android (`max(1, zoom × maxRatio)`). The UI keeps
one level 0–1; `mobile/.../pairing/zoomScale.ts` maps it per curve (iOS `level × 0.4`,
Android `0.125 × 4.8^level`, tuned for an 8× camera). Rejected: patching expo-camera to
report max zoom (patch upkeep on every bump). Reverse: patch it and show real × values.

## 2026-10-31 — Landing "Try it" listens in the browser behind a DemoListener port

The demo uses the browser's speech recognition (en-IN, one phrase, 6 s), keyword rules in
`listening/matchSpoken.ts`, and prerecorded ElevenLabs clips (`pnpm voice:landing`). No server call, so no cost or
abuse surface, but Firefox has no API and Brave/Arc expose it and then fail. Rejected for now:
a public speech-to-text route (cost, rate limits, privacy line changes). Reverse or extend:
add a recorder listener in `listening/` and return it from `createDemoListener()`.

## 2026-10-31 — Web cn built with createCn, shadcn alias points at it

`web/src/lib/cn.ts` is `createCn` from `cn/config` listing the Ahoy text sizes, and
`components.json` `utils` plus the shadcn primitives import it. The default merger read
`text-button` and `text-body` as colours and dropped them beside `text-white` inside `Button` and
`Input`. Rejected: a landing-only cn (shadcn merges inside its own files). Reverse: point the
imports back at `cn` and drop custom sizes from shadcn class names.

## 2026-10-31 — Landing CSP allows inline scripts so `/` stays static

Production headers come from `next.config.ts` with `script-src 'self' 'unsafe-inline'`.
Nonces force every request through the server. Experimental SRI only adds `integrity` to the
external chunks (12 on `/`); the 18 inline RSC payload scripts would still need
`'unsafe-inline'` or per-build hashes Next does not emit. Rejected: nonce in `proxy.ts`.
Reverse: nonce proxy plus dynamic `/`.

## 2026-10-31 — Waitlist answers the same for new, existing and honeypot signups

`POST /api/waitlist` inserts with `on conflict do nothing` and replies `{ joined: true }` either
way; a filled `company` field saves nothing. The form cannot reveal who signed up. Rate limit
keys on `x-real-ip`, which Vercel sets itself (pairing claim switched to it too). Rejected: a
409 for duplicates, raw `x-forwarded-for`. Reverse: branch on the insert result in the route.

## 2026-10-31 — Landing tokens live beside the shadcn variables

Ahoy colours, sizes and radii are new `@theme` names in `globals.css`; the shadcn variables
(`--background`, `--primary`) are untouched so `/sign-in`, `/sign-up` and `/account` look the
same. Fonts load in `features/landing/fonts` and apply on the landing root as `font-display`,
`font-text`, `font-label`, not `font-sans`. Rejected: remapping shadcn variables (restyles the
auth pages). Reverse: map shadcn variables to the Ahoy tokens.

## 2026-10-31 — Mac remove and sign-out confirm in the renderer, not a native dialog

`unpairPhone` and `signOutMac` no longer call `dialog.showMessageBox`; the designed
`ConfirmDialog` (Flare header) asks first and the IPC call acts straight away. Keeping both would
ask twice. Rejected: native dialog only (off-brand, no Flare header). Reverse: restore
`confirmed()` in `shell/pairing/unpairPhone.ts` and drop the renderer dialogs.

## 2026-10-31 — Mac popover is a vibrancy BrowserWindow sized by its content

The tray popover is a frameless, transparent `BrowserWindow` (`vibrancy: "popover"`) built once
and hidden, so it reopens instantly; the renderer reports its height over IPC. Rejected: the
native tray menu (cannot hold a switch, cards or status pills), a fixed height (transparent
rows would swallow clicks). Reverse: delete `shell/popoverWindow.ts`, set a context menu again.

## 2026-10-31 — Mac colour tokens use CSS light-dark()

Each token is `light-dark(light, dark)`, resolved where it is used, so the app follows the system
appearance and a gallery frame with `color-scheme: dark` previews dark beside light. `cn` is
built with `createCn` listing the custom font sizes, because the merger otherwise reads
`text-ui` as a colour and drops `text-on-ink`. Rejected: a `prefers-color-scheme` block (no
side-by-side preview). Reverse: move dark values back into a media query.

## 2026-10-31 — Phone fonts: static Fraunces cuts embedded at build time

React Native cannot set variable-font axes, so `mobile/scripts/buildFonts.py` pins Fraunces
(opsz 72, SOFT 100, WONK 1, weights 300/400, upright and italic), Inter (opsz 14) and Geist
Mono into static files whose PostScript name equals the file name, so one `fontFamily` works on
iOS and Android. The `expo-font` config plugin embeds them, so text never waits on a font load.
Rejected: `@expo-google-fonts/*` (normal cuts, no WONK), runtime `useFonts`. Reverse: delete
`assets/fonts/` and the plugin entry.

## 2026-10-31 — Orb gradient: React Native backgroundImage, not a class

Uniwind compiles any `radial-gradient()` to an empty string, so the orb reads its stop colours
from `global.css` tokens with `useCSSVariable` and passes `backgroundImage` to the view as its
one inline style. React Native 0.88 draws radial gradients natively on iOS and Android.
Rejected: Skia (a native dependency for one gradient). Reverse: once Uniwind supports radial
gradients, move the gradient into a `@utility` class.

## 2026-10-30 — Voice replies: server voice first, device voice as the fallback

Text becomes speech only in web's `/api/speak` (ElevenLabs Flash, a small streamed MP3, voice
ids from server env); the phone and Mac only play it. If it fails or the first audio is not
ready in 1.5 s, the device voice speaks instead (expo-speech, `say`) and the server request is
aborted, so the two never overlap. Replies play on the phone by default and speak only
questions and problems; successes chime. Rejected: an SDK or key in the apps, device voice only
(poorer quality). Reverse: set every phone to Off.

## 2026-10-30 — Spoken choices: tap-only confirm, no auto-listening, a careful delegate

A short answer (4 words or fewer) to an open choice is parsed on the Mac, including Hindi and
sound-alike numbers; anything longer is a new command that replaces the choice. A destructive
confirm is never accepted by voice, and Ahoy never opens the microphone on its own. "You pick"
chooses option one through the normal safe-click path and says which, but never for a
destructive choice. Threads grow only when Ahoy asked something back, three follow-ups at most.
Rejected: voice confirm (one mishearing deletes), time-based threads. Reverse: drop
`parseChoiceAnswer` from `commandTurns.ts`.

## 2026-10-29 — Safety: a fail-closed app blocklist and a sticky breaker

Clicks, shortcuts and typing are refused when the front app's bundle id is on the blocklist
(password managers, Keychain Access, Passwords by default; editable in `settings.json`) or when
the front app cannot be read. The breaker trips at more than 30 commands a minute or 5
failures in a row and stays tripped until the person taps Resume, which is a one-option
choice on the phone. Rejected: banking apps by default (bundle ids vary by bank and country),
an automatic reset after a minute. Reverse: drop `core/safety`.

## 2026-10-29 — Choosing and confirming reuse one mechanism on both devices

Several matches, a miss and a destructive target all answer `needsChoice` with numbered
`choices`; a destructive one sets `confirm`. The Mac keeps one pending choice for 20 s and drops
it on a new command or a front window change; a pick is `command.choose` with a new id and the
parent id, so dedupe still works. The phone shows a `@gorhom/bottom-sheet` that closes when
the choice expires. Rejected: a separate confirm frame, reusing the parent id. Reverse: remove
`choices`, `confirm` and `command.choose` (additive, so older peers never noticed).

## 2026-10-29 — Clicks happen only after two snapshots agree

A click resolves the label, waits 100 ms, snapshots again and clicks only when the app, window
frame and element count are the same and the target has the same label in the same place.
The helper presses first and falls back to a coordinate click, which needs no conversion
because accessibility and mouse events share global points; the overlay is the one place
that converts (`cocoaRect`). Rejected: clicking from the first snapshot (stale after any
animation). Reverse: call `click` straight after resolving.

## 2026-10-29 — The resolver is pure core, matching five accessibility fields in three tiers

`core/resolver` keeps enabled, non-empty elements whose centre is in the window, matches title,
description, help, placeholder and subrole after removing ellipses and punctuation, and tries
exact, whole word, then contains. Nested copies at one frame collapse to the pressable one.
Rejected: fuzzy scores (they guess; the plan says ask). Reverse: replace `resolveTarget`.

## 2026-10-28 — Email drafts carry AI-written text; they never send

`draftEmail { to, subject, body, via }` lets the AI write the subject and body, unlike `type`,
whose text must be what the person said. Allowed because the result is a draft the person
reads and sends: Ahoy opens a `mailto:` URL or Gmail's compose page and stops. A draft must
be the only action, so no Return or ⌘Return can follow it, and the letters of the address's
local part must appear in what was said. Rejected: sending (one mistake reaches someone),
verbatim-only bodies (useless for "about the hackathon credits"). Reverse: drop the action.

## 2026-10-28 — Browser tabs through JXA by index; matching on the Mac, not in the script

Tabs in Chrome, Brave, Edge, Arc and Safari are listed with one JXA script (title and URL by
window and tab index, because Arc throws on `tabs()` but answers indexed access), matched in
`core/commands/matchTabs.ts`, then closed or switched by index. The spoken match never enters
a script. No browser named → the frontmost browser, else the default browser
(`NSWorkspace URLForApplicationToOpenURL`). Several matches answer with the titles and do
nothing until choices arrive in step 7. Rejected: matching inside AppleScript (the match would
reach the script), screen reading (step 7). Reverse: drop `tabIntents.ts` and the two actions.

## 2026-10-28 — Shortcuts: any key with a modifier, minus a denylist; supersedes the 25-key allowlist

The AI's `keys` action is now `{ key, modifiers }` with semantic modifiers (`mod`, `shift`,
`alt`, `control`) mapped to ⌘⇧⌥⌃ on the Mac, so it can use app shortcuts it knows (Notion ⌘P,
⌘M, ⌃⌘F). Key names are exactly the Swift helper's table, so power and eject cannot be sent.
The Mac refuses ⌘Q in every variant, ⌥⌘Esc, ⌘Delete in every variant, a letter without a
modifier (that is typing), and Return unless the person said enter, send, submit or return.
Rejected: keeping the allowlist (every new app shortcut a code change). Reverse: restore
`KEY_COMBOS` in the protocol.

## 2026-10-28 — Pairings belong to an account on the phone

The phone stores the user id that owns its pairings. When a different user signs in, paired
Macs, pending unlinks and device keys are wiped before any card connects, the same data the
sign-out button clears. An automatic sign-out (session gone on the server) keeps the pairing,
so the same user signing back in reconnects without re-pairing. The phone signs out on
`unauthorized` only after `getSession()` confirms the session is gone with no error. Rejected:
unpairing on every automatic sign-out (re-pairing after each expiry). Reverse: delete
`pairingOwner.ts`.

## 2026-10-28 — A garbled session cache cookie is retried without it, not answered 500

`readSession` in web calls Better Auth's `getSession`; if it throws, it retries once without
the `session_data` cache cookie, so the signed token alone decides. A database failure still
throws and stays a 500. iOS had merged its own cookie jar into the phone's cookie header;
the phone now sends `credentials: "omit"` like Better Auth's own Expo client. Rejected:
mapping every `getSession` error to 401 (a database blip would sign the phone out).
Reverse: call `auth.api.getSession` directly again.

## 2026-10-28 — Smart commands send text to Anthropic by default; onboarding must say so

Command text and installed app names go to Anthropic when the parser can't handle a command,
with no setting for now. Onboarding must disclose both cloud hops before first use: ElevenLabs
hears the voice, Anthropic reads the words. A "Smart commands" off-switch (parser only) comes
in hardening. Rejected: asking for consent now (no onboarding screen yet). Reverse: gate the
interpreter call on the setting.

## 2026-10-28 — Slim LLM tier: the parser first, the AI only when it can't, the Mac validates

`/api/interpret` turns a command the parser can't handle into at most 3 actions from a closed
set (open, quit, hide, switch, search, openSite, keys from an allowlist, type). The Mac runs
nothing unless every action passes: exact installed app names, plain site hosts, Return only
when the person said enter / send / submit / return, typed text only if it appears in what they
said. Model `claude-haiku-5-5` through `@ai-sdk/anthropic` (thinking off, low effort, 5 s, no
retry, system prompt and app list cached). Models come only from the allowlist in
`web/src/features/interpret/server/models.ts`, resolved through `createProviderRegistry`: a new
model is one entry, a new provider one registry line, a per-user choice one function; the Mac
never knows the model. No interpreter port: the AI SDK is the abstraction. Rejected: the AI
first (slower and paid for commands the parser already knows), the AI Gateway (one more
account for one provider). Reverse: route everything through `interpret()` or drop the tier.

## 2026-10-27 — Speech uses the official ElevenLabs SDK, not the AI SDK; partly reverses the voice entry below

`/api/transcribe` calls `speechToText.convert` from `@elevenlabs/elevenlabs-js` in one file
(`features/speech/server/transcribeAudio.ts`), model `scribe_v2`, language auto-detected,
with about 60 keyterms (command words and app names). The AI SDK's ElevenLabs provider only
passes keyterms for realtime streaming, and short command words need the bias. Rejected: AI
SDK without keyterms. Reverse: swap that one file for `transcribe` from `ai`.

## 2026-10-27 — App-level commands run through fixed templates before screen matching

"Open Safari", "quit Music" and similar run through a closed set of `open` / `osascript`
templates on the Mac, tried before any screen matching. The text only fills a template slot
(an installed app name), runs with `execFile` and never reaches a shell, so the "no shell
verb" rule holds. Rejected: matching on-screen elements for app commands (slow, needs the app
visible), model-written scripts (unsafe). Reverse: drop the template tier; commands go
straight to matching.

## 2026-10-27 — 5b Bonjour is skipped; the next transport is a hosted realtime channel

Bonjour only fixes a changed IP on the same Wi-Fi. A hosted realtime channel (Ably or
Pusher), added alongside the LAN link after 31 Oct, fixes that and off-Wi-Fi use too. It
carries our Noise-encrypted frames unchanged, with server-issued tokens per active pairing,
so the service never sees plaintext. Not built now. Rejected: Bonjour first, our own
WebSocket relay with Redis presence (more to run). Reverse: add Bonjour discovery as a LAN
address source.

## 2026-10-27 — Voice: the phone records a file, our server transcribes, only text reaches the Mac

The phone records an m4a file and posts it to `/api/transcribe`; web calls ElevenLabs
through the Vercel AI SDK and returns text; the phone sends `command { commandId, text }`
over the encrypted session. One place holds the key, the Mac never handles audio, and a
command frame is tiny on any transport. The AI SDK is our provider abstraction. Reverses
streaming audio to the Mac and the Transcriber/Speaker ports. Rejected: streaming PCM over
Noise to the Mac (parked on `parked/custom-talk` in agent and mobile), speech on the Mac.
Reverse: merge `parked/custom-talk`.

## 2026-10-25 — Pairing ends are explicit: goodbye frames, signed refusals, server links

Every ending carries a reason: `goodbye` frames on the live socket (fast), a revoked
`device_links` row with `revoke_reason` (reliable), and Mac tombstones that answer a removed
phone with a Noise-authenticated `not_paired`. A timeout or closed socket never ends a
pairing. Outboxes on both devices retry unlinks until done or `not_found`.
Rejected: treating a closed socket as removal (false "no longer trusts" on Wi-Fi blips),
server-only checks (slow), LAN-only messages (lost when offline).
Reverse: drop tombstones and the links check; endings fall back to unreachable.

## 2026-10-25 — A Mac has one phone, a phone has many Macs

Pairing a new phone revokes the old link as `replaced` inside the complete transaction, so
the old phone keeps working until the new one is in. Fewer states than many phones per Mac
and it matches "my phone controls my Mac". Changes the corner-case row that allowed several.
Rejected: several phones per Mac (who is active, concurrent commands, more UI states).
Reverse: remove `replacePreviousPhones` and let the Mac hold a list again.

## 2026-10-24 — Pairing is Better Auth device authorization plus our proof; reverses courier-lan first (spec Part 16.3)

The Mac asks Better Auth's `deviceAuthorization` plugin (RFC 8628) for a code, the phone
claims it (our route approves it), the Mac polls and receives a Better Auth session via the
`bearer` plugin. Works with the phone on mobile data. The QR's one-time secret never reaches
web: the phone proves it scanned the QR with keyed BLAKE2b over both public keys, stored in
`pairing_claims`, and the Mac checks it, so a lying server cannot pair a stranger.
Rejected: courier-lan (fails off Wi-Fi), our own code/poll/token tables (security code the
auth library already maintains), Redis for codes.
Reverse: add courier-lan as a second strategy; the QR already carries `lan`.

## 2026-10-24 — Noise IK over a WebSocket on the LAN, pure-JS libsodium

`noise-handshake` with `sodium-javascript` on both sides: Metro takes it through the
`browser` field, Vite through an alias, so no native addon ships. Fresh ephemeral keys per
connection give forward secrecy and replay safety without counters or clocks. The protocol
version rides in the encrypted hello, so a mismatch gets a readable refusal.
Rejected: XX (the phone already knows the Mac key), sodium-native (Electron rebuilds),
TLS with self-signed certs.
Reverse: swap the pattern inside `@getahoy/crypto/secureChannel.ts`.

## 2026-10-24 — Error status comes from the error code

`ErrorResponse(code, message, field?)` maps the code to the HTTP status through one
exhaustive record, so a route cannot pair `not_found` with 400. `field` names the invalid
input path (Stripe's `param`). `request()` reads `x-request-id` into its typed error.
Rejected: a status argument per call (drifts), RFC 9457 problem+json (more fields than needed).
Reverse: add an optional status override to `ErrorResponse`.

## 2026-10-24 — The Mac's credential is a Better Auth session

The device-code poll returns a normal session token, kept in `identity.bin` (`safeStorage`)
and sent as `Authorization: Bearer`. Revoking it is revoking a session; no token table of
ours. It expires like any session (7 days, refreshed on use), so the relay in 5b must keep
it fresh. Claims are written before approval, so the Mac never polls a token whose claim is
missing; a refused approval deletes its claim, stale claims are swept on insert.
Rejected: our own hashed device token (a second credential system), JWTs (not revocable).
Reverse: issue a dedicated device token in `completePairing` and store its hash.

## 2026-10-23 — Agent window is React with Tailwind tokens; menubar is a native menu

The window renders three onboarding screens from one `AgentState` pushed over IPC and
Zod-parsed in the preload. Tokens live in `src/renderer/tokens.css`. The menubar uses a native
tray menu (status lines, Open, Quit) until it needs live content, then becomes a popover.
Rejected: shadcn in the agent (no vite config for its CLI to detect), a popover window now.
Reverse: swap `AgentTray` for a popover `BrowserWindow` anchored to the tray bounds.

## 2026-10-23 — Helper protocol lives in the agent; capture returns a file path

Only the agent and the Swift helper speak it, so the Zod schemas sit in
`agent/src/core/helper/`, not `@getahoy/protocol` (created at feature 5 for phone to Mac).
Element refs are opaque and valid until the next snapshot. `capture` writes a PNG to the temp
directory and returns its path instead of streaming base64 over stdout.
Rejected: helper schemas in shared (no second speaker), base64 frames (megabytes per line).
Reverse: move the schemas to `@getahoy/protocol` if the phone ever talks to the helper.

## 2026-10-23 — Sparse tree retry triggers below 10 elements

A Chromium window with accessibility off exposes only its chrome (window, group, three
traffic-light buttons, measured on VS Code), which slipped past a threshold of 5. Below 10
the helper sets `AXManualAccessibility`, waits 300 ms and walks again.
Rejected: always setting the flag (an extra 300 ms on every native app).
Reverse: change `sparseTreeThreshold` in `snapshot.swift`.

## 2026-10-23 — Agent is Electron Forge with Vite and TypeScript, not Next.js

The agent UI is a tray popover, settings and overlays; there is nothing to route or render on
a server. Next inside Electron means either a static export that drops every server feature
or a local Next server that slows launch and opens a port.
Rejected: Next static export, Next server in the main process, Nextron.
Reverse: swap the renderer for a Next static export loaded over a custom `app://` protocol.

## 2026-10-22 — App scheme and password maximum live in `@getahoy/lib`

`APP_SCHEME` feeds the server's `trustedOrigins` and the phone's registered scheme
(`app.json` became `app.config.ts` to import it). A mismatch passes in dev, where the expo
plugin trusts `exp://`, and fails only in production with `INVALID_ORIGIN`.
`MAXIMUM_PASSWORD_LENGTH` (128) is set explicitly on Better Auth and checked by the
sign-up schema, so a long password fails on the form, not after a round trip.
Rejected: sharing the Better Auth version as code, and a CI check comparing the pins (a
cross-repo read token is more upkeep than the bug; drift breaks mobile sign-in on the next
test). Both pins are bumped in the same session.
Reverse: inline the constants in each app.

## 2026-10-22 — Credentials schemas live in `@getahoy/lib`

`MINIMUM_PASSWORD_LENGTH` is the server's rule (`minPasswordLength` in Better Auth), so both
clients and the server read one constant. `credentialsSchemas(messages)` takes the
validation copy as input, because copy stays in each app's `authCopy`.
Rejected: messages inside lib (moves copy into shared), copy keys as messages (string keys
across the boundary).
Reverse: copy the schema file back into each app.

## 2026-10-22 — Shared is lib only; protocol, crypto and tokens removed

Revises the entry below. `protocol` and `crypto` were empty stubs nothing imported.
`tokens` is UI work, which is not started, so each app keeps its own theme CSS
(`web/src/app/globals.css`, `mobile/src/global.css`).
Rejected: keeping stubs and a token pipeline ahead of the features that need them.
Reverse: recreate each package with its feature (protocol in 3, crypto in 5, tokens with UI).

## 2026-10-22 — Shared shrinks to protocol, crypto, tokens, lib; auth returns to the apps

Reverses the auth half of "Shared is split by domain" (below). Auth changes daily, so a
shared package made the rebuild-and-resync cost worst exactly where work is fastest. Each
app now owns `features/auth`, mobile with its own copy. Shared keeps only what makes the
product wrong when the two sides disagree: protocol (messages), crypto (key derivation),
tokens (look), and lib (error codes, parseInput). Duplicated auth code is accepted.
Rejected: keeping `@getahoy/auth` (cost without a correctness reason).
Reverse: recreate the package from `web/src/features/auth`, ask first.

## 2026-10-22 — Shared is split by domain: auth and tokens leave lib

`@getahoy/auth` holds headless auth (schemas, `authCopy`, provider error mapping, hooks
taking `signIn`/`signUp`/`signOut`/`onSuccess` as injected deps). `@getahoy/tokens` holds
plain design values. `lib` keeps only generic `ErrorCode`, `errorMessages`, `parseInput`.
`ErrorCode` stays in `lib` because `auth → lib`; moving it to `auth` would invert that.
No JSX in shared: React DOM and React Native elements cannot cross platforms, so each app
writes its own markup against shared props contracts. No platform prop: a `platform`
switch grows a branch per OS in every shared file, while an injected dependency adds a
platform without editing shared at all.
Rejected: one `lib` holding everything (no boundaries, every app pulls every concern).
Reverse: merge the packages back; app imports are the only thing that changes.

## 2026-10-22 — Shared packages reach apps as `file:` copies, never `link:`

Measured on the mobile iOS bundle: `link:` shipped two Reacts (19.3.0 and 19.3.1) and two
Zods, because the symlink resolves `shared`'s own `node_modules`. `file:` copies only
`dist` and resolves peers from the app: one React, one Zod. A rebuild plus `pnpm install`
re-syncs the copy in under 150 ms.
Rejected: `link:` (duplicate React, "Invalid hook call"), yalc (unmaintained pre-release).
Reverse: switch the dependency specifier; Metro and Next need dedupe aliases first.

## 2026-10-20 — Mobile component kit is React Native Reusables, copied in

The deciding reason is copy-in: the component source lands in our repo, so it can be made
to obey CLAUDE.md (file size, comments, exports, tokens). An installed kit is a black box
that can never meet those rules. Uniwind variant, to match the styling decision below.
Rejected: HeroUI Native (installed package, rules unenforceable inside it).
Reverse: delete `components/ui/` and install another kit; screens keep their hooks.

## 2026-10-20 — Hooks: react-haiku first, usehooks.com second

react-haiku is the maintained one: last release August 2025, while `@uidotdev/usehooks`
has not released since October 2023 and declares `react-dom` as a required peer. Both are
DOM-first, so on React Native each hook is checked, then adapted or rewritten under the
same name in `shared`. A third hooks library is never added.
Rejected: usehooks.com as the first source (unmaintained since 2023, hard `react-dom`
peer).
Reverse: swap the order in CLAUDE.md; hooks already written keep their names.

## 2026-10-20 — Mobile styling is Uniwind only, on Tailwind 4

Uniwind 1.13 is stable on Tailwind 4, the same Tailwind as `web`, so token syntax matches
across repos. It is Metro-only: no Expo config plugin, no Babel plugin.
Rejected: NativeWind v5 (release candidate, verified only on Expo 57), NativeWind v4
(Tailwind 3, a migration waiting to happen), `@expo/ui` SwiftUI views (no `className`, so
they cannot follow the tokens rule).
Reverse: swap the Metro wrapper and re-run the component kit CLI with another styling
library; components keep their `className` strings.

## 2026-10-20 — Mobile animation is Reanimated only, plugin left to Expo

One animation engine runs on the UI thread and matches the component kits. On Expo SDK
54+ `babel-preset-expo` already includes the worklets plugin, so there is no
`babel.config.js` and the plugin is never added by hand (it would transform twice).
Rejected: the `Animated` API (JS thread, drops frames under load).
Reverse: none planned; adding a library means auditing every animated component.

## 2026-10-20 — Pairing in Phase A is `courier-lan`: the phone carries the Mac's tokens

The Mac shows a QR (LAN address + public key). The phone connects directly, completes the key
exchange, asks `web` to link that public key to the account and issue its tokens, then hands
the tokens to the Mac over the encrypted socket. The server never sees a private key, the Mac
never logs in through a browser, and the Mac needs no internet at pairing time.
Why: "one QR does auth and pairing" assumed a server-issued token, but local-first pairing had
no server in the path. Rejected: Redis rendezvous (adds a service for the 5% case), device-code
login on the Mac (a second login step the first-run flow forbids).
Reverse: add `server-rendezvous` or `device-code` as further `EnrollmentStrategy` entries
(system-design Part 16). `courier-lan` stays as the first entry.

## 2026-10-20 — API errors carry a closed `code`, not just a message

`{ success: false, error: { code: ErrorCode, message } }` instead of zyraa's bare string.
Callers switch on `code`; matching on message text breaks on a copy edit. `miss` and
`ambiguous` are command outcomes, not error codes.
Rejected: zyraa's `error: string` (style rule loses to architecture here).
Reverse: collapse `error` back to `message` in `@getahoy/lib` and every caller's switch.

## 2026-10-20 — `shared` is only `protocol`, `crypto`, `lib`

`resolver`, `grammar` and `llm` live in `agent/src/core/`. Only the agent uses them; the phone
never resolves. They stay pure and fixture-testable inside the agent.
Rejected: publishing them from `shared` because they felt important.
Reverse: when a second app needs one, move it to `shared` as a new package.

## 2026-10-20 — Drizzle uses the `neon-serverless` driver, not `neon-http`

`neon-http` only supports non-interactive transactions. The pairing claim (verify, link,
issue, return) needs an interactive one.
Rejected: `neon-http` for reads plus a second client for transactions (two clients for six
tables).
Reverse: swap the driver in `web`'s DB module; queries are unchanged.
