# Telepítési útmutató

Ez a dokumentum végigvezet a Supabase backend felhúzásán és a Netlify-os
hosting beállításán. Az app egy statikus HTML/JS — nincs build lépés, nincs
szerver oldali kód, minden a böngészőben fut és közvetlenül a Supabase REST
API-t hívja.

---

## 1. Supabase setup

### 1.1 Projekt létrehozása

1. Lépj be: <https://supabase.com>
2. **New project** → adj nevet (pl. `etterem-kalauz`), válassz régiót
   (`Central EU (Frankfurt)` ajánlott), és állíts be DB jelszót (mentsd el
   valahova, később már nem fogod látni).
3. Várd meg, amíg a projekt elindul (~2 perc).

### 1.2 Tábla létrehozása + RLS

A bal oldali menüben **SQL Editor** → **New query**, és futtasd le ezt:

```sql
-- Éttermek alaptáblája
create table public.restaurants (
  id          text primary key,
  name        text not null,
  address     text not null,
  category    text,
  rating      integer,             -- DEPRECATED: a régi (egyetlen csillag) modellből maradt, az új kód a restaurant_ratings táblát használja
  note        text,
  recommender text,
  link        text,
  lat         double precision not null,
  lng         double precision not null,
  updated_at  timestamptz default now()
);

-- Egyéni értékelések (több ember adhat csillagot ugyanarra az étteremre,
-- az appban a megjelenített érték = átlag).
create table public.restaurant_ratings (
  id            bigserial primary key,
  restaurant_id text not null references public.restaurants(id) on delete cascade,
  rater         text not null,
  value         integer not null check (value between 1 and 5),
  created_at    timestamptz default now(),
  updated_at    timestamptz default now(),
  unique (restaurant_id, rater)   -- egy név egy étteremre csak egyet rakhat (de azt felülírhatja)
);

create index on public.restaurant_ratings (restaurant_id);

alter table public.restaurants        enable row level security;
alter table public.restaurant_ratings enable row level security;

create policy "anyone can read"   on public.restaurants for select using (true);
create policy "anyone can insert" on public.restaurants for insert with check (true);
create policy "anyone can update" on public.restaurants for update using (true);
create policy "anyone can delete" on public.restaurants for delete using (true);

create policy "anyone can read"   on public.restaurant_ratings for select using (true);
create policy "anyone can insert" on public.restaurant_ratings for insert with check (true);
create policy "anyone can update" on public.restaurant_ratings for update using (true);
create policy "anyone can delete" on public.restaurant_ratings for delete using (true);
```

> ⚠️ A négy "anyone can ..." policy bárkinek megengedi az írást/törlést is.
> Ez egy belső céges ajánló-appnál elfogadható, **publikus internetre nem**.
> Ha publikussá teszed, érdemes a write policy-kat `auth.role() = 'authenticated'`
> feltételhez kötni és Supabase Auth-tal bejelentkeztetni a felhasználókat.

### 1.3 API kulcsok kinyerése

**Project Settings → API**, két érték kell:

- **Project URL** — pl. `https://xxxx.supabase.co`
- **Publishable key** (más néven `anon` key) — `sb_publishable_...` vagy
  `eyJ...` kezdetű string. Ez bizalmas-nak tűnik, de tényleg publikus, a
  böngészőbe kerül és a fenti RLS policy-k védik (illetve nem védik, lásd
  fent).

---

## 2. Lokális futtatás

1. Klónozd a repót: `git clone <repo-url>` és `cd etterem-kalauz`
2. Másold át a configot: `cp config.example.js config.js`
3. A `config.js`-be írd be a Supabase URL-t és a Publishable Key-t.
4. Nyisd meg az `index.html`-t a böngészőben.

   Ha a `file://` protokoll miatt valami akadozik (CORS, geocoding), futtasd
   egy mini szerverrel:

   ```sh
   python3 -m http.server 8080
   # majd: http://localhost:8080
   ```

### Teszt adatok betöltése

A repóban van egy `test-import.json` — kb. 7 étterem a Váci út / Árpád híd
környékéről. Töltés:

1. Az `index.html`-ben a `ADMIN_TOKEN` konstanst nézd meg, és nyisd meg az
   appot a megfelelő hash-sel, pl. `http://localhost:8080/#admin=<token>` —
   ettől megjelennek az **Import** / **JSON** / **XML** gombok.
2. Kattints az **📤 Import** gombra → válaszd ki a `test-import.json`-t.

---

## 3. Netlify deploy

### 3.1 Repó bekötése

1. <https://app.netlify.com> → **Add new site → Import from Git** → válaszd
   ki a GitHub repót.
2. **Build settings**:
   - Build command: *(üres — a `netlify.toml` megadja)*
   - Publish directory: `.`

A repóban lévő [netlify.toml](netlify.toml) build parancsa egy soros: a
`config.js`-t a deploy pillanatában generálja le a Netlify environment
változókból, így a kulcsokat nem kell commit-olni.

### 3.2 Environment változók

**Site settings → Environment variables**, ezeket vedd fel:

| Név                       | Érték                                       |
| ------------------------- | ------------------------------------------- |
| `SUPABASE_URL`            | a Supabase Project URL (1.3-ból)            |
| `SUPABASE_PUBLISHABLE_KEY`| a Publishable / anon key (1.3-ból)          |

Mentés után **Deploys → Trigger deploy → Clear cache and deploy site**, hogy
a friss env-ek bekerüljenek a generált `config.js`-be.

### 3.3 Admin token (Import/Export gombok elrejtése)

Az Import/Export gombok alapból csak akkor látszanak, ha az URL hash-ben
benne van a token, pl. `https://<site>.netlify.app/#admin=titkos-token`.

A tokent az [index.html](index.html) tetején, a `<script>` blokk elején lévő
`ADMIN_TOKEN` konstans tartalmazza — itt írhatod át. (Fontos: ez nem valódi
biztonság, mivel a JS forrás publikusan letölthető, és aki megnézi, azonnal
megtalálja a tokent. Csak véletlen kattintgatás ellen szűr.)

---

## 4. Hibakeresés

| Tünet                                            | Mit ellenőrizz                                                                        |
| ------------------------------------------------ | ------------------------------------------------------------------------------------- |
| ⚠️ "Hiányzik a config.js" banner                  | Lokálisan: `config.js` létezik-e. Netlify-on: env változók be vannak-e állítva.       |
| `Supabase hiba (401)` betöltéskor                | A Publishable Key rossz, vagy a `restaurants` táblán nincs SELECT policy.             |
| `Supabase hiba (404)`                             | Nincs `restaurants` tábla, vagy más néven van — futtasd újra az 1.2 SQL-t.            |
| Mentés/törlés `(403)` hibával                    | Hiányzik az INSERT/UPDATE/DELETE policy — futtasd újra az 1.2 SQL-t.                  |
| Import nem írja ki a JSON elemeket                | Minden elemnek kell `id`, `name`, `lat`, `lng` — a többi mező opcionális.             |
| Cím nem található geokódolásnál                  | Próbáld pontosabban (irányítószámmal, kerülettel), vagy kattints a térképen a helyre.|
