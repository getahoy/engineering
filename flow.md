# Flow

Entry points per app, then one call chain per major flow. `║` marks a process, network or port
boundary.

## Entry points

| App | Entry | Starts |
|---|---|---|
| web | `src/app/**/page.tsx`, `src/app/api/auth/[...all]/route.ts` | Next.js pages, Better Auth API |
| web | `src/proxy.ts` | Session check before `/sign-in`, `/sign-up`, `/account` render |
| mobile | `src/app/_layout.tsx` | Expo Router stack, session guards |
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
