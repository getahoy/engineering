# Launch, session and local-data consistency — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Signed-in cold launch goes splash → LaunchIntro → Home with no SigningIn flash, and
the phone's session and stored Macs stay consistent with the server.

**Architecture:** The launch gate trusts the SecureStore-cached session plus a stored-cookie
probe instead of waiting for `/get-session`; every decision is a pure function under
`node:test`, wrapped by thin hooks. App API calls go through one session-recovering
wrapper. Stored Macs get a versioned payload and one write queue.

**Tech Stack:** Expo 58 / React Native, expo-router `Stack.Protected`, better-auth +
@better-auth/expo 1.7.11, expo-secure-store, react-native-mmkv, Reanimated, Next.js +
better-auth server, `node --test`.

**Spec:** `engineering/launch-session-spec.md` (read it first; this plan argues from it).

## Global Constraints

- Never touch `shared/`. Edit/Write only; no sed/python/script rewrites.
- `better-auth` and `@better-auth/expo` stay exactly `1.7.11` in `mobile` and `web`.
- Phase 1 = Tasks 1–9 (spec Parts 1 + 2). After Task 9: STOP, uncommitted, for the user's
  device check. Phase 2 (Tasks 10–15) starts only when the user says so.
- No branches (user override): commit on `main` in each repo, never push unless told.
- **No commits per task (user override of the small-commits rule).** One commit per repo at
  the very end (Task 15), after asking the user and showing staged paths + message.
  Explicit paths only. Every "checkpoint" step below means: nothing staged, move on.
- Commit messages: `type: lowercase imperative`, ≤72 chars, short bullets, no quotes or
  apostrophes, no trailer, no footer.
- **Tests are build-time only (user override).** Write and run them per task as below
  (`node:test` `*.test.ts` next to pure files; relative imports inside pure files use the
  `.ts` extension, see `src/features/pairing/endedPairing.ts`; run `cd mobile && pnpm test`),
  then delete every `*.test.ts` this plan adds before the final commit. The existing
  `mobile/src/features/auth/sessionGate.test.ts` is rewritten in Task 2 and deleted too,
  since the old one no longer matches the new signature. Keep pure files pure anyway.
- Checks per task: `pnpm typecheck` and `pnpm lint` clean (zero warnings).
- Files < 200 lines, no comments except a real why, no `!`, no `as` escapes, banned names
  (`data` only as the wire field, `result`, `value`, …) per CLAUDE.md.
- Logging only through `logger` (`mobile/src/features/logging/logger.ts`), never contents.

## Review Focus

1. Offline cold launch, cookie present, no cache: gate waits for the 6 s give-up, then shows
   Welcome; reconnect then moves to Home. Never Welcome at once. → Task 2 test
   `offline with cookie and no cache waits for give-up`.
2. Cached session but cookie expired (session_token past expiry): signed out at launch,
   never Home for one frame. → Task 2 test `cookie absent ignores cached session`.
3. Two screens mounting `usePairedMacs` at once right after sign-in: one claim, one link
   sync. → Task 8 test `oncePerKey runs once for concurrent callers`.
4. A rejected `oncePerKey` run (storage throw) must not stick: the next call retries.
   → Task 8 test `oncePerKey forgets a rejected run`.
5. Retry after 401 on transcribe sends a fresh `FormData` (consumed body never reused).
   → Task 10 test `retry calls build again`.

---

## Phase 1 — mobile (spec Parts 1 + 2)

### Task 1: singleFlight and oncePerKey

**Files:**
- Create: `mobile/src/lib/singleFlight.ts`, `mobile/src/lib/oncePerKey.ts`
- Test: `mobile/src/lib/singleFlight.test.ts`, `mobile/src/lib/oncePerKey.test.ts`

**Interfaces:**
- Produces: `singleFlight<T>(run: () => Promise<T>): () => Promise<T>` — concurrent callers
  share one in-flight promise; cleared on settle (resolve or reject).
- Produces: `oncePerKey<T>(run: (key: string) => Promise<T>): (key: string) => Promise<T>`
  — first call per key runs; later calls get the same promise for the app run; a rejected
  run is forgotten so the next call retries.

- [ ] **Step 1: Write failing tests**
  - `singleFlight shares one run between concurrent callers`: two calls before resolve →
    run count 1, both get the same answer.
  - `singleFlight runs again after settling`: call, await, call → run count 2.
  - `oncePerKey runs once for concurrent callers`: two calls `"a"` → count 1.
  - `oncePerKey keeps keys apart`: `"a"`, `"b"` → count 2.
  - `oncePerKey forgets a rejected run`: first run rejects; second call → count 2, resolves.
- [ ] **Step 2: Run `pnpm test`** → FAIL (modules missing).
- [ ] **Step 3: Implement both** (module-local `Map` / variable inside the returned closure).
- [ ] **Step 4: Run `pnpm test`** → PASS.
- [ ] **Step 5: Checkpoint.**

### Task 2: Launch gate decision

**Files:**
- Modify: `mobile/src/features/auth/sessionGate.ts`
- Create: `mobile/src/features/auth/launchCookie.ts`
- Test: `mobile/src/features/auth/sessionGate.test.ts` (rewrite),
  `mobile/src/features/auth/launchCookie.test.ts`

**Interfaces:**
- Produces in `launchCookie.ts`:
  `type LaunchCookie = "pending" | "present" | "absent" | "unreadable"`;
  `readLaunchCookie(read: () => Promise<string>): Promise<Exclude<LaunchCookie, "pending">>`
  — non-empty → `present`, `""` → `absent`, throw → `unreadable`.
- Produces in `sessionGate.ts`:
  `type SessionFailure = "none" | "unauthorized" | "other"`;
  `type GateInput = { cookie: LaunchCookie; hasSession: boolean; isPending: boolean; failure: SessionFailure; gaveUp: boolean; showSigningIn: boolean }`;
  `sessionReady(input: GateInput): boolean`;
  `nextSessionGate(previous: SessionGate, input: GateInput): SessionGate`;
  `launchSignedIn(input: Pick<GateInput, "cookie" | "hasSession" | "isPending">): boolean`
  — while `isPending`, `hasSession && cookie !== "absent"`; after, `hasSession`.
- Ready rule (spec): `(hasSession && cookie === "present") || cookie === "absent" || (!isPending && (hasSession || failure !== "other")) || gaveUp`.
  `nextSessionGate`: ready sticky → `"ready"`; ready → `"ready"`; `showSigningIn` →
  `"signingIn"`; else `"waiting"`. Comment above it states the why: only the first load may
  hold the app; later refetches flip `isPending` and must not unmount the navigator.

- [ ] **Step 1: Write failing tests** (named, one assertion group each):
  `cached session with cookie opens at once`; `cookie absent opens signed out`;
  `cookie absent ignores cached session` (`launchSignedIn` false);
  `server answer opens`; `401 opens`; `offline with cookie and no cache waits for give-up`
  (`isPending:false, failure:"other", hasSession:false, cookie:"present"` → not ready; with
  `gaveUp:true` → ready); `unreadable cookie never opens early`
  (`cookie:"unreadable", isPending:true, hasSession:true` → not ready);
  `pending cookie waits`; `ready is sticky` (from `"ready"` with every input false →
  `"ready"`); `signing in shows only when asked`;
  `readLaunchCookie classifies present, absent and a throw`.
- [ ] **Step 2: Run `pnpm test`** → FAIL.
- [ ] **Step 3: Implement** both files.
- [ ] **Step 4: Run `pnpm test`** → PASS.
- [ ] **Step 5: Checkpoint.**

### Task 3: Reinstall wipe

**Files:**
- Create: `mobile/src/features/auth/installWipe.ts` (pure),
  `mobile/src/features/auth/storage/installMarker.ts` (MMKV `createMMKV({ id: "ahoy.install" })`,
  key `installSeen`, `readInstallSeen(): boolean`, `markInstallSeen(): void`),
  `mobile/src/features/auth/client/wipeAuthStorage.ts`
  (`wipeAuthStorage(): Promise<void>`: `storageAdapter(SecureStore)` from
  `@better-auth/expo/client`, `setItemAsync` `"{}"` on `` `${APP_SCHEME}_cookie` `` and
  `` `${APP_SCHEME}_session_data` ``),
  `mobile/src/features/auth/hooks/useInstallWipe.ts` (`useInstallWipe(): boolean` — `true`
  once settled; synchronous `true` on first render when the flag exists)
- Test: `mobile/src/features/auth/installWipe.test.ts`

**Interfaces:**
- Produces: `runInstallWipe(steps: { seen: () => boolean; wipe: () => Promise<void>; markSeen: () => void }): Promise<void>`
  — seen → return; else `try { await wipe() } catch { swallow, the catch is the why } finally { markSeen() }`.
  A failed wipe is logged via `logger.error("auth.install", "wipe failed", error)` by the hook,
  not the pure function (pure file has no logger import) — so `runInstallWipe` rethrows
  after `finally`, and the hook catches + logs.

- [ ] **Step 1: Failing tests**: `seen install does nothing` (wipe count 0);
  `new install wipes then marks`; `failed wipe still marks seen` (wipe rejects → markSeen
  called once, promise rejects).
- [ ] **Step 2: `pnpm test`** → FAIL. **Step 3:** implement all four files.
- [ ] **Step 4: `pnpm test`** → PASS; `pnpm typecheck`.
- [ ] **Step 5: Checkpoint.**

### Task 4: Wire the gate, probe and splash into the root

**Files:**
- Create: `mobile/src/features/auth/hooks/useLaunchCookie.ts`
  (`useLaunchCookie(): LaunchCookie` — `readLaunchCookie(authClient.getCookie)` once on mount)
- Create: `mobile/src/features/auth/components/sessionRoot.tsx` — the current body of
  `RootLayout` (gate, intro, Stack, overlays) moves here unchanged except as below
- Modify: `mobile/src/app/_layout.tsx` — `RootLayout` calls `useInstallWipe()`; `false` →
  `return null` (splash stays: `preventAutoHideAsync` runs at module import); `true` →
  `<SessionRoot />`. Order is wipe → probe → mount because `useLaunchCookie` and
  `useSession` live only in `SessionRoot`.
- Modify: `mobile/src/features/auth/hooks/useSplashHold.ts` — `export const SPLASH_LIMIT_MS`
- Modify: `mobile/src/features/auth/hooks/useSessionGate.ts`

**Interfaces:**
- Consumes: Task 2 `nextSessionGate`, `sessionReady`, `launchSignedIn`; Task 3
  `useInstallWipe`.
- `useSessionGate()` returns `{ gate: SessionGate; signedIn: boolean }` (same shape).
  Builds `GateInput` from `authClient.useSession()` (`failure`: `error === null` → `"none"`,
  `error.status === 401` → `"unauthorized"`, else `"other"`), `useLaunchCookie()`,
  `gaveUp = useElapsedOver(isPending, SESSION_WAIT_LIMIT_MS)`.
  `waiting = gate !== "ready" && !sessionReady(input)`.
  SigningIn timing: `showSigningIn = useSpinnerVisible(useElapsedOver(waiting, SPLASH_LIMIT_MS - spinnerTiming.showAfterMs))`
  so it mounts no earlier than `SPLASH_LIMIT_MS` and keeps the 500 ms minimum.
  `useSplashHold(!waiting)`. `signedIn = launchSignedIn(input)`.

- [ ] **Step 1:** implement the five changes.
- [ ] **Step 2:** `pnpm typecheck && pnpm lint && pnpm test` → clean.
- [ ] **Step 3: Device smoke (iOS dev build)**: signed in on wifi → no SigningIn frame;
  signed out → Welcome; airplane mode signed in → Home.
- [ ] **Step 4: Checkpoint.**

### Task 5: SigningIn layout matches the splash

**Files:** Modify `mobile/src/features/auth/components/signingIn.tsx`

- [ ] **Step 1:** wordmark container `absolute inset-x-0 top-1/2 -mt-8 h-16 items-center justify-center`
  (same classes as `launchIntro.tsx:39`); orb and text containers move from `top-21/50` to
  `top-1/2`, keeping their `mt-*` offsets below the wordmark.
- [ ] **Step 2:** typecheck + lint. Device: sign in with Google (overlay) and a slow launch
  (Network Link Conditioner "Very Bad Network", cookie present, cache expired) → wordmark
  does not move between splash and SigningIn, light and dark.
- [ ] **Step 3: Checkpoint.**

### Task 6: Launch intro latches in render

**Files:**
- Create: `mobile/src/features/welcome/introPhase.ts` (pure)
- Modify: `mobile/src/features/welcome/hooks/useLaunchIntro.ts`
- Test: `mobile/src/features/welcome/introPhase.test.ts`

**Interfaces:**
- Produces: `type IntroPhase = "pending" | "playing" | "done"`;
  `latchIntro(phase: IntroPhase, ready: boolean, signedIn: boolean): IntroPhase` — only
  `"pending"` + `ready` moves (`signedIn` → `"playing"`, else `"done"`); everything else
  returns `phase`.
- `useLaunchIntro(ready, signedIn)` keeps its return shape; holds the phase in a `useRef`,
  applies `latchIntro` during render, `finish` sets `"done"` and forces a render
  (`useReducer` counter). No `useEffect`.

- [ ] **Step 1: Failing tests**: `first ready frame latches playing when signed in`;
  `first ready frame latches done when signed out`; `not ready stays pending`;
  `later sign-in does not replay the intro` (`latchIntro("done", true, true)` → `"done"`).
- [ ] **Step 2–4:** FAIL → implement → PASS.
- [ ] **Step 5: Checkpoint.**

### Task 7: Sign-out on 401 with loop guard, and the welcome fade

**Files:**
- Create: `mobile/src/features/auth/signOutGuard.ts` (pure),
  `mobile/src/features/auth/client/signOutOnce.ts` (the app instance:
  `createSignOutGuard({ readCookie: authClient.getCookie, signOut: () => authClient.signOut() })`),
  `mobile/src/features/auth/hooks/useSignOutOnUnauthorized.ts`
- Modify: `sessionRoot.tsx` (call the hook), `mobile/src/app/_layout.tsx` or
  `sessionRoot.tsx` Stack: `welcome` options `animation: "fade"`
- Test: `mobile/src/features/auth/signOutGuard.test.ts`

**Interfaces:**
- Produces: `createSignOutGuard(deps: { readCookie: () => Promise<string>; signOut: () => Promise<unknown> }): () => Promise<void>`
  — reads the cookie; `""` → skip; equal to the last cookie it signed out → skip; else
  remember it and call `signOut`. A `readCookie` throw → skip (never sign out on a failed
  read).
- `useSignOutOnUnauthorized()`: when `useSession().error?.status === 401`, call
  `signOutOnce()` (effect keyed on the error object).

- [ ] **Step 1: Failing tests**: `signs out once per cookie`; `empty cookie skips`;
  `unreadable cookie skips`; `401 then signOut then refetch does not loop` (second call with
  the same cookie → signOut count 1; after cookie becomes `""` → still 1).
- [ ] **Step 2–4:** FAIL → implement → PASS.
- [ ] **Step 5:** Device: revoke the session in Neon (`delete from session where id = …`),
  foreground the app → Home fades to Welcome; relaunch → Welcome directly (cache cleared).
  Also `curl -i $API/api/auth/get-session -H "cookie: <revoked>"` and note status + body
  in the Task 9 report.
- [ ] **Step 6: Checkpoint.**

### Task 8: Prepare pairings once per user, before Home

**Files:**
- Create: `mobile/src/features/pairing/sync/preparePairings.ts`
- Modify: `mobile/src/features/pairing/hooks/usePairedMacs.ts`
- Test: covered by Task 1 `oncePerKey` tests (this file imports SecureStore, so no direct test).

**Interfaces:**
- Consumes: Task 1 `oncePerKey`; existing `claimPairingsFor`, `flushPendingUnlinks`,
  `endRevokedMacs`.
- Produces: `preparePairingsFor: (userId: string) => Promise<void>` =
  `oncePerKey(async (userId) => { await claimPairingsFor(userId); await linkSyncWithin(LINK_SYNC_CAP_MS) })`,
  `LINK_SYNC_CAP_MS = 1500`; `linkSyncWithin` races `flushPendingUnlinks().then(endRevokedMacs)`
  (errors swallowed — failures fall through, `useLinkSync` retries on foreground) against a
  timer.
- `usePairedMacs.reload` becomes `preparePairingsFor(userId).then(settleStoredEndedMacs).then(readPairedMacs).then(setMacs)`.
- Storage reads that throw: `readStoredMacs` already returns `[]` on parse failure; wrap the
  `getItemAsync` call too so a throw reads as "no data" (spec Part 2).

- [ ] **Step 1:** implement; wrap `getItemAsync` in `readStoredMacs` and `readPendingUnlinks`.
- [ ] **Step 2:** typecheck + lint + test.
- [ ] **Step 3: Measure**: temporary dev log (not committed) of ms from gate ready to first
  `setMacs`, 5 cold launches each on wifi, airplane, Very Bad Network. If every run is
  under 900 ms (`INTRO_MS`), no hold. Otherwise add to `LaunchIntro` a
  `holdFade: boolean` prop (fade starts when it turns false, capped at
  `LINK_SYNC_CAP_MS`), fed from a `macsReady` flag the Home screen reports through the
  existing `LiveMacLinksProvider` context. Record the numbers in the Task 9 report.
- [ ] **Step 4: Checkpoint.**

### Task 9: Phase 1 close-out — STOP

- [ ] **Step 1:** `pnpm typecheck && pnpm lint && pnpm test` clean.
- [ ] **Step 2:** device matrix for Parts 1–2 (spec), iOS + Android, light + dark:
  signed in fast wifi / airplane / Very Bad Network; signed out fresh install;
  delete-and-reinstall iOS; cached session expired with cookie valid; revoked session;
  expired cookie with cache left over; restart before unlock (Android: adb reboot, open
  before unlocking is impossible — note it).
- [ ] **Step 3:** `engineering/flow.md` launch-gate chain (`_layout.tsx` → `useInstallWipe`
  → `SessionRoot` → `useSessionGate` → `useLaunchCookie` / `authClient.useSession`
  [SecureStore boundary, network boundary `/api/auth/get-session`]).
  `engineering/decisions.md` (newest first): launch trusts cached session + stored cookie;
  `ahoy.install` / `installSeen` must never be renamed.
- [ ] **Step 4:** report: what changed, files (all uncommitted), revoked-session curl result,
  macs timing numbers. **STOP. Wait for the user's device check.**

---

## Phase 2 — after the user's go-ahead (spec Parts 3–5)

### Task 10: Session-recovering requests and renew on use

**Files:**
- Create: `mobile/src/features/auth/sessionRecovery.ts` (pure),
  `mobile/src/features/auth/client/authedRequest.ts`
- Modify: `mobile/src/features/talk/client/transcribeApi.ts`,
  `mobile/src/features/voice/client/speakApi.ts`,
  `mobile/src/features/voice/client/voicesApi.ts`,
  `mobile/src/features/pairing/client/pairingApi.ts`,
  `mobile/src/features/talk/hooks/useTalk.ts` (drop `signOutIfSessionGone`; recovery owns it)
- Test: `mobile/src/features/auth/sessionRecovery.test.ts`

**Interfaces:**
- Produces in `sessionRecovery.ts`:
  `type SessionCheck = "valid" | "gone" | "failed"`;
  `hasSessionCache(cookieHeader: string): boolean` — true when any cookie name ends in
  `session_data`;
  `createSessionRecovery(deps: { readCookie: () => Promise<string>; checkSession: () => Promise<SessionCheck>; signOutOnce: () => Promise<void>; renewSession: () => void })`
  → `{ send<R>(attempt: { build: (cookie: string) => Promise<R>; unauthorized: (response: R) => boolean }): Promise<R> }`.
  `send`: read cookie; if `!hasSessionCache(cookie)` call single-flight `renewSession`
  (non-blocking); `build(cookie)`; not unauthorized → return; else single-flight
  `checkSession`: `"valid"` → `build(await readCookie())` once and return it whatever it
  says; `"gone"` → `signOutOnce()`, return the first response; `"failed"` → return the
  first response.
- `authedRequest.ts` (app instance): `checkSession` maps `authClient.getSession()`
  (`data` → `valid`; `error.status === 401` or `data === null && error === null` → `gone`;
  else `failed`); `renewSession` = `authClient.$store.notify("$sessionSignal")`;
  `signOutOnce` from Task 7. Exports
  `authedRequest<T>(options: () => Omit<RequestOptions<T>, "fetch" | "headers">): Promise<RequestResult<T>>`
  — `build` calls `options()` fresh (new `FormData` each attempt), adds
  `headers: { cookie }` and `fetch` with `credentials: "omit"`; `unauthorized` =
  `error.kind === "api" && error.status === 401`. And
  `authedFetch(url: string, init: () => RequestInit): Promise<Response>` for speak;
  `unauthorized` = `response.status === 401`.
  `RequestOptions` is not exported by `@getahoy/lib`: derive with
  `Parameters<typeof request>[0]`.
- Clients: delete every `fetchWithOwnCookie` / `OWN_COOKIE_ONLY` and inline cookie header;
  transcribe passes `() => ({ …, form: buildForm(uri) })`.

- [ ] **Step 1: Failing tests**: `passes through a success`;
  `concurrent 401s share one session check`; `expired cookie with valid server session
  retries once and succeeds`; `retry calls build again` (build count 2, distinct objects);
  `revoked session signs out and returns the 401`; `offline check never signs out`;
  `second 401 after retry is returned, not retried again`;
  `missing session_data renews once for concurrent calls without delaying the request`;
  `present session_data does not renew`.
- [ ] **Step 2–4:** FAIL → implement pure + app instance + clients → PASS; typecheck, lint.
- [ ] **Step 5:** device: talk with a revoked session mid-request → existing error line +
  fade to Welcome, no crash; talk after 6 min idle → one `/get-session` renewal.
- [ ] **Step 6: Checkpoint.**

### Task 11: Library guard tests

**Files:** Test: `mobile/src/features/auth/betterAuthPin.test.ts`,
`mobile/src/features/auth/sessionAtom.test.ts`

- [ ] **Step 1:** `betterAuthPin.test.ts`: reads `mobile/package.json` via `node:fs`;
  asserts `dependencies["better-auth"] === "1.7.11"` and
  `dependencies["@better-auth/expo"] === "1.7.11"`.
- [ ] **Step 2:** `sessionAtom.test.ts`: `node:http` server on port 0 serving
  `/api/auth/get-session` from a mutable response mode; `createAuthClient` from
  `better-auth/client` with that `baseURL`; subscribe to `useSession`; tests
  `200 session then 401 clears data`, `200 session then 500 keeps data`,
  `200 session then a destroyed socket keeps data` (refetch via
  `$store.notify("$sessionSignal")`, await `isPending === false`).
- [ ] **Step 3:** `pnpm test` → PASS. If `better-auth/client` cannot load in node, stop and
  report; do not add a runner dependency.
- [ ] **Step 4: Checkpoint.** (Both files are build-time checks; deleted in Task 15 with the
  rest.)

### Task 12: Server session lifetime and fresh pairing reads (web)

**Files:**
- Modify: `web/src/features/auth/server/auth.ts` — `session: { expiresIn: SESSION_LIFETIME_SECONDS, updateAge: SESSION_ROLL_SECONDS, cookieCache: … }`,
  `SESSION_LIFETIME_SECONDS = 60 * 24 * 60 * 60`, `SESSION_ROLL_SECONDS = 24 * 60 * 60`
- Modify: `web/src/features/auth/server/readSession.ts` — add
  `readFreshSession(headers: Headers)` = `auth.api.getSession({ headers, query: { disableCookieCache: true } })`
  (no cache cookie fallback needed: the cache is skipped)
- Modify: `web/src/app/api/pairing/{claim,complete,unlink}/route.ts` — `readFreshSession`

- [ ] **Step 1:** implement. Audit (done during planning, re-check on the diff): every
  `unauthorized` in `web/src/app/api/**` is the first branch after the session read, before
  any write; nothing else returns 401.
- [ ] **Step 2:** `pnpm typecheck && pnpm lint` in `web`.
- [ ] **Step 3:** `next build && next start`; time 20 `/get-session` calls with a valid
  cookie (cache hit and miss); log `/get-session` hits for 10 min with the phone in use →
  ≤ 1 renewal per 300 s window plus focus/reconnect.
- [ ] **Step 4: Checkpoint.**

### Task 13: User switch resets the navigator

**Files:**
- Create: `mobile/src/features/auth/sessionChange.ts` (pure),
  `mobile/src/features/auth/hooks/useNavigatorKey.ts`
- Modify: `sessionRoot.tsx` — `<Stack key={navigatorKey} …>`
- Test: `mobile/src/features/auth/sessionChange.test.ts`

**Interfaces:**
- Produces: `sessionChange(previousUserId: string | null, nextUserId: string | null): "same" | "signedIn" | "signedOut" | "switched"`;
  `navigatorKey(lastUserId: string | null, nextUserId: string | null): string | null` —
  returns the last non-null user id (`next ?? last`).
- `useNavigatorKey(userId)`: ref holding the last non-null id; returns it (`"anonymous"`
  while none has been seen).

- [ ] **Step 1: Failing tests**: four `sessionChange` cases; `null transitions keep the
  key`; `A to B changes the key`.
- [ ] **Step 2–4:** FAIL → implement → PASS.
- [ ] **Step 5: Checkpoint.**

### Task 14: Versioned stored Macs and one write queue

**Files:**
- Create: `mobile/src/features/pairing/storage/storedMacsPayload.ts` (pure: schemas +
  parse), `mobile/src/lib/serialQueue.ts`
- Modify: `mobile/src/features/pairing/storage/pairedMacs.ts`,
  `mobile/src/features/pairing/sync/endedMacs.ts`
- Test: `mobile/src/features/pairing/storage/storedMacsPayload.test.ts`,
  `mobile/src/lib/serialQueue.test.ts`

**Interfaces:**
- Produces: `STORED_MACS_SCHEMA_VERSION = 2`;
  `parseStoredMacs(raw: unknown): { macs: StoredMac[]; dropped: number[] }` — bare array →
  v1 entries; `{ schemaVersion: 2, macs }` → v2; anything else → `{ macs: [], dropped: [] }`;
  each entry `safeParse`d alone, failing indexes in `dropped`.
  `serializeStoredMacs(macs: PairedMac[]): string`. `pairedMacSchema`, `storedMacSchema`,
  `PairedMac`, `StoredMac` move here (pure), re-exported types used by `pairedMacs.ts`.
- Produces: `createSerialQueue(): <T>(task: () => Promise<T>) => Promise<T>` — tasks run one
  at a time in call order; a rejected task does not block the next.
- `pairedMacs.ts`: one module queue `const writeMacs = createSerialQueue()`;
  `savePairedMac`, `updatePairedMac`, `removePairedMac`, `clearPairedMacs` run their whole
  read-modify-write inside it; new `changeStoredMacs(change: (macs: StoredMac[]) => StoredMac[] | null): Promise<void>`
  (null → no write) used by `endPairedMac` and `settleStoredEndedMacs`. `writePairedMacs`
  stops being exported. Dropped entries: `if (__DEV__) logger.info("pairing.storage", \`dropped ${count} stored mac entries\`)`
  (indexes only).

- [ ] **Step 1: Failing tests**: `corrupt entry is dropped and the rest kept`;
  `v1 bare array migrates`; `v2 payload parses`; `unknown shape reads empty`;
  `serialized payload carries schemaVersion 2`; `serialQueue runs tasks in order`;
  `two concurrent updates both land` (two read-modify-write tasks on a shared array
  through the queue → both changes present); `serialQueue continues after a rejection`.
- [ ] **Step 2–4:** FAIL → implement → PASS; typecheck, lint.
- [ ] **Step 5: Checkpoint.**

### Task 15: Phase 2 close-out, review, PRs

- [ ] **Step 1:** typecheck, lint, test clean in `mobile` and `web`.
- [ ] **Step 2:** remaining device matrix rows (spec): foreground after 10 min revalidates;
  reconnect after offline revalidates; in-flight talk with revoked session; user switch
  A → B shows B's empty Macs.
- [ ] **Step 3:** living docs: `flow.md` authed request chain (client → `authedRequest` →
  `request()` [network] → route → `readSession` / `readFreshSession` [Neon]);
  `decisions.md`: authed single-flight recovery + renew on use; stored Macs schemaVersion;
  60-day rolling sessions.
- [ ] **Step 4:** three reviewers in parallel on the uncommitted diff (`git diff main` plus
  untracked files) in each repo, since nothing is committed yet; fix what they flag.
- [ ] **Step 5:** run the full test suite one last time, then delete every `*.test.ts` this
  plan added plus `mobile/src/features/auth/sessionGate.test.ts`; re-run typecheck + lint.
- [ ] **Step 6:** ask the user, showing staged paths and message; one commit per repo:
  mobile `fix: open the app from the cached session and keep it in sync`, web
  `fix: roll sessions for 60 days and read pairing sessions fresh`, each with short bullets.
- [ ] **Step 7:** push both branches, `gh pr create` each; report + 3–5 multiple-choice
  questions per CLAUDE.md. STOP.
