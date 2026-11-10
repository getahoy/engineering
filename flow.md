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
| web | `src/app/page.tsx` | Landing page at `/` (static), `features/landing/screens/landingScreen.tsx` |
| web | `src/app/api/waitlist/route.ts` | Early-access signup from the landing page capsules |
| web | `src/app/api/history/**/route.ts` | Command history: list (GET), write batch (POST), search (POST `/search`), clear (DELETE), `[id]` delete / restore / report, settings; logic in `features/history/server/` |
| web | `src/app/api/history/retention/route.ts` | Daily Vercel Cron (`vercel.json`), `Bearer CRON_SECRET` only; purges history over 90 days and stale soft deletes |
| web | `scripts/evalInterpret.ts` (`pnpm eval:interpret`) | Manual run of the cases in `scripts/interpretCases.ts` against the interpret route, checked automatically, with a pass/fail table |
| mobile | `src/app/_layout.tsx` | Reinstall wipe, then `features/auth/components/sessionRoot.tsx` (launch gate, Expo Router stack, session guards); `getRandomValues` polyfill |
| mobile | `src/app/pair.tsx` | QR scanner and pairing claim |
| agent | `src/main.ts` | Electron main process, composition root |
| agent | `src/renderer/main.tsx` | One renderer bundle; `?surface=window` main window, `popover` menu bar popover, `gallery` dev-only component gallery |
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
3. Session signal → `sessionRoot.tsx` `Stack.Protected` guard flips → `index.tsx` account

## Cold launch (mobile)

1. `app/_layout.tsx` → `useInstallWipe` (MMKV `ahoy.install`/`installSeen`; missing →
   `client/wipeAuthStorage.ts` ║ SecureStore ║ writes `{}` to `ahoy_cookie` and
   `ahoy_session_data`). Native splash stays up; nothing below mounts until this settles.
2. `components/sessionRoot.tsx` → `hooks/useSessionGate.ts`:
   - `useLaunchCookie` → `authClient.getCookie()` ║ SecureStore ║ → `launchCookie.ts`
     (`present` / `absent` / `unreadable`)
   - `authClient.useSession()` → expo plugin `init` hydrates the cached session ║ SecureStore ║
     then `/api/auth/get-session` ║ HTTP ║ web Better Auth (keeps running in the background)
   - `sessionGate.ts` `sessionReady`: cache + cookie, or no cookie, or a server answer
     (data, null, or 401), or the 6 s give-up → `ready` (sticky). SigningIn only after
     `SPLASH_LIMIT_MS` (`hooks/useSplashHold.ts`).
3. Ready + signed in → `welcome/hooks/useLaunchIntro.ts` latches in render →
   `LaunchIntro` over Home; `pairing/hooks/usePairedMacs.ts` →
   `sync/preparePairings.ts` (once per latest user: `claimPairingsFor`) → settle → read
   ║ SecureStore ║ → dialling starts; `useLinkSync` settles revoked Macs alongside ║ HTTP ║
   and reloads the list.
4. Ready + signed out → splash fades into `welcome`.
5. `hooks/useClearStaleSession.ts`: a later 401 on `/get-session` →
   `client/signOutOnce.ts` (once per cookie), or a cached session with no stored cookie →
   `authClient.signOut()`; either clears the cached session → `Stack.Protected` fades to
   `welcome`.

## Sign in with Google

- web: `useSocialSignIn` → Better Auth redirect ║ Google ║ `/api/auth/callback/google` →
  `/account`, or `/sign-in?error=<code>` → `socialSignInError.tsx` → `socialErrorMessage`
- mobile: `useSocialSignIn` → expo client opens `expo-web-browser` sheet ║ Google ║ web callback
  ║ `ahoy://` back into the app → session or the generic `error.social` message

## Join the waitlist (web)

1. `components/waitlistCapsule.tsx` (hero `source: "hero"`, CTA band `"cta"`) →
   `hooks/useWaitlistCapsule.ts` checks the email with `forms/waitlistRequest.ts`
   `waitlistEmailSchema`; invalid stops here with the inline message
2. `hooks/useJoinWaitlist.ts` → `request()` (`@getahoy/lib`) ║ HTTP POST `/api/waitlist` ║
   `apiRoute("waitlist")`
3. `server/waitlistRateLimit.ts` → `consumeRateLimit("waitlist:ip:<x-real-ip>", 5/min)` ║ Neon
   → over the limit `rate_limited` 429
4. `readJson(waitlistRequestSchema)` → filled `company` honeypot answers success, saves nothing
5. `server/joinWaitlist.ts` → insert `waitlist_signups` on conflict do nothing ║ Neon → the same
   `{ joined: true }` for new and existing emails; the log line holds the outcome only
6. Capsule: `useSpinnerVisible` (300 ms delay, 500 ms minimum) → success in place, or
   `errors/waitlistNotice.ts` maps the `RequestError` to one line plus retry

## Agent startup

1. `main.ts`: single-instance lock, hide the Dock, read `settings.json` (`shell/settingsStore.ts`)
2. Build `MacosPlatform`, `AgentController`, `HelperClient`; `shell/shellSurfaces.ts` builds
   `MainWindow`, `PopoverWindow` (kept hidden, reopens instantly) and `AgentTray`;
   `shell/bridgeIpc.ts` registers the IPC handlers and pushes state to every window
3. `HelperClient.start()` ║ spawn ║ `ahoyHelper` (debug build in dev,
   `Contents/Resources/ahoyHelper` when packaged); `ready` and `exited` update the controller
4. Permission polling every 2 s (`systemPreferences`) and `powerMonitor` lock events update the
   controller, which swaps the tray pennant icon (`menubarStatus`) and pushes state to the windows
5. Onboarding not done → `MainWindow.show()` (Dock shown, focus stolen) ║ IPC ║ `preload.ts`
   bridge ║ `renderer/agentApp.tsx` picks a `setup/` step or `window/windowApp.tsx` from
   `state.screen`
6. A button press ║ IPC ║ `AgentController.completeStep` → writes `settings.json` → new state

## Menu bar popover (agent)

1. Tray left click → `PopoverWindow.toggle(trayBounds)`: placed under the icon on the display
   that holds it, always on top while open; blur or Esc hides it
2. Renderer `popover/popoverApp.tsx` reads state over the same bridge; a `ResizeObserver` sends
   its height ║ IPC `setPopoverHeight` ║ window resized
3. Open Ahoy / Show QR code ║ IPC `openWindow` → main window shows, then popover hides, so a
   code started while the popover was open survives the switch. Right click
   → native menu (Open Ahoy, Quit Ahoy, Component gallery in dev)

## Appearance (agent)

1. Startup: `shell/settingsStore.ts` reads `appearance` (`system` default) →
   `shell/shellSurfaces.ts` sets `nativeTheme.themeSource` before any
   window exists, so window backgrounds, popover material and `prefers-color-scheme` match
2. Renderer load: `preload.ts` ║ sync IPC `agent:appearance` ║ `shell/appearanceIpc.ts` →
   `initialAppearance`; `renderer/main.tsx` seeds next-themes' localStorage copy from it,
   then `appearance/appearanceProvider.tsx` (next-themes, class + inline `color-scheme`)
3. Settings › Appearance (`appearance/appearanceGroup.tsx`): ring moves at once; 60 ms later
   `useAppearanceSwitch` → `document.startViewTransition` → ║ IPC `agent:change-appearance` ║
   `themeSource` set, `settings.json` written, resolved theme returned → `schemeSettled`
   waits for `prefers-color-scheme` → `flushSync(setTheme)`; 240 ms crossfade
4. Popover: next-themes' `storage` event (same origin) plus the new `themeSource`

## Appearance (mobile)

1. `app/_layout.tsx` imports `appearance/applyStoredAppearance.ts` first: reads the system
   scheme, then MMKV `ahoy.appearance` → `Uniwind.setTheme` (which also sets
   `Appearance.setColorScheme`) before the first render
2. Launch: setting the theme also overrides the native appearance, so iOS repaints the
   splash in the chosen theme straight away; `hooks/useRootBackground.ts` in
   `sessionRoot.tsx` sets the native root background (`expo-system-ui`) to the themed
   Sailcloth on launch and on every change, behind modals, sheets and the keyboard
3. Settings › Appearance (`components/appearanceGroup.tsx`): selection haptic, radio and ring
   move at once; 60 ms later `useAppearance().setTheme` writes MMKV and sets Uniwind;
   `components/appearanceFade.tsx` in `sessionRoot.tsx` sees the theme change in render and
   fades the previous canvas out over 240 ms (any theme change, including System at night)

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

1. `shell/pairing/surfacePairing.ts` is the only owner of `shell/pairing/pairingSession.ts`.
   On a surface showing or hiding, and a microtask after any state change (a phone ending
   the pairing): no visible window or popover → pause; `needsPairingCode`
   (`core/appState/appState.ts`) → start; otherwise resume, decided by
   `core/pairing/pairingMove.ts`. Pause keeps a ready code and stops all network; resume
   polls the same code again if it has 30 s left, else makes a new one. Anything not ready
   is discarded on pause. `stop()` (discard) is only Cancel, sign-out and quit. The
   renderer never starts or stops a session; IPC `startPairing` is only "Get a new code" and
   the popover's Show QR code (then `openWindow`), through the same visibility gate. A
   session that shows Scanned is never stopped, so an approved claim always completes
2. `PairingSession` runs the code machine `core/pairing/pairingCode.ts` (events in, status
   and next effect out): begin → `starting` (or `offline` when `net.isOnline()` is false,
   checked every 2 s until back) → `slow` at 6 s → `offering`; a missed request retries
   once silently, then `failed`. At expiry → `refreshing` (old code kept) → new `offering`,
   or `expired` when the refresh misses. Poll approved → `scanned` → complete → `paired`
3. `shell/pairing/deviceFlow.ts` → Better Auth client `device.code` ║ HTTP ║
   `/api/auth/device/code` (`deviceAuthorization`, client `ahoy-desktop`, 5 min) ║ Neon
   `device_code`. 15 s deadline per request and per poll (`core/pairing/deadline.ts`); a
   late poll counts as a network miss and polls again
4. Renderer `pairing/pairingCode.tsx` draws one `QrFrame` everywhere (168 px, 216 px on
   first run): the code is an SVG from `qrcode-generator` at level H with the pennant on a
   9×9-module Paper square (`components/ui/qrCode.tsx`); spinner only after 300 ms and held
   500 ms (`useShownCode`); a three-line `aria-live` caption per state (`pairingCodeCopy`).
   Payload: Mac key, name, user code, one-time secret (never sent to web), LAN addresses +
   port 7421, expiry. Polls `device.token` at the returned interval, slowing down when told
5. Phone `pairScreen.tsx` → `ScannerPreview` (expo-camera) → `usePairScanner` →
   `useClaimMac` (state machine in `scannerFlow.ts`, one bottom `ScannerMessageSlot`) →
   `enrollment/claimScannedCode.ts`: proof = keyed BLAKE2b(secret, phone key ‖ Mac key);
   claim errors map to typed kinds in `enrollment/claimFailure.ts`
6. `POST /api/pairing/claim` with the session cookie ║ HTTP ║ `claimPairing`: one
   transaction upserts the phone in `devices` and writes `pairing_claims` (user code, phone,
   Mac key, proof), then `auth.api.deviceApprove`; a refusal deletes the claim
7. Next agent poll returns a session token (Mac shows Scanned) ║ `POST /api/pairing/complete` with
   `Authorization: Bearer` ║ `completePairing`, one transaction: match the claim to the
   session user and Mac key, upsert the Mac, link both in `device_links`, burn the claim,
   return the phone and proof ║ agent checks the proof (`verifiedPhone`); a mismatch refuses
   the pairing. Saves the phone and session in `identity.bin` (`safeStorage`)
8. Phone saves the Mac (key, name, addresses) in SecureStore and returns home

## Phone connects to Mac (LAN)

1. Agent `PhoneLink` → `LanListener` binds 127.0.0.1 + each private LAN IPv4 on 7421,
   re-syncing addresses every 5 s, at most 20 handshakes per minute per address. It runs
   whether or not "Listen for my phone" is on
2. Phone `LiveMacCard` → `useMacConnection` → `MacConnection` → `connectToMac` →
   `lanTransport` races the saved addresses ║ WebSocket on Wi-Fi ║ `WebSocketFrameStream`
3. Noise IK message 1 carries the phone hello ║ agent `PhoneSession` → `acceptHandshake`:
   undecryptable → plaintext `unknown_mac_key`; unknown key → signed `not_paired` with the
   tombstone reason or `unknown` (payload never parsed); other version → `version_mismatch`;
   else `welcome`
4. Phone sends an encrypted `ready`; `PhoneSession` checks listening then, not at hello,
   so a switch flipped mid-handshake applies. Listening on: the Mac counts it connected, sends
   `status { locked }` and repeats it on every lock change. Listening off: the Mac sends
   `goodbye turned_off`, closes, and `ListenSwitch` (`shell/lan/listenSwitch.ts`) shows
   "Your iPhone tried to connect" at most once per 10 min. The phone counts itself connected
   and registers the live session only on the first `status`
5. Timeout or close before a reply → phone `unreachable`, auto-retry in the foreground at
   300 ms, 1 s, 2 s, then every 5 s (`connection/linkTiming.ts`; off/asleep/quit wait 5 s).
   Only a signed refusal ends the pairing: `sync/endedMacs.ts` removes the Mac and saves
   the one-time notice
6. Background: the session stays open 20 s, then closes. Foreground: `resumeLink` pings
   the open session (500 ms); a pong keeps it, silence or no session redials. Device keys
   stay in memory after the first SecureStore read. Dev builds log per-stage dial timing
   (`connection/dialMac.ts`, `macLink.timing`)

## Permissions (mobile)

1. Camera: `pairScreen.tsx` → `useCameraGate` → `permissions/hooks/usePermissionGate.ts`
   reads `permissionAccess.camera` on mount and on every AppState `active` → stage
   (`permissionStage.ts`: explainer / again / guide / granted) → `PermissionGateView`.
   `CameraView.onMountError` → couldn't start
2. Microphone: hold to talk → `TalkFlow.press` → `useFileRecorder.start` reads the mic
   permission fresh and returns `recording`, `needed` or `failed` ║ `needed` →
   `/microphone` (explainer, again, guide); `failed` → `/microphone?failed=1`. Nothing
   records on that press
3. Open Settings: `rememberSettingsReturn` (MMKV `resumePairing` / `resumeTalk`) →
   `Linking.openSettings()` ║ OS Settings ║ Android resumes → gate re-reads, clears the
   flag; iOS restarts → `macsHomeScreen` → `useTurnedOnNotice` takes the flag → `/pair`
   with `resumed=1` (haptic, Camera is on) or Home with Microphone is on
4. Local Network (iOS): pairing done → `/local-network` explainer before any dial
   (`useLocalNetworkIntro` holds `useKeepMacsConnected`) → Continue → first LAN dial ║ iOS
   popup. `useLocalNetworkCheck` opens the guide once when the Mac is `unreachable`, the
   phone is on Wi-Fi and `hasReachedOnLan` is false; otherwise the Can't reach card links to
   it. `MacConnection` marks `reachedOnLan` on every handshake; the guide closes to Home
   with Local Network is on when the Mac connects

## Pairing ends (either side)

1. Remove / sign out / pair a new phone: the acting side sends `goodbye { reason }` on the
   live socket, then queues the unlink in its outbox (`identity.bin` on the Mac,
   `pendingUnlinks` in SecureStore on the phone) ║ HTTP ║ `/api/pairing/unlink` revokes the
   `device_links` row with `revoked_by` + `revoke_reason`; a new phone is revoked inside
   `completePairing` as `replaced`
2. The other side, if connected, handles the goodbye at once, only if its own record still
   has the peer. Mac `phoneEnded.ts` → `phoneEnding` (core): tombstone, overlay, speech and
   choices cleared, `endedNotice` in state, `PairingSession.restartAfterPhoneEnded` (fresh
   QR unless one is showing), a notification only when no Ahoy window is visible → U1 in
   Home, Pair a new phone and the popover. Phone `MacConnection` → `endPairedMac`: Mac
   removed, MMKV notice `{ reason, deviceName, at, whileClosed }` → U1 on "Pair your Mac"
   or U2 on the remaining Mac's home, until a new pairing, sign-out, ✕ / next command (U2)
   or 24 h
3. If it was offline: Mac `LinkMonitor` (start, every 15 min, wake, window or popover open) and phone
   `useLinkSync` (start, foreground) read `/api/pairing/links` and treat a revoked link like
   the goodbye. A later handshake gets the signed `not_paired` from the Mac's tombstone
4. Pause: phone `goodbye paused` + stored `paused`; Mac shows "paused". Ahoy off, quit and
   sleep send `turned_off`, `quitting`, `sleeping`; the phone keeps that reason per Mac in
   MMKV `ahoy.macGoodbyes` (`connection/macLinkMemory.ts`), shows it on launch and on every
   miss instead of Can't reach, and clears it on the next `status`. Turning off from the Mac
   stores `lanOffUntil` (1 hour, next 08:00, or none) in settings; `ListenSwitch` turns
   listening back on once it passes (checked at start and every 30 s). Completing a pairing
   turns listening on

## Talk (mobile ⇄ web ⇄ agent)

1. Hold the button: `pairing/screens/macsHomeScreen.tsx` → `talk/hooks/useTalk.ts` →
   `talk/sending/talkFlow.ts` `press` → `talk/recorder/useFileRecorder.ts` (expo-audio, mono
   AAC m4a, stops at 30 s). The target Mac comes from `talk/hooks/useTalkTarget.ts`;
   `pairing/components/macHome.tsx` reads one steady link state for the chip and the talk
   block (`talk/talkBlock.ts`: connecting never blocks). If the link is still dialling,
   `talk/sending/askMac.ts` waits for it (`waitForSession`, 5 s, "Check your Mac" at 3 s)
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

## History (mobile ⇄ web, agent ⇄ web)

1. The result frame also carries `app`, `via`, `onMacMs` and `reason`
   (`agent/src/shell/commands/resultFrame.ts`, reason from `core/commands/outcomeDetails.ts`)
2. Phone writes: `talk/components/talkPanel.tsx` → `history/hooks/useHistoryWriter.ts` watches
   the talk states → `history/historyTracker.ts` → `historyRecords.ts` (status mapping,
   follow-ups, timings) → MMKV outbox (`history/storage/historyStore.ts`) →
   `history/sync/flushHistory.ts` on write, foreground and reconnect ║ HTTPS ║
   `POST /api/history` → `features/history/server/writeHistory.ts` (rev-guarded upsert)
3. Phone reads: Settings → History (`app/history/index.tsx`) → `history/hooks/useHistoryList.ts`
   (MMKV cache first) ║ HTTPS ║ `GET /api/history` or `POST /api/history/search` →
   `listHistory.ts`; unsent outbox entries are merged in by `history/localEntries.ts`
4. Mac reads: window History item → `renderer/history/useHistoryList.ts` → `window.ahoy` ║ IPC ║
   `shell/history/historyIpc.ts` → `historyApi.ts` (bearer, `macKey` = this Mac) ║ HTTPS ║ web
5. Run again: phone entry → `history/hooks/useRunAgain.ts` selects the Mac and goes Home with
   the words → `talk/hooks/useRunAgainOnArrival.ts` → `talkFlow.runText` (step 3 of Talk).
   Mac entry ║ IPC ║ `shell/history/runLocally.ts` → `CommandReceiver`, silent, not recorded
6. Report: entry → report sheet or dialog ║ HTTPS / IPC ║ `POST /api/history/:id/report` →
   `reportProblem.ts` copies the stored entry into `problem_reports`
7. Retention: Vercel Cron ║ HTTPS ║ `GET /api/history/retention` (`Bearer CRON_SECRET`)

## Click and choose (mobile ⇄ agent ⇄ helper)

1. "Click Save" → `core/commands/parseScreenCommand.ts` (or the AI's `click` action) →
   `commandRunner.ts` `runGuarded`: front bundle id from `frontApp.ts` against
   `core/safety/blockedApps.ts` (unknown → refused)
2. `shell/commands/clickIntent.ts` → helper `snapshot` ║ stdio ║ `snapshotWalk.swift`
   (title, description, help, placeholder, subrole, enabled) → `core/resolver/resolveTarget.ts`
3. One match: destructive (`core/resolver/destructiveLabels.ts`) → confirm choice; otherwise
   wait 100 ms, snapshot again, `core/resolver/stableTarget.ts` (same app and window; the
   target has the same label and place) → helper `click` (`AXPress`, else a coordinate click)
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

## Voice out (agent ⇄ web ⇄ mobile)

1. Every phone `command` carries `speakOn` from `voice/storage/voiceSettings.ts` (MMKV)
2. Mac `commandReceiver.ts` → `resultFrame.ts` → `core/voice/buildSpeech.ts` (a short line,
   never screen text) → `command.result { threadId, message, speech }` ║ LAN, Noise ║ phone
3. Phone `voice/hooks/useVoiceReplies.ts` → `voice/replyAction.ts` (policy, current, age,
   recording, screen reader) → speak, chime (`assets/sounds/done.m4a` + haptic), announce
   (`announceForAccessibility`) or quiet
4. Speak: `voice/engine/fallbackVoice.ts` → `elevenLabsVoice.ts` (`speechCache.ts`, else
   `client/speakApi.ts`) ║ HTTPS ║ web `app/api/speak/route.ts` → `features/speech/server/
   elevenLabs.ts` ║ HTTPS ║ ElevenLabs → MP3 → `audioPlayback.ts` (expo-audio); no audio in
   1.5 s or an error → request aborted → `deviceVoice.ts` (expo-speech)
5. `speakOn` includes the Mac: `shell/voice/macVoice.ts` → the same `/api/speak` with the Mac
   bearer → temp file → `afplay` ║ process ║; failure → `say` (argument list)
6. Press talk: phone `talkFlow.press` → voice stop (80 ms fade) → `speech.stop` and
   `choice.extend` ║ LAN ║ Mac `commandReceiver.ts` (only from the owning session)
7. Voice preview: Settings → `voice/components/voiceSheets.tsx` → `previewPill.tsx` →
   `hooks/useVoicePreview.ts` → `engine/voicePreview.ts` (`previewState.ts`) →
   `engine/previewAudio.ts` → `previewSource.ts`: `speechCache.ts`, else the bundled
   `assets/voice-samples/<voice>.mp3` while `manifest.json` matches, else `prepareSpeech`
   ║ HTTPS ║ `/api/speak`, else `deviceVoice.ts` → `engine/previewPlayer.ts` (expo-audio).
   Talk losing focus stops any reply first (`useVoiceReplies.ts`)
8. Samples are built offline: web `scripts/buildVoiceSamples.ts` (`pnpm voice:samples`) →
   `features/speech/server/speechVoices.ts` (model, format, voice id) ║ HTTPS ║ ElevenLabs
   → `mobile/assets/voice-samples/`

## Spoken choice (mobile ⇄ agent)

1. A choice is open in `choiceBook.ts`; the user holds talk and says "two" → `command`
2. `commandTurns.ts` at run time: same session, choice open, ≤ 4 words →
   `core/voice/parseChoiceAnswer.ts` → pick, out of range, cancel or delegate; anything else
   expires the choice (replaced) and runs as a new command
3. `choiceAnswers.ts`: a pick → `ChoiceBook.pick` → the safe-click path; a destructive confirm
   is tap only; "you pick" → option one unless destructive (`core/voice/threadRules.ts`); more
   than three follow-ups → "Let's start over."
4. The result keeps the root `threadId` → phone `talk/hooks/useThread.ts` →
   `talk/threads/threadStore.ts` → `talk/components/threadView.tsx`

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
