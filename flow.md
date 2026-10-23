# Flow

Entry points per app, then one call chain per major flow. `║` marks a process, network or port
boundary.

## Entry points

| App | Entry | Starts |
|---|---|---|
| web | `src/app/**/page.tsx`, `src/app/api/auth/[...all]/route.ts` | Next.js pages, Better Auth API |
| web | `src/proxy.ts` | Session check before `/sign-in`, `/sign-up`, `/account` render |
| web | `src/app/api/pairing/{claim,complete}/route.ts` | Pairing API, each wrapped by `features/api/server/apiRoute.ts`; device codes live under `/api/auth/device/*` (Better Auth) |
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
2. Route: authenticate → `readJson(request, schema)` → work → `SuccessResponse` or
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
   re-syncing addresses every 5 s; tray "Ahoy On" off closes every socket
2. Phone `PairedMacItem` → `useMacConnection` → `connectToMac` → `lanTransport` races the
   saved addresses ║ WebSocket on Wi-Fi ║ `WebSocketFrameStream` (ping every 15 s)
3. Noise IK message 1 carries the phone hello ║ agent `PhoneSession` →
   `acceptHandshake`: unknown phone key → closed before parsing; other protocol version →
   `refused`; else `welcome` with the Mac hello
4. Phone sends an encrypted `ready` frame; only then does the Mac count it connected and the
   tray icon turns to `iphone.radiowaves.left.and.right`
5. Socket close → phone shows Wi-Fi vs mobile data (`expo-network`), Mac drops the phone

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
