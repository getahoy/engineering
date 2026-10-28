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
| 15 | "Draft a mail to samar abbas at gmail dot com about the hackathon credits" | Mail opens a draft to samarabbas@gmail.com with a subject and body; nothing is sent; "Drafted an email to samarabbas@gmail.com. Check it and press Send." |
| 16 | "Write a Gmail to rahul at example dot com saying I'll be late" | Gmail compose opens in the browser, filled in, not sent |
| 17 | "Send an email to x at y dot com saying hi" | A draft opens; it is not sent and no Return is pressed |
| 18 | "Email this to my boss" | Nothing opens; the parser's message |
| 19 | Chrome is the default browser: "open gmail in Arc", then "search cats on Safari" (bug 9) | Gmail opens in Arc; the search opens in Safari; "Opened mail.google.com in Arc" |
| 20 | "Open YouTube in Brave" with Brave not installed | Nothing opens; no other browser is used instead |
| 21 | Say something like "go over to the Notion tab" (bug 8) | It switches to the tab or does nothing; it never closes it |
| 22 | console.groq.com open in Arc, Chrome in front: "close the groq tab" (bug 10) | The Groq tab closes in Arc, even if speech heard "grok" |
