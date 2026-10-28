# Feature 6 checks (talk)

Run on a real phone and Mac. Before starting: `ELEVENLABS_API_KEY` in `web/.env.local`,
restart the web dev server, rebuild the phone app, `pnpm start` in agent.

| # | Do this | Expect |
|---|---|---|
| 1 | Hold, say "open Chrome" | Phone: Heard "open chrome" → "Opened Google Chrome"; Chrome opens |
| 2 | "Close Terminal" with Terminal open | Terminal quits; "Closed Terminal" |
| 3 | "Search cats" | The browser opens a Google search for cats |
| 4 | "Pause" with music playing | Cut from this build: "I can't do that yet" |
| 5 | Tap and release instantly | "Hold to talk", nothing sent |
| 6 | Hold in silence for 3 s | "Didn't catch that" |
| 7 | "Open Xyzzy" | "Can't find an app called Xyzzy" |
| 8 | "Make me a sandwich" | "I can't do that yet" |
| 9 | "Open Microsoft" with Word, Excel and PowerPoint installed | Asks "Which one: Microsoft Excel, Microsoft PowerPoint or Microsoft Word?", does nothing |
| 10 | Lock the Mac, then talk | Button disabled, "is locked" |
| 11 | Wi-Fi off after "Understanding…", then back on, Try again | Runs once |
| 12 | Airplane mode, then talk | "Speech needs the internet" |
| 13 | Get a call while holding | Discarded, button resets |
| 14 | Hindi-English: "Chrome open karo" | Opens Chrome (note the result if not) |
| 15 | 10 commands quickly in a row | Each runs once, in order |
| 16 | Type in TextEdit without saving, then "close TextEdit" | After 3 s the phone shows "Check your Mac"; then "Asked TextEdit to quit. It may be waiting for you to save." The save dialog stays on the Mac |
| 17 | First "close Terminal", deny the Automation prompt | "Allow Ahoy to control Terminal in System Settings → Privacy & Security → Automation." |

| 18 | Hold, drag your finger off the button while talking, let go off the button | Recording keeps going while held; letting go anywhere sends it |
| 19 | VoiceOver on: focus the talk button, double-tap and hold, speak, let go | Records while held and sends on release |
| 20 | Reduce Motion on (Settings → Accessibility → Motion), then hold | No pulsing rings or spin; a steady glow while listening |
| 21 | Use the phone for more than 5 minutes, then talk (bug 3) | The command runs; web log shows `POST /api/transcribe 200`, never 500 |
| 22 | Stop the web server's database access briefly, then talk (bug 4) | "Speech didn't work this time"; you stay signed in |
| 23 | Sign out, sign in as a different account on the same phone (bug 5) | No Macs listed; the old Mac cannot be controlled |
| 24 | Reload the app with a Mac paired and awake (bug 6) | The talk button is active as soon as the card says Connected |
| 25 | "Hide VS Code" with VS Code open (bug 7) | VS Code hides; "Hid Visual Studio Code" |
| 27 | Reload the phone app (bug 11) | Under the title: "Dev build <commit>, loaded <time>", the time of this reload |
| 26 | Watch the card while the Mac connects (after Disconnect → Connect, or a Mac restart) | The card turns Connected and the talk button is active in the same instant; a locked Mac shows "is locked" on both |

"Check your Mac" comes from the phone: 3 s after `command.received` with no result yet.
