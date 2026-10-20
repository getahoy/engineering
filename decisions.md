# Decisions

Append-only. Newest first. About four lines each: what, why, what was rejected, how to
reverse. A later decision that reverses one adds a new entry linking back; history is never
edited.

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

## 2026-10-20 — Living docs live in `engineering/`, a private repo; `CLAUDE.md` symlinked at root

`CLAUDE.md`, `decisions.md` and `flow.md` sit outside `docs/` so they can become their own
private repo. The workspace folder is not a repo, so a root file would be unversioned; Claude
Code walks up the tree, so the root symlink loads `engineering/CLAUDE.md` from every sibling
repo.
Rejected: keeping them in `getahoy/docs` (that spec repo may not stay private).
Reverse: move the files back and repoint the symlink.
