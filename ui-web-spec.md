# Website: the landing page with the early-access form

Build **one page**: the Ahoy landing page at `/`, with a working early-access (waitlist) form.
Nothing else: no dashboard, no new account pages, no extra marketing pages.

## Inputs (in `~/Developer/Ahoy/design/`)

| File | Use it for |
|---|---|
| `DESIGN.md` | The rules: colours, type, spacing, motion, the waitlist capsule |
| `tailwind-v4.css`, `css-variables.css`, `design-tokens.json` | The tokens; never invent values |
| `Ahoy Web.dc.html` | **The landing page mockup. Follow it** (hero, product shot, pinned statement, How it works, It asks, founder note + trust, dark CTA, footer) |
| `Ahoy Web Capsule.dc.html`, `Ahoy Web States.dc.html` | The waitlist capsule in every state, the link preview image |

- Ignore `Ahoy Web v1/v2` if present, `Ahoy Web Pages.dc.html` (other pages), and every phone and
  Mac file.
- The mockups are HTML: read them for layout, sizes, copy and states, then rebuild them as clean
  components. Where the mockup and DESIGN.md disagree, DESIGN.md wins.

## Scope

- **Only `/`.** Leave the existing `/sign-in`, `/sign-up` and `/account` pages exactly as they are,
  and don't link to them from the landing page.
- **Never change or remove** the auth and API routes (`/api/auth/*`, pairing, transcribe,
  interpret, speak): the phone and Mac depend on them.
- Commit each step on main once typecheck, lint and tests pass. No push.

## Stack (already in `web/`)

- Next.js App Router + Tailwind v4 + **shadcn/ui** ("base-nova", Base UI) restyled to our tokens,
  lucide icons at strokeWidth 1.5.
- **Motion** (`motion` package, imported from `motion/react`) for animation. **Never
  `framer-motion`.** The pinned scroll reveal can use Magic UI's Text Reveal pattern or Motion
  `useScroll`; copy the component in, don't add a big UI package.
- Fonts with `next/font/google`: **Fraunces** with `axes: ["SOFT", "WONK", "opsz"]` (always
  `"SOFT" 100, "WONK" 1`), Inter 400/500, Geist Mono 500.
- New dependencies: list them first and wait for my OK.

## The page (follow the mockup)

1. **Nav:** 56px, the wordmark on the left, a small Ink pill "Join waitlist" on the right that
   scrolls to the hero form. It becomes the floating centred pill after 80px of scroll. **No "Sign
   in" link, no menu.**
2. **Hero (a full screen of its own):** "Talk to *your Mac*, from your phone.", the subtitle, the
   waitlist capsule, and "One email when it's ready. No spam." The headline animates in word by
   word.
3. **Product shot (its own screen):** Version B from the mockup, a Mac window plus the iPhone
   acting out "pause YouTube" → Done, as a short looping animation. Real-looking UI, nothing cut
   off. Leave a clearly marked slot for a future video (Version A) but don't show it.
4. **Pinned statement:** the paragraph pins for about 250vh (200vh on phone), and the words light
   from Fog to Ink as you scroll. "asks." comes last, in Fraunces italic. Then it releases. No
   scroll snapping.
5. **How it works:** three steps, the same layout for each (no zig-zag).
6. **It asks instead of guessing:** the Mac window and the phone side by side with the same
   numbered tags, the phone showing only the "Which one?" sheet.
7. **Founder note + trust strip.**
8. **Dark CTA band** with the dark capsule.
9. **Footer:** the wordmark plus tagline, social icons **only for accounts I give you** (leave them
   out until then), "© 2026 Ahoy", "Made in India", and the giant wordmark. **No links to pages
   that don't exist** (no Privacy, Terms, Support or Changelog yet).

**Responsive, mobile-first:** default = phone (390), `sm` ≥ 640, `lg` ≥ 1024. Under 380px the
capsule stacks the input over a full-width button. No horizontal scroll. Reduced motion: no
pinning and no word animation; the text simply shows fully lit.

**Metadata:** title, description, and the 1200×630 link preview image from the mockup (the orb
**without** a mic button), as a static file or `opengraph-image`.

## The early-access form (works for real)

**Database:** a new Drizzle table `waitlist_signups`:
- `id`
- `email` (lowercased and trimmed, **unique**)
- `created_at`
- `source` (`"hero"` or `"cta"`)

Nothing else: no IP, no user agent, no name. Generate and apply the migration with the existing
`db:generate` / `db:migrate` scripts, and tell me the command.

**Route:** `POST /api/waitlist`
- Validate with zod: email ≤ 254 characters, `source` in the allowed list, plus a hidden honeypot
  field `company` that must be empty. **A filled honeypot gets the normal success reply and saves
  nothing.**
- Rate-limit with the existing `consumeRateLimit` per client IP (for example 5 a minute). Over the
  limit gets a clear error code.
- Insert with "on conflict do nothing". **New and existing emails get the exact same success
  reply**, so the form can't reveal who signed up.
- Reply with the shared API envelope (`SuccessResponse` / `ErrorResponse`) and error codes from the
  existing error setup. **Never log the email** (log only the outcome).
- The request schema lives in the web app (not `shared`) unless you ask me first.

**Capsule states (from the mockup):**
- default, focus
- invalid: "That email doesn't look right." No shake.
- joining: the button says "Joining…" and is disabled. Follow the spinner rule (nothing under
  300ms, at least 500ms once shown).
- success: "You're on the list. We'll write when it's ready." The capsule keeps its size.
- network or server error: one calm line plus a retry, with the typed email kept.
- The light (hero) and dark (CTA) versions share one component.

**Security on the page:** keep or add security headers in `next.config` for production (a CSP
allowing only our own scripts and fonts, HSTS, `frame-ancestors 'none'`, `nosniff`). No
third-party scripts, no analytics, no cookies on this page.

## Rules

- All copy in a copy file (`features/landing/copy/`). No inline text.
- Tokens only: no raw hex values or arbitrary values. One component per file, files under 200
  lines. Server-render everything that doesn't need the browser; client components only for
  motion and the form.
- Accessible: real `<form>` and `<label>`, visible focus rings, `aria-live` on the capsule
  messages, the decorative animation hidden from screen readers.
- Delete any placeholder home page code you replace.

## How to work

- First reply: the file list, the dependencies, the table plus route design, and anything in the
  mockup you can't match. Wait for my OK.
- Then, one commit each on main, no push:
  1. landing layout and sections
  2. motion (hero words, product shot loop, pinned statement)
  3. the waitlist table, route and tests (validation, honeypot, duplicate = same reply, rate limit)
  4. the capsule wired to the route in every state
  5. metadata, OG image and security headers
- End with: commits, differences from the mockup, the migration command, and a check list (phone
  width and desktop, a real signup, a duplicate signup, a bad email, offline).
