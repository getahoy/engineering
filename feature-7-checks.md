# Feature 7 checks (smart commands, shortcuts, tabs, drafts)

Run on a real phone and Mac. Before starting: `ANTHROPIC_API_KEY` in `web/.env.local`,
restart the web dev server after any `shared` change, reload the phone app, `pnpm start` in
agent. `pnpm eval:interpret` in web should pass every case first.

| # | Do this | Expect |
|---|---|---|
| 1 | Stop the web server, then say "open a new tab in Chrome" (bug 1) | "Smart commands aren't working right now"; the Mac log says why, never the words |
| 2 | Chrome in the background, say "open a new tab in Chrome" (bug 2) | Chrome comes to the front, then the new tab opens in Chrome |
| 3 | "Minimize this window" | The front window minimizes; "Pressed ⌘M" |
| 4 | Notion open, "search in Notion" | Notion's search opens |
| 5 | "Full screen" | The front window goes full screen; "Pressed ⌃⌘F" |
| 6 | "Force quit" | Nothing happens; the parser's message |
| 7 | "Quit everything with command Q" | Nothing happens |
| 8 | Two Chrome tabs, one Grok: "close the Grok tab" | Only that tab closes; "Closed the Grok tab" |
| 9 | Two Grok tabs: "close the Grok tab" | Nothing closes; "2 tabs match: …. Say which one." |
| 10 | "Close the YouTube tab" with none open | "No tab matching youtube in Google Chrome" |
| 11 | Arc in front, "go to my Gmail tab" | Arc switches to the Gmail tab |
| 12 | Safari closed, "close the Grok tab in Safari" | "Safari isn't open"; Safari does not launch |
| 13 | First tab command, deny the Automation prompt | "Allow Ahoy to control Google Chrome in System Settings → Privacy & Security → Automation." |
| 14 | "Close tab" | The current tab closes; no app named Tab… quits |
| 15 | "Draft a mail to jake bauer at gmail dot com about the hackathon credits" | Mail opens a draft to jakebauer@gmail.com with a subject and body; nothing is sent; "Drafted an email to jakebauer@gmail.com. Check it and press Send." |
| 16 | "Write a Gmail to rahul at example dot com saying I'll be late" | Gmail compose opens in the browser, filled in, not sent |
| 17 | "Send an email to x at y dot com saying hi" | A draft opens; it is not sent and no Return is pressed |
| 18 | "Email this to my boss" | Nothing opens; the parser's message |
| 19 | Chrome is the default browser: "open gmail in Arc", then "search cats on Safari" (bug 9) | Gmail opens in Arc; the search opens in Safari; "Opened mail.google.com in Arc" |
| 20 | "Open YouTube in Brave" with Brave not installed | Nothing opens; no other browser is used instead |
| 21 | Say something like "go over to the Notion tab" (bug 8) | It switches to the tab or does nothing; it never closes it |
| 22 | console.groq.com open in Arc, Chrome in front: "close the groq tab" (bug 10) | The Groq tab closes in Arc, even if speech heard "grok" |
| 23 | TextEdit with an unsaved document, ⌘S open: "click Save" | Save is clicked; phone shows "Clicked Save" |
| 24 | Two buttons with the same name on screen (e.g. two Reply buttons in a Gmail thread): "click Reply" | Numbered badges on both; the phone sheet shows 1 and 2; picking one clicks only that one |
| 25 | Finder with a file selected, "click Delete" or "click Move to Trash" | Nothing is clicked yet; the phone asks "… can't be undone. Confirm on your phone." with Yes and Cancel; Yes clicks it once |
| 26 | "Scroll down", "page up", "go to the top", "go to the bottom" in a long web page | Page Down, Page Up, ⌘↑, ⌘↓ |
| 27 | "Click Xyzzy" where nothing looks like it | No badges; "I couldn't find Xyzzy." |
| 36 | A live page (a ticking clock or a feed) with a Save button: "click Save" (bug 12) | Save is clicked, never "The screen changed" |
| 37 | A button titled OK whose tooltip or description says delete (bug 13) | The phone asks to confirm before clicking |
| 38 | A long list with Save visible at the top and another Save scrolled out of view: "click Save" (bug 14) | The visible Save is clicked, no numbers |
| 39 | A page where the word Send appears as text next to a Send button: "click Send" (bug 15) | The button is clicked, no numbers (Send still asks to confirm) |
| 40 | "Click the close button" or "click minimize button" on a window (bug 16) | The window's close or minimize button; choice labels read "Close button", never blank |
| 35 | A dialog with Save and Save As: say "click sav" (or a near miss speech gives) | At most 9 badges, closest first: 1 on Save, 2 on Save As; "Can't find Sav. Pick a number on the screen." |
| 28 | Badges showing, then switch to another app or window | Badges disappear at once; the phone sheet closes; a late tap answers "The screen changed, say it again." |
| 29 | Badges showing, wait 20 s | Badges disappear and the sheet closes |
| 30 | Ask for a click, then move the window before the click lands (within 100 ms is hard; drag during "Sending…") | "The screen changed, say it again."; nothing is clicked |
| 31 | 1Password (or Keychain Access) in front: "click Unlock" or "press command c" | Refused: "Ahoy doesn't click or type in this app." |
| 32 | Say a command that fails 5 times in a row (e.g. "click Xyzzy" then cancel) | The next command answers "Ahoy paused…" with a Resume button; tapping it brings Ahoy back |
| 33 | "Hit the blue continue button" on a page with a Continue button | The AI turns it into a click; Continue is clicked |
| 34 | Badges on a second monitor and on a Retina display | Each badge sits on the top-left corner of its element |
