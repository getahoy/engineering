# Voice replies checks

Run on a real phone and Mac. Before starting: set `ELEVENLABS_VOICE_DEFAULT` and
`ELEVENLABS_VOICE_INDIA` in `web/.env.local` and restart web; rebuild the phone app (it adds
native modules); `pnpm start` in agent.

| # | Do this | Expect |
|---|---|---|
| 1 | "Open Calculator" (default policy) | Opens; soft sound + haptic; no speech |
| 2 | Policy Everything, "open Calculator" | Speaks "Done. Opened Calculator." through the speaker, not the earpiece |
| 3 | "Click save" with two Save buttons | Speaks the two-option line; tags on Mac; sheet on phone |
| 4 | …then hold and say "two" | Clicks the second; "Done. Clicked Save."; one thread, folded after |
| 5 | …same, say "doosra" | Same as 4 |
| 6 | …same, say "five" | "There are only two. Say one or two."; choice stays open |
| 7 | …same, say "you pick" | "Picked one, the Save in the toolbar." and clicks it |
| 8 | "Delete" with two Delete buttons, say "you pick" | "I won't pick for a delete…"; nothing clicked |
| 9 | Confirm sheet open, say "yes" | Not confirmed; "Tap Delete on your phone to confirm." |
| 10 | Press talk while Ahoy is speaking | Speech stops instantly; recording starts |
| 11 | Hold for 25 s answering an open choice | Choice doesn't expire mid-answer |
| 12 | Answer a choice, then say "open Spotify" instead | Choice closes as replaced; Spotify opens |
| 13 | Four rounds of non-answers | "Let's start over."; thread closed |
| 14 | Airplane mode after the result arrives | Text shows; device voice speaks (fallback) |
| 15 | Slow network (Link Conditioner) | Device voice within about 1.5 s, never both voices |
| 16 | Silent switch on, default settings | No sound; text shows |
| 17 | Silent switch on, "Speak in silent mode" on | Speaks |
| 18 | VoiceOver on | VoiceOver reads it once; our voice stays quiet |
| 19 | AirPods connected, a full talk then a reply | Reply plays clearly in AirPods |
| 20 | Speak on: Mac | The Mac speaks; the phone is quiet |
| 21 | Speak on: Mac, press talk while the Mac is speaking | The Mac stops |
| 22 | "Draft a mail to samar at gmail dot com saying hi" | "Drafted an email to Samar. Check it and press Send." No address read out |
| 23 | Same phrase twice | Second plays instantly (cache) |
| 24 | Two commands quickly; the first's result arrives late | Only the latest is spoken |
| 25 | Background the app mid-speech | Speech stops |
| 26 | Remove the voice env vars and restart web, then policy Everything | The device voice speaks; no error shown |
| 27 | Two Save buttons both in the toolbar | "I see two Save buttons. Say one or two." with no areas |
