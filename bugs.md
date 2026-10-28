# Bugs

Report a bug as "bug: I said X, expected Y, got Z". Each one gets a row here, and its fix adds
a row to the matching checks file (`feature-6-checks.md`, `feature-7-checks.md`) so it cannot
come back unnoticed.

Status is `open`, or `fixed in <repo> <commit>`.

| # | Date | What I did | Expected | What happened | Status |
|---|---|---|---|---|---|
| 1 | 2026-10-28 | Said "open a new tab in Chrome" | A new tab in Chrome | "Can't find an app called a new tab in chrome": the AI call failed (system messages in `messages`) and the parser message was shown | fixed in web d6e2b1a, agent 97d3a80; check 7.1 |
| 2 | 2026-10-28 | Said "open a new tab in Chrome" after fix 1 | New tab in Chrome | New tab pressed in whatever app was in front | fixed in web d6e2b1a, agent 97d3a80; check 7.2 |
| 3 | 2026-10-28 | Held the talk button and spoke | The command runs | "Speech didn't work this time": iOS added its own cookie jar to our cookie header, the server could not read it and answered 500 | fixed in mobile ec82071, web 8bc8189; check 6.21 |
| 4 | 2026-10-28 | Used the phone normally | Stay signed in | Signed out on its own after one unauthorized answer | fixed in mobile c97a740; check 6.22 |
| 5 | 2026-10-28 | Signed in again after the automatic sign-out | Only my own Macs | The pairing survived sign-out, so any account signing in on the phone would control the Mac | fixed in mobile 2579b97; check 6.23 |
| 6 | 2026-10-28 | Reloaded the app | Talk button active when the card says Connected | Card said Connected, button said "Connect to your Mac to talk" | fixed in mobile 90fd959; check 6.24 |
| 7 | 2026-10-28 | Said "hide VS Code" (found in review) | VS Code hides | Failed: its process is named Code, not Visual Studio Code | fixed in agent 1ca0114; check 6.25 |
| 9 | 2026-10-28 | Said "open gmail in arc" | Gmail opens in Arc | Opened in Chrome: sites, searches and Gmail drafts used the default browser | fixed in agent, web, shared (browser field); check 7.19 |
| 8 | 2026-10-28 | Held-out eval: "switch to the notion tab" sent straight to the AI | switchTab notion | closeTab notion. On the phone the Mac's parser catches this phrase first, so only other wordings reach the AI | fixed in agent 294ce91 (a tab closes only when close was said); check 7.21 |
| 10 | 2026-10-28 | Said "close the groq tab" with console.groq.com open | That tab closes | "No tab matching": speech heard a sound-alike, and only the front or default browser was searched | fixed in agent 294ce91; check 7.22 |
| 12 | 2026-10-29 | Code review: "click Save" on a live page | Save is clicked | "The screen changed": the two snapshots were compared by total element count, which live pages change constantly | fixed in agent (app, window and target only); check 7.36 |
| 13 | 2026-10-29 | Code review: click a button titled OK whose tooltip says Delete permanently | Ask to confirm | Clicked at once: only the first label was checked for destructive words | fixed in agent (every text field checked); check 7.37 |
| 14 | 2026-10-29 | Code review: "click Save" with a second Save scrolled out of view in a list | Click the visible one | Both counted, so it asked to pick a number: scroll areas were not checked, only the window | fixed in agent and helper (nearest scroll area clip); check 7.38 |
| 15 | 2026-10-29 | Code review: "click Send" where a text label and the Send button both match | Click the button | Asked to pick between them: a plain label counted as much as a button | fixed in agent (pressable wins within a tier); check 7.39 |
| 16 | 2026-10-29 | Code review: "click the close button" on a window | The window's close button | Not found, and unnamed buttons showed as blank choices: window buttons only have a subrole | fixed in agent (friendly subrole names); check 7.40 |
| 11 | 2026-10-28 | Reloaded the phone app | Know which JavaScript is running | No way to tell an old bundle from a new one | fixed in mobile 4e7613f (dev build stamp); check 6.27 |
