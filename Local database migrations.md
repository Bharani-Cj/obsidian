
**Status:** nothing in this guide is built yet. Items marked **NEW** are planned files. Everything else is how your app works today, taken from the real code.

**Assumption:** "DB" here means the **local SQLite database on the phone**. The remote backend (the quantman API) is drawn too, because the two work together.

---

## 1. The big picture

There are two places data can live:

```
                  ┌────────────────────────────────────┐
                  │  BACKEND  (quantman API server)    │
                  │  the source of truth, needs net    │
                  └────────────────▲───────────────────┘
                                   │  HTTPS requests (axios)
                                   │
┌──────────────────────────────────┴───────────────────────────────────┐
│  YOUR PHONE                                                          │
│                                                                      │
│   ┌────────────┐   asks for data   ┌──────────────┐   read / write   ┌─────────────┐
│   │  SCREENS   │ ────────────────► │  APP LOGIC   │ ◄──────────────► │   SQLite    │
│   │  (React)   │ ◄──────────────── │  (services)  │                  │  quantman.db│
│   └────────────┘    gives data     └──────────────┘                  │  (a file)   │
│                                                                      └─────────────┘
└──────────────────────────────────────────────────────────────────────┘
```

- **The backend** owns the real data. It needs internet and takes time.
- **SQLite** is a small database engine that lives *inside your app*. It is not a server. Your data is one file, `quantman.db`, in the app's private storage on the phone.
- **The local DB is a saved copy.** The first time the app needs something, it asks the backend and saves a copy. After that it can read the copy instantly, with no network.
- **Deleting the app deletes the DB.** That's fine for a cache, because it can always be fetched again.

---

## 2. The layers: who talks to whom

The one rule that keeps things tidy: **each layer only talks to the layer directly below it.** Screens never write SQL.

```
 #  LAYER                 EXAMPLE                                         KNOWS SQL?
 ───────────────────────────────────────────────────────────────────────────────────
 1  Screen                the Dashboard calendar                          no
        │  calls
        ▼
 2  Data function         fetchTransactionWithRunsFlattern()              no
        │                 components/DashboardComponents/subscription.ts
        ▼
 3  Service               getCachedDay() / setCachedDay()                 no
        │                 src/services/cache/dashboardCache.ts
        ▼
 4  Repository   [NEW]    getDay() / putDay() / evictBeyond()             YES (only here)
        │                 src/services/db/repositories/dashboardCacheRepo.ts
        ▼
 5  DB client    [NEW]    getDb()  opens the file, sets PRAGMAs,          no
        │                 runs migrations  → src/services/db/client.ts
        ▼
 6  expo-sqlite  (lib)    openDatabaseAsync, runAsync, getFirstAsync      -
        │
        ▼
 7  SQLite file on disk   quantman.db                                     -
```

Why so many layers? You said more tables are coming. If SQL is scattered through screens, changing one column means hunting through the whole app. With a repository per table, a change lives in one file.

---

## 3. The starting point: what happens when the app launches

This is the real startup path in your app, with the database added as one more task.

```
 App launch
   │
   ├─ index.ts ──► src/startup/instrument.ts      Sentry starts, splash screen is held
   │               └─► expo-router/entry
   │
   └─ app/_layout.tsx  ──►  useAppBoot()  ──►  startBoot()
          │
          │   Every boot task is started AT THE SAME TIME (not one after another):
          │
          ├── fonts          [blocking]   loads fonts
          ├── baseUrl        [blocking]   reads which API URL to use
          ├── pnlEngine      [blocking]   sets up the PnL engine
          ├── database       [NEW]        calls getDb() ─────────────────┐
          ├── network                     watches connectivity           │
          ├── analytics                   starts analytics               │
          └── releaseNotes                checks What's New              │
                                                                         ▼
                                       getDb() does this, once:
                                         1. open quantman.db (create it if missing)
                                         2. PRAGMA journal_mode = WAL, foreign_keys = ON
                                         3. migrate()  ← create/upgrade tables
                                         4. remember the open database for everyone
          │
          ▼
   when all [blocking] tasks are done:  ready = true
          │
          ▼
   splash screen hides ──► <Stack> mounts ──► your screens appear
```

**Two things to understand:**

1. **`getDb()` is the single front door.** Every repository calls `await getDb()` before doing anything. The first call starts the opening work. Every later call gets the *same* result. So it doesn't matter whether the screen asks for data before or after the boot task finishes. Everyone waits for the same one job.
2. **Why `database` is not `[blocking]`.** Blocking tasks hold the splash screen. A cache is optional. If the DB is slow or broken, the app should still open and fall back to the API. If you'd rather guarantee the DB is ready before any screen, set `blocking: true`. It's a one-word change.

---

## 4. Walkthrough A: opening a past day on the dashboard (first time)

This follows `fetchTransactionWithRunsFlattern("2026-10-20")` in `subscription.ts`.

```
fetchTransactionWithRunsFlattern("2026-10-20")
   │
   ▼
Is the date today?  ──── yes ───► skip the cache, call the API
   │ no (a past day)               (today's data is still changing, so it is
   ▼                                never cached)
getCachedDay("2026-10-20")
   │
   ▼
Already in the in-memory Map? ── yes ──► return it immediately
   │ no
   ▼
Repository runs SQL:
   SELECT * FROM dashboard_cache WHERE date = '2026-10-20'
   │
   ├── no row found (or an error) ──► returns null
   │                                    │
   │                                    ▼   ← CACHE MISS: this walkthrough
   │                          getRuns()  +  getRecentTransaction()   (API calls)
   │                                    │
   │                                    ▼
   │                          setCachedDay(...)
   │                              ├─ put it in the in-memory Map
   │                              ├─ INSERT OR REPLACE INTO dashboard_cache ...
   │                              └─ DELETE the oldest rows beyond the newest 30 days
   │                                    │
   │                                    ▼
   │                          return { orders, runs } to the screen
   │
   └── row found ───► see Walkthrough B
```

## 5. Walkthrough B: opening the same past day again (cache hit)

```
fetchTransactionWithRunsFlattern("2026-10-20")
   │
   ▼
getCachedDay("2026-10-20")
   │
   ▼
SELECT ... WHERE date = '2026-10-20'   ──►  1 row found
   │
   ▼
turn the JSON text back into objects, remember in the Map
   │
   ▼
return it   (no network used at all)
   │
   ▼
screen shows the day
```

The existing code adds a 500 ms delay on a hit (`addDelayOf(500)` at `subscription.ts:32`). That is existing behavior and is not part of this change.

## 6. Walkthrough C: what if the database is broken?

```
getDb() fails  (disk full, corrupted file, ...)
   │
   ▼
repository throws
   │
   ▼
getCachedDay catches it, logs it, returns null      ← same as "no cached row"
   │
   ▼
the app goes to the API as normal
```

The app keeps working. This works because this table is a *cache*. For tables holding data that exists only on the device, a failure must be shown to the user and reported, not hidden.

---

## 7. What's actually in the table

`dashboard_cache` has one row per cached day:

```
 date          runs (JSON text)           transactions (JSON text)     cached_at (ms)
 ─────────────────────────────────────────────────────────────────────────────────────
 2026-10-20    [{"id":..., ...}, ...]     {"orders":[...]}              1761800000000
 2026-10-19    [{"id":..., ...}, ...]     {"orders":[...]}              1761790000000
 ...           (at most 30 rows; the oldest is deleted when a 31st arrives)
```

---

## 8. Walkthrough D: updating the app when the database shape changes

Say version 1.5 of the app adds a new table. That's **migration 2**.

```
 User updates the app 1.4 ──► 1.5, then opens it
   │
   ▼
 getDb() opens quantman.db
   │
   ▼
 PRAGMA user_version  ──►  1          "this file was built with migration 1"
   │
   ▼
 the app now knows 2 migrations, so run #2:
   │
   ├─ inside one transaction:
   │     create the new table
   │     set user_version = 2
   │
   ├── success ──► continue, app works as normal
   └── failure ──► everything rolls back, user_version stays 1,
                    error goes to Sentry, tried again next launch
```

A fresh install starts at version 0 and runs migration 1, then 2. Both paths end up with the same tables. (Full rules for migrations are in `db-migrations.md`.)

---

## 9. Adding a new thing later: the recipe

Example: remember the user's favorite symbols on the device.

```
 1. Migration     migrations/0002_create_favorite_symbols.ts   "make the table"
        │
        ▼
 2. Repository    repositories/favoriteSymbolsRepo.ts          list / add / remove (SQL here)
        │
        ▼
 3. (only if the screen must update LIVE when data changes)
    Store         src/stores/external/FavoritesStore.ts        keeps a snapshot, calls notify()
        │         (same pattern as RunStore: rebuild the snapshot as a NEW reference first)
        ▼
 4. Hook          src/hooks/useExternalStore.ts                useFavoriteSymbols()
        │
        ▼
 5. Screen        reads the hook, shows the list
```

If the data is only read once (like the cache), skip step 3 and call the service from a data function.

---

## 10. Glossary

| Term | Plain meaning |
|---|---|
| **SQLite** | A database engine that runs inside your app and stores everything in one file |
| **Table** | A grid of data, like a spreadsheet tab (`dashboard_cache`) |
| **Row / column** | One record / one field of every record |
| **SQL** | The language for asking the database things (`SELECT`, `INSERT`, `DELETE`) |
| **`?` parameter** | A safe placeholder for values in SQL. Never paste values into the SQL text |
| **Transaction** | A group of changes that either all happen or none do |
| **Migration** | A numbered, run-once script that creates or changes tables |
| **PRAGMA** | A SQLite setting command (`user_version` stores the migration number) |
| **Repository** | The one file per table that contains its SQL |
| **Singleton** | One shared instance. `getDb()` always returns the same open database |
| **`await` / Promise** | Database work happens in native code and takes time. JS gets a promise and `await`s it |

---

## 11. Common questions

**Is the database a server I connect to?** No. It's a library plus a file, inside the app.

**Does the backend know about this database?** No. It's a private local copy. The backend stays the source of truth.

**Does the data sync to other devices?** No. It lives only on this phone.

**Why not keep using AsyncStorage?** It only stores text under a key. To find the oldest 30 days, the current code loads and parses every entry. SQL lets the database do that in one statement, and it handles many tables much better.

**What goes in SQLite and what stays in SecureStore?** Cached and structured data goes in SQLite. Secrets (the JWT, device ids) stay in `expo-secure-store` because SQLite is not encrypted.

**Can two things write at once?** SQLite handles it by queuing writes. Wrap multi-step changes in a transaction.

---

## 12. Suggested build order, with a check after each step

| Step | Do | How you know it worked |
|---|---|---|
| 1 | `npx expo install expo-sqlite`, rebuild the native app | App launches normally on a dev build |
| 2 | Add `client.ts` with `getDb()` | `console.log` from `getDb()` once, no errors |
| 3 | Add `migrate.ts` and migration 1 (`dashboard_cache`) | `PRAGMA user_version` reads `1` |
| 4 | Add `dashboardCacheRepo.ts` | A test `putDay` then `getDay` returns the same row |
| 5 | Switch `dashboardCache.ts` to the repo | Open a past day twice: the second open makes no API call |
| 6 | Add the `database` boot task | Sentry shows no `boot.task: database` errors |
| 7 | Remove `@react-native-async-storage/async-storage` | App builds, nothing imports it |

Each step can be checked on its own, so you can stop after any step with a working app.