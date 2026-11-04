# Mobile enhancements

Everything left for the phone app after the bug-fix rounds, from a code read on 2026-11-05.
Tick items off as they ship; add the commit next to each one.

In progress now (prompts handed over): speech pace cleanup, pairing keys this-device-only,
haptics, Android back button.

## Open from earlier work

- [ ] Phase 1 device check (launch from the cached session, commit 8058700)
- [ ] Phase 2 of the launch/session plan: authedFetch with single-flight 401 recovery,
      renew-on-use, user switch, saved Mac list schemaVersion + per-entry parse + write
      queue. Keep `betterAuthPin.test.ts` and the 401 guard test.
- [ ] Reconnect speed: step-0 stage timings, then decide on native crypto for the Noise
      handshake (lives in shared/, ask first)
- [ ] Launch polish check on device: one Ahoy, held ~1 s (commit c64af23)

## Bugs and risks in the code

- [ ] Tests for the riskiest untested areas: auth (session gate, launch probe, install
      wipe), welcome timeline, settings. Auth has 28 files and 0 tests.
- [ ] Remove the four copied `fetchWithOwnCookie` helpers (done by Phase 2's authedFetch)
- [ ] Large text (Dynamic Type): check Home, Settings, Pair and the sheets at the largest
      sizes; font scaling is capped in places (carousel `NO_FONT_SCALING`)

## Improvements to existing screens

- [ ] Talk button: slide away to cancel while holding
- [ ] Talk button: lock for hands-free longer commands
- [ ] Talk button: tap-to-talk option for people who can't hold
- [ ] Home: "do it again" chips for the last two or three commands
- [ ] Settings: app version and build
- [ ] Settings: Help / Send feedback
- [ ] Settings: privacy policy and terms links
- [ ] Settings: Smart commands switch (decisions.md 2026-10-28)
- [ ] Pairing: make the Mac's QR a universal link, so the iPhone Camera app opens Ahoy on
      the pair screen (today it holds JSON)
- [ ] Mac card: rename a Mac from the phone
- [ ] Android parity pass: edge-to-edge (RN 0.88), microphone/camera denied flows on real
      devices, voice sheet, launch handoff

## Needed before beta / App Store

- [ ] Sign in with Apple (Google is offered, so Apple requires an equivalent)
- [ ] Delete account in the app (App Store rule)
- [ ] First-run onboarding: hold to talk, pairing, and the cloud disclosure (ElevenLabs
      hears the voice, Anthropic reads the words), required by decisions.md 2026-10-28
- [ ] History screen (designed: `history` / `historyOpen` in design/Ahoy iPhone.dc.html)
- [ ] Crash reporting (Sentry for Expo), transcripts and labels redacted
- [ ] EAS build + TestFlight + over-the-air updates (`expo-updates`); no eas.json yet
- [ ] iOS privacy manifest (`ios.privacyManifests`) and App Privacy labels

## Future

- [ ] Talk without opening the app: Action Button, Control Center button, Lock Screen
      widget, Siri Shortcut ("Ask Ahoy"), Home Screen quick action ("Talk to my Mac")
- [ ] Streaming transcription (ElevenLabs scribe_v2_realtime): words appear while speaking
- [ ] Contacts lookup for email drafts (plan: contacts-lookup-plan.md)
- [ ] Hindi / Hinglish (copy already lives in copy/*.ts files)
- [ ] More voices and a speaking-speed setting
- [ ] Apple Watch hold to talk
- [ ] iPad support (iPhone only today)
