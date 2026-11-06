# History checks

Run on a real phone and Mac against the dev server and dev database. Before starting:
`pnpm db:migrate` in web, restart the web dev server after any `shared` change, a new
phone dev build (expo-clipboard is a native module), `pnpm start` in agent.

| # | Part | Do this | Expect |
|---|---|---|---|
| 1 | 1 | Without `CRON_SECRET` in `web/.env.local`: start the dev server, `curl -i localhost:3000/api/history/retention` | Server starts; 503. Then set `CRON_SECRET` (`openssl rand -hex 32`), restart: no header 401, `Authorization: Bearer <CRON_SECRET>` 200; the log says only `purged N entries` |
| 2 | 2 | Say "open Pages", "click save" on a screen with two Saves, then "click flibber" | Mac log shows only ids and statuses; each entry later shows "In Pages", Built-in rules, and the right Why on the failure |
| 3 | 3 | Say "open Notes"; then turn on airplane mode and say "open Notes" again | First entry Done; second shows "Not synced yet" and the offline banner; after reconnect it syncs and the line goes |
| 4 | 3 | Say "click save" with two Saves, then force quit the app before answering; reopen | The entry is Cancelled with "What happened" |
| 5 | 3 | Sign out, sign in as another account | History page is empty, no rows from the first account |
| 6 | 4 | Settings → History | List by day, summary "N in the last 7 days", search, All · Done · Failed; Failed shows failed and blocked only |
| 7 | 4 | Open an entry with a follow-up | Ahoy asked, the chosen number in Signal, "You said · after Ns", time bar with the wait noted |
| 8 | 4 | Delete an entry, tap Undo within 3 s; delete another and wait | First comes back; second stays gone after a refresh |
| 9 | 4 | Clear history…, confirm | Only the button is Flare; then "History cleared." |
| 10 | 4 | Turn Keep history off, then open History | Everything gone; "History is off." with a switch; new commands do not appear |
| 11 | 4 | Two Macs paired: tap the Mac chip, pick one | List and summary show only that Mac |
| 12 | 4 | Light and dark, largest Dynamic Type, Reduce Motion, VoiceOver on a row | Readable, fades only, each row reads heard, status, what Ahoy did and time |
| 13 | 5 | Mac window → History | Only this Mac's entries; list left, entry right; Copy shows "Copied"; offline shows "Couldn't refresh" over the last page |
| 14 | 5 | Mac Settings → Keep history off, then on; Clear history… | Same as the phone; the phone's list follows |
| 15 | 6 | Phone entry "open Pages" → Run again on {Mac} | Home opens on that Mac and runs it; a new entry appears; hidden on a blocked entry; off with the Mac disconnected |
| 16 | 6 | Run again on an entry that said "quit" or "draft an email" | Asks "Run this again?" first, on both apps |
| 17 | 6 | Report a problem with a note, on both apps | The sheet lists exactly heard, did, status, app, reason, timings; "Thanks. We'll look at this one."; a `problem_reports` row with that snapshot |
| 18 | 7 | Fresh Mac onboarding welcome step; phone Settings under Keep history | The 90-day history line and "Speech audio is never stored." |
