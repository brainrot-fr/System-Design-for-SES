# Dashboard

## The short version

The home screen — pulls together content that otherwise lives scattered across every other feature. No new data, purely a read-and-arrange layer.

## Layout, top to bottom

1. **Today** — daily Ayah/Hadees/Naql cards, plus a "next prayer" element. Sits at the top since prayer timing is the one genuinely time-sensitive piece.
2. **Happening near you** — 2–3 upcoming event cards (both event types, region + gender filtered per [[feat-nearby-events]]), with a "see all" link into the full feed.
3. **Explore** — quick-access grid: Quran, Hadees, Nuqool, Silsila-e-Tarbiyat, Silsila-e-Faqiri, Ziyarah list. *(Timeline features — RasoolAllah SAWS / Imam Mahdi AS — deferred to v2; not included in this grid for now.)*
4. **Nearby people** — a card into [[feat-nearby-discovery]] if `discovery_visible` is already on, otherwise a low-pressure opt-in prompt. Never nagging, no badge/notification-dot.
5. **From your Murshid** — a visible stub placeholder for bahr-e-aam/urs updates ([[feat-notifs]] Part 3, not yet designed). Likely just the Murshid's name/silsila reference from [[feat-murshidObject]] for now, no live feed.

## Data sourcing

```js
// Today's content — reuses the same row feat-notifs already computed for the push
const today = new Intl.DateTimeFormat('en-CA', { timeZone: 'Asia/Kolkata' }).format(new Date());
const { data: daily } = await supabase.from('daily_content').select('*').eq('date', today).single();

// Next prayer
const { data: prayers } = await supabase
  .from('region_prayer_times')
  .select('prayer, fires_at')
  .eq('region_id', myRegion)
  .gte('fires_at', new Date().toISOString())
  .order('fires_at')
  .limit(1);

// Events preview
const { data: events } = await supabase
  .from('public_events')
  .select('*')
  .eq('region_id', myRegion)
  .in('gender_scope', myGender ? ['all', myGender] : ['all'])
  .gte('starts_at', new Date().toISOString())
  .order('starts_at')
  .limit(3);
```

New RLS needed — `daily_content` was only ever built for a cron job/Edge Function to read, never a client:

```sql
create policy "daily content is readable by anyone"
on daily_content for select
using (true);
```

## Offline behavior (confirmed)

Today/Events/Prayer sections **drop out entirely** when offline, rather than showing cached data. Default recommendation: pair this with a lightweight top-of-screen "you're offline — some sections unavailable" indicator, so the missing sections read as an intentional state rather than a bug. Explore tiles are bundled on-device and always work regardless of connectivity.