# Email address, browser and mail app checks

Run on a real phone and Mac after relaunching Ahoy (`pnpm start` in agent) and restarting the
web dev server. Use a made-up address of the same shape as a real one, never a real person's.

## Spoken addresses (plan A1)

| # | Say | Expected |
|---|---|---|
| 1 | "draft an email to rahul verma 1404 at gmail dot com about the demo" | To: rahulverma1404@gmail.com |
| 2 | "… rahul verma one four zero four at gmail dot com …" | Same |
| 3 | "… rahul verma fourteen oh four at gmail dot com …" | Same |
| 4 | "… r a h u l verma 1404 at the rate gmail dot com …" | Same |
| 5 | "… rahul dot verma at gmail dot com …" | To: rahul.verma@gmail.com |
| 6 | Name misheard (say it fast) | Address filled; reply says "Check the address, then press Send." |
| 7 | "email Samar about lunch" | No address (unchanged) |
| 8 | "draft an email to myself about groceries" | To: my account email |
| 9 | Form field focused: "type rahul verma 1404 at gmail dot com" | Types the address |
| 10 | Form field focused: "type my email" | Types my account email |
| 11 | "type hello at noon" | Types "hello at noon" |
| 12 | "email rahul verma 1404 at gmail dot com and priya at example dot com about lunch" | Both addresses in To |

## Browser and mail app

| # | Say | Expected |
|---|---|---|
| 13 | "search for cats" | Opens in Chrome (or the default browser without Chrome) |
| 14 | "search cats in Safari" | Opens in Safari |
| 15 | "search cats in Opera" with Opera not installed | "Opera isn't installed." Nothing opens |
| 16 | "send a mail to rahul about lunch" | Draft in the default mail app; "I drafted it. I don't send emails yet. Check it and press Send." |
| 17 | "write an email in Gmail to rahul about lunch" | Gmail compose opens in Chrome |
| 18 | "draft an email in Outlook about lunch" with Outlook not installed | "Microsoft Outlook isn't installed." |
| 19 | Tabs open in Safari (in front) and Chrome, both with a Grok tab: "close the grok tab" | The Safari one closes |
| 19a | A browser set as the default email app (System Settings or Mail → Settings → Default email reader): "draft a mail to rahul verma 1404 at gmail dot com regarding how are you" | Gmail compose opens in Chrome with To, subject and body filled |
| 19b | Mail set as the default email app: the same command | Mail opens the draft, as before |

## Email writing step

| # | Say | Expected |
|---|---|---|
| 20 | "email rahul verma 1404 at gmail dot com about moving our call to 5" | Subject of 3 to 7 words; body opens with the ask, keeps "5", "Hi," (no name guessed from the address), ends "Thanks," then my first name |
| 21 | "mail priya at example dot com saying sorry I'll be ten minutes late start without me" | Body is my words with punctuation fixed, nothing added but greeting and sign-off |
| 22 | "write a formal email to hr at example dot com asking for leave on friday for a family function" | "Hello,", formal tone, no invented reason or date |
| 23 | "email rahul verma 1404 at gmail dot com in hindi saying I'll be late" | Body in Hindi (Devanagari) |
| 24 | "email rahul verma 1404 at gmail dot com subject demo update body the demo moved to friday" | Subject "Demo update", body "The demo moved to Friday." |
| 25 | Any draft: the body has no "[", no "hope this email finds you well", no emojis, no dashes | Plain text only |
| 26 | Any draft, then read the web server log | One line per draft: `composed in … ms` or `fallback: <reason> in … ms`, never the email text; on fallback the rough draft still opens |
