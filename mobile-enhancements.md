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
- [ ] Launch polish check on device: one Ahoy, held ~1 s (commit c64af23)

## Bugs and risks in the code

- [x] **"Can't reach" after the Mac's IP changes (top priority).** The phone saves the
      Mac's LAN addresses once, from the QR at pairing, and never updates them; the Mac gets
      a new IP from the router (new day, restart) and the phone dials the old one.
      Fixed with the .local hostname and refreshing addresses after every handshake
      (shared 3dc7882, agent e52f7b8, mobile ac119d3 + bb09757). Bonjour browsing
      (_ahoy._tcp, matched by key fingerprint) still open. Device check pending.
- [ ] Tests for the riskiest untested areas: auth (session gate, launch probe, install
      wipe), welcome timeline, settings. Auth has 28 files and 0 tests.
- [ ] Remove the four copied `fetchWithOwnCookie` helpers (done by Phase 2's authedFetch)
- [x] Backgrounding during a send loses the result: interrupt() only cancels while
      recording, but iOS closes the socket in the background while the Mac still runs the
      command, so the phone showed "No answer" and History marked a command that worked as
      failed. Fixed with command.lookup on return (shared aebcbb5, agent 92ba9e9, mobile
      a125f7f + f938593). Device check pending.
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
      hears the voice, Anthropic reads the words), required by decisions.md 2026-10-28.
      The disclosure carries the history line: "Ahoy keeps a history of what you said and
      what it did in your account for 90 days. You can clear it or turn it off in
      Settings." Speech audio is never stored.
- [ ] Privacy policy page (none exists yet): include the same history line
- [x] History screen (Settings → History)
- [ ] Crash reporting (Sentry for Expo), transcripts and labels redacted
- [ ] EAS build + TestFlight + over-the-air updates (`expo-updates`); no eas.json yet
- [ ] iOS privacy manifest (`ios.privacyManifests`) and App Privacy labels

## Future

Moved to mobile-future-enhancements.md.
