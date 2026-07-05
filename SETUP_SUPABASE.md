# Setup Supabase (Map Marketplace + Chat/Login)

Game ini pakai Supabase buat 2 fitur:

1. **🗺️ Map Marketplace** — upload/vote map custom, Hall of Fame
2. **💬 Chat Komunitas + Login** — akun email/password, chat real-time

Repo ini sengaja **tidak menyertakan** URL & API key Supabase (biar aman buat
open source). Kamu perlu bikin project Supabase sendiri (gratis) dan isi
kredensialnya sendiri. Boleh pakai **1 project untuk keduanya**, atau
**2 project terpisah** kalau mau beban kepisah.

---

## 1. Bikin project Supabase

1. Daftar/masuk ke https://supabase.com
2. Klik **New Project**, kasih nama bebas, tunggu sampai statusnya aktif.
3. Buka **Project Settings → API** — nanti kamu butuh:
   - **Project URL** (`https://xxxxxxxx.supabase.co`)
   - **anon / public key**

---

## 2. Jalankan SQL setup

Buka **SQL Editor** di dashboard Supabase, lalu jalankan skrip di bawah.

### A. Map Marketplace (tabel `maps`, `votes`, `hall_of_fame`)

> Skema ini direkonstruksi dari kode game (kolom yang dipakai saat select/insert).
> Silakan sesuaikan lagi kalau ada kebutuhan tambahan.

```sql
create table if not exists public.maps (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  author text not null,
  config jsonb not null,
  period int not null default 1,
  upvotes int not null default 0,
  vote_count int not null default 0,
  created_at timestamptz not null default now()
);

create table if not exists public.votes (
  id uuid primary key default gen_random_uuid(),
  map_id uuid not null references public.maps(id) on delete cascade,
  user_id text not null,
  vote int not null,
  created_at timestamptz not null default now(),
  unique (map_id, user_id)
);

create table if not exists public.hall_of_fame (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  author text not null,
  config jsonb not null,
  votes int not null default 0,
  period int not null,
  rank int not null,
  theme text,
  inducted_at timestamptz not null default now()
);

-- Marketplace bersifat publik & anonim (tanpa akun), jadi RLS dibuka
-- untuk anon key. Kalau mau lebih ketat, tambahkan validasi/limit sendiri.
alter table public.maps enable row level security;
alter table public.votes enable row level security;
alter table public.hall_of_fame enable row level security;

create policy "maps readable by anyone" on public.maps for select using (true);
create policy "maps insertable by anyone" on public.maps for insert with check (true);
create policy "maps updatable by anyone" on public.maps for update using (true);

create policy "votes readable by anyone" on public.votes for select using (true);
create policy "votes insertable by anyone" on public.votes for insert with check (true);
create policy "votes updatable by anyone" on public.votes for update using (true);

create policy "hof readable by anyone" on public.hall_of_fame for select using (true);
```

### B. Chat Komunitas + Login (akun asli via Supabase Auth)

```sql
-- Profil (username publik, terhubung ke akun auth.users)
create table if not exists public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  username text not null,
  created_at timestamptz not null default now()
);

alter table public.profiles enable row level security;
create policy "Profiles are viewable by everyone" on public.profiles for select using (true);
create policy "Users can insert their own profile" on public.profiles for insert with check (auth.uid() = id);
create policy "Users can update their own profile" on public.profiles for update using (auth.uid() = id);

create or replace function public.handle_new_user()
returns trigger language plpgsql security definer set search_path = public as $$
begin
  insert into public.profiles (id, username)
  values (new.id, coalesce(new.raw_user_meta_data->>'username', split_part(new.email,'@',1)))
  on conflict (id) do nothing;
  return new;
end;
$$;

drop trigger if exists on_auth_user_created on auth.users;
create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();

-- Chat messages
create table if not exists public.chat_messages (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  username text not null,
  message text not null check (char_length(message) between 1 and 200),
  created_at timestamptz not null default now()
);

create index if not exists chat_messages_created_at_idx on public.chat_messages (created_at desc);
create index if not exists chat_messages_user_id_idx on public.chat_messages (user_id);

alter table public.chat_messages enable row level security;
create policy "Chat viewable by logged-in users" on public.chat_messages for select to authenticated using (true);
create policy "Users can insert their own messages" on public.chat_messages for insert to authenticated with check (auth.uid() = user_id);

-- Batas 6 chat per user per 2 hari (dijaga di server, gak bisa dibypass client)
create or replace function public.enforce_chat_limit()
returns trigger language plpgsql security definer set search_path = public as $$
declare recent_count integer;
begin
  select count(*) into recent_count from public.chat_messages
  where user_id = new.user_id and created_at > now() - interval '2 days';
  if recent_count >= 6 then
    raise exception 'CHAT_LIMIT_REACHED';
  end if;
  return new;
end;
$$;

drop trigger if exists trg_enforce_chat_limit on public.chat_messages;
create trigger trg_enforce_chat_limit
  before insert on public.chat_messages
  for each row execute procedure public.enforce_chat_limit();

-- Kunci fungsi trigger biar gak bisa dipanggil manual lewat RPC publik
revoke execute on function public.enforce_chat_limit() from public;
revoke execute on function public.handle_new_user() from public;

-- Auto-hapus chat yang lebih tua dari 2 hari (jalan tiap jam via pg_cron)
create extension if not exists pg_cron;
select cron.schedule(
  'delete-old-chat-messages',
  '0 * * * *',
  $$ delete from public.chat_messages where created_at < now() - interval '2 days'; $$
);
```

> **Catatan soal konfirmasi email:** secara default Supabase Auth mewajibkan
> user klik link konfirmasi di email sebelum bisa login. Kalau mau user
> langsung bisa main tanpa verifikasi email, matikan di dashboard:
> **Authentication → Providers → Email → matikan "Confirm email"**.

---

## 3. Pasang URL & Key ke kode

Buka file game (`.html`), cari 2 tempat berikut dan ganti placeholder-nya:

**Bagian Map Marketplace** (cari `SUPABASE_URL`):
```js
const SUPABASE_URL = 'YOUR_SUPABASE_URL_HERE';
const SUPABASE_KEY = 'YOUR_SUPABASE_ANON_KEY_HERE';
```

**Bagian Chat + Login** (cari `CHAT_URL`):
```js
const CHAT_URL = 'YOUR_SUPABASE_URL_HERE';
const CHAT_KEY = 'YOUR_SUPABASE_ANON_KEY_HERE';
```

Ganti `YOUR_SUPABASE_URL_HERE` dan `YOUR_SUPABASE_ANON_KEY_HERE` dengan
Project URL & anon key dari project Supabase kamu (boleh sama untuk
keduanya, atau beda project). Simpan, upload/host filenya — selesai.

---

## Kenapa gak disertakan langsung?

Repo ini open source, jadi kalau API key ikut ke-push ke GitHub, siapa
pun bisa pakai (baca/tulis) database kamu. Makanya kredensialnya
sengaja dikosongkan — tinggal isi punya kamu sendiri, gratis & dalam
kendali penuh kamu.
