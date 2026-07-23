# Nearby Events

## The short version

Two needs live under one feature because they turned out to share almost everything: Must-have #1 ("show nearby happening/upcoming events") and the volunteer hooks from Should-haves #1/#2 ("show organizers their volunteers" / "show volunteers nearby events to help with"). Rather than a separate events table and a separate volunteer-events table, one `public_events` row carries a `needs_volunteers` flag — a hook, not the volunteer-matching logic itself. The actual "who signed up to help" mechanics are deliberately left to [[feat-volunteer-matching]] (not yet designed) so this spec stays scoped to "can I see what's happening near me, and can an organizer post something."

## What the user sees

- A feed of upcoming events, filtered to their region and (if set) their gender scope.
- Each event: title, description, when, where (name + optional map pin), and — if `needs_volunteers` is true — a "looking for volunteers" badge.
- [[feat-dashboard]] already shows a 2–3 card preview of this feed ("Happening near you"); this spec is the full "see all" feed plus the write path organizers use to create events.

## Data model

Reuses `regions` from [[feat-notifs]] — same "where is this user" concept, not solved twice.

```sql
create type event_gender_scope as enum ('all', 'male', 'female');

create table public_events (
  id               uuid primary key default gen_random_uuid(),
  organizer_id     uuid not null references auth.users(id),
  region_id        uuid not null references regions(id),
  title            text not null,
  description      text,
  gender_scope     event_gender_scope not null default 'all',
  starts_at        timestamptz not null,
  ends_at          timestamptz,
  location_name    text,
  location_lat     double precision,
  location_lng     double precision,
  needs_volunteers boolean not null default false, -- hook for feat-volunteer-matching; no matching logic lives here
  created_at       timestamptz not null default now()
);

create index on public_events (region_id, starts_at);
```

One table, not two — an event either needs volunteers or it doesn't; making that a column instead of a parallel schema means the region+gender+time query below never has to check two tables to answer "what's happening near me."

## RLS

```sql
create policy "events are readable by anyone"
on public_events for select
using (true);

create policy "anyone can create an event under their own identity"
on public_events for insert
with check (auth.uid() = organizer_id);

create policy "organizers can edit their own events"
on public_events for update
using (auth.uid() = organizer_id);

create policy "organizers can delete their own events"
on public_events for delete
using (auth.uid() = organizer_id);
```

No approval/moderation gate on creation for v1 — any identity, anonymous or upgraded (per [[feat-userObject]]), can post an event under their own `organizer_id`. Flagged below since it's the one thing here that gets harder to bolt on later.

## The feed query

Same shape [[feat-dashboard]] already previews, just without the `.limit(3)`:

```js
const { data: events } = await supabase
  .from('public_events')
  .select('*')
  .eq('region_id', myRegion)
  .in('gender_scope', myGender ? ['all', myGender] : ['all'])
  .gte('starts_at', new Date().toISOString())
  .order('starts_at');
```

`myGender` only exists once the contextual gender-aware flow ([[feat-onboarding]]) has run. If it hasn't, the `.in()` clause quietly falls back to `['all']` — someone who's never touched a gender-scoped feature just sees the gender-neutral events, no forced prompt. The prompt only fires the moment they tap into a gender-scoped section — a female-only event here, or [[feat-nearby-discovery]] — same contextual pattern onboarding already established, just triggered from a second entry point now.

## Event reminders

Must-have #3 covers this: "upcoming event updates via feat-notifs." Same ticker shape as the prayer-time dispatcher in [[feat-notifs]] Part 2 — ask "what's due in the next N minutes," not one cron job per event:

```sql
select cron.schedule(
  'dispatch-event-reminders',
  '*/10 * * * *',
  $$ select net.http_post(url := '.../functions/v1/dispatch-event-reminders') $$
);
```

```js
// supabase/functions/dispatch-event-reminders/index.js
Deno.serve(async () => {
  const supabase = adminClient();
  const windowStart = new Date(Date.now() + 55 * 60_000); // reminder fires ~1hr before start
  const windowEnd   = new Date(Date.now() + 65 * 60_000);

  const { data: due } = await supabase
    .from('public_events')
    .select('id, title, region_id, gender_scope')
    .gte('starts_at', windowStart.toISOString())
    .lt('starts_at', windowEnd.toISOString());

  for (const event of due ?? []) {
    const tokens = await tokensForRegionAndGender(supabase, event.region_id, event.gender_scope);
    await sendToTokens(tokens, `Starting soon: ${event.title}`, '');
  }
  return new Response('ok');
});
```

`tokensForRegionAndGender` is one filter away from the existing `tokensForRegion` helper in [[feat-notifs]] — add a gender check on top of the region join, same `device_tokens → profiles` path, reusing `sendToTokens` from the shared plumbing.

One-hour-before is a starting assumption, not a locked number — easy to make configurable per event later (organizer picks 1hr vs 1 day) without touching the ticker's shape.

## Offline behavior

Matches [[feat-dashboard]]'s existing rule: the events feed drops out entirely when offline rather than showing stale cached events — an event you're viewing offline could already be moved or cancelled. No local bundling here — unlike Quran/Hadees/Nuqool, events are inherently live data, not static content.

## Cost

Same free-tier plumbing as [[feat-notifs]]: one more pg_cron job, one more Edge Function, both comfortably inside Supabase's free allowance. No new infra category introduced.

## Open questions

- **Moderation**: nothing stops spam/duplicate/bad-faith event posts right now. Fine at "team size 1, early days" scale; revisit once organizers outside a trusted circle start posting.
- **Reminder timing**: fixed 1-hour-before for now — organizer-configurable, or even user-configurable, later?
- **Handoff to feat-volunteer-matching**: `needs_volunteers` is the only thing this spec commits to. The actual "show organizers their volunteers" / "show volunteers nearby events" flows (Should-haves #1/#2) are still fully unspecified — next after [[feat-nearby-discovery]] per the build order.