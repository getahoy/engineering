# Decisions

Append-only. Newest first. About four lines each: what, why, what was rejected, how to
reverse. A later decision that reverses one adds a new entry linking back; history is never
edited.

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
