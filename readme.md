# Jófogás|Vatera étterem kalauz 🍴

Céges étterem ajánló, OpenStreetMap térképpel és Supabase backenddel.
Egyetlen statikus HTML/JS oldal — nincs build, nincs szerver kód.

## Funkciók

- 🗺️ Térkép kattintással új helyet rakhatsz fel (Nominatim reverse geocoding)
- 🔍 Keresés név / cím / ajánló alapján, kategória szűrő
- ⭐ Csillagos értékelés, ajánló neve, link és megjegyzés
- 📥 JSON / XML export, 📤 JSON import — token mögé rejtve (`#admin=...`)

## Gyors indítás

```sh
cp config.example.js config.js
# töltsd ki a config.js-t a Supabase adatokkal
python3 -m http.server 8080
# nyisd meg: http://localhost:8080
```

A `test-import.json` egy 7 elemes mintaadatbázis a Váci út / Árpád híd
környéki éttermekről — az Import gombbal betölthető.

## Részletes telepítés

Lásd a [SETUP.md](SETUP.md)-t:

- Supabase projekt + tábla + RLS policy-k (a teljes `create table` SQL-lel)
- Lokális futtatás
- Netlify deploy + environment változók
- Admin token (Import/Export elrejtése)
- Hibakeresési tippek

## Fájlok

| Fájl                  | Mire való                                                       |
| --------------------- | --------------------------------------------------------------- |
| `index.html`          | A teljes app — HTML, CSS és JS egy fájlban.                     |
| `config.js`           | Supabase URL + publishable key. **Nincs commit-olva** (`.gitignore`). |
| `config.example.js`   | Sablon a `config.js`-hez.                                       |
| `netlify.toml`        | Netlify build config — env-ből generálja a `config.js`-t.       |
| `test-import.json`    | Minta adatok importhoz (7 étterem Váci út / Árpád híd környékén).|
| `SETUP.md`            | Részletes telepítési útmutató.                                  |
