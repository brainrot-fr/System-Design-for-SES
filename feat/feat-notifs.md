# Notifications

## The short version

Two independent notification systems live under this one feature, because they behave completely differently:

1. **Daily wisdom** — one Ayah, one Hadees, one Naql, picked at random once a day, plus a fixed Ayah at 11 AM that never changes. Same content, sent to every user, at the same clock time, every day.
2. **Prayer reminders** — Fajr, Dhuhr, Asr, Maghrib, Isha. These can't use a fixed clock time at all — they follow the sun, and the sun doesn't run on a schedule. Every user gets these at the _correct_ time for wherever they actually are.

Both are delivered the same way underneath: a Postgres cron job wakes up, an Edge Function decides what to send, and Firebase Cloud Messaging pushes it to the phone — even if the app hasn't been opened in weeks. There's no client-side scheduling anywhere in this design. The app's only job is to hand over a device token once, then wait.

---

## Part 1 — Daily Wisdom (Ayah, Hadees, Naql)

### What the user sees

|Time (IST)|Notification|Rotates daily?|
|---|---|---|
|5:00 AM|Ayah of the Day|Yes — random pick|
|7:00 AM|Hadees of the Day|Yes — random pick|
|9:00 AM|Naql of the Day|Yes — random pick|
|11:00 AM|A fixed Ayah|No — same one, every day, forever|

That's 4 pushes a day, sent to _everyone_ — no per-user personalization, and no prayer-time coupling. An earlier version of this design tied the 5 AM Ayah to Fajr and the 5 PM Hadees to Asr; we deliberately pulled those apart. Fajr's time is about to become something that moves every day (see Part 2), and there's no reason a Quran verse should have to move with it.

### The idea: pick once, read many times

The old approach picked a random row from the whole corpus _at the moment each notification fired_. That works, but "no repeats this week" or "make today's three picks feel connected" would each need to be solved separately, at every send site.

Instead: once a day, a cron job reaches into the full corpus, makes exactly one pick per type, and writes it down. Every notification that day just _reads what was already decided_ — nothing at send time does any picking. Any smarter logic later (avoid repeats, weight by length, whatever) only has to be written in one place.

### The trap: UTC midnight is not our midnight

This is the one part of this system that will quietly break if it isn't built exactly like this.

Postgres, by default, thinks in UTC. But **IST is UTC + 5:30** — and that offset is the trap: UTC's midnight lands at 5:30 AM IST, right in the middle of our morning notifications, between the 5 AM and 7 AM sends. If the daily pick is keyed off the database's default idea of "today," the 5 AM send and the 7 AM send could each see a _different_ "today" — and one of them finds nothing waiting for it.

The fix: never let the database's default clock decide the date. Always ask explicitly for the IST date:

```sql
(now() at time zone 'Asia/Kolkata')::date
```

Everything below uses this — the picker writes with it, the sender reads with it.

### Data model

Two tables, doing two very different jobs:

```sql
-- The full corpus. Every Ayah/Hadees/Naql lives here, mirrored into
-- Postgres purely so a server-side job can pick from it — the in-app
-- reading experience still uses its own locally bundled copy.
create type notif_type as enum ('ayah', 'hadees', 'naql');

create table notification_content (
  id    uuid primary key default gen_random_uuid(),
  type  notif_type not null,
  title text not null,
  body  text not null,
  ref   text  -- optional, e.g. "51:56" or a hadees number — your own bookkeeping, never shown to the user
);

-- Today's pick. One row per day, wide on purpose: a "day" is naturally
-- one row with three columns, not three rows that need joining.
create table daily_content (
  date         date primary key,  -- always the IST date, see above
  ayah_title   text not null,
  ayah_body    text not null,
  hadees_title text not null,
  hadees_body  text not null,
  naql_title   text not null,
  naql_body    text not null
);

-- The 11 AM Ayah. Its own tiny table instead of a hardcoded string in
-- code, so it can be changed later without a redeploy — but it's one
-- row, and it never rotates.
create table fixed_content (
  slot  text primary key,  -- 'ayah_11am'
  title text not null,
  body  text not null
);
```

### The jobs

**Job 1 — pick today's content**, once a day, well before the first send:

```sql
select cron.schedule(
  'pick-daily-content',
  '0 21 * * *',  -- 21:00 UTC = 2:30 AM IST — safely before the 5 AM send
  $$
  insert into daily_content (date, ayah_title, ayah_body, hadees_title, hadees_body, naql_title, naql_body)
  select
    (now() at time zone 'Asia/Kolkata')::date,
    a.title, a.body, h.title, h.body, n.title, n.body
  from
    (select title, body from notification_content where type = 'ayah'   order by random() limit 1) a,
    (select title, body from notification_content where type = 'hadees' order by random() limit 1) h,
    (select title, body from notification_content where type = 'naql'   order by random() limit 1) n
  on conflict (date) do nothing;
  $$
);
```

**Job 2 — send**, one job per time slot, all pointing at the same Edge Function with a different `slot` in the request body:

|IST time|UTC cron|Cron expression|slot|
|---|---|---|---|
|5:00 AM|23:30 (prev. day)|`30 23 * * *`|`ayah`|
|7:00 AM|01:30|`30 1 * * *`|`hadees`|
|9:00 AM|03:30|`30 3 * * *`|`naql`|
|11:00 AM|05:30|`30 5 * * *`|`ayah_11am`|

```js
// supabase/functions/send-content/index.js
Deno.serve(async (req) => {
  const { slot } = await req.json();
  const supabase = adminClient();
  const today = new Intl.DateTimeFormat('en-CA', { timeZone: 'Asia/Kolkata' }).format(new Date());

  let title, body;

  if (slot === 'ayah_11am') {
    const { data } = await supabase.from('fixed_content').select('title, body').eq('slot', slot).single();
    ({ title, body } = data);
  } else {
    const { data } = await supabase.from('daily_content').select('*').eq('date', today).single();
    title = data[`${slot}_title`];
    body  = data[`${slot}_body`];
  }

  await sendToAllTokens(supabase, title, body); // shared helper, see "Shared plumbing" below
  return new Response('ok');
});
```

No `order by random()` anywhere near send time anymore — just a lookup.

---

## Part 2 — Prayer Time Reminders

### What the user sees

Fajr, Dhuhr, Asr, Maghrib, Isha — one push each, at the moment that prayer actually begins, wherever that user happens to be.

### Why this can't be a fixed clock time

Every one of these five moments is defined by where the sun is, not by a clock: Fajr by first light, Dhuhr by the sun crossing its highest point, Asr by a shadow reaching a certain length, Maghrib by sunset, Isha by full dark. None of that holds still — it shifts a little every day and swings by well over an hour across the year. Hardcoding "Fajr = 5:00 AM" would be right for a few weeks and quietly wrong the rest of the year.

The fix: don't compute this once and forget it — recompute it daily, for wherever the user actually is.

### How it works, step by step

1. Once a day, for every region that has at least one user in it, ask a prayer-times service for tomorrow's five times.
2. Store those as plain, absolute timestamps — no timezone math needed at read time, no cron expression to maintain per region.
3. A small job wakes up every 10 minutes, checks "is anything due right now," and if so, sends only to the users in that region.

Checking on some interval is unavoidable — there's no built-in way to ask Postgres to fire at one exact arbitrary instant without building a small job-queue system on top of pg_cron, which is more moving parts than this needs. So the real question is just how coarse that interval can be. 10 minutes means a prayer notification is never more than 10 minutes late, and costs about 144 Edge Function invocations a day — a rounding error against Supabase's free-tier allowance no matter what interval is picked. Going to hourly instead is a one-line change (`'0 * * * *'`) if fewer jobs matters more than timeliness — but it means a notification could land up to 59 minutes after the prayer actually started.

This deliberately avoids one cron job per region per prayer — that turns into a growing pile of scheduled jobs to manage (and clean up) as new cities join. A fixed pair of jobs that scales through table rows instead of more cron jobs stays maintainable no matter how many regions get added later.

### Data model

```sql
-- Shared with the nearby-events/nearby-people feature — one canonical
-- "where is this user" concept, reused here instead of solved twice.
create table regions (
  id      uuid primary key default gen_random_uuid(),
  city    text not null,
  country text not null,
  lat     double precision not null,
  lng     double precision not null,
  unique (city, country)
);

alter table profiles add column region_id uuid references regions(id);

-- Rewritten daily. One row per prayer per region, always for "today."
create table region_prayer_times (
  region_id uuid references regions(id),
  prayer    text not null,        -- fajr | dhuhr | asr | maghrib | isha
  fires_at  timestamptz not null, -- absolute instant — nothing to convert when reading this
  primary key (region_id, prayer)
);
```

**One change this forces on the existing schema:** `device_tokens` used to be an anonymous list — fine when every push went to everyone. Now that prayer pushes are region-specific, a token has to be traceable back to a user, and from there to a region:

```sql
alter table device_tokens add column user_id uuid not null references auth.users(id);
```

### The jobs

**Job 1 — compute tomorrow's times**, once a day, comfortably before the earliest Fajr anywhere this applies:

```sql
select cron.schedule(
  'compute-region-times',
  '0 12 * * *',  -- 12:00 UTC = 5:30 PM IST — a full overnight buffer before the next Fajr
  $$ select net.http_post(url := '.../functions/v1/compute-region-times') $$
);
```

```js
// For every active region, ask Aladhan for today's timings and store
// them as absolute instants. School defaults to Hanafi (school=1) —
// confirm that's right for the community rather than leaving it
// unset, since it changes the Asr calculation specifically.
for (const region of activeRegions) {
  const res = await fetch(
    `https://api.aladhan.com/v1/timingsByCity?city=${region.city}&country=${region.country}&method=1&school=1`
  );
  const timings = (await res.json()).data.timings;
  for (const prayer of ['Fajr', 'Dhuhr', 'Asr', 'Maghrib', 'Isha']) {
    await supabase.from('region_prayer_times').upsert({
      region_id: region.id,
      prayer: prayer.toLowerCase(),
      fires_at: toUtcTimestamp(timings[prayer], region), // combine today's date + the returned HH:mm
    });
  }
}
```

**Job 2 — the ticker**, checking every 10 minutes for anything due:

```sql
select cron.schedule(
  'dispatch-prayer',
  '*/10 * * * *',
  $$ select net.http_post(url := '.../functions/v1/dispatch-prayer') $$
);
```

```js
// supabase/functions/dispatch-prayer/index.js
Deno.serve(async () => {
  const supabase = adminClient();
  const { data: due } = await supabase
    .from('region_prayer_times')
    .select('region_id, prayer')
    .gte('fires_at', new Date().toISOString())
    .lt('fires_at', new Date(Date.now() + 10 * 60_000).toISOString());

  for (const { region_id, prayer } of due ?? []) {
    const tokens = await tokensForRegion(supabase, region_id); // one join: device_tokens → profiles → region_id
    await sendToTokens(tokens, `Time for ${capitalize(prayer)}`, '');
  }
  return new Response('ok');
});
```

---

## Shared plumbing (both parts use this)

Both subsystems end at the same place: hand a title, a body, and a list of tokens to Firebase.

FCM's v1 API needs an OAuth token minted from a service-account key — there's no static "server key" anymore. This OAuth step is between our backend and Google only; it has nothing to do with how users log into SES via Supabase Auth. `firebase-admin` isn't used here on purpose: its internals depend on Node APIs that don't exist in the Deno runtime Edge Functions run on. Instead, the JWT is built and signed by hand with the Web Crypto API (`crypto.subtle`), then traded for an access token at Google's OAuth endpoint. Zero extra dependencies.

```js
// getAccessToken() is the actual "prove we're allowed to call FCM" step —
// SA is the Firebase service-account JSON, loaded once from a Supabase secret.
function b64url(input) {
  const bytes = typeof input === 'string' ? new TextEncoder().encode(input) : new Uint8Array(input);
  return btoa(String.fromCharCode(...bytes)).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

async function getAccessToken() {
  const now = Math.floor(Date.now() / 1000);
  const unsigned = `${b64url(JSON.stringify({ alg: 'RS256', typ: 'JWT' }))}.${b64url(JSON.stringify({
    iss: SA.client_email,
    scope: 'https://www.googleapis.com/auth/firebase.messaging',
    aud: 'https://oauth2.googleapis.com/token',
    iat: now, exp: now + 3600,
  }))}`;
  const pem = SA.private_key.replace(/-----[^-]+-----|\n/g, '');
  const key = await crypto.subtle.importKey(
    'pkcs8', Uint8Array.from(atob(pem), c => c.charCodeAt(0)),
    { name: 'RSASSA-PKCS1-v1_5', hash: 'SHA-256' }, false, ['sign']
  );
  const sig = await crypto.subtle.sign('RSASSA-PKCS1-v1_5', key, new TextEncoder().encode(unsigned));
  const res = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion=${unsigned}.${b64url(sig)}`,
  });
  return (await res.json()).access_token;
}

async function sendToAllTokens(supabase, title, body) {
  const { data } = await supabase.from('device_tokens').select('token');
  return sendToTokens(data.map(t => t.token), title, body);
}

async function sendToTokens(tokens, title, body) {
  const accessToken = await getAccessToken();
  const dead = [];
  for (const token of tokens) {
    const res = await fetch(`https://fcm.googleapis.com/v1/projects/${PROJECT_ID}/messages:send`, {
      method: 'POST',
      headers: { Authorization: `Bearer ${accessToken}`, 'Content-Type': 'application/json' },
      body: JSON.stringify({ message: { token, notification: { title, body } } }),
    });
    if (!res.ok) {
      const err = await res.json().catch(() => null);
      if (err?.error?.status === 'UNREGISTERED' || res.status === 404) dead.push(token);
    }
  }
  if (dead.length) await supabase.from('device_tokens').delete().in('token', dead);
}
```

**Registering a token** hasn't changed shape, it just gained a `user_id`:

```ts
await fetch('https://<project-ref>.supabase.co/functions/v1/register-token', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ token, platform: Capacitor.getPlatform(), user_id: currentUser.id }),
});
```

---

## Cost

Still $0. Everything here runs on Supabase's free tier (Postgres + pg_cron + pg_net + Edge Functions) and Firebase's free Spark plan (FCM delivery only — no Cloud Functions, no Blaze plan, no card on file). The prayer-time ticker adds roughly 144 extra Edge Function invocations a day on top of the content sends — a few percent of the free tier's monthly allowance even before accounting for the content sends. Worth a glance at Supabase's current numbers before launch since they do get revised, but invocation count isn't the constraint here at any reasonable ticker interval.

## Part 3 — Bahr-e-aam / Urs updates (not designed yet)

Per-Murshid updates — each user follows a different Murshid (see feat-murshidObject), so this is closer in shape to Part 2 (per-user, not "same for everyone") than Part 1. Deliberately not designed in this pass; noting it here so the doc doesn't quietly lose track of it. Whatever shape it takes, it plugs into the same shared plumbing above (`sendToTokens`, the token-registration flow) rather than needing its own delivery mechanism.

## Open questions

- **Regions outside IST**: `compute-region-times`'s run-time and the content system's IST date-keying both assume one timezone for now. Needs a second look the day a region outside India shows up.
- **Corpus quality**: a fully random pick from the whole Ayah/Hadees/Naql corpus can occasionally land on something long or awkward as a standalone notification. Not fixing this now — a future content-curation pass, not a schema change.
- **Repeats**: nothing currently stops the same Ayah/Hadees/Naql from being picked again a few days later. Easy to add later (exclude anything shown in the last N days) — left out for now to keep the picker simple.