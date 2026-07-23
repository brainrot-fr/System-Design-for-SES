# Nearby Discovery (Nearby People)

## The short version

Must-have #2 — same-gender, same-region people-matching. Reuses two things already built: `region_id` from [[feat-notifs]]/[[feat-nearby-events]], and the gender/age-band/interests/display-name fields [[feat-userObject]] said would be collected contextually, not during onboarding. This spec is the trigger point for that contextual flow, plus the data model and query for the feed itself.

Scope note: this is a **display-only match feed** — "show nearby brothers/sisters of similar age/interests," not a chat or contact-exchange system. How two matched people actually connect isn't designed yet (see open questions), same as bahr-e-aam updates were left open in [[feat-notifs]].

## Data model

```sql
create table discovery_profiles (
  user_id           uuid primary key references auth.users(id),
  region_id         uuid not null references regions(id),
  gender            text not null check (gender in ('male','female')),
  age_band          text not null,        -- e.g. '18-24', '25-34'
  interests         text[] not null default '{}',
  display_name      text not null,
  discovery_visible boolean not null default false,
  updated_at        timestamptz not null default now()
);

create index on discovery_profiles (region_id, gender, discovery_visible);
```

One row per user, not folded into the main profile table — this data only exists once someone opts in, and keeping it separate means the RLS policy below only has to reason about one table.

## RLS

```sql
-- you can always see and manage your own row, regardless of visibility
create policy "own discovery profile"
on discovery_profiles for select
using (auth.uid() = user_id);

create policy "insert own discovery profile"
on discovery_profiles for insert
with check (auth.uid() = user_id);

create policy "update own discovery profile"
on discovery_profiles for update
using (auth.uid() = user_id);

-- everyone else's row is visible only if: they opted in, same gender, same region
create policy "browse same-gender same-region discoverable people"
on discovery_profiles for select
using (
  discovery_visible = true
  and gender     = (select gender from discovery_profiles where user_id = auth.uid())
  and region_id  = (select region_id from discovery_profiles where user_id = auth.uid())
);
```

Postgres OR's multiple `select` policies together, so a user's actual visible set is "my own row" ∪ "discoverable same-gender same-region rows" — no separate query path needed for "am I looking at myself."

## The feed

```js
const { data: people } = await supabase
  .from('discovery_profiles')
  .select('user_id, display_name, age_band, interests')
  .eq('region_id', myRegion)
  .eq('gender', myGender)
  .eq('discovery_visible', true)
  .neq('user_id', myUserId);

// interest-overlap ranking done client-side — region-scoped result sets are
// small enough that this doesn't need a DB function yet
const ranked = people
  .map(p => ({ ...p, sharedCount: p.interests.filter(i => myInterests.includes(i)).length }))
  .sort((a, b) => b.sharedCount - a.sharedCount);
```

`age_band` filters to an exact match for now (not a range) — simplest possible v1; loosening to "adjacent bands" is a one-line change later if exact-match feels too narrow in practice.

## Triggering the opt-in

Per [[feat-onboarding]] and [[feat-userObject]]: the first time someone taps into this feature, a short contextual flow collects `display_name`, `gender`, `age_band`, `interests`, and `region_id` (if not already set), then flips `discovery_visible = true`. Turning it back off later is just one `update` — no re-onboarding needed to opt back in afterward, since the row persists.

## Offline behavior

Same rule as [[feat-nearby-events]] and [[feat-dashboard]]: drops out entirely offline rather than showing a stale feed — live data, no local bundling.

## Open questions

- **Connection mechanism**: once someone sees a nearby match, how do they actually reach out — in-app messaging, or just a display name and "ask your Sohbat organizer to introduce you"? Deliberately unspecified — building this without a moderation story first is exactly the kind of thing that goes wrong.
- **Age-band granularity**: exact-match vs adjacent-band — revisit once real usage shows whether exact match feels too restrictive.
- **Region-only matching ceiling**: same as events — no cross-region discovery even for someone traveling. Fine at current scale, worth a second look once regions outside IST/India show up (same flag as [[feat-notifs]]'s open questions).