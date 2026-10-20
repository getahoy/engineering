# Decisions

Append-only. Newest first. About four lines each: what, why, what was rejected, how to
reverse. A later decision that reverses one adds a new entry linking back; history is never
edited.

## 2026-10-20 — Mobile component kit is React Native Reusables, copied in

The deciding reason is copy-in: the component source lands in our repo, so it can be made
to obey CLAUDE.md (file size, comments, exports, tokens). An installed kit is a black box
that can never meet those rules. Every CLI-added component gets a cleanup pass in the
same commit. Uniwind variant, to match the styling decision below.
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
