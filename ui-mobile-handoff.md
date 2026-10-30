# Mobile UI build: handoff

Spec: `engineering/ui-mobile-spec.md`. Work is committed straight to `main` in `mobile`
(not pushed). Order for the rest: 2.3 home + talk, 2.4 Mac states, 2.5 sheets, 2.6 pairing,
2.7 settings, 2.8 error fallback and leftovers. Commit each step on `main` after typecheck,
lint and tests pass.

## Done (committed)

| Commit | What |
|---|---|
| `feat: add ahoy tokens, fonts and motion foundation` | Part 0: tokens in `src/global.css` (light + dark), static fonts from `scripts/buildFonts.py`, splash wordmark PNGs, `src/lib/{cn,motion,haptics,useSpinnerVisible}.ts` |
| `feat: add ahoy ui primitives` | Part 1: `src/components/ui/*` (text, button, input, card, switch, statusPill, statusDot, choiceTag, sheet + drainBar/backdrop/background, listGroup, listRow, banner, icon, orb), old screens moved onto them |
| `feat: add dev component gallery behind the dev menu` | `/dev/components`, opened from the Expo dev menu ("Component gallery"), `Stack.Protected guard={__DEV__}` |
| `feat: add the welcome screen with the opening animation` | 2.1: `src/features/welcome/`, splash hold (`useSplashHold`, max 3 s), signed-out redirect to `/welcome` |
| `feat: restyle sign in and add the signing in screen` | 2.2: pennant (`ui/pennant`, PNG mask from `scripts/buildBrandAssets.py`), auth layout, errors split (credential under fields, Google under buttons), `SigningIn` overlay after the Google return (Cancel after 6 s), cold-start gate `useSessionGate` (splash ≤ 3 s, signing-in after, gives up at 6 s). Shared `authCopy` gained `session.signingIn/slow/cancel` (shared commit `2834572`) |

| `feat: rebuild home and talk with mac states` | 2.3 + 2.4 in one commit (the old cards owned the connections, so they could not go separately). `MacConnections` (in `LiveMacLinksProvider`) keeps every paired Mac connected; Home calls `useKeepMacsConnected`. Selected Mac in MMKV (`storage/selectedMac.ts`); `useTalkTarget` = selected → connected → first. `macStatus.ts` maps every state to chip label, dot tone and message (ended reason bug fixed). `HomeMac` = chip + P1/P2/P3 card; `TalkPanel` = Heard line (30 px), `ReplyCard`, `TalkBar` (76 px, orb 300×220). Pure view logic in `talkView.ts` (tested) and `talkBarView.ts` |

| `feat: replace alerts with sheets and rebuild settings` | 2.5 + 2.7: `ui/confirmSheet`, `ui/optionRow`, More options with "Talk to", Which one? with expired state, Settings (Macs, Voice replies sheets, account, sign out). `ui/sheet` now renders through the root `Portal` |
| `feat: restyle pairing and the error fallback` | 2.6 + 2.8: pairing screens, error fallback, legacy shadcn aliases removed |

**The phone UI build in the spec is complete.** Left for later: network banner wiring
(`design/handoff/network-state.md`), duration on the reply eyebrow, Android check.

## Earlier plan for 2.5–2.8 (built)

- **2.5:** replace the `Alert.alert` menu in `pairing/hooks/useMacCardActions.ts` (`openMenu`) and
  in `useUnpairMac` / `useConfirmSignOut` with `ui/sheet`. More options: title = Mac name,
  status line, with 2+ Macs a "Talk to" group first (`macCopy.talkTo`; one row per Mac with
  its chip word, Signal checkmark on the selected one, tap → `useSelectedMac().select`), then
  Connect/Disconnect (Ink), Remove this Mac (outline, Flare text), Cancel. Remove and sign-out
  confirms use `tone="danger"`. Restyle `talk/components/choiceSheet.tsx` + `choiceButtons.tsx`
  on `ui/sheet` (drain bar over a 2 px Rope track, rows 64 px, "None of these" + "or say a
  number", expired state: rows at 40 % and not tappable, title "That screen changed. Ask
  again.", Close). Add the 2-Mac sheet to the gallery.
- **2.7:** Settings must bring back **Add another Mac** (removed from Home in 2.3, it is only
  in the mockup's Settings now) and list every paired Mac.
- Reply eyebrow has no duration ("Done · 0.8s"): the answer time is not stored; add it only
  if the thread keeps a received time.
- `src/features/pairing/macStatus.test.ts` was deleted on disk after this step; I left it out.

## Notes gathered for 2.3 (built)

Mockup (`Ahoy iPhone.dc.html`, `vHomeB`):
- Header row 56 px, padding 16 left / 10 right. Left: Mac chip (h-9 pill, Paper, Rope border,
  status dot, Mac name Inter 15/500, chip label Inter 13 Driftwood, chevron down) opens More
  options. Right: one 44 px account/settings button (no History button).
- Mac message card (P1/P2/P3) under the header: Paper, radius 24, padding 16/20, message
  Inter 15 Driftwood, actions h-10 pills (primary Ink or outline Rope).
- Middle (flex, centred, gap 20, px 6): Heard line Fraunces italic 400 **30 px** Driftwood
  centred (DESIGN.md says 24; mockup 1b uses 30), then the reply card: Paper, radius 24,
  padding 16/20, eyebrow Geist Mono 11 in the status colour, message Inter 17, optional
  follow-up line Inter 13 Driftwood, "Try again" outline pill h-9 on the right when retryable.
- Bottom (px 20, pb 42, gap 14): disabled reason Inter 13 Driftwood centred, then the talk
  bar: full width, **76 px** pill, Ink (Linen + Fog when disabled), mic icon + label Inter
  17/500 ("Hold to talk" / "Understanding…" / "Sending to …"), Geist Mono 15 timer while
  listening, scale 1.02 while held, orb 300×220 behind it while listening/understanding.
- Status labels: Done (Kelp, with " · 0.8s" from `sentAt`), Needs your choice (Signal),
  Can’t do that (= `refused`, Driftwood), Failed (Flare).
- Connection ownership: each `LiveMacCard` calls `useMacConnection`, which starts and stops
  that Mac's connection. A home that shows only one chip must still mount a connection per
  paired Mac, or the others stop connecting. Waiting on the user's answer for 2+ Macs.

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
- After adding a token name (text size, font, radius, spacing) also add it to
  `src/lib/cn.ts`. Fraction classes like `top-21/50` and half steps like `bottom-42.5`
  compile; check new ones in an `expo export --no-bytecode` bundle.
- Rebuilding brand assets rewrites the font files (timestamps only); `git checkout -- assets/fonts`
  afterwards.
- `authCopy["form.pending"]` and `["session.loading"]` are no longer used by mobile
  (buttons show the spinner rule instead); they stay in shared for web.
- Sheet details from the mockup still to apply in 2.5: a 2 px Rope track under the Signal
  drain bar, title padding 16/20/8, rows 64 px with a 14 px gap.
