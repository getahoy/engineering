# Flow

Entry points per app, then one call chain per major flow. `║` marks a process, network or port
boundary.

## Entry points

| App | Entry | Starts |
|---|---|---|
| web | `src/app/**/page.tsx`, `src/app/api/auth/[...all]/route.ts` | Next.js pages, Better Auth API |
| web | `src/proxy.ts` | Session check before `/sign-in`, `/sign-up`, `/account` render |
| web | `src/app/api/pairing/{claim,complete}/route.ts` | Pairing API, each wrapped by `features/api/server/apiRoute.ts`; device codes live under `/api/auth/device/*` (Better Auth) |
| web | `src/app/api/transcribe/route.ts` | Speech to text for the phone's talk button |
| web | `src/app/api/interpret/route.ts` | Smart commands: text to up to 3 actions, called by the Mac |
| web | `scripts/evalInterpret.ts` (`pnpm eval:interpret`) | Manual run of the cases in `scripts/interpretCases.ts` against the interpret route, checked automatically, with a pass/fail table |
| mobile | `src/app/_layout.tsx` | Expo Router stack, session guards, `getRandomValues` polyfill |
| mobile | `src/app/pair.tsx` | QR scanner and pairing claim |
| agent | `src/main.ts` | Electron main process, composition root |
| agent | `src/renderer/main.tsx` | Main window React app |
| agent | `src/dev/helperConsole.ts` (`pnpm helper`) | Dev console driving the Swift helper |
| helper | `native/macos-helper/Sources/ahoyHelper/main.swift` | Swift process reading JSON lines on stdin |

## Sign in with email (web)

1. `screens/signInScreen.tsx` → `screens/credentialsScreen.tsx` → `useCredentialsForm`
   (`@getahoy/lib`) parses with `signInSchema`
2. `authClient.signIn.email` ║ HTTP ║ `app/api/auth/[...all]/route.ts` → Better Auth →
   `db/client.ts` ║ Neon
3. Success → `useAuthNavigation.goToAccount` → `/account` ║ `proxy.ts` session check ║
   `gates/accountGate.tsx`

## Sign in with email (mobile)

1. `app/sign-in.tsx` → `screens/signInScreen.tsx` → `useCredentialsForm` (`@getahoy/lib`)
2. `authClient.signIn.email` (expo client, SecureStore cookie) ║ HTTP to `EXPO_PUBLIC_API_URL` ║
   web Better Auth
3. Session signal → `_layout.tsx` `Stack.Protected` guard flips → `index.tsx` account

## Sign in with Google

- web: `useSocialSignIn` → Better Auth redirect ║ Google ║ `/api/auth/callback/google` →
  `/account`, or `/sign-in?error=<code>` → `socialSignInError.tsx` → `socialErrorMessage`
- mobile: `useSocialSignIn` → expo client opens `expo-web-browser` sheet ║ Google ║ web callback
  ║ `ahoy://` back into the app → session or the generic `error.social` message

## Agent startup

1. `main.ts`: single-instance lock, hide the Dock, read `settings.json` (`shell/settingsStore.ts`)
2. Build `MacosPlatform`, `AgentController`, `HelperClient`, `MainWindow`, `AgentTray`;
   `shell/bridgeIpc.ts` registers the IPC handlers
3. `HelperClient.start()` ║ spawn ║ `ahoyHelper` (debug build in dev,
   `Contents/Resources/ahoyHelper` when packaged); `ready` and `exited` update the controller
4. Permission polling every 2 s (`systemPreferences`) and `powerMonitor` lock events update the
   controller, which re-renders the tray menu and pushes state to the window
5. Onboarding not done → `MainWindow.show()` (Dock shown, focus stolen) ║ IPC ║ `preload.ts`
   bridge ║ `renderer/agentApp.tsx` picks welcome, permissions or pair from `state.screen`
6. A button press ║ IPC ║ `AgentController.completeStep` → writes `settings.json` → new state

## API request (any web route)

1. `apiRoute(scope, work)` gives the request a UUID in `AsyncLocalStorage`
   (`features/logging/server/logger.ts`), so every log line carries it
2. Route: authenticate (`features/auth/server/readSession.ts`: Better Auth `getSession`,
   retried once without the `session_data` cache cookie if that cookie is unreadable) →
   `readJson(request, schema)` → work → `SuccessResponse` or
   `ErrorResponse(code, message, field?)` (`@getahoy/lib`, status derived from the code)
3. A throw is a bug: logged with its stack, answered `internal` 500. The response gets
   `x-request-id` and one access line `POST 201 42ms`
4. Callers use `request()` (`@getahoy/lib`): never throws, returns `success` or a typed
   `api | malformed | network` error

## Pairing (agent ⇄ web ⇄ mobile)

1. Agent pair screen mounts ║ IPC `startPairing` ║ `shell/pairing/pairingSession.ts`
2. `shell/pairing/deviceFlow.ts` → Better Auth client `device.code` ║ HTTP ║
   `/api/auth/device/code` (`deviceAuthorization`, client `ahoy-desktop`, 5 min) ║ Neon
   `device_code`
3. Agent draws the QR (`core/pairing/pairingTerms.ts`, `react-qrcode-logo`): Mac key, name,
   user code, one-time secret (never sent to web), LAN addresses + port 7421, expiry. Polls
   `device.token` at the returned interval, slowing down when told
4. Phone `pairScreen.tsx` → `QrScanner` (expo-camera) → `useClaimMac` →
   `enrollment/claimScannedCode.ts`: proof = keyed BLAKE2b(secret, phone key ‖ Mac key)
5. `POST /api/pairing/claim` with the session cookie ║ HTTP ║ `claimPairing`: one
   transaction upserts the phone in `devices` and writes `pairing_claims` (user code, phone,
   Mac key, proof), then `auth.api.deviceApprove`; a refusal deletes the claim
6. Next agent poll returns a session token ║ `POST /api/pairing/complete` with
   `Authorization: Bearer` ║ `completePairing`, one transaction: match the claim to the
   session user and Mac key, upsert the Mac, link both in `device_links`, burn the claim,
   return the phone and proof ║ agent checks the proof (`verifiedPhone`); a mismatch refuses
   the pairing. Saves the phone and session in `identity.bin` (`safeStorage`)
7. Phone saves the Mac (key, name, addresses) in SecureStore and returns home

## Phone connects to Mac (LAN)

1. Agent `PhoneLink` → `LanListener` binds 127.0.0.1 + each private LAN IPv4 on 7421,
   re-syncing addresses every 5 s, at most 20 handshakes per minute per address
2. Phone `LiveMacCard` → `useMacConnection` → `MacConnection` → `connectToMac` →
   `lanTransport` races the saved addresses ║ WebSocket on Wi-Fi ║ `WebSocketFrameStream`
3. Noise IK message 1 carries the phone hello ║ agent `PhoneSession` → `acceptHandshake`:
   undecryptable → plaintext `unknown_mac_key`; unknown key → signed `not_paired` with the
   tombstone reason or `unknown` (payload never parsed); other version → `version_mismatch`;
   else `welcome`
4. Phone sends an encrypted `ready`; the Mac counts it connected, sends `status { locked }`
   and repeats it on every lock change
5. Timeout or close before a reply → phone `unreachable`, auto-retry every 5 s in the
   foreground. Only a signed refusal marks the stored Mac `ended`

## Pairing ends (either side)

1. Remove / sign out / pair a new phone: the acting side sends `goodbye { reason }` on the
   live socket, then queues the unlink in its outbox (`identity.bin` on the Mac,
   `pendingUnlinks` in SecureStore on the phone) ║ HTTP ║ `/api/pairing/unlink` revokes the
   `device_links` row with `revoked_by` + `revoke_reason`; a new phone is revoked inside
   `completePairing` as `replaced`
2. The other side, if connected, handles the goodbye at once: Mac `phoneEnded.ts` (banner,
   one notification, tombstone); phone `MacConnection` (stored `ended`, ended card)
3. If it was offline: Mac `LinkMonitor` (start, every 15 min, on wake) and phone
   `useLinkSync` (start, foreground) read `/api/pairing/links` and treat a revoked link like
   the goodbye. A later handshake gets the signed `not_paired` from the Mac's tombstone
4. Pause: phone `goodbye paused` + stored `paused`; Mac shows "paused". Ahoy off, quit and
   sleep send `turned_off`, `quitting`, `sleeping`; the phone keeps that reason while it
   retries

## Talk (mobile ⇄ web ⇄ agent)

1. Hold the button: `pairing/screens/macsHomeScreen.tsx` → `talk/hooks/useTalk.ts` →
   `talk/sending/talkFlow.ts` `press` → `talk/recorder/useFileRecorder.ts` (expo-audio, mono
   AAC m4a, stops at 30 s). The target Mac comes from `talk/hooks/useTalkTarget.ts`, which
   reads each card's state from `LiveMacLinks`
2. Release: `talkFlow.ts` → `talk/client/transcribeApi.ts` (multipart with `expo/fetch`,
   session cookie) ║ HTTPS ║ `app/api/transcribe/route.ts`: session, 20/min rate limit,
   `features/speech/server/readAudio.ts` (2 MB cap) → `transcribeAudio.ts`
   ║ HTTPS ║ ElevenLabs `scribe_v2` with keyterms → `{ text }` or `nothing_heard`
3. The phone deletes the recording, then `talk/sending/sendCommand.ts` sends
   `command { commandId, text }` on the `MacSession` ║ LAN, Noise ║ agent
   `core/connection/phoneSession.ts` → `shell/commands/commandReceiver.ts`
4. The receiver replies `command.received` at once, checks `core/commands/commandLog.ts`
   (last 100 ids: a repeat gets its stored result), queues the command, shows "Heard" in
   the window, refuses when the screen is locked
5. `shell/commands/commandRunner.ts` → `core/commands/parseCommand.ts` (tab phrases first,
   `parseTabCommand.ts`) with `shell/commands/installedApps.ts` → `runIntent.ts`: `open -a`,
   `osaScript.ts` AppleScript templates with `on run argv`, or `open <url>` ║ macOS. Keys
   and typing go through `helperIntents.ts` (`mod`/`alt` mapped to ⌘/⌥) ║ stdio ║ the
   helper's `key` and `type` verbs. Tabs: `tabIntents.ts` picks the named, frontmost or
   default browser → `browserScripts.ts` JXA lists tabs by index ║ Apple Events ║ the browser
   → `core/commands/matchTabs.ts` → close or switch by index. Drafts:
   `core/commands/draftUrl.ts` → `open mailto:` or Gmail compose, never sent
5a. When the parser has no match or no such app: `shell/interpret/interpretApi.ts` (Mac session
   bearer, 7 s) ║ HTTPS ║ `app/api/interpret/route.ts` → `features/interpret/server/interpret.ts`
   (model from `models.ts`) ║ HTTPS ║ Anthropic → up to 3 actions →
   `core/commands/validateActions.ts` (all or nothing; `shortcutRules.ts` denylist,
   `emailRules.ts`, a draft only alone) → run in order, waiting in `frontApp.ts` until an
   opened or switched app is in front before keys or typing. AI unreachable, timed out or
   invalid → "Smart commands aren't working right now"; no actions → the parser's message
6. `command.result { status, message }` ║ LAN ║ the phone shows Heard "…" and the message.
   No `command.received` in 5 s → "Can't reach"; no result 15 s after it → "No answer".
   Try again resends the same `commandId`, or re-uploads the same file if transcription
   failed

## Click and choose (mobile ⇄ agent ⇄ helper)

1. "Click Save" → `core/commands/parseScreenCommand.ts` (or the AI's `click` action) →
   `commandRunner.ts` `runGuarded`: front bundle id from `frontApp.ts` against
   `core/safety/blockedApps.ts` (unknown → refused)
2. `shell/commands/clickIntent.ts` → helper `snapshot` ║ stdio ║ `snapshotWalk.swift`
   (title, description, help, placeholder, subrole, enabled) → `core/resolver/resolveTarget.ts`
3. One match: destructive (`core/resolver/destructiveLabels.ts`) → confirm choice; otherwise
   wait 100 ms, snapshot again, `core/resolver/stableTarget.ts` (same app, window, element
   count; same label and place) → helper `click` (`AXPress`, else a coordinate click)
4. Several matches or a miss: helper `overlay` ║ stdio ║ `overlay.swift` badges (converted by
   `cocoaRect` in `displays.swift`) → `command.result { needsChoice, choices, confirm }`;
   `choiceBook.ts` keeps it 20 s and drops it on a new command or `frontWindowChanged`
   (`frontWindowWatch.swift`)
5. Phone `talk/components/choiceSheet.tsx` (`@gorhom/bottom-sheet`) → `talkFlow.choose` →
   `command.choose { commandId (new), parentId, number }` ║ LAN, Noise ║
   `commandReceiver.ts` → `ChoiceBook.pick` → the same checks as step 3; stale → "The screen
   changed, say it again"
6. `commandReceiver.ts` records each outcome in `core/safety/breaker.ts`; tripped → every
   command answers a one-option Resume choice until it is picked

## Helper command (agent)

1. Caller (`dev/consoleCommands.ts` today, the pipeline in feature 7) →
   `core/helper/helperClient.ts` `request(verb, args)`
2. One JSON line with an `id` ║ stdin ║ `main.swift` reader thread → main queue →
   `router.swift`
3. `router.swift` → `snapshot.swift` / `clickElement.swift` / `keyboard.swift` /
   `screenCapture.swift` ║ macOS AX, CGEvent, ScreenCaptureKit
4. One JSON line back ║ stdout ║ `helperClient.ts` parses with `helperProtocol.ts`, matches
   the `id`, returns a `HelperOutcome`
5. Helper events (`ready`, `displaysChanged`) go to `onEvent`. Helper exit fails in-flight
   requests with `helper_exited`, restarts with backoff, replays nothing.
