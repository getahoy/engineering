# Mobile UI build: handoff

Spec: `engineering/ui-mobile-spec.md`. Branch: `mobile` on `feature/ahoy-design-system`
(not pushed, no PR yet).

## Done (committed)

| Commit | What |
|---|---|
| `feat: add ahoy tokens, fonts and motion foundation` | Part 0: tokens in `src/global.css` (light + dark), static fonts from `scripts/buildFonts.py`, splash wordmark PNGs, `src/lib/{cn,motion,haptics,useSpinnerVisible}.ts` |
| `feat: add ahoy ui primitives` | Part 1: `src/components/ui/*` (text, button, input, card, switch, statusPill, statusDot, choiceTag, sheet + drainBar/backdrop/background, listGroup, listRow, banner, icon, orb), old screens moved onto them |
| `feat: add dev component gallery behind the dev menu` | `/dev/components`, opened from the Expo dev menu ("Component gallery"), `Stack.Protected guard={__DEV__}` |
| `feat: add the welcome screen with the opening animation` | 2.1: `src/features/welcome/`, splash hold (`useSplashHold`, max 3 s), signed-out redirect to `/welcome` |

## Next, in the spec's order

2.2 sign in / create account, 2.3 home + talk (layout 1b), 2.4 Mac card and chip states,
2.5 sheets, 2.6 pairing, 2.8 settings (with voice replies), 2.9 signing-in screen.
One commit per item. The copy-to-pattern mapping from the first reply of this session still
holds (see `design/handoff/phone-states.md` plus the mismatches it lists).

Agreed answers to carry forward:
- Signing-in copy goes into shared `authCopy` as `session.signingIn`, `session.slow`,
  `session.cancel`; rebuild `shared`, then `pnpm install` in each app. Cancel only after the
  Google return. Cold start: over 6 s, continue with the cached session (or sign-in).
- No local network priming, Mac checklist or accessibility sheet in pairing.
- `voiceCopy.playsOn` stays "Play replies on". New voice help lines and footer go in
  `voiceCopy`.
- Network banner: gallery only until `design/handoff/network-state.md` ships.
- Expired choice: rows faded to 40 %, not tappable, nothing sent.

## Gotchas

- **Expo native versions:** `npx expo install` can pull a module built against a newer
  `expo-modules-core` than the SDK ships, and the app then dies at launch with
  "Symbol not found". `expo-font` is pinned to `58.0.9`. Check the lockfile resolves to
  versions released with `expo-modules-core@58.0.23` before adding any native module.
- **Uniwind drops `radial-gradient()`:** the orb passes `backgroundImage` inline (see
  `decisions.md`). Unused theme vars are pruned, so values read with `useCSSVariable` go in
  `@theme static`.
- **`cn` must know custom tokens:** add every new text size, font, radius or spacing name to
  `src/lib/cn.ts`, or `twMerge` silently drops a class.
- **Legacy shadcn colour names** (`bg-background`, `text-muted-foreground`, …) are aliases
  at the end of `global.css`. Delete each alias when its last screen is rebuilt.
- **Typed routes:** after adding a route, run `npx expo start` once so `.expo/types`
  regenerates, or `tsc` rejects the new path.
- **Not verified on device yet:** the welcome orb rise and buttons (the dev menu covered
  them on the simulator), Android rendering (no emulator on this Mac).
- **`design/DESIGN.md`** still describes the old welcome ("Talk to your Mac.", 44 px).
  Built from the Screens mockup instead, as agreed.
- **`mobile` has no `.githooks/` and no `prepare` script**, which CLAUDE.md requires. Not
  fixed here; it needs its own change.
- Sheet details from the mockup still to apply in 2.5: a 2 px Rope track under the Signal
  drain bar, title padding 16/20/8, rows 64 px with a 14 px gap.
