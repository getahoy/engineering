# Listen-off refuses, it doesn't go deaf — plan

**Goal:** with "Listen for my phone" off, a paired phone learns "Ahoy is turned off" (live, on
launch, after an app restart) and every Mac surface says off with a Turn on button.

**Architecture:** the LAN listener always runs. The handshake decides `refused` for a paired
key while off; the session waits for the phone's `ready`, sends `goodbye turned_off`, closes.
A shell `ListenSwitch` owns turn on/off, the off expiry check and the throttled notification.
The phone reports `connected` only on the Mac's first `status` frame (refusal never flashes
Connected) and keeps the last goodbye per Mac in MMKV.

**Spec:** the bug report in the session (items 1–9, tests, device checks a–e).

## Global constraints

- No `shared/` change: `turned_off` is already in `goodbyeReasonSchema`.
- Copy: "Ahoy is off. Your iPhone can't reach this Mac.", button "Turn on", durations
  "For 1 hour" / "Until tomorrow" / "Until I turn it back on", notification
  "Your iPhone tried to connect. Ahoy is off." with action "Turn on", tooltip "Ahoy is off".
- Notification at most once per 10 minutes.
- Tests are `node:test` in `agent/src/core` and pure mobile files only.
- Commit on `main` in agent and mobile, ask before each commit, no push.

## Calls made (not in the spec)

- "Until tomorrow" = the next 08:00 local after now (at 02:00 that is 08:00 the same day).
- Expiry checked every 30 s (survives sleep without a resume hook) and once at startup.
- Off duration choice is inline buttons under the switch (popover header, Settings row,
  sidebar footer); a popup menu would be clipped by the popover window.
- Bridge `setLanEnabled(boolean)` becomes `turnOn()` and `turnOff(duration)`.
- Notification click opens the window (fallback when macOS hides the action button).
- Phone "paused" is already persisted (`PairedMac.paused`); the Mac never sends it. The
  remembered goodbyes are `turned_off`, `quitting`, `sleeping`.

## Review focus

1. Phone connects normally when on: `connected` must still arrive (status frame) — device check.
2. Agent closes before `status` (crash, version race): phone must not sit on "Connecting".
3. Turn off while a phone is connected still sends `turned_off` live.
4. Pairing while off: QR still carries LAN addresses; completion turns listening on.
5. Expiry passed while the Mac was asleep or quit: first check turns it on.

---

### Task 1 (agent, core): refusal decision, off expiry, throttle, pairing turns on

**Files:** modify `src/core/connection/acceptHandshake.ts`; create
`src/core/connection/acceptHandshake.test.ts`, `src/core/appState/listenOff.ts`,
`src/core/appState/listenOff.test.ts`.

- `acceptHandshake(input & { listening: boolean })` adds outcome
  `{ outcome: "refused"; reply; channel; phone }` for a paired key after the version check.
  Unknown key keeps `closing` with `not_paired` `unknown`.
- `listenOff.ts`: `type OffDuration = "hour" | "tomorrow" | "untilOn"`,
  `offUntil(duration, now: Date): number | null`, `offExpired(until: number | null, now: number): boolean`,
  `REFUSED_NOTICE_GAP_MS = 600_000`, `shouldNoticeRefusal(lastAt: number | null, now: number): boolean`,
  `turnsOnListening(status: PairingStatus): boolean`.
- Tests: paired key + `listening: false` → `refused`, reply opens to `welcome`; unknown key →
  reply opens to `not_paired`/`unknown`; `listening: true` → `accepted`; hour → +3 600 000;
  tomorrow at 22:00 → next day 08:00, at 02:00 → same day 08:00; untilOn → null, never expires;
  throttle: null → true, 9 min → false, 10 min → true; `paired` → true, `offering` → false.
- Run `pnpm test`, commit `fix: refuse a paired phone while listening is off` (with Task 2).

### Task 2 (agent, shell): listener always on, ListenSwitch, notification, tooltip

**Files:** `src/core/connection/phoneSession.ts` (phase `awaitingReady.refusing`, on `ready`
→ `end("turned_off")` + `onRefused(phone)`), `src/shell/lan/phoneLink.ts` (`isListening`,
`onRefused`), `src/shell/settingsStore.ts` (`lanOffUntil: z.int().nullable().default(null)`),
`src/shell/agentController.ts` (`turnListeningOn()`, `turnListeningOff(until)`, `lanOffUntil()`),
create `src/shell/lan/listenSwitch.ts` (`turnOn`, `turnOff(duration)`, `start/stop` expiry
interval, `phoneRefused()` throttled `Notification` with action), `pairingServices.ts`
(paired → turn on), `main.ts` (always `phoneLink.start()`), `shellSurfaces.ts`, `bridgeIpc.ts`,
`agentBridge.ts`, `preload.ts`, `agentTray.ts` (tooltip "Ahoy is off" when off).

### Task 3 (agent, renderer): off everywhere with Turn on

`useListenSwitch` hook + `ListenSwitch` + `OffDurationChoices` components; `OffNotice`
(copy + primary Turn on). Home: off → `OffNotice` instead of the status card;
`PairingDone` while off swaps "Now say something" for `OffNotice`. Phone page and popover
show `OffNotice`. Sidebar footer = status line + switch. Settings + popover header use
`ListenSwitch`.

### Task 4 (mobile): remembered goodbye, connected on status, copy

**Files:** create `connection/goodbyeMemory.ts` (+ `.test.ts`): `RememberedGoodbye`,
`nextGoodbyeMemory(memory, event)`, `rememberedState(memory)`, `stateAfterMiss(memory, unreachable)`;
`storage/macGoodbyes.ts` (MMKV `ahoy.macGoodbyes`, Zod-parsed); `macConnection.ts` (seed from
storage, show it instead of connecting, connected + clear on first `status`, closed before
status → unreachable/remembered); `macLinkState.ts` uses the pure map; `macCopy.notFound`
adds "and turned on".

### Task 5: docs

`bugs.md` rows 17–19, `decisions.md` entry "Off refuses, it doesn't go deaf", `flow.md` if the
LAN chain changes. Left uncommitted.
