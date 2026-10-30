# Mac app UI build (menu bar, popover, main window)

Rebuild the Mac app's UI from the Claude Design files. The logic already works; this is the
look, the structure and the motion. **No behaviour changes** unless this file says so.

## Inputs (in `~/Developer/Ahoy/design/`)

| File | Use it for |
|---|---|
| `DESIGN.md` | The rules: colours, type, spacing, radius, motion, component looks |
| `tailwind-v4.css`, `css-variables.css`, `design-tokens.json` | The tokens. Copy values from here, never invent new ones |
| `Ahoy Mac.dc.html` | The Mac canvas: menu bar icon states, popover states, main window pages, first-run setup, overlay |
| `Ahoy Mac Popover.dc.html`, `Ahoy Mac Window.dc.html` | The popover and main window components, in light and dark |
| `Ahoy Components.dc.html` | Shared pieces (pills, status pill, choice tags) |

- The HTML files are mockups. Read them for layout, spacing, sizes, copy and states, then rebuild
  them as clean React components.
- Ignore the phone, web and onboarding files, `support.js` and `image-slot.js`.
- Where a mockup and DESIGN.md disagree, DESIGN.md wins. Where either disagrees with the real
  app behaviour, the app wins: tell me.

## Scope

- **Mac app only** (`agent/`). Don't touch mobile, web or shared without asking.
- **Never change** pairing, connection, command, resolver, helper protocol or safety logic. UI and
  IPC wiring only.
- Commit each part on main as soon as typecheck, lint and tests pass. No push.

## Stack

The renderer is a web page inside Electron, so web tools apply here (unlike the phone):
- **React + Tailwind v4** (already installed).
- **shadcn/ui** components, the same family as the website (Base UI primitives, `cva`, `cn()`),
  copied into `src/renderer/components/ui/` and restyled to the tokens.
- **Motion** (`motion` package, imported from `motion/react`) for animation. **Never
  `framer-motion`.**
- **lucide-react** icons at strokeWidth 1.5.
- **Fonts bundled locally, no CDN** (the app must work offline and keep a strict CSP): Fraunces
  variable via `@fontsource-variable/fraunces` (full axes, so `"SOFT" 100, "WONK" 1` works),
  Inter 400/500, Geist Mono 500.
- New dependencies: list them in your first reply and wait for my OK.

## Only what exists

**The UI shows only features the Mac app already has.** The mockups include things that aren't
built; leave them out rather than adding dead controls:
- **No code-in-browser sign-in screen.** The Mac signs in when the phone scans its QR. Setup is
  Welcome → Permissions → Pair → done.
- **No "Launch at login", no "pause shortcut",** and no multiple phones (one phone per Mac).
- **"Never click in these apps":** the list exists in the settings store (`blockedApps`). Show it
  only if an IPC to read and change it already exists; otherwise skip it and tell me.

**Everything that does exist must get UI, even if the mockups missed it.** Before building, list
every string in `agentCopy`, `helperStatusCopy` and `pairingFailureCopy` and map each to a
designed screen or component. At minimum:
- **Setup:** welcome; permissions (Accessibility, Screen Recording, Open System Settings, the
  relaunch hint plus Restart Ahoy, Continue).
- **Pair:** starting, the QR plus instructions, the 5-minute refresh, `notOnLan`, every pairing
  failure plus Try again, paired with phone plus account, Done, "Pair a new phone" plus Cancel.
- **Home:** the phone status (connected, not connected, paused, screen locked, Ahoy off), the last
  heard line, the ended notices (removed, signed out), remove phone plus confirm, sign out
  (with and without a phone) plus confirm, and the helper status (starting, running, restarting).
- **Menu bar:** every status line, "Ahoy On", Open Ahoy, Quit Ahoy, and the first-run
  notification.

Show the mapping in your first reply. If something needs a new pattern, ask first.

## Part 0: foundation

1. Tokens from `tailwind-v4.css` into `src/renderer/tokens.css` (Tailwind v4 `@theme`), light and
   dark. **Follow the system appearance** (`prefers-color-scheme`, kept in sync with Electron's
   `nativeTheme`).
2. Local fonts (above). A `font-display` utility always sets `"SOFT" 100, "WONK" 1`.
3. `src/renderer/lib/motion.ts`: durations, easings and reduced motion, from DESIGN.md.
4. shadcn/ui setup for the renderer (`components.json`, `cn`), plus a strict CSP still passing.

## Part 1: primitives (`src/renderer/components/ui/`)

Button (primary, secondary, ghost, destructive; pill; loading), Switch, StatusPill, Card,
ListGroup/ListRow, Dialog (confirm, Flare header for destructive), Sidebar item, Text styles
(display, title, body, eyebrow, heard), QR frame. Delete the old components as you replace them.

## Part 2: screens and surfaces, in order

1. **Main window** (about 760×540, hidden title bar with traffic lights, vibrancy-free Paper or
   Night background): a sidebar with Home, Phone, Permissions, Settings, Account (the "Phones"
   page from the mockup becomes "Phone", since there's one phone). The status card and the
   last-heard line on Home.
2. **First-run setup** in the same window: Welcome → Permissions (each row ticks live when
   granted) → Pair (big centred QR, "Scan with the Ahoy app on your iPhone", the 5-minute refresh
   with a countdown) → "Now say something on your phone". Every pairing failure state.
3. **Menu bar popover** replacing the native context menu on left-click:
   - A frameless, transparent `BrowserWindow` with macOS vibrancy, about 340px wide, 12px radius,
     positioned under the tray icon, closing when it loses focus.
   - **Right-click keeps a small native menu** (Open Ahoy, Quit Ahoy).
   - The content follows the mockup: wordmark, the "Listen for my phone" switch (= `lanEnabled`),
     the phone card with its status pill, the last command, a problem row only when something is
     wrong (permission missing, helper restarting), and the footer (Open Ahoy, Pair a phone,
     Quit ⌘Q).
   - Every state: no phone paired, connected, not connected, paused, screen locked, permission
     missing, off. (The mockup's "signed out" state becomes "Pair your iPhone to get started"
     with a "Show QR code" button that opens the main window's pair screen.)
4. **Menu bar icon:** a template image of the pennant mark (from the mockup) in every state,
   chosen by shape not colour: connected (solid), no phone (outline), off or locked (outline with
   slash), needs permission (outline with dot). **Remove the menu bar title text** that shows the
   heard words; the popover and overlay show it instead. Update the first-run notification copy,
   which still says "phone icon", to "the Ahoy icon".
5. **Overlay choice tags (Swift helper, last and separate):** restyle the existing marks in
   `overlay.swift` to the slanted tag: Ink fill, -10° skew, 7px radius, white SF Mono numeral
   counter-skewed, a 2px Sailcloth ring plus a soft shadow so they read on light and dark apps.
   **Drawing only:** no new behaviour, no pill (the mockup's pill doesn't exist yet). Say when the
   helper must be rebuilt and Accessibility re-granted.

## Rules

- Copy comes from `agentCopy` (add keys there for new strings; no inline text).
- Tokens only: no raw hex values, no arbitrary values. Inline style only for Motion values.
- Keyboard and accessibility: every control is focusable with a visible focus ring, has labels
  and roles, and the popover closes with Esc.
- Reduced motion: replace movement with a 150ms fade.
- One component per file, files under 200 lines, delete dead UI code.
- A dev-only gallery window (opened from a dev menu item, never in release builds) showing every
  component and state, light and dark.

## How to work

- First reply: the file list, the dependencies, which old components get deleted, and the full
  "Only what exists" mapping. Wait for my OK.
- Then build in order, one commit per part or item on main, no push. After each, 2–3 lines: what
  changed and where to look.
- If the session gets long, stop after a committed item and write `engineering/ui-mac-handoff.md`
  (done, next, gotchas).
- At the end: commits, differences from the mockups and why, anything not built, a check list on
  the Mac, and the rebuild steps (app and helper).
