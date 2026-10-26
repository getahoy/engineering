# Pairing and connection states

What both screens show in every state, and what the other device sees when something
changes. Written after reading the code on 25 Oct 2026 (uncommitted 5a work included).
Build from this, one numbered step at a time (bottom of the file).

---

## Where it stands

The plumbing is good: Noise IK encryption, key-based identity, device codes, proof check.
The gaps are all in **what happens when a pairing changes**. Nothing tells the other
device *why* the connection ended, so every ending looks the same: a closed socket. The phone
guesses, and sometimes guesses "this Mac no longer trusts you" when it was only a Wi-Fi
blip. The Mac window has one screen, the QR, so it shows a pairing code even while a phone
is paired and connected.

---

## Three decisions (recommended defaults, change them here if you disagree)

**1. A Mac has one phone. A phone can have several Macs.**
Pairing a new phone to a Mac replaces the old one, with a clear warning. Fewer states, fewer
bugs, and it matches how people think about it ("my phone controls my Mac"). A phone with a
home iMac and a work MacBook is a real case, so the phone keeps a list of Macs.
This changes the corner-case row "Several phones, one Mac: allow it".

**2. Three different ways to stop, with different meanings.**

| Action | Where | What it means | Coming back |
|---|---|---|---|
| **Pause** | Phone: "Disconnect". Mac: "Ahoy On" off | Stop connecting for now. Pairing kept | Tap Connect / turn Ahoy on. No QR |
| **Remove** | Either device | End the pairing on both devices and on the server | Scan a new QR |
| **Sign out** | Either device | Remove every pairing on this device, then sign out | Sign in, scan a new QR |

Phone sign-out removes its Macs so the next account signed in on that phone can't inherit
them. It also deletes the phone's device keys, so each account gets a fresh identity.

**3. The server decides whether a pairing still exists. The LAN message is the fast path.**
When one side removes the pairing it sends a `goodbye` over the live connection (instant)
*and* tells the server (reliable). The other side checks the server on start, on foreground
and every 15 minutes, so it finds out even if it was offline at the time.

---

## What's broken today (checked against the code)

| # | Problem | Where |
|---|---|---|
| 1 | The Mac window always shows a QR after onboarding, even with a phone paired and connected. Opening the window starts a new pairing round | `appState.ts` `windowScreen` returns `"pair"` for every finished state; `usePairingSession` starts pairing on mount |
| 2 | After a later pairing succeeds, the button says "Pair another phone" and pressing it starts another QR straight away | `renderer/screens/pairScreen.tsx` `finish` / `finishLabel` |
| 3 | The Mac window never says whether the phone is connected. Only the tray does | `pairedPhoneList.tsx` has no status |
| 4 | Connected phones are tracked by display name. Two phones called "iPhone" are one entry | `agentController.ts` `setConnectedPhones` |
| 5 | Remove on the Mac just closes the socket. The phone isn't told why | `unpairPhone.ts` → `phoneLink.disconnect` → `stream.close()`; `frameSchema` only has `ready` |
| 6 | A phone the Mac doesn't know is dropped with no reply, so it can't tell "removed" from "network problem" | `acceptHandshake.ts` returns `DROPPED` |
| 7 | **The phone shows "This Mac no longer trusts this phone" on any handshake timeout or socket close.** A Wi-Fi blip mid-handshake looks like being removed | `macHandshake.ts`: the 5 s timer and `onClose` both resolve `REJECTED` |
| 8 | Remove on the phone never tells the Mac. The Mac keeps listing the phone and **keeps trusting its key** | `useUnpairMac.ts` calls the server and local storage only |
| 9 | Failed server unlinks are dropped. The Mac logs a warning, the phone swallows the error. Nothing retries, so the server link stays active | `unpairPhone.ts`, `useUnpairMac.ts` `.catch(() => undefined)` |
| 10 | `/api/pairing/unlink` answers `unlinked: true` even when it changed nothing (devices not found, or owned by another account) | `unlinkDevices.ts` returns early; route always sends `SuccessResponse` |
| 11 | Pairing a phone from a second account overwrites the Mac's session token and desktop id. Unlinking the first phone then runs under the wrong account and silently does nothing (see 10) | `identityStore.ts` `completePairing` |
| 12 | Phone sign-out keeps the paired Macs and device keys. The next account on the phone inherits them | `useSignOut.ts` only calls `authClient.signOut()` |
| 13 | Turning "Ahoy On" off closes sockets without a reason. The phone says "Not found on this Wi-Fi" instead of "Ahoy is off on your Mac" | `main.ts` `setLanEnabled` → `phoneLink.stop()` |
| 14 | The Mac knows the screen is locked, but the phone is never told | `screenLocked` lives only in agent state |
| 15 | "Pair again" on the phone opens the scanner with no explanation and leaves the dead Mac in storage | `pairedMacItem.tsx` |
| 16 | Phone home: "Pair another Mac" is always under the cards, a red Remove is always visible on every card, account details sit on the same screen | `pairedMacsSection.tsx`, `macLinkStatus.tsx`, `app/index.tsx` |

#7 is the most urgent. It shows a frightening message in a normal situation.

---

## Words used everywhere

- **Paired**: both devices hold each other's key and the server link is active.
- **Connected**: paired, and an encrypted socket is open right now.
- **Ready**: paired, not connected, nothing wrong. The normal state when the phone is in a
  pocket. Never show it as an error.
- **Paused**: the user stopped connections on purpose.
- **Ended**: the pairing is over, and we know why (removed, replaced, signed out).

---

## The Mac window

The pair screen splits into a **Home** screen and a **Pair** screen. The window opens on
Home. The QR only appears when there is no phone, or after the user chose to pair a new one.

### Home, by state

| State | Shows | Actions |
|---|---|---|
| No phone | The QR, with "Open Ahoy on your phone, sign in, and scan this code." | — |
| Connected | Phone name, account email, green dot, **"Connected"** | Turn off Ahoy · Remove phone · Pair a new phone |
| Ready | Grey dot, **"Not connected. Open Ahoy on {phone} to connect."** Below it: "Last connected 2 minutes ago" | Same as above |
| Paused by phone | **"{phone} paused the connection."** | Same as above |
| Ahoy off | **"Ahoy is off. Your phone can't connect."** | Turn on Ahoy |
| Ended by phone | Banner: **"{phone} removed this Mac."** / **"{phone} signed out of Ahoy."** Then the QR | Dismiss banner |

Footer on every Home state: **"Signed in as {email} · Sign out"**, plus the helper status line.

### Actions

- **Remove phone**: native dialog. *"Remove {phone}? It will stop working with this Mac right
  away. To use it again, scan a new code."* [Remove] [Cancel]. Sends `goodbye: removed`,
  queues the server unlink, returns to the QR.
- **Pair a new phone**: opens the Pair screen with: *"Scan this code with the new phone.
  {phone} keeps working until the new phone scans it, then it's removed from this Mac."*
  [Cancel] returns to Home with nothing changed. On success, the old phone gets
  `goodbye: replaced` and its server link is revoked.
- **Sign out**: *"Sign out of Ahoy on this Mac? {phone} will be removed and you'll need to scan
  a new code to use Ahoy again."* Sends `goodbye: signed_out`, unlinks, revokes the Mac's
  session, clears it from `identity.bin`, shows the QR.
- **After a successful pairing**: "Paired with {phone}" with one button, **Done**, which goes
  to Home. Never a "pair another" button there.

### Tray lines

| State | Line |
|---|---|
| Connected | Connected to {phone} |
| Ready | {phone} is not connected |
| Paused by phone | {phone} paused the connection |
| Off | Ahoy is off |
| Locked | Screen locked, Ahoy is paused |
| No phone | No phone paired |

### macOS notifications (once per event, never repeated)

- "{phone} removed this Mac from Ahoy."
- "{phone} signed out of Ahoy on this Mac."

---

## The phone

### Home

- **Header**: "Ahoy" on the left, a gear on the right that opens **Account** (name, email,
  Sign out).
- One **card per Mac**. "Add another Mac" lives at the bottom of the list as a small text
  button, or in the gear screen. Not a big button under connected cards.
- No Macs: one centred block. *"Pair your Mac. On your Mac, click the Ahoy icon in the menubar
  and choose Open Ahoy. Then scan the code it shows."* [Scan code]

### Card, by state

Each card: Mac name, a status line, **at most one** primary button, and a `⋯` menu with
**Disconnect** (or **Connect** when paused) and **Remove this Mac**.

| State | Status line | Primary button |
|---|---|---|
| Connecting | Connecting… | — |
| Connected | Connected · encrypted | — |
| Mac locked | {mac} is locked. Unlock it to use Ahoy. | — |
| Paused by you | Paused. Ahoy won't connect until you tap Connect. | Connect |
| Ahoy off on Mac | Ahoy is turned off on {mac}. Turn it on from the menubar. | — (keeps retrying) |
| Ahoy quit on Mac | Ahoy isn't running on {mac}. Open it on the Mac. | — (keeps retrying) |
| Mac asleep | {mac} is asleep. | — (keeps retrying) |
| On mobile data | You're on mobile data. Join the same Wi-Fi as {mac}. | — |
| Can't reach (unknown cause) | Can't reach {mac}. Check it's awake, Ahoy is running, and you're on the same Wi-Fi. | Try again |
| Mac too old / too new | (existing copy) | — |
| Mac doesn't recognise phone | {mac} doesn't recognise this phone. Ahoy may have been reinstalled there. | Pair again |

### Ended cards (stay until the user acts, survive app restarts)

| Reason | Message | Buttons |
|---|---|---|
| Mac removed this phone | **{mac} removed this phone.** | Pair again · Dismiss |
| Mac paired another phone | **{mac} is now paired with a different phone.** | Pair again · Dismiss |
| Mac signed out | **You signed out of Ahoy on {mac}.** | Pair again · Dismiss |

**Pair again** opens the scanner with this hint above the viewfinder: *"On {mac}, open Ahoy.
If it shows another phone, choose Pair a new phone."* Dismiss deletes the stored Mac.

### Phone actions

- **Disconnect** (`⋯`): no dialog, reversible. Sends `goodbye: paused`, saves `paused` on
  the stored Mac, stops auto-retry.
- **Remove this Mac**: *"Remove {mac}? Ahoy on {mac} will stop accepting this phone. To use it
  again, scan a new code on the Mac."* [Remove] [Cancel]. Sends `goodbye: removed`, queues
  the server unlink, deletes the stored Mac.
- **Sign out**: *"Sign out? This phone will also be removed from {mac list}. You'll need to scan
  a code to pair again."* Sends `goodbye: signed_out` to connected Macs, queues the unlinks,
  clears stored Macs and device keys, signs out.
- **After a successful scan**: a check mark and "Paired with {mac}" for about 1 second, then
  Home. Not a silent close.

---

## When one side changes, what the other sees

| Event | Mac shows | Phone shows |
|---|---|---|
| Phone connects | Connected | Connected · encrypted |
| Phone goes to background | Ready (not an error) | — |
| Phone taps Disconnect | {phone} paused the connection | Paused card |
| Phone removes the Mac, online | Banner + notification, QR | Card gone |
| Phone removes the Mac, Mac offline | Same, at the next server check or connection attempt | Card gone |
| Mac removes the phone | QR | Ended card: removed |
| Mac pairs a new phone | New phone's details | Old phone: ended card, replaced |
| Mac signs out | QR | Ended card: signed out |
| Phone signs out | Banner: signed out, QR | Sign-in screen |
| Mac turns Ahoy off | Ahoy is off | Ahoy is turned off on {mac} |
| Mac app quits | — | Ahoy isn't running on {mac} |
| Mac sleeps | — | {mac} is asleep |
| Mac screen locks / unlocks | Tray: Screen locked | {mac} is locked / Connected |
| Wi-Fi drops mid-handshake | — | Can't reach {mac} (**never** "no longer trusts") |
| Ahoy reinstalled on the Mac | QR (fresh identity) | {mac} doesn't recognise this phone |
| Ahoy reinstalled on the phone | Old phone still listed. User picks Pair a new phone | No Macs, Pair your Mac |
| Both sides press Remove at once | QR | Card gone (both operations are idempotent) |

The last known reason (off, quit, asleep) stays on the phone card until the Mac is reachable
again, so a later failed retry doesn't overwrite it with "Can't reach".

---

## Changes needed underneath

### Protocol (`@getahoy/protocol`)

- `handshakeReplySchema` `refused.reason`: add `not_paired`, with
  `detail: "removed" | "replaced" | "signed_out" | "unknown"`. The Mac sends it inside the Noise
  reply, so the phone knows it came from the real Mac and can trust it.
- `frameSchema`, both directions: `goodbye` with
  `reason: "removed" | "replaced" | "signed_out" | "paused" | "turned_off" | "quitting" | "sleeping"`.
  Send it, then close.
- `frameSchema`, Mac to phone: `status { locked: boolean }`, sent on connect and on every change.

### Mac

- **Tombstones**: keep removed phone keys with their reason for 30 days in `identity.bin`.
  `acceptHandshake` answers a tombstoned key with an authenticated `refused: not_paired`, and any
  other unknown key with `detail: "unknown"`. Rate-limit handshakes per IP, since each costs a
  key exchange.
- When the first message can't be decrypted at all (the phone holds an old Mac key after a
  reinstall), send a plaintext `{ type: "unknown_mac_key" }` and close. It's unauthenticated,
  so the phone only shows the hint and never deletes anything because of it.
- Send `goodbye` before closing: Remove, replace, sign out, Ahoy off, `before-quit`,
  `powerMonitor` `suspend`.
- Key connected phones by public key, not name.
- **Outbox**: pending server calls (unlink, sign-out) saved in `identity.bin`, retried with
  backoff until they succeed or come back `not_found`.
- **Server check** on start, every 15 minutes and on network change: if the phone's link was
  revoked, handle it exactly like `goodbye: removed`. This call also keeps the 7-day session
  fresh.
- One phone: pairing a new phone revokes the old one only after the new pairing completes.

### Phone

- `macHandshake.ts`: a timeout or close before a reply means **unreachable**. Only an
  authenticated `refused` means removed.
- The stored Mac gets two optional fields: `paused: boolean` and
  `ended: { reason, at } | null`.
- Same outbox and server check as the Mac, run on app start and when the app comes to the foreground.
- Sign-out runs the full sign-out from decision 2.

### Server

- `unlinkDevices` returns whether it revoked anything. The route answers `not_found` when it
  didn't.
- `device_links`: add `revoked_by` (device id) and `revoke_reason`.
- New `GET /api/pairing/links` for the calling device: active links, plus links revoked in the
  last 30 days with their reason.
- `completePairing`: in the same transaction, revoke the Mac's other active link with reason
  `replaced`.
- Rate limit `/api/pairing/claim` (already a known gap).

---

## Build order

Each step is one small commit you can test on your own phone and Mac. Don't start the next
until the current one passes its checks below.

1. **Phone: stop the false "no longer trusts".** Timeout or close means unreachable. *(Fixes #7. Tiny.)*
2. **Protocol: `goodbye`, `status`, `refused: not_paired`.** Shared package only.
3. **Mac: tombstones, authenticated refusal, `goodbye` on Remove.** *(Fixes #5, #6.)*
4. **Phone: ended cards.** Handle `goodbye` and refusals, store `ended`, Pair again with a hint. *(Fixes #15.)*
5. **Phone: `⋯` menu, Disconnect (pause), Remove sends `goodbye`.** *(Fixes #8 on the LAN.)*
6. **Mac window: Home and Pair screens.** Status in the window, no QR when paired, Done goes Home,
   Pair a new phone flow, phones keyed by public key. *(Fixes #1–4.)*
7. **Goodbye for off, quit and sleep. Lock status frame.** *(Fixes #13, #14.)*
8. **Server: honest unlink, revoke reasons, links endpoint, replace on complete. Outbox and
   server check on both apps.** *(Fixes #8 offline, #9, #10, #11.)*
9. **Sign out on both sides.** *(Fixes #12.)*
10. **Phone home layout: header, gear, Account screen, Add another Mac moved.** *(Fixes #16.)*

---

## Checks to run on real devices

Run the ones for the step you just built, then all of them before calling feature 5 done.

| # | Do this | Expect |
|---|---|---|
| 1 | Pair, then open the Mac window | Home with the phone's status. No QR |
| 2 | Phone open, same Wi-Fi | Both show Connected within 2 s |
| 3 | Lock the phone | Mac shows Ready, not an error |
| 4 | Turn Wi-Fi off on the phone during "Connecting…" | "Can't reach". Never "no longer trusts" |
| 5 | Mac: Remove phone, with the phone open | Phone shows "removed this phone" within 1 s |
| 6 | Mac: Remove phone while the phone app is closed, then open it | Phone shows "removed this phone" |
| 7 | Phone: Remove Mac, with the Mac window open | Mac banner and notification, QR |
| 8 | Phone: Remove Mac with the Mac offline, then bring the Mac online | Mac finds out within 15 min or on restart |
| 9 | Phone: Disconnect, then Connect | Mac shows paused, then connected. No QR anywhere |
| 10 | Mac: Pair a new phone, then Cancel | Old phone still works |
| 11 | Mac: Pair a new phone, scan with a second phone | Old phone shows "paired with a different phone" |
| 12 | Mac: turn Ahoy off | Phone: "Ahoy is turned off" |
| 13 | Quit Ahoy on the Mac | Phone: "isn't running" |
| 14 | Close the MacBook lid | Phone: "is asleep" |
| 15 | Lock the Mac screen | Phone: "is locked". Unlock → Connected |
| 16 | Phone: Sign out, sign in with another account | No Macs listed |
| 17 | Mac: Sign out | Phone ended card; Mac shows the QR and no account |
| 18 | Press Remove on both devices within a second of each other | Both end cleanly, no error |
| 19 | Delete and reinstall the phone app, try to pair | Mac requires Pair a new phone; old entry replaced |
| 20 | Delete `identity.bin` on the Mac, open the phone | Phone: "doesn't recognise this phone" |
| 21 | Server down, Remove on either side, server back | Unlink goes through on its own (outbox) |
| 22 | Kill the app mid-pairing on each side | No half-paired state; scanning again works |

Add a row to `corner-cases` for anything new you hit.
