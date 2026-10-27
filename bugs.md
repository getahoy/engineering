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
| 8 | 2026-10-28 | Held-out eval: "switch to the notion tab" sent straight to the AI | switchTab notion | closeTab notion. On the phone the Mac's parser catches this phrase first, so only other wordings reach the AI | open |
