# Volunteer-Matching

## The short version

Two directions, both region-scoped, both staying in the same "browse pools, no signups" shape as [[feat-nearby-events]] (browse-only) and [[feat-nearby-discovery]] (card-only, no messaging) — not a third interaction pattern.

## Direction 1 — volunteers browsing events

No new table — reuses [[feat-nearby-events]]'s feed with one extra filter:

```js
const { data } = await supabase
  .from('public_events')
  .select('*')
  .eq('region_id', myRegion)
  .eq('needs_volunteers', true)
  .in('gender_scope', myGender ? ['all', myGender] : ['all'])
  .gte('starts_at', new Date().toISOString())
  .order('starts_at');
```

Reaching out about a specific event uses `event_hosts.contact_number`, already public per the eventObject design — no new contact mechanism needed for this direction.

## Direction 2 — organizers browsing volunteers

```sql
alter table profiles add column willing_volunteer boolean not null default false;
alter table profiles add column volunteer_contact text; -- only ever exposed via the view below

create view willing_volunteers as
select id, display_name, region_id, interests, volunteer_contact
from profiles
where willing_volunteer = true;
```

**Who can see this pool (confirmed):** anyone who's created an event — not just `is_organizer`-flagged accounts. A student running a study circle can browse it too.

```sql
create policy "event creators see regional volunteers"
on willing_volunteers for select
using (
  region_id = (select region_id from profiles where id = auth.uid())
  and exists (select 1 from events where created_by = auth.uid())
);
```

**Contact mechanism (confirmed):** a volunteer who opts in provides `volunteer_contact` specifically for this — not reused from anywhere else on their profile — and it's visible only to eligible event-creators per the RLS policy above. Not exposed publicly, not exposed until opted in.

## v1 scope

Browse-only, same as events and discovery. No in-app signup/application flow, no messaging. Coordination happens off-platform once someone's found in either direction.