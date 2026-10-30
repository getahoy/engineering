# Mobile UI build (iPhone first, Android-safe)

Rebuild the phone app's UI from the Claude Design files, as clean reusable components. The logic
already works; this is the look, the structure and the motion. **No behaviour changes** unless
this file says so.

## Scope

- **Phone app only.** Don't touch the Mac app.
- **The website stays a static marketing site for now.** Don't build or restyle any web sign-in,
  sign-up, account or auth pages, and don't add a web auth loading page. **Never remove or change
  the web auth API** (`/api/auth/*`, Better Auth, the pairing, transcribe, interpret and speak
  routes): the phone and Mac depend on them. Leave the existing web auth pages as they are; they
  get their own spec later.
- Sign in with Apple and the Mac's sign-in screens are out of scope.

## Inputs (in `~/Developer/Ahoy/design/`)

| File | Use it for |
|---|---|
| `DESIGN.md` | The rules: colours, type, spacing, radius, motion, every component's look and states |
| `tailwind-v4.css`, `css-variables.css`, `design-tokens.json` | The tokens. Copy values from here, never invent new ones |
| `Ahoy iPhone Screens.dc.html` | Every phone screen and state, laid out side by side. The main reference |
| `Ahoy iPhone.dc.html` | The clickable phone prototype (layout 1b is the chosen one) |
| `Ahoy Components.dc.html` | The component sheet: tags, pills, buttons, sheets, reply cards |
| `Ahoy Onboarding.dc.html` | Only the Welcome screen for now |

- The HTML files are **web mockups**. Read them for layout, spacing, sizes, copy and states. Do
  not copy their HTML structure; rebuild in React Native.
- Ignore `support.js`, `image-slot.js`, the `Web*` and `Mac*` files, and layout 1a.
- Where a mockup and DESIGN.md disagree, DESIGN.md wins. Where either disagrees with the real
  app behaviour, the app wins: tell me.

## Stack (already installed, use it)

- **Uniwind** (Tailwind v4 for React Native) for all styling, through `className`.
- **React Native Reusables pattern**: `@rn-primitives/*` + `class-variance-authority` + `cn()`
  (clsx + tailwind-merge). UI primitives live in `src/components/ui/`, shadcn-style: one
  component per file, variants with `cva`, a `className` prop merged with `cn()`.
- **Reanimated 4** for all animation (its CSS-style transitions and animations for simple state
  changes; shared values and `useAnimatedStyle` for gesture or audio-driven motion).
  `react-native-gesture-handler` for press-and-hold.
- **@gorhom/bottom-sheet** for every sheet. **expo-haptics** for haptics.
- **Motion (motion.dev) is web only.** It doesn't run on React Native, so the phone uses
  Reanimated. (The web build later uses the `motion` package, imported from `motion/react`.
  Never `framer-motion`.)

New dependencies need my OK first. Likely candidates; propose each one with one line of reasoning:
- `@shopify/react-native-skia` for the orb's radial glow (React Native has no radial gradient)
- `expo-symbols` for SF Symbols on iOS, with an `@expo/vector-icons` fallback on Android behind
  one `Icon` component
- `@expo-google-fonts/inter`, plus the font files below

## Part 0: foundation

1. **Tokens.** Put the colours, radii, spacing, type scale and shadows from `tailwind-v4.css`
   into `src/global.css` with Uniwind's `@theme`. Include the dark set (Uniwind's dark theme
   variant). Tokens stay in the app's own CSS; nothing goes into `shared`.
2. **Fonts** via expo-font, loaded before the splash screen hides:
   - Inter 400 and 500. Geist Mono 500.
   - Fraunces: React Native can't set variable axes, so generate static instances once with a
     script in `mobile/scripts/` (`fonttools varLib.instancer`: opsz 72, SOFT 100, WONK 1, at
     wght 300 and 400, upright and italic) named `FrauncesAhoy-Light`, `-LightItalic`,
     `-Regular`, `-Italic`. Commit the generated files plus the script, and note the font
     licence (OFL) in the README.
   - Text variants map to these files. Never set `fontWeight` on Fraunces; pick the file.
3. **Motion tokens** in `src/lib/motion.ts`: durations (150 / 240 / 480 ms), easings
   (`ease-out`, `ease-spring`, `ease-in-out` from DESIGN.md as Reanimated `Easing.bezier`), the
   word stagger (60 ms), and a `useReducedMotion()` wrapper. Every animation reads these. With
   Reduce Motion on, replace movement with a fade of 150 ms or less.
4. **Haptics** in `src/lib/haptics.ts`: named moments (`talkStart`, `talkEnd`, `choicePicked`,
   `confirmOpen` warning, `success`), so screens never call expo-haptics directly.
5. **A dev gallery route** (`src/app/dev/components.tsx`, dev builds only): every UI primitive
   and feature component in every state, light and dark. This is how I review the work on my
   phone.

## Part 1: UI primitives (`src/components/ui/`)

First audit the existing components and say which get replaced, merged or deleted. No duplicates
at the end.

| Component | Variants and states |
|---|---|
| `Text` | display, heading, title, heard (Fraunces italic), body, bodySmall, caption, eyebrow (Geist Mono uppercase) |
| `Button` | primary (Ink), secondary (Paper with Rope border), ghost, destructive (Flare); sizes md (52) and sm (40); pill; loading (no spinner under 300 ms, at least 500 ms once shown); disabled |
| `Input` | pill or field shape, focus ring (Signal plus Signal Mist), error, secure entry with an eye toggle |
| `Card` | Paper, 24 radius, no shadow |
| `StatusPill` | connected (Kelp dot), connecting (pulsing Signal dot), locked, paused, can't reach, offline (Fog) |
| `ChoiceTag` | slanted parallelogram (skewX -10°, radius 7, Ink), numeral counter-skewed in Geist Mono; selected = Signal with scale 1.1 |
| `Sheet` | the @gorhom wrapper: Paper, 28 top radius, grabber, optional draining time bar (2px Signal), optional Flare Mist header |
| `Switch`, `ListGroup`, `ListRow` | the iOS grouped settings style |
| `Banner` | the network banner (28px Ink pill, fade plus an 8px drop) |
| `Icon` | one icon component (SF Symbols on iOS, fallback on Android) |

## Part 2: feature components and screens

### Only what exists

**The UI shows only features the app already has.** The mockups contain a few things that aren't
built yet; leave them out rather than adding dead buttons:
- **No History screen and no clock icon in the header** (history is a later step). The current
  thread on Home stays.
- **Account Name and Email are read-only** (there's no name editing yet).
- **No Sign in with Apple and no "Forgot password?" flow** unless it already exists in the code.
- Choice rows have no thumbnails.

**Everything that does exist must get UI, even when the mockups missed it.** Before part 2, list
every state and action in `macCopy`, `talkCopy`, `pairingCopy`, `voiceCopy` and `authCopy`, and
map each to a designed screen or component. Known ones the mockups don't show:
- **Mac states:** Mac turned off, Ahoy not running, asleep, on mobile data, can't reach, Mac app
  too old, phone app too old, Mac doesn't recognise this phone.
- **Ended states:** removed, replaced by another phone, signed out on the Mac.
- **Mac actions:** the "More options" menu (Connect / Disconnect, Remove this Mac, Pair again).
- **Sign out:** the online and offline sign-out confirm.

Use the existing designed patterns for these (Mac card or chip with a message plus one action,
the confirm sheet). Show the mapping in your first reply, and put every state in the dev gallery.
If something important truly needs a new UI pattern, ask first.

### Order and structure

Keep the mirrored feature layout (`src/features/<feature>/components|screens|hooks|copy/`).
Screens compose components; components hold no logic; hooks own state and return what the
component needs; all copy comes from the copy files (no inline strings).

Build in this order:
1. **Starter (Welcome):** the wordmark (matching the launch screen) → it slides up and fades
   while the orb rises → "Your Mac, *on a word.*" word by word → the buttons fade in. About
   2 s; tap anywhere to skip to the end. Then "Get started" / "I have an account" go to sign-up
   and sign-in. **Onboarding screens 2–7 are not built now.**
2. **Sign in and Create account:** existing logic, new look, including the error states,
   and Continue with Google. Leave room for Sign in with Apple and "Forgot password?" later (see
   decisions), but don't show them yet.
3. **Home / talk (layout 1b):**
   - A header chip for the Mac's status
   - The current thread (main prompt plus follow-ups, folded when finished) or the single reply
   - The Heard line
   - The docked full-width talk bar with every state: idle, listening (orb glow driven by mic
     level), understanding, sending, and disabled with its reason
   - The network banner states
   - The "No Macs yet" empty state
4. **Mac card / chip states:** connecting, connected, locked, paused, can't reach, ended (Pair
   again / Dismiss), offline, no internet.
5. **Sheets:** Which one? (with the time bar; a numbered row has no thumbnail for now), expired
   ("That screen changed. Ask again."), the destructive confirm.
6. **Pairing:** camera priming, scanner, local network priming, the Mac checklist, success, and
   every edge state from the Screens file.
7. (History is skipped for now, see "Only what exists".)
8. **Settings:** Macs (each paired Mac with its status, Connect / Disconnect, Remove this Mac,
   Add another Mac → the existing pair screen), account (read-only), and the sign-out sheet, as in the
   mockups. **Plus a Voice replies group** (designed in the Screens file under "Voice replies";
   if it's missing there, build it anyway with the guardrails below) and wire it
   to the existing voice settings: Plays on (This iPhone / Mac / Both / Off), Speak (Questions and
   problems / Everything / Off), Voice (Default / Indian English), Speak in silent mode (switch).
   Guardrails for anything not in the mockups:
   - Use only existing primitives (`ListGroup`, `ListRow`, `Switch`, `Text`) and tokens. No new
     visual pattern, colour, size or component.
   - Copy the look of the nearest designed screen: the Settings rows, the group headers in Geist
     Mono uppercase Driftwood, the chevron rows.
   - A choice with 3 or 4 options opens a small sheet with a checkmark list (the same `Sheet`
     wrapper). No dropdowns, no segmented controls.
   - Add the screen to the dev gallery and list it in the end summary as "built without a
     mockup", so I review it.
9. **Signing-in screen** (designed in the Screens file under "Signing in"; if it's missing, use
   the guardrails from 8). After "Continue with Google" returns
   to the app, and while the session is checked on a cold start, show a calm in-between screen
   instead of a blank or white flash:
   - Sailcloth background, the wordmark, the small orb breathing slowly, and "Signing you in…"
     in Inter 15 Driftwood.
   - Spinner timing rules apply: show it only if the wait passes 300 ms, and keep it at least
     500 ms once shown. Then make one forward screen change (home, or "Pair your Mac" if no Mac
     is paired).
   - Failure or cancel: back to sign-in with the existing "Google sign-in did not finish. Try
     again." error. Never a dead end.
   - **No white flash anywhere:** hold the splash screen until fonts and the session check are
     ready, and set the root and system background to Sailcloth (expo-system-ui, plus the native
     splash background) so even the first frame is cream.

## Rules

- **Styling:** `className` with tokens only. No raw hex values, no arbitrary values; if a value
  is missing, add a token and say so. 4px spacing scale. Inline `style` only for Reanimated
  animated styles.
- **Layout:** safe areas through `react-native-safe-area-context`. Screen padding 20. Tap
  targets at least 44pt (use `hitSlop` where the visual is smaller). Long names cut off with an
  ellipsis.
- **The CLAUDE.md UI rules apply:** instant final state for local actions, a waiting state for
  remote ones, one forward screen change per action, the spinner timing, and buttons disabled
  while their action runs.
- **Accessibility:** a label and role on every control. VoiceOver order follows the visual
  order. Dynamic Type allowed, capped with `maxFontSizeMultiplier` on the display sizes.
  Contrast per DESIGN.md (Fog never for readable text).
- **Android-safe:** no iOS-only API without a Platform check. Note anything that needs Android
  work later.
- Delete dead UI code you replace. One component per file. Typecheck, lint and tests pass.

## How to work

- First: list every file you'll create, change or delete, the dependencies you want, and which
  existing components you'll replace. Wait for my OK.
- One commit per part (0, 1, then each numbered item of part 2).
- After each part, tell me what to open in the dev gallery and on which screens.
- At the end: what's built, anything weaker than the mockups, what couldn't be checked, and the
  native rebuild command if needed.

## Decisions (already made)

- Layout **1b** (header chip, single reply or current thread, docked talk bar).
- **The Mac signs in by the phone scanning its QR.** The Mac mockups show a code-in-browser
  step; that's a possible future design only. Don't build it.
- Choice rows have no screenshot thumbnail until screen capture exists.
- **Sign in with Apple** is required by App Store guideline 4.8 because we offer Google. It
  belongs to the "complete auth" roadmap item; leave the button slot and flag it, don't build
  it here.
