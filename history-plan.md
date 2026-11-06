# History plan

Full command history for the phone and the Mac. Written 2026-11-07. Build in the parts at
the end, one commit set per part, in order. No extra fluff: build what is listed, nothing
decorative beyond it.

Design reference: `design/Ahoy History Canvas.dc.html` (direction 1a "A diary") and the
History section of `design/handoff/phone-states.md`. Where the design and this file differ,
this file wins.

## Decisions

- History belongs to the **account**, stored on the server. Same login on a new phone or
  after a reinstall shows the same history.
- **The phone writes it.** Every command starts on the phone, which is online (it needs the
  server for speech-to-text) and signed in. Writes never block a command.
- **Home is unchanged.** Cold start shows the carousel; after a command, Home shows only the
  latest conversation. Older history lives only on the History page.
- **Entry points:** iPhone: a "History" row in Settings. Mac: a "History" sidebar item
  under Home. The Mac sidebar keeps "Phone" (one phone per Mac).
- **Retention:** the user chooses "Keep history for: 3 months / 18 months / Until I delete
  it", default 18 months; older entries are deleted automatically. "Keep history" switch
  (default on).
- **Remove phone, re-pair, Mac sign-out:** history is kept (it's the account's). Only Clear
  history, Keep history off, retention and account deletion remove it.
- Never stored: audio, screen contents, window titles beyond the result line, email bodies.

## Data model (web, Postgres)

```
command_history
  id            uuid primary key      -- the threadId; every write is an idempotent upsert
  user_id       text  → user.id       on delete cascade
  phone_id      uuid  → devices.id    on delete set null
  desktop_id    uuid  → devices.id    on delete set null
  mac_name      text  not null        -- snapshot at write time
  heard         text  not null        -- first thing said, ≤ MAX_COMMAND_TEXT_CHARS
  did           text                  -- final result message, ≤ MAX_RESULT_MESSAGE_CHARS
  status        text  not null        -- running | done | choice | failed | blocked | cancelled
  app           text                  -- app the command acted on, e.g. "Pages"
  via           text                  -- rules | ai
  reason        text                  -- closed failure code (see below), failed/blocked/cancelled only
  timings       jsonb                 -- { hearingMs, understandingMs, onMacMs, waitedMs }
  follow_ups    jsonb                 -- [{ question, options[{number,label}], chosen, said, waitedMs }], max 5
  started_at    timestamptz not null
  finished_at   timestamptz
  deleted_at    timestamptz           -- soft delete for the 3 s Undo; purged after 1 minute
index (user_id, started_at desc) where deleted_at is null
index (desktop_id, started_at desc) where deleted_at is null

history_settings
  user_id       text primary key → user.id on delete cascade
  keep_history  boolean not null default true
  retention     text not null default '18m'   -- 3m | 18m | forever

problem_reports
  id uuid primary key, user_id → user.id cascade, history_id uuid, snapshot jsonb,
  note text (≤ 500), created_at
```

Status mapping from the Mac's `command.result`: `done` → done (or **choice** when the thread
had a follow-up); `needsChoice` → running until answered; `refused` → blocked; `failed` →
failed; choice expired, dismissed, or the app died before a result → cancelled.

Failure reasons (closed set, mapped to plain-word "Why" copy on the phone and Mac):
`macUnreachable`, `noAnswer`, `nothingHeard`, `transcribeFailed`, `unknownApp`,
`nothingSelected`, `targetNotFound`, `blockedApp`, `screenChanged`, `offline`, `other`.

## Protocol (shared/protocol: additive and optional; ask before changing shared)

- `command.result` gains optional `app` (≤ 60), `via` (`rules` | `ai`), `onMacMs` (int) and
  `reason` (the closed set above). Old phones ignore them; old Macs omit them.
- History API schemas: `historyEntrySchema`, `historyPageSchema` (entries + cursor),
  `historyWriteSchema` (batch of upserts, ≤ 20), `problemReportSchema`.

## API (web)

All behind `readSession` (phone cookie, Mac bearer), per-user rate limits, nothing logged
beyond ids and counts.

- `POST /api/history` batch upsert (≤ 20 entries, idempotent by id). Ignored when
  keep_history is off. Rejects device ids that aren't the caller's.
- `GET /api/history?cursor=&status=all|done|failed&q=&desktopId=` newest first, 30 per page.
  `q` searches heard, did and app (case-insensitive). The summary line comes back with
  page one: `{ thisWeek, didntGoThrough }`.
- `DELETE /api/history/:id` soft delete (Undo window), `POST /api/history/:id/restore`.
- `DELETE /api/history` clear all.
- `GET/PUT /api/history/settings` keep_history and retention; turning keep_history off
  clears everything.
- `POST /api/history/:id/report` stores a problem report (snapshot of the entry + note).
- Retention: a daily job deletes, per user, rows older than their retention ("forever"
  never deletes) and purges soft deletes older than 1 minute.

## Flows

**Phone writes (outbox).**
On send: upsert `running` with heard, Mac, started_at and hearing time.
On result: upsert the final status, did, app, via, reason, timings and follow-ups.
On a follow-up answer: update the same thread row.
If the app dies or the choice expires: finalise as `cancelled` on the next launch.
Writes go into a small persisted outbox (MMKV) that flushes in the background, retrying on
foreground and reconnect. Unsent entries show "Not synced yet".

**Phone reads.** The History screen shows the cached last 50 entries instantly (MMKV), then
fetches page one and pages on scroll. Account switch wipes the cache and outbox (same path as
`claimPairingsFor`).

**Timings.** hearingMs and understandingMs come from the phone's stage clock; onMacMs from
the Mac's result; waitedMs (time the user took to answer) is stored but excluded from the
total, as the design notes.

**Mac reads.** The History page calls `GET /api/history?desktopId=<this Mac>` with its
bearer token. Offline: shows the last loaded page with a quiet "couldn't refresh".

**Run again.** Re-sends the stored `heard` text to that Mac as a normal `command` frame (no
speech). Hidden on blocked entries; disabled when the Mac isn't connected; confirm first when
the original action was close, quit or an email draft. On the Mac page it runs locally.

**Report a problem.** A sheet shows exactly what will be sent (the entry: heard, did, status,
app, reason, timings) plus an optional note; nothing else is attached.

## UI (both apps, design 1a, no extras)

- List grouped by day; row = heard (Fraunces italic, 3-line clamp) + time, then one status
  mark + did. Marks: done check (Kelp), asked-first mark, failed circle-x (Flare), blocked
  ban (Driftwood), cancelled dash (Driftwood), running breathing dot.
- Top: one summary line ("38 this week · 2 didn't go through"), search, All · Done ·
  Failed. No chart. The summary hides while searching or filtering.
- iPhone with 2+ Macs: Mac picker chip and sheet. The Mac page has no picker.
- Detail: eyebrow (day · time · Mac), heard, follow-ups, status + did + "In <app> · 1.6s",
  Why card (failed/blocked/cancelled), Where the time went (bar + legend), Understood by
  ("Smart (AI)" / "Built-in rules"), actions: Copy, Run again (secondary), Report a
  problem, Delete.
- Delete: no confirm, Undo toast for 3 s. Clear history: the calm confirm card (button in
  Flare only), then "History cleared."
- States: running, offline/not synced, empty, couldn't load + Try again, no search
  results, nothing failed, cleared, copied (Mac toast).
- Settings (iPhone and Mac): "History" row/page link, "Keep history" switch, "Clear
  history…".
- Light and dark, reduced motion (fades only), Dynamic Type on iPhone, VoiceOver labels on
  every row and mark.

## Privacy

Add one line to onboarding's cloud disclosure and the privacy policy: "Ahoy keeps a history
of what you said and what it did in your account (18 months unless you change it). You can
clear it or turn it off in Settings." Speech audio is never stored.

## Edge cases

| Case | Behaviour |
|---|---|
| Force quit / reinstall | History page loads from the server after sign-in; Home shows the carousel |
| New phone, same login | Full history from the server |
| Offline when commanding | Entry shows "Not synced yet", flushes later |
| App dies mid-command | Finalised as cancelled on next launch |
| Keep history off | Clears everything; the outbox stops writing; the page shows a short "History is off" with a switch |
| Account switch on the phone | Cache and outbox wiped before the new user's data loads |
| Mac reinstalled (new device id) | Mac page starts empty; phone still shows old rows under mac_name |
| Delete account | Cascade removes history, settings and reports |
| Very long heard text | 3-line clamp in the list, full in the detail |

## Parts

1. **Server + protocol.** Tables + migration, API routes, retention job, protocol schemas
   (shared: show the diff first). Tests for the upsert, filters, search, soft delete/restore,
   clear, keep_history off and retention.
2. **Mac sends the extras.** `command.result` carries app, via, onMacMs and reason from the
   command runner. Tests per outcome type.
3. **Phone writes.** Outbox, status mapping, timings, cancelled-on-relaunch, cache, account-
   switch wipe. Tests for the mapping, the outbox (retry, idempotent, offline), relaunch
   finalising.
4. **Phone History screen.** List, search, filters, Mac picker, detail, delete + Undo,
   clear, states, Settings row + Keep history switch.
5. **Mac History page.** Sidebar item, two-pane list and detail, states, Settings switch and
   clear.
6. **Run again + Report a problem** on both apps.
7. **Disclosure copy** in onboarding and the privacy policy text.

Each part: typecheck, lint and tests pass; light and dark device check; a row in the
relevant checks file; decisions.md entries for the account-history and retention decisions.
