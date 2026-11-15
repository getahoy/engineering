# Mac actions checks (scroll, media, volume, brightness, windows, lock)

Run on a real phone and Mac. Before starting: restart the web dev server (the shared schema
changed), quit and relaunch Ahoy on the Mac with `pnpm start` in agent (it rebuilds the Swift
helper; the running app still has the old one), reload the phone app. Accessibility must be
on for Ahoy. Successful scroll, media, volume, brightness, window and lock results show a
card and haptic but say nothing aloud.

| # | Do this | Expect |
|---|---|---|
| 1 | Safari on a long page, pointer parked over another window: "scroll down" | The Safari page moves about one screenful; "Scrolled down" |
| 2 | "Scroll up a little", then "scroll down a lot", then "scroll down 3 times" | Small move up; about three screens down; three screens down; lines say so |
| 3 | "Scroll down 20 times" | Ten screens at most; "Scrolled down 10 times" |
| 4 | "Go to the top", then "scroll to the end" | Page jumps to the start, then the end; "At the top", "At the bottom" |
| 5 | "Neeche karo", then "upar karo" | Scrolls down, then up, without the AI (History shows rules) |
| 6 | "Scroll to the pricing section" | Nothing scrolls; "I can't do that yet" |
| 7 | "Keep scrolling" | Nothing moves; "I can't keep scrolling yet. Say scroll down, then more." |
| 8 | "Scroll down", then within a minute "more", "a bit more", "a lot more" | Same direction each time, smaller then larger |
| 9 | Wait over a minute after a scroll, or say "open notes" first, then "more" | Nothing happens; "More of what?" |
| 10 | Safari, "go back", "go forward", "reload" | Safari goes back, forward, reloads; "Went back", "Went forward", "Reloaded" with no voice; app is Safari |
| 11 | Spotify or Music playing: "pause", "play", "next", "previous song", "stop the music" | Playback toggles and changes track; no voice over the music |
| 12 | Nothing has played since login: "play" | Apple Music opens (macOS behaviour) |
| 13 | Volume at 40: "volume up", "turn it up a little", "volume up by 15", "louder" | 50, 55, 70, 85 ("louder" repeats the last step, 15); each line "Volume X% → Y%" |
| 14 | "Volume 30", "half volume", "volume 7 out of 10", "volume 5", "volume 0" | 30, 50, 70, 5, 0; volume 0 is not muted |
| 15 | "Volume 150" | 100; "Volume is at the maximum, 100%." |
| 16 | At 100, "louder"; at 40, "volume 40" | Nothing changes; "Volume is already 100%.", "Volume is already 40%." |
| 17 | At 30, "mute", then "volume up" | Muted, then unmuted at 40; "Volume 30% → 40%" |
| 17a | At 30, "mute", then "volume 30" | Unmuted at 30; "Volume is already 30%." |
| 18 | "Awaaz badhao", "awaaz kam karo", "it's too loud" | Up 10, down 10, down 10, without the AI |
| 19 | "Spotify volume to 50" | System volume unchanged; "I can change the Mac's volume, not one app's yet." |
| 20 | MacBook screen at 50: "brightness up", "brightness to 40", "full brightness", "a little brighter" | 60, 40, 100, then "Brightness is already 100%." |
| 21 | "Dim my external monitor", "turn up the keyboard backlight" | Nothing changes; the one-line refusals |
| 22 | Lid closed on an external display: "brightness up" | "I can change your Mac's own screen. External displays aren't supported yet." |
| 23 | Safari in front: "make it full screen", then "exit full screen" | Full screen and back; "Safari is full screen", "Safari left full screen" |
| 24 | "Minimize", "maximize", "left half", "right half" | The window does each; lines name Safari |
| 25 | Two displays: "move it to the other screen"; one display: same | Moves with the same relative frame; one display: "There's only one screen." |
| 26 | System Settings (fixed size) in front: "maximize", "make it full screen" | Nothing changes; "That window can't be resized." |
| 27 | Finder with no window open: "left half" | "No window is open in front." |
| 28 | "Lock my Mac" (also "lock kar do") | Mac locks; "Locked"; the phone shows the locked state |
| 29 | A blocked app in front (add it to `blockedApps` in the settings file): "scroll down" | Nothing scrolls; "Ahoy doesn't click or type in this app…" |
| 30 | "Thodi awaaz badha do", "screen thodi dim karo", "put this on the left side" | Through the AI: up 5-10, brightness down 5-10, left half |
| 31 | Disconnect the phone (wifi off and on), then "more" right after a scroll | "More of what?" (the step memory cleared) |
