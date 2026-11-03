# Launch, session and local-data consistency

One branch per repo (`mobile`, `web`). Parts 1 + 2 are committed first, then the build
stops for the user's device check of the launch before Parts 3–5. Server is the source of truth for
identity and revocation; the phone owns device keys, addresses and UI flags. Cached data is
for display only: anything that acts (talk, unpair, pair) is verified by the server or the
Mac at that moment, and its error lands in the existing message slot.

Not touched: `shared/`. Edits with Edit/Write only. `better-auth` and `@better-auth/expo`
stay pinned at `1.7.11` in `mobile` and `web`.

## Why

Signed-in cold launch shows splash → SigningIn for a split second → Home. Causes:

1. `useSessionGate` waits for `isPending === false`, the `/get-session` round trip. The expo
   plugin hydrates the cached session from SecureStore inside the fetch's `init`
   (`hydrateSessionCache`), so `data` is set within ms while `isPending` stays `true`.
2. `useSpinnerVisible` flips the gate to `signingIn` at 300 ms, under the splash; the splash
   fade reveals it and the 500 ms minimum holds it.
3. SigningIn's wordmark sits at `top-21/50`; splash, LaunchIntro and Welcome use `top-1/2`.
4. `useLaunchIntro` picks `playing` in an effect, so the first ready frame is bare Home.

Separately: app API calls (`transcribeApi`, `speakApi`, `voicesApi`, `pairingApi`) use plain
fetch with `authClient.getCookie()`, so `Set-Cookie` on their responses is dropped. The
`session_data` cache cookie expires after 5 min and is never renewed while the app is open,
so talk routes fall back to a Neon lookup per command and a rolled expiry never reaches the
phone.

## Facts verified in 1.7.11 (no code needed)

- `@better-auth/expo/client` calls `setupExpoFocusManager()` and `setupExpoOnlineManager()`
  on import when `Platform.OS !== "web"`. `expo-network` is installed;
  `refetchOnWindowFocus` defaults to `true`; online-again refetches. Foreground and
  reconnect revalidation already work. Verify on device only.
- Session atom: a 401 sets `data: null`; a 5xx or thrown fetch keeps `latest.data`.
- Cache hydration skips a cached session whose `session.expiresAt` is past.
- On `/get-session` success the plugin writes the body to `ahoy_session_data` (so a `null`
  body clears it) and saves `Set-Cookie` (so a cleared token cookie drops out).
- `authClient.signOut()` clears `ahoy_cookie`, `ahoy_session_data` and the atom in the fetch
  `init`, before the network call, so it clears locally even offline.
- If the SecureStore read inside hydration throws, `$fetch` throws and the atom ends
  `isPending: false, data: null, error` — indistinguishable from signed out unless the gate
  checks `error`.
- Server: `cookieCache { enabled: true, maxAge: 300 }` is already on (`auth.ts`). Per-user
  rate limits already exist on transcribe (20/min), interpret (30/min), speak (60/min) via
  `consumeRateLimit`. Talk routes write nothing keyed to the user except the rate-limit row
  (text key, no FK).
- Android: the `expo-secure-store` config plugin defaults `configureAndroidBackup: true` and
  we have no custom backup rules, so SecureStore is excluded from Auto Backup. MMKV is not
  excluded; restored MMKV flags with an empty SecureStore are harmless.

## Part 1: Launch gate (mobile)

**Launch auth probe.** At launch read `authClient.getCookie()` once, wrapped:
`"present" | "absent" | "unreadable"` (`"pending"` until the read settles). A throw is
`"unreadable"`, never `"absent"`.

**Signed in during launch** = cached session exists AND cookie `"present"`. Cookie
`"absent"` → signed out, ignore any cached session. Once the server answers
(`isPending === false` after a real response), the atom alone decides.

**Gate is ready** when any of:

- cached session AND cookie `"present"`
- cookie `"absent"`
- `isPending === false` AND (`data !== null` OR no `error` OR `error.status === 401`)
- the existing 6 s give-up (`SESSION_WAIT_LIMIT_MS`)

`isPending === false` with `data === null` and a non-401 error (offline, 5xx, storage
throw) does NOT count: it waits for the give-up. Ready stays sticky (`nextSessionGate`);
update its comment for the new conditions.

**SigningIn** appears only for the real slow case (cookie present, no usable cache, server
slow). It never mounts before `SPLASH_LIMIT_MS` (3000, exported from `useSplashHold`), so it
is never hidden under the splash; once shown it keeps the 500 ms minimum. Its wordmark moves
to `top-1/2 -mt-8 h-16` like the splash, with orb and text below it. The sign-in flow's
overlay (`useSigningInOverlay`) uses the same component and gets the same layout.

**useLaunchIntro** latches `playing`/`done` synchronously in render the moment `ready`
flips (no effect), so the first ready frame is already covered when signed in. Pure latch
function, tested.

**Signed in:** splash → LaunchIntro (same wordmark position) fades and lifts into Home /
NoMacsYet. Measure when `usePairedMacs` first resolves relative to the 900 ms intro; if it
can lose the race, LaunchIntro holds its fade until `macs !== null`, capped so it never
hangs.

**Signed out:** splash fades straight into Welcome; Welcome's first wordmark frame is
pixel-aligned with the splash in light and dark.

## Part 2: Launch hardening (mobile)

**Reinstall.** MMKV instance `ahoy.install`, flag `installSeen`. Order is wipe → launch
probe → mount. If the flag is missing: write `"{}"` to `ahoy_cookie` and
`ahoy_session_data` through `storageAdapter(SecureStore)` from `@better-auth/expo/client`
(the library's own wrapper, which clears chunk keys `key.i` and `key.slot.i`). Set the flag
in `finally`, even if the wipe throws, so a failing wipe cannot sign the user out on every
launch. The root does not mount `useSession` until this settles, so the splash stays up
meanwhile (first launch only, a few ms). The `ahoy.install` instance and `installSeen` key
are never renamed (a rename re-wipes every existing install; logged in `decisions.md`).
Pairing keys
(`ahoy.device.keys`, `ahoy.paired.macs`, `ahoy.pairing.owner`, `ahoy.pending.unlinks`) are
kept. Android uninstall wipes everything; the code assumes neither platform's behaviour.

**After sign-in** (fresh sign-in, not a cached launch): `claimPairingsFor(userId)` →
`flushPendingUnlinks` → `endRevokedMacs`, before Home renders, so a different user gets a
wipe and revoked Macs are settled first. `claimPairingsFor` becomes single-flight (this sync
and `usePairedMacs` can race). The sign-in overlay stays up for this, with a cap;
network failures fall through (Home shows the claimed local list, link sync retries on
foreground as today).

**Storage read throws** (Android restored backup, device not yet unlocked): pairings read as
"no data"; auth reads as `"unreadable"` → wait for `/get-session` and the give-up. Never a
crash, never treated as signed out.

**401 from `/get-session`** → `authClient.signOut()` (clears cached `session_data` and the
cookie locally, best-effort server call), so the next launch does not show Home. Verify what
the server returns for a revoked session (expected `200 null`, already self-clearing).

**Sign-out loop guard.** `signOut()` triggered by `authedFetch` or by the `/get-session` 401
path runs at most once per cookie value; if `getCookie()` is already empty it is skipped.
Pure guard, tested: 401 → signOut → refetch does not sign out again.

**Background sign-out** (the check returns null/401 after Home showed): Stack.Protected
moves to `/welcome`; `welcome` gets `animation: "fade"` so the move fades (the initial
route never animates, so launch is unchanged).

## Part 3: Session refresh (mobile)

**Renew on use.** No timer (a 240 s tick misses the cache: valid at 240, expired from 300
to 480). When `authedFetch` sees that `getCookie()` has no `session_data` entry (expired
entries are already filtered out), it starts one non-blocking, single-flight session
refetch via `authClient.$store.notify("$sessionSignal")` and sends the request without
waiting. The refetch goes through the plugin, which saves the renewed cookies and
`session_data`. Focus/reconnect refetch stays as is, for revocation.

**authedFetch** (`mobile/src/features/auth/client/authedFetch.ts`, a `typeof fetch`):

- `credentials: "omit"` plus our `cookie` header from `authClient.getCookie()`, replacing
  the four copied `fetchWithOwnCookie` helpers and inline cookie headers.
- On a 401: one single-flight `authClient.getSession()` (concurrent 401s share one call),
  then retry the request exactly once with the fresh cookie. A transcribe upload is retried
  at most once.
- Retry is safe only because routes return 401 solely from the session check, before any
  side effect; every other auth failure is 403. Audit claim, complete, unlink, transcribe,
  speak and voices for this and fix any route that breaks it.
- A retry rebuilds the request (callers pass a body factory, not a body); a consumed
  `FormData` is never reused.
- Refetch confirms no session → `authClient.signOut()` (fade to `/welcome`), the caller gets
  its 401 result and shows the existing message. Refetch fails on network/5xx → return the
  original 401 result, never sign out.
- Passed as `fetch` to `request()` (transcribe, voices, pairing); used directly by speak.
- The orchestration is a pure function taking `send`, `refetchSession`, `signOut`; tested.

## Part 4: Server (web)

- `auth.ts`: `session.expiresIn` 60 days, `session.updateAge` 1 day (rolling). Keep
  `cookieCache` maxAge 300.
- `readFreshSession(headers)` next to `readSession`, passing
  `query: { disableCookieCache: true }`, used by unlink, claim and complete (the Mac
  pairing handshake). Talk routes keep the cache; a revoked or deleted user can pass them
  for up to 5 minutes (accepted).
- Talk routes with a deleted user: confirm `readSession` → 401 path after cache expiry and
  no FK write anywhere in the route. No code expected.
- Measure `/get-session` time against `next build && next start`.
- Log `/get-session` calls for 10 minutes with the app open and in use: at most one renewal
  per 300 s cache window plus focus/reconnect, never a loop.

## Part 5: Local-vs-server consistency (mobile)

**Session differences** — pure `sessionChange(previousUserId, nextUserId)` →
`"same" | "signedIn" | "signedOut" | "switched"`:

- `signedOut` (null/401): fade to `/welcome`, cached session cleared (Part 2).
- `switched`: hard reset — root `Stack` keyed by the last non-null user id, so it changes
  only on a real A → B switch (remount drops screen state). Null transitions keep the
  navigator, so the welcome fade and the sign-in transition still animate. And
  `claimPairingsFor(newId)` runs before any read of that user's Macs (already the order in
  `usePairedMacs`).
- `same` with changed fields: applied silently by the atom, no navigation.

**readStoredMacs.** Stored payload becomes `{ schemaVersion: 2, macs: [...] }`. A bare array
(no version) is v1 and migrates. Each entry parses on its own; valid ones are kept, dropped
ones are logged in dev only (count and index, never contents). Never `[]` because one entry
is bad. Pure `parseStoredMacs(raw: unknown)` in a file with no SecureStore import, tested.

**Write queue.** Every read-modify-write of `ahoy.paired.macs` (save, update, remove,
settle-ended, clear) runs through one promise queue owned by the storage module. Pure
`createSerialQueue()`, tested with two concurrent updates.

## Tests (node:test, pure files only)

- `sessionGate.test.ts`: each ready condition; non-401 error waits; give-up; sticky ready;
  cookie absent ignores cache; unreadable never ready early.
- launch probe classifier: throw → `unreadable`, empty → `absent`.
- intro latch: first ready frame latches, later `signedIn` changes do not re-latch.
- authedFetch orchestration: concurrent 401s → one refetch; expired cookie with valid server
  session → silent retry success; revoked → signOut; offline/5xx refetch → no signOut;
  retry happens once only; retry calls the body factory again.
- renew on use: no `session_data` → one refetch for concurrent calls, request not delayed;
  `session_data` present → no refetch.
- sign-out guard: once per cookie; empty cookie skips; 401 → signOut → refetch no loop.
- reinstall: flag set even when the wipe throws.
- `claimPairingsFor` single-flight: two concurrent calls run one claim.
- `sessionChange`: same, switched, signedOut, signedIn.
- `parseStoredMacs`: corrupt entry kept-others, v1 array migrates, unparseable JSON → `[]`.
- serial queue: two concurrent updates both land.
- `betterAuthPin.test.ts`: `better-auth` and `@better-auth/expo` are exactly `1.7.11`.
- `better-auth/client` session atom against a local `node:http` server: 200 session then 401
  → `data` null; 500 → kept; socket destroyed → kept.

The "cache loads before the network fetch" ordering cannot run in node
(`@better-auth/expo/client` imports react-native); the pin test guards it and the device
matrix checks it.

## Device matrix (iOS + Android, light and dark)

Signed in: fast wifi, airplane mode, slow 3G. Signed out fresh install. Delete-and-reinstall
on iOS. Cached session expired with cookie valid. Revoked session on the server. Server
returning 401. Expired cookie with cached session left over. Device restarted, not yet
unlocked. Foreground after 10 min (revalidates). Reconnect after offline (revalidates).
In-flight talk request when the session is revoked (clean error, fade to Welcome).

## Living docs

`flow.md`: launch gate chain and authedFetch recovery. `decisions.md`: launch trusts the
cached session; authedFetch single-flight recovery and renew-on-use; macs payload
schemaVersion; 60-day rolling sessions; `ahoy.install` / `installSeen` must never be renamed.

## Builder calls (marked as mine, not agreed)

- Reinstall wipe via the library's `storageAdapter` writing `"{}"`, not hand-rolled chunk
  deletes.
- 401 cache clear via `authClient.signOut()`.
- `welcome` animation changes to `fade`.
- Post-sign-in sync rides the existing sign-in overlay with a cap.
- Tests pass plain functions into the pure orchestrators; no mocking library.
