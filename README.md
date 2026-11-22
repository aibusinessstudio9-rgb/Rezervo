
Skedari 1: emri BUILD_BRIEF.md
# Rezervo: build brief (phase 1)

Give this file, `schema.sql`, `README.md` and the spec/backlog documents to a developer or to Claude Code, and ask it to build phase 1 below. Albanian (sq) is the main language; en, de, it and fr are supported.

## Goal
A real booking app for barbers, restaurants, cafes and salons, replacing the demo prototype. Customers book; venue owners manage; an admin approves venues.

## Stack
- App: Expo (React Native, TypeScript, Expo Router). Also export for web so it works as an installable web app, reachable by QR code without an app store.
- Backend: Supabase in an EU region (Postgres, Auth, Storage, Edge Functions).
- Auth: phone login with SMS OTP, plus email login.
- i18n: i18next, JSON files per language, `sq` as base and fallback.
- Run `schema.sql` as the first migration. Do not change its security rules without asking.

## Phase 1 scope (must have)
1. Sign up and login (SMS OTP, email); profile with name and language; delete account in the app.
2. Browse and search venues by category, city and name; venue page with photo, services and reviews.
3. Booking flow: choose service (barber, cafe, beauty) or party size and stay length (restaurant), choose a date and a free time, confirm. Free times come from `busy_slots()`; booking goes through `book_slot()`.
4. My bookings: list, change (atomic reschedule, old slot freed only if the new one succeeds), cancel, status shown; review after status "done".
5. Owner area: create venue (starts as `pending`), services with price and duration, weekly hours, closed days and vacation, photo upload to Storage, calendar of bookings, set status (confirmed, done, noshow), cancel.
6. Admin area (service role or a protected admin role, never exposed to normal users): approve, suspend or delete venues, remove reviews, simple stats, toggle sample venues off.
7. Legal: privacy policy and terms linked in the app; consent text where needed.

## Behaviours to keep from the prototype
- Restaurant: party size is a free number field (1 to 500); the client picks how long they stay (1 to 6 hours in 30-minute steps); that duration blocks the table time.
- Other venues: the chosen service duration blocks time; no slot may pass closing time.
- Closed weekdays and vacation dates cannot be picked.
- Clients can change or cancel their own bookings; past bookings can only be cancelled.
- Venue owner sees the client name, service, time, duration and party size; phone is visible only to the venue the client booked.
- A festive banner under the language buttons from 15 November to 10 January, text in all five languages, static when the device prefers reduced motion.
- Light and dark mode; layout works on small phones.

## Not in phase 1
Payments, WhatsApp or Viber, waiting lists, restaurant table capacity and staff scheduling (see backlog CAP-1 and STAFF-1), maps.

## Working agreement
- Scaffold first, then run the schema, then build features in the order above, one commit per feature.
- Keep keys in `.env` and provide `.env.example` (`SUPABASE_URL`, `SUPABASE_ANON_KEY`). Never put the service role key in the app.
- Write database tests for: a client cannot read others' bookings; two simultaneous bookings for the same slot cannot both succeed; a review is only allowed after status "done".
- Ask before adding any paid service or dependency not listed here.
- Finish by running through the QA checklist and reporting what passes and what does not.

## What the owner of the project must prepare
- A Supabase project (EU region) and an SMS provider (Twilio, Vonage or MessageBird), with coverage and price checked for Albania, Kosovo, Germany, Italy and France.
- Apple Developer and Google Play Console accounts, only when ready to publish.
- A domain and a contact email (the demo uses aibusinessstudio9@gmail.com for now).
- The company details to fill in the privacy policy and terms, reviewed by a lawyer.
Skedari 2: emri schema.sql
-- Rezervo: PostgreSQL schema for Supabase (EU region).
-- Run in the Supabase SQL editor. Phone/email login is handled by Supabase Auth (auth.users).

create extension if not exists btree_gist;

-- ---------- Tables ----------
create table profiles (
  id uuid primary key references auth.users on delete cascade,
  name text,
  phone text,
  lang text not null default 'sq' check (lang in ('sq','en','de','it','fr')),
  push_token text,
  created_at timestamptz not null default now()
);

create table venues (
  id uuid primary key default gen_random_uuid(),
  owner_id uuid not null references profiles(id) on delete cascade,
  name text not null check (char_length(name) between 2 and 80),
  kind text not null check (kind in ('barber','restaurant','cafe','beauty')),
  city text not null,
  address text,
  phone text,
  description text,
  photo_path text,
  tz text not null default 'Europe/Tirane',
  resources int not null default 1,
  status text not null default 'pending' check (status in ('pending','active','suspended')),
  created_at timestamptz not null default now()
);

create table services (
  id uuid primary key default gen_random_uuid(),
  venue_id uuid not null references venues(id) on delete cascade,
  name text not null check (char_length(name) between 2 and 60),
  minutes int not null check (minutes between 5 and 480),
  price_cents int not null default 0 check (price_cents >= 0),
  active boolean not null default true
);

create table opening_hours (
  venue_id uuid not null references venues(id) on delete cascade,
  weekday smallint not null check (weekday between 0 and 6),
  opens time not null,
  closes time not null check (closes > opens),
  primary key (venue_id, weekday)
);

create table closures (
  id uuid primary key default gen_random_uuid(),
  venue_id uuid not null references venues(id) on delete cascade,
  from_date date not null,
  to_date date not null check (to_date >= from_date),
  reason text
);

create type booking_status as enum ('confirmed','done','noshow','cancelled');

create table bookings (
  id uuid primary key default gen_random_uuid(),
  venue_id uuid not null references venues(id) on delete cascade,
  service_id uuid not null references services(id),
  user_id uuid not null references profiles(id),
  resource int not null default 1,
  starts_at timestamptz not null,
  ends_at timestamptz not null,
  guests int not null default 1 check (guests between 1 and 500),
  status booking_status not null default 'confirmed',
  note text check (char_length(note) <= 300),
  created_at timestamptz not null default now(),
  check (ends_at > starts_at),
  exclude using gist (
    venue_id with =, resource with =, tstzrange(starts_at, ends_at) with &&
  ) where (status in ('confirmed','done'))
);
create index on bookings (venue_id, starts_at);
create index on bookings (user_id, starts_at);

create table reviews (
  booking_id uuid primary key references bookings(id) on delete cascade,
  venue_id uuid not null references venues(id) on delete cascade,
  user_id uuid not null references profiles(id),
  rating int not null check (rating between 1 and 5),
  comment text check (char_length(comment) <= 200),
  created_at timestamptz not null default now()
);

-- ---------- Row level security ----------
alter table profiles enable row level security;
alter table venues enable row level security;
alter table services enable row level security;
alter table opening_hours enable row level security;
alter table closures enable row level security;
alter table bookings enable row level security;
alter table reviews enable row level security;

create function is_owner(v uuid) returns boolean language sql stable security definer
  set search_path = public as $$
  select exists (select 1 from venues where id = v and owner_id = auth.uid())
$$;

create function is_active(v uuid) returns boolean language sql stable security definer
  set search_path = public as $$
  select exists (select 1 from venues where id = v and status = 'active')
$$;

create policy "own profile" on profiles for all using (id = auth.uid()) with check (id = auth.uid());

create policy "venues readable" on venues for select using (status = 'active' or owner_id = auth.uid());
create policy "venues insert" on venues for insert with check (owner_id = auth.uid() and status = 'pending');
create policy "venues update" on venues for update using (owner_id = auth.uid())
  with check (owner_id = auth.uid() and status in ('pending','active'));
create policy "venues delete" on venues for delete using (owner_id = auth.uid());

create policy "services read" on services for select using (is_active(venue_id) or is_owner(venue_id));
create policy "services write" on services for all using (is_owner(venue_id)) with check (is_owner(venue_id));
create policy "hours read" on opening_hours for select using (is_active(venue_id) or is_owner(venue_id));
create policy "hours write" on opening_hours for all using (is_owner(venue_id)) with check (is_owner(venue_id));
create policy "closures read" on closures for select using (is_active(venue_id) or is_owner(venue_id));
create policy "closures write" on closures for all using (is_owner(venue_id)) with check (is_owner(venue_id));

create policy "bookings read" on bookings for select using (user_id = auth.uid() or is_owner(venue_id));
create policy "bookings owner update" on bookings for update using (is_owner(venue_id)) with check (is_owner(venue_id));

create policy "reviews read" on reviews for select using (true);
create policy "reviews insert" on reviews for insert with check (
  user_id = auth.uid()
  and exists (select 1 from bookings b where b.id = booking_id and b.user_id = auth.uid()
              and b.status = 'done' and b.venue_id = reviews.venue_id)
);

-- ---------- Booking functions ----------
create function book_slot(p_venue uuid, p_service uuid, p_start timestamptz,
                          p_guests int default 1, p_note text default null)
returns bookings language plpgsql security definer set search_path = public as $$
declare
  v venues; s services; b bookings;
  ls timestamp; le timestamp; wd int; r int;
begin
  if auth.uid() is null then raise exception 'not authenticated'; end if;
  select * into v from venues where id = p_venue and status = 'active';
  select * into s from services where id = p_service and venue_id = p_venue and active;
  if v.id is null or s.id is null then raise exception 'unknown venue or service'; end if;
  if p_start < now() then raise exception 'start is in the past'; end if;

  ls := p_start at time zone v.tz;
  le := ls + make_interval(mins => s.minutes);
  wd := extract(dow from ls);
  if le::date <> ls::date or not exists (
       select 1 from opening_hours h
       where h.venue_id = v.id and h.weekday = wd and ls::time >= h.opens and le::time <= h.closes)
  then raise exception 'venue is closed at that time'; end if;
  if exists (select 1 from closures c where c.venue_id = v.id and ls::date between c.from_date and c.to_date)
  then raise exception 'venue is closed on that date'; end if;

  for r in 1..v.resources loop
    begin
      insert into bookings (venue_id, service_id, user_id, resource, starts_at, ends_at, guests, note)
      values (v.id, s.id, auth.uid(), r, p_start, p_start + make_interval(mins => s.minutes), p_guests, p_note)
      returning * into b;
      return b;
    exception when exclusion_violation then
      continue;
    end;
  end loop;
  raise exception 'slot taken';
end $$;

create function cancel_booking(p_id uuid) returns void language sql security definer
  set search_path = public as $$
  update bookings set status = 'cancelled'
  where id = p_id and user_id = auth.uid() and status = 'confirmed' and starts_at > now()
$$;

create function busy_slots(p_venue uuid, p_from timestamptz, p_to timestamptz)
returns table (resource int, starts_at timestamptz, ends_at timestamptz)
language sql stable security definer set search_path = public as $$
  select resource, starts_at, ends_at from bookings
  where venue_id = p_venue and status in ('confirmed','done')
    and starts_at < p_to and ends_at > p_from and is_active(p_venue)
$$;

grant execute on function book_slot, cancel_booking, busy_slots to authenticated;
Skedari 3: emri README.md
# Rezervo: udhëzues ngritjeje për zhvilluesin

Ky paketim përmban skemën e databazës (`schema.sql`) dhe hapat për ta kthyer prototipin në aplikacion real.

## 1. Backend (Supabase, rajon BE)

1. Krijo projekt në Supabase dhe zgjidh një rajon në Evropë (p.sh. Frankfurt).
2. Hap SQL Editor dhe ekzekuto `schema.sql`.
3. Authentication, Providers, Phone: aktivizo hyrjen me telefon dhe lidhe me një ofrues SMS (Twilio, Vonage ose MessageBird). Kontrollo mbulimin dhe çmimin për Shqipërinë, Kosovën, Gjermaninë, Italinë dhe Francën. Aktivizo edhe hyrjen me email si alternativë.
4. Pas hyrjes së parë, krijo rreshtin në `profiles` (me një trigger mbi `auth.users` ose nga aplikacioni).
5. Storage: krijo bucket `venue-photos` (publik për lexim, shkrim vetëm për pronarin e vendit).
6. Vendet e reja ruhen me `status = 'pending'`. Administratori i kalon në `active` me çelësin e shërbimit (service role), jo nga aplikacioni.

## 2. Njoftimet

- SMS/email konfirmimi: një Edge Function e thirrur pas çdo rezervimi (Database Webhook mbi `bookings`) dërgon mesazhin në gjuhën `profiles.lang`.
- Kujtesa një orë para takimit: punë e planifikuar (`pg_cron`) që dërgon push.
- Push: Firebase Cloud Messaging dhe APNs, ose Expo Push. Tokeni ruhet te `profiles.push_token`.
- Shabllonet e mesazheve në pesë gjuhët: sq, en, de, it, fr.

## 3. Aplikacioni

- Sugjerim: React Native me Expo dhe `supabase-js`. Alternativë më e lirë për fillim: PWA.
- Gjuhët: `i18next` me skedarë JSON për sq (bazë), en, de, it, fr.
- Ora lokale e vendit përdor `venues.tz`.

## 4. Publikimi në dyqane

- Apple Developer Program dhe Google Play Console. Çmimet ndryshojnë, kontrollo faqet zyrtare.
- Të detyrueshme: politikë privatësie, kushte përdorimi, fshirje llogarie brenda aplikacionit.

## 5. Privatësia (GDPR)

- Ruaj të dhënat në BE; shfaq emrin dhe telefonin e klientit vetëm pronarit të vendit ku ka rezervuar.
- Ofro eksport dhe fshirje të të dhënave nga profili.
- Kopje rezervë ditore dhe plan rikuperimi