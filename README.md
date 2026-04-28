# Moo Run

A voice-controlled cow runner that plays in the browser, with the player's
webcam as the background. **High-pitched** sound jumps the cow over fences,
**low-pitched** sound ducks under overhead beams. Top 5 scores are stored in
Supabase.

```
moo-run.html      # the entire game (HTML + CSS + JS in one file)
vercel.json       # Vercel routing so `/` serves the game
README.md         # this file
LICENSE           # MIT
.gitignore
```

## 1. Set up Supabase (≈ 3 minutes)

1. Go to [supabase.com](https://supabase.com) and create a free project.
   Pick any region close to you.
2. Open the project's **SQL Editor** (left sidebar) and run the SQL below.
   This creates the `leaderboard` table and lets the game read/write it
   anonymously (which is what the public anon key does).

   ```sql
   create table public.leaderboard (
     id          uuid primary key default gen_random_uuid(),
     username    text not null check (length(username) between 1 and 32),
     score       int  not null check (score >= 0),
     cow_color   text,
     created_at  timestamptz default now()
   );

   create index leaderboard_score_idx
     on public.leaderboard (score desc, created_at desc);

   alter table public.leaderboard enable row level security;

   create policy "Anyone can read scores"
     on public.leaderboard for select
     using (true);

   create policy "Anyone can post a score"
     on public.leaderboard for insert
     with check (
       length(username) between 1 and 32
       and score >= 0
       and score < 100000
     );
   ```

3. Open **Project Settings → API**. Copy two values:
   - **Project URL** (looks like `https://abcd1234.supabase.co`)
   - **anon public** key (a long `eyJhbGciOi...` string)

4. Open `moo-run.html` and replace the placeholders near the top of the
   `<script>` block:

   ```js
   const SUPABASE_URL      = 'https://abcd1234.supabase.co';
   const SUPABASE_ANON_KEY = 'eyJhbGciOi...';
   ```

That's it for the backend — no server code, no functions, no auth.

> **Why is the anon key safe in the frontend?** It only has the permissions
> the RLS policies above grant: read everyone's scores, insert a row that
> matches the size/range checks. It cannot delete, update, or read any
> other table. If someone abuses the insert endpoint you can tighten the
> policy or add Supabase rate limits later.

## 2. Run locally

You can just double-click `moo-run.html` — but browsers block
`getUserMedia` on `file://` URLs, so the camera/mic won't work. Serve it
over HTTP instead:

```bash
# any of these works:
npx serve .          # easiest, no install
python3 -m http.server 8000
```

Then open `http://localhost:8000/moo-run.html` and grant mic + camera
permissions when prompted.

## 3. Deploy to Vercel (≈ 2 minutes)

### Option A — drag & drop (fastest)

1. Sign in to [vercel.com](https://vercel.com).
2. New Project → "Import" → **"Deploy without Git"**.
3. Drag this folder (the one containing `moo-run.html` + `vercel.json`)
   into the upload area.
4. Click Deploy. Done — your URL will look like `your-project.vercel.app`.

### Option B — via GitHub

1. Push this folder to a new GitHub repo.
2. In Vercel: **Add New… → Project → Import** that repo.
3. Framework preset: **"Other"** (it's a static site, no build needed).
4. Click Deploy.

The included `vercel.json` rewrites `/` to `/moo-run.html` so visitors
hit the game directly. It also sets a permissions policy header that
allows the page to request the mic and camera.

> **HTTPS is required.** Vercel gives you HTTPS automatically, which is
> what browsers need before they'll prompt for mic/camera access.

## How the game works (quick mental model)

- **Camera** comes from `getUserMedia({ video: true })` and is rendered as a
  full-screen `<video>` element behind the game canvas, mirrored.
- **Pitch detection** uses the Web Audio API's `AnalyserNode` to grab a
  2048-sample window every frame and runs autocorrelation on it. On game
  start it spends ~1.2 seconds calibrating to the player's neutral pitch,
  then triggers jump/duck whenever they go ±2 semitones from that baseline
  — so the same thresholds work for low and high voices.
- **Track** is a perspective trapezoid drawn on a `<canvas>`. Obstacles
  scroll from a `z = 1` horizon toward `z = 0` (the camera). The cow is
  fixed at `z ≈ 0.12`. Collision is a simple `Math.abs(z - cowZ) < 0.04`
  window with a check on jumping/ducking state.
- **Collision flash:** on a missed obstacle the game enters a `HIT` state
  for ~0.45 s where the cow is drawn with a pulsing red outline + glow,
  then it goes to game over.
- **Leaderboard** is a single `leaderboard` table in Supabase. On game
  over the game inserts `{username, score, cow_color}`. Top-5 query is
  `order by score desc, created_at desc limit 5`.

## Tweaks you might want

- **Stricter username uniqueness** — drop the `check` constraint and add a
  unique index on `lower(username)`. The frontend handles non-unique
  names fine today.
- **More cow colors** — add entries to the `COW_COLORS` object near the
  top of the script. The picker auto-builds from those keys.
- **Persist personal best** — change the leaderboard query to `eq('username',
  game.username)` for a "your best" panel, or use `upsert` with a unique
  constraint on `(username)` to keep just one row per name.
- **Daily/weekly board** — add `where created_at > now() - interval '7 days'`
  to the select.

## Troubleshooting

- **"Mic permission denied"** — check the site permission in your browser's
  URL bar. Safari on iOS requires the page to be HTTPS *and* explicitly
  allow each kind of media.
- **Cow doesn't move with my voice** — try a sustained "wooo" (jump) or a
  growly "huh" (duck). Some quiet ambient noise is fine, but the
  autocorrelator needs a clear pitched sound.
- **Score not saving** — open the browser console; if you see a Supabase
  401/403, your anon key or RLS policies are off. The SQL above is the
  exact set the game expects.
