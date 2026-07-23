`The whole On-Boarding process is done and described here`

## Steps and Information Collected

**Mandatory:**
1. Language Selection
2. Murshid

**Optional:**
1. Location-based updates
2. Sign-in/Sign-up to backup data

This stays the shortest possible path to a usable app. Language Selection is also what satisfies the app's "Must have Multilingual Support" requirement — every content feature just respects whatever gets picked here; not a separate feature unless something like RTL layout or partial-translation fallback turns out to need real design later.

## Step 0 — the identity nobody sees

Before language selection even renders, an anonymous Supabase identity gets created silently in the background (per [[feat-userObject]]). Nothing changes about what the user experiences — they still just see "pick a language" first. This is what region, Murshid choice, and everything downstream attaches to.

## The discovery/gender-aware fields — deferred, not front-loaded

Display name, gender, age band, interests are **not** collected during main onboarding. They live in a small, contextual flow that only appears the first time someone opts into something that actually needs them — currently [[feat-nearby-discovery]], and potentially other gender-aware features (e.g. gender-scoped event browsing) as they come up. Confirmed: this stays purely opt-in, tied to Must-haves #1 /#2 (nearby people, nearby events), not part of the guaranteed onboarding path.

That contextual flow asks for exactly what's needed in one short pass, and also asks for a region if one was never set during onboarding (since discovery/events can't function without it).

## How this connects

- Region reuse holds — same `region_id` whether set during onboarding or during a later contextual flow
- Uninstall-wipes-everything policy (per [[feat-userObject]]) applies here too — skip backup, skip a contextual flow, lose both on uninstall
- No new schema beyond what [[feat-userObject]] and [[feat-nearby-discovery]] already specify — this doc is about *when/where* fields get asked for, not new columns