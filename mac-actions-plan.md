# Mac actions, batch 1: scroll, media, volume, windows, lock

More things the Mac can do from the couch. Written 2026-11-15. Build in the parts at the end,
one commit set per part, in order. No extras beyond what is listed.

## Why these first

Ahoy's use is "the Mac is across the room". Today the Mac opens, quits, hides and switches
apps, searches, opens sites, presses shortcuts, types, clicks by name, closes or switches tabs
and drafts emails. What's missing for that use: reading (scrolling, back, reload), media
(pause, next, volume) and arranging windows. They are all safe (nothing is deleted or sent)
and easy to check by eye.

## What the person says → what happens

| Say | Action | How the Mac does it |
|---|---|---|
| scroll down / up, a little, a lot, 3 times, page down / up | `scroll { direction: up \| down, amount: small \| page \| large, times: number }` (the Mac caps at 10) | Helper posts scroll-wheel events at the centre of the front window (see Amounts) |
| go to the top / bottom, scroll to the end | `scroll { direction: top \| bottom }` | mod+up / mod+down |
| more, again, keep going, a bit more, louder, quieter | `again { amount?: small \| normal \| large }` | Repeats the last scroll, volume or brightness step (see Amounts) |
| go back / forward, reload | `navigate { to: back \| forward \| reload }` | mod+[ / mod+] / mod+r (browsers, Finder) |
| play, pause, resume, stop the music | `media { action: playPause }` | System media key from the helper |
| next / skip, previous / last song | `media { action: next \| previous }` | System media key |
| volume up / down, turn it up, a little louder, up by 15 | `volume { action: up \| down, percent: number \| null }` (the step; null = default) | AppleScript `set volume output volume` (see Amounts) |
| volume to 30, half volume, full volume, volume 0 | `volume { action: set, percent: number }` (the level) | AppleScript |
| mute, unmute | `volume { action: mute \| unmute }` | AppleScript `set volume output muted` |
| brightness up / down, to 40, a little brighter, by 20 | `brightness { action: up \| down \| set, percent: number \| null }` | DisplayServices in the helper, brightness keys as fallback (see Amounts) |
| full screen, exit full screen | `window { action: fullScreen \| exitFullScreen }` | Accessibility `AXFullScreen` on the front window |
| minimize | `window { action: minimize }` | `AXMinimized` |
| maximize, fill the screen | `window { action: fill }` | Accessibility frame = the screen's visible frame |
| left half, right half | `window { action: leftHalf \| rightHalf }` | Accessibility frame = half the visible frame |
| move it to the other screen | `window { action: nextDisplay }` | Same relative frame on the next display |
| lock my Mac | `lock` | ctrl+cmd+Q from the helper |

## Amounts

Vague amounts never get a question back. A question costs a whole extra hold-and-speak round
trip; a sensible step plus "more" is faster, and the reply always says where it landed so the
next correction is easy. The same approach as Siri and Alexa's "turn it up".

**Volume** (0–100, always clamped):
- "up / down", "louder", "quieter", "turn it up": ±10.
- "a little / a bit / slightly": ±5. "a lot / way up / much louder": ±25.
- "up by 15", "down 20": ±that number. A bare "volume 30" or "to 30" is set, not plus.
- "half": 50. "full / max / all the way up": 100. "lowest / all the way down": 0 (not
  mute, so unmute isn't needed after). "too loud": −10.
- Reply with old → new: "Volume 40% → 50%".

**Scroll:**
- "scroll down / up": one page (80% of the front window's height), so reading from the couch
  moves a screenful at a time with a few lines of overlap.
- "a little / a bit / slowly / slow": small (≈ 3 lines). "a lot / fast / way down": large
  (3 pages).
- "3 times", "2 pages": `times`, capped at 10.
- "to the top / bottom / end": mod+up / mod+down. On a feed that loads forever, that goes as
  far as the page has loaded; that's acceptable.
- Speed words change distance, not animation: every scroll is a short smooth burst
  (~250 ms) of small wheel events.
- "Scroll until I say stop" and "keep scrolling slowly" (auto-scroll) are not in this
  batch; reply that it can't do that yet.
- Reply: "Scrolled down", "Scrolled down a lot", "At the bottom".

**Percent means points.** "Up by 20%" from 60 is 80, not 72. Siri, Alexa and the macOS
volume display all work this way, and the reply ("Volume 60% → 80%") shows what happened, so a
wrong guess is one sentence to fix. "20 percent more than now" is also points; no question.

**Brightness** (built-in display, 0–100):
- Read and set it exactly through DisplayServices (`DisplayServicesGetBrightness` /
  `SetBrightness`, the private framework MonitorControl and other utilities use). Ahoy isn't
  sandboxed or in the App Store, so this is allowed; it can break with a macOS update, so it
  sits behind a fallback.
- Fallback when DisplayServices is missing or fails: the brightness keys. Relative steps as
  before; "to 40%" presses down 16 times to reach 0, then up 6 (16 steps, so about 38%), and
  the reply says "about 40%". No reading back in the fallback, so replies say "Brightness up",
  not a number.
- Same words as volume: up/down 10, "a little" 5, "a lot" 25, "by 20", "to 40", half, full,
  lowest. Reply "Brightness 50% → 70%".
- External displays: "I can change your Mac's own screen. External displays aren't supported
  yet." (DDC control comes later, if ever.)

**Edge cases (volume and brightness alike).** Never a question, never a silent failure: do the
nearest sensible thing and say what happened, or say in one line what Ahoy can do instead.

| Case | Ahoy does | Reply |
|---|---|---|
| Above 100 or below 0 ("volume 150") | Clamps | "Volume is at the maximum, 100%." |
| Already at the limit ("louder" at 100) | Nothing | "Volume is already at 100%." |
| Muted, then "volume up" or "to 40" | Unmutes, then sets | "Volume 0% → 40%" |
| "7 out of 10", "level 7" | 70 | "Volume 70%" |
| A bare small number ("volume 5") | 5%: too quiet is safe, too loud isn't | "Volume 5%" |
| One app's volume ("Spotify volume to 50") | Nothing | "I can change the Mac's volume, not one app's yet." |
| Keyboard backlight, external display | Nothing | One line naming what it can change instead |
| Words with no number it can read | AI picks the nearest field; if none, the usual "couldn't do that" | — |

Every result line goes into History as is, so wrong guesses show up in the Failed filter and
the weekly review.

**Again / more.** The Mac remembers the last scroll, volume or brightness step for 60 s.
"more", "again", "keep going", "a bit more", "louder" or "quieter" right after it repeats the
same direction, scaled by `amount` (small = half, large = double). "louder" and "quieter"
pick volume when the last step wasn't volume. After 60 s, or when the last command was
something else, "more" alone replies "More of what?" and does nothing. The memory belongs to
this Mac and clears when the phone disconnects.

Rules take the plain forms (words above, digits, "up by N", "to N", "N times"); the AI takes
everything fuzzier ("bahut tez hai", "can't hear it", "make it way quieter") into the same
fields, never a question.

## How

**Rules first, AI second.** `core/commands/parseSystemCommand.ts`, checked before the web and
app rules, takes the fixed English phrases above (with fillers stripped by
`normaliseCommand`). Rule matches skip the AI: no network round trip, no cost, and they still
work when Smart commands are down. Everything else (other wording, Hindi and Hinglish such as
"awaaz badhao", "neeche scroll karo") goes to the AI, which gets the same new action kinds.

**Helper (Swift), three new requests:**
- `scroll { lines | pages | direction }`: CGEvent scroll-wheel events with their location set
  to the centre of the front window, so the window you're looking at scrolls, not whatever
  is under a forgotten mouse pointer. Several small events, not one jump, so pages scroll
  the way a trackpad does.
- `brightness { get | set level }`: DisplayServices, loaded at runtime with `dlopen` so a
  missing symbol fails with `unavailable` instead of crashing the helper; the agent then falls
  back to the brightness keys.
- `systemKey { key: playPause | next | previous | brightnessUp | brightnessDown }`: NSEvent
  `systemDefined` (subtype 8, NX_KEYTYPE_PLAY / NEXT / PREVIOUS / BRIGHTNESS_UP / DOWN), key
  down then key up. CGEvent keyboard events can't produce media keys; these reach whatever
  macOS treats as Now Playing (Spotify, Music, YouTube in a browser).
- `window { action }`: Accessibility on the focused window of the front app. Works the same in
  every standard app; keyboard shortcuts for this differ per app and the globe key can't be
  sent reliably. A window that refuses the change (fixed size, no full screen) fails with
  `notResizable`.

Add `scroll`, `systemKey` and `window` to the helper capabilities.

**Lock** is its own action, not a `keys` shortcut: the shortcut filter blocks any mod+Q (quit)
on purpose, and that rule stays. The lock action sends exactly ctrl+cmd+Q and nothing else.
After it the Mac reports locked, so the phone shows its locked state as it does today.

**Volume and navigate** reuse what exists: `osaScript` for volume, `pressKeys` for
navigate and top/bottom.

## Protocol (shared, additive)

- `interpretActionSchema` gains `scroll`, `navigate`, `media`, `volume`, `brightness`,
  `window`, `lock` and `again`, with the fields in the table.
- Volume and brightness share one shape: `action` plus one nullable `percent` (the step for
  up/down, the level for set). Separate `by` + `level` fields push the model's compiled grammar
  past Anthropic's size limit.
- Number limits live on the Mac, not in the schema. The model doesn't reliably keep to
  min/max, and an out-of-range number would fail the whole answer ("Smart commands aren't
  working"). The schema keeps only integer and non-negative, with loose ceilings (percent up to
  1000, times up to 100); the Mac clamps and replies with the edge-case lines in Amounts.
- `interpretRequestSchema` gains optional `supports: string[]`, the action kinds this Mac can
  run. The server drops any returned action whose kind isn't listed; a request without
  `supports` (an older Mac) gets only today's kinds. Without this, a new server answering an
  old Mac with `scroll` would fail its schema check and show "Smart commands aren't working".

## AI prompt (web)

Add the new actions to `INTERPRET_INSTRUCTIONS` with one line each, in the same style.
- "Turn it up" is volume unless an app or window is named.
- "Stop" is pause.
- "Scroll to <something>" is not a scroll (it needs finding something on screen): empty
  list.
- Add Hinglish examples: "awaaz badhao" (volume up), "awaaz kam karo" (volume down),
  "neeche karo" (scroll down), "gaana rok do" (pause).
- Lock only when the person says lock.
- Add eval cases in `scripts/interpretCases.ts` for every new action, each Hinglish line, and
  the near misses ("scroll to the pricing section", "stop" with nothing playing is still
  pause). Run the eval and paste the score.

## Safety

- `scroll` counts as a screen action, like click, keys and type: it fails closed on the
  blocked-apps list and when the front app is unknown.
- Media, volume, brightness, window and lock don't read or change what's inside an app; they
  aren't blocked.
- No Mac sleep and no shutdown: the phone can't wake the Mac again. (Display sleep and Mac
  sleep come later with Wake-on-LAN.)

## Replies

- Short result lines for the card and History: "Scrolled down", "Paused", "Next track",
  "Volume 40%", "Muted", "Brightness up", "Safari is full screen", "Moved Safari to the left
  half", "Moved Safari to the other screen", "Locked". Each failure gets a plain line too:
  - "That window can't be resized."
  - "There's only one screen."
  - "No window is open in front."
- No spoken reply for media, volume, brightness, scroll and window: you hear or see the
  result, and a voice talking over the music it just started is worse. The phone still shows
  the card and plays its haptic. Lock speaks nothing either.
- `app` on the result is the front app for scroll, navigate and window, and none for the
  rest.

## Known macOS behaviour

- If nothing has played since login, the play key opens Apple Music. Nothing reliable and
  public tells us what's playing (Apple shut MediaRemote to third parties in macOS 15.4), so
  this stays as macOS does it.
- Brightness keys move only the built-in display.

## Parts

1. **Protocol + server:** the new action kinds, `supports`, the server filter, the prompt, the
   eval cases (paste the score). Tests for the schemas and the filter.
2. **Helper:** `scroll`, `systemKey`, `window`, with the capability names.
3. **Agent:** `parseSystemCommand` rules, the `Intent` kinds, `runIntent` cases, the lock
   action, `supports` sent with every interpret call, reply copy, no speech for these kinds,
   scroll in `SCREEN_INTENTS`, the `again` memory. Tests for the rules (each phrase in the
   table and in Amounts, fillers, "scroll to X" goes to the AI, "volume 30" is set and "up 30"
   is plus, "by 20%" is points), every row of the edge-case table for volume and brightness,
   the brightness key fallback math, `again` (repeats, scales, expires after 60 s, "More of
   what?"),
   and blocked-app scroll.
4. **Checks file:** `engineering/mac-actions-checks.md`, one row per table line, run on the
   Mac from the phone. Also try: scroll in a blocked app (refused), "make it full screen" on
   a fixed-size window (plain failure), two displays if you have them.

Each part: typecheck, lint, tests pass; a decisions.md entry for "rules first, AI second for
system actions" and "lock is its own action".

## Not in this batch

Mac sleep and shutdown, files and folders, answering questions out loud, sending messages.
Do Not Disturb, calendar and reminders come through Shortcuts (next section), and longer
multi-step tasks through agent mode.

## After this batch, in order

Each one builds on what exists; nothing is replaced.

1. **Shortcuts bridge.** Run the user's Shortcuts and apps' own Shortcuts actions with the
   `shortcuts` command-line tool (`shortcuts list`, `shortcuts run`). The AI gets the
   shortcut names like it gets installed apps. This unlocks Focus / Do Not Disturb, Notes,
   Reminders, Home, and any app that ships Shortcuts actions, with no UI driving. Apple's new
   Siri works the same way: the model picks from actions apps expose.
2. **Front-app context for the AI.** Send the front app's name (and maybe the window title)
   with each interpret call, so "this", "that" and "close it" resolve better. App name only
   by default: a small privacy step, covered by the existing cloud disclosure.
3. **Agent mode for open tasks.** "Reply to Daniel's last email saying I'm in" needs look →
   act → look. A separate loop, used only when the one-shot path returns nothing. It reads
   the accessibility tree, uses the same tools, has a step cap and a stop from the phone, and
   tiered approvals: reading is silent, acting is logged, and sending, deleting or paying
   asks first.
4. **Vision fallback for click.** When an app exposes no accessibility labels (some Electron
   or canvas apps), "click X" fails today. Use a screenshot (the helper can already capture)
   only after the accessibility search finds nothing.

## How Ahoy compares (2026-11-15)

| Product | How it acts | Compared with Ahoy |
|---|---|---|
| Apple Voice Control, Talon | Fixed voice phrases, plus numbers on screen to click | Ahoy's numbered click is the same idea; Ahoy adds AI, so you don't learn exact phrases |
| Wispr Flow Command Mode | Edits or drafts text in the focused text box, or runs a web search; its docs don't describe controlling apps | Ahoy acts on the whole Mac: "Wispr Flow that acts" holds |
| Apple's new Siri (WWDC 2026) | AI picks from actions the system and apps expose, plus on-screen awareness | Same shape as Ahoy: a model choosing from real actions, not clicking around. Siri also gets apps' own actions; Ahoy gets them with the Shortcuts bridge |
| Screenshot agents (computer-use style) | Loop: screenshot → think → click → screenshot | About 2–5 s per step and around 20% success on open desktop tasks, with failures compounding over steps. Ahoy answers most commands in about a second with one call |
| Microsoft UFO2 / HCU (research) | Hybrid: app APIs and code first, accessibility tree next, screen vision last; the model picks per step | The research's best practice. Ahoy has the first two layers (AppleScript and browser scripts, then accessibility); vision is item 4 above |

**In one line:** Ahoy takes the fast, reliable path first (local rules, then one AI call
choosing from safe, tested actions) and keeps the slow, screen-watching agent for the tasks
that need it. That's the order Apple and Microsoft's research both point to.

Sources:
- [Wispr Flow: How to use Command Mode](https://docs.wisprflow.ai/articles/4816967992-how-to-use-command-mode)
- [Apple Newsroom: Siri AI](https://apple.com/newsroom/2026/06/apple-introduces-siri-ai-a-profoundly-more-capable-and-personal-assistant/)
- [Microsoft Research: UFO2, The Desktop AgentOS](https://www.microsoft.com/en-us/research/publication/ufo2-the-desktop-agentos/)
- [AI Weekly: Microsoft HCU tops OSWorld 2.0](https://aiweekly.co/alerts/microsoft-hcu-tops-osworld-20-with-three-agent-channels)
- [Computer Use Agents in Production](https://tianpan.co/blog/2026-04-10-computer-use-agents-when-pixels-replace-apis)
- [Apple: Voice Control number and grid overlays](https://support.apple.com/en-asia/guide/mac-help/mchl26854b08/12.0/mac)
