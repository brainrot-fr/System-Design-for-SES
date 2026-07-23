# User Object

## The short version

Every other feature quietly assumes a "user" exists — the Murshid chosen, the region prayer times/events revolve around, the tokens push notifications land on, whether strangers can see you in nearby-discovery. This feature defines what a "user" is in SES, given that [[feat-onboarding]] deliberately makes sign-in/sign-up optional.

## Identity, two layers

**Anonymous by default.** On first launch, before language or Murshid is chosen, the app quietly creates an anonymous identity via Supabase Auth's anonymous sign-in — no email, no password, no UI shown. Device tokens, region, and Murshid selection all attach to this identity.

**Upgradeable, never mandatory.** When someone takes the optional "back up my data" step, that anonymous identity gets linked to a real email/phone — same identity underneath, now portable across reinstalls. Skip that step, and the identity lives and dies with the install.

**Uninstalling without backup wipes everything.** Accepted trade-off — someone who skipped backup either isn't deeply invested in cross-device continuity or isn't familiar with the concept. Reinstalling just means walking through onboarding again.

## What a profile holds

- **Language** — mandatory, from onboarding
- **Murshid** — mandatory, from onboarding; a stable reference/slug, not a copy of murshid content (bundled separately per [[feat-murshidObject]])
- **Region** — optional, only if location-based updates opted into; same `region` concept anchored in [[feat-notifs]]
- **Discovery visibility** — off by default, opt-in only
- **Gender, age band, interests, display name** — needed for nearby-people matching, collected contextually the first time discovery (or another gender-aware feature) is touched, not during main onboarding — see [[feat-onboarding]]
- **Device token(s)** — for push, tied 1:many to the identity

## Local vs. cloud

- **On-device, always present** — source of truth for the reading/offline experience.
- **Mirrored server-side, when online** — a thin row keyed to the identity: region, Murshid slug, discovery flag, gender/age/interests once collected. No religious content lives here.

## Content storage architecture

Content (Quran, Hadees, Nuqool, Murshid/silsila data) gets its own server-side rows too, but **stays in the same Supabase project as user data**, isolated via a separate Postgres schema (`content`) with its own RLS policies rather than a second project. Reasoning:
- Free tier caps at 2 projects total across the whole org — spending both on user-data vs. content leaves nothing for staging/future services
- Two projects means two independent 7-day pause timers to babysit instead of one
- Today, content is fully bundled on-device — there's no live traffic to isolate yet; a schema split gives the same "a leak in one can't touch the other" guarantee without the cost

**Live content updates are a real future goal** (confirmed: not just app-update-only) — so the `content` schema should be designed so it *could* eventually get its own RLS-gated live read path without a rewrite. Not building that path now — just not architecting it out of reach either. Revisit as its own ADR once actual live-update needs materialize.

## How this feeds the rest of the app

- [[feat-notifs]] — device tokens trace to this identity; region drives prayer-time targeting
- [[feat-murshidObject]] — the Murshid chosen here is just the pointer
- [[feat-nearby-discovery]], [[feat-nearby-events]], [[feat-volunteer-matching]] — region, gender, discovery/volunteer flags all live here
- [[feat-dashboard]] — reads the on-device copy to decide what to render

## Open questions

- Does a Murshid ever need more than a slug server-side, or is a plain reference enough forever? Only matters once bahr-e-aam/urs (feat-notifs Part 3) gets designed.