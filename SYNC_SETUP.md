# ☁️ Cloud Sync Setup (multi-device, just you)

This makes your trades & plans sync across devices using **Supabase** (free hosted Postgres + auth). The app stays **localStorage-first** (instant + offline); the cloud is a synced mirror. No Vercel serverless functions needed — your static page talks to Supabase directly.

**Time: ~5–10 minutes.** You do steps 1–4 once.

---

## 1. Create a Supabase project

1. Go to <https://supabase.com> → sign up (free) → **New project**.
2. Pick a name, a strong **database password** (you won't need it for the app), and a region near you.
3. Wait ~2 min for it to provision.

## 2. Create the tables + security rules

Open **SQL Editor** (left sidebar) → **New query** → paste this and click **Run**:

```sql
-- Tables: one row per trade / plan, owned by the signed-in user.
create table if not exists public.trades (
  id         text primary key,
  user_id    uuid not null default auth.uid() references auth.users(id) on delete cascade,
  payload    jsonb not null,
  updated_at timestamptz not null default now()
);

create table if not exists public.plans (
  id         text primary key,
  user_id    uuid not null default auth.uid() references auth.users(id) on delete cascade,
  payload    jsonb not null,
  updated_at timestamptz not null default now()
);

-- Row Level Security: each user can only see/touch their OWN rows.
alter table public.trades enable row level security;
alter table public.plans  enable row level security;

create policy "own trades" on public.trades
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

create policy "own plans" on public.plans
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
```

> **Why this is safe to use from the browser:** RLS means even though the page holds the public "anon" key, the database will only ever return or accept rows belonging to the logged-in user. Without a valid login, the tables return nothing.

## 3. (Recommended) Turn OFF email confirmation for a personal app

So you can sign in immediately without clicking a confirmation email:

**Authentication → Providers → Email** → toggle **"Confirm email" OFF** → Save.
(Leave it ON if you prefer; you'll just confirm via email once before the first sign-in.)

Email/password auth is on by default — nothing else to enable.

## 4. Copy your keys into `index.html`

In Supabase: **Project Settings → API**. Copy:
- **Project URL** (e.g. `https://abcdxyz.supabase.co`)
- **anon public** key (a long `eyJ...` string — the *anon*, NOT the `service_role` key)

Open `index.html`, find the **CLOUD CONFIG** block near the top of the `<script>`:

```js
const SUPABASE = {
  url: '',       // paste Project URL here
  anonKey: ''    // paste the anon public key here
};
```

Fill both in, save, deploy.

> ⚠️ **Only ever put the `anon` key in the file. NEVER the `service_role` key** — that one bypasses RLS and would expose all data.

---

## 5. Use it

1. Open the app → **⚙️ Data** tab → the **Cloud Sync** card now shows **"sign in"**.
2. Click **Create account** (enter an email + password) once. Then **Sign in**.
3. Do the same on your phone/other laptop with the **same email + password**.
4. New trades & plans auto-sync. Hit **🔄 Sync now** any time to force a full two-way merge.

That's it — log a trade on your laptop, open the app on your phone, hit Sync (or just reload), and it's there.

---

## How the sync behaves

- **localStorage-first:** the UI is always instant and works offline. Cloud writes happen in the background.
- **Merge rule:** on pull, records are merged by `id`, newest `updated` timestamp wins. So editing the same trade on two devices keeps the most recent edit.
- **Deletes:** propagate immediately when you're online & signed in. **Caveat:** if you delete something on Device A while Device B is offline, Device B may re-push it on its next sync (no tombstones). Fix: delete it again while online. For a personal journal this is rarely an issue.
- **Not signed in / no config:** everything still works locally exactly as before.

---

## Deploying to Vercel

This is a static site — no build needed.

1. Push your repo (already done: `khaaali/orderflowlab`).
2. On <https://vercel.com> → **Add New → Project → Import** your GitHub repo.
3. Framework preset: **Other**. Build command: *(none)*. Output dir: *(root / leave default)*.
4. Deploy. Your app is live at `https://<project>.vercel.app`.

Because the Supabase **anon key is safe to expose** (RLS protects the data), you can simply keep it in `index.html` — no Vercel environment variables or build step required.

Optional `vercel.json` (only if you want to force clean routing):
```json
{ "cleanUrls": true }
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Card still says **"off"** | `SUPABASE.url`/`anonKey` not filled, or the `@supabase/supabase-js` CDN didn't load (check the browser console / your connection). |
| **"Invalid login credentials"** | Wrong email/password, or email confirmation is ON and not yet confirmed (see step 3). |
| Sign-up says check email | Email confirmation is ON — confirm via the email, or turn it off (step 3). |
| Rows not appearing on Device B | Hit **🔄 Sync now**; confirm you signed in with the **same** account; check RLS policies ran (step 2). |
| `permission denied for table` | RLS policies weren't created — re-run the SQL in step 2. |

---

## Alternatives (if you ever outgrow Supabase)

- **Vercel Postgres / Vercel KV** + serverless `/api` routes — more code, you build the auth yourself.
- **Firebase Firestore** — similar model to Supabase, Google ecosystem.
- **Turso / Upstash Redis** — lightweight, but you'd still write an API layer + auth.

Supabase is the best fit here: hosted Postgres, built-in auth, RLS, generous free tier, and it works directly from a static page.
