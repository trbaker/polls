# Reader Poll — embeddable polling widget

A single-question poll (3–6 answers) that you embed in any publication with one URL.
The poll definition is encoded in the link itself, votes are logged to Supabase, the
tally is live, and it resets at **midnight Pacific** each day.

## Files

| File | What it is |
|---|---|
| `supabase-setup.sql` | Run once in Supabase. Creates the votes table, the two access functions, grants, and the 25-hour cleanup job. |
| `index.html` | The embeddable widget. Reads the poll from `?p=…`, casts votes, shows live results. |
| `create.html` | The poll builder. Fills in a question + answers and gives you the share link and embed code. |

## Setup (about 5 minutes)

1. **Create a Supabase project** at supabase.com (free tier is fine).
2. **Run the SQL.** Open *SQL Editor → New query*, paste all of `supabase-setup.sql`, Run.
   - If the `pg_cron` line errors, enable it first under *Database → Extensions* (search "pg_cron"), then re-run the `cron.schedule(...)` block.
3. **Get your keys.** *Project Settings → API* → copy the **Project URL** and the **anon / publishable key** (the public one — safe to ship in client code).
4. **Configure the widget.** In `index.html`, set `SUPABASE_URL` and `SUPABASE_KEY` near the top of the `<script>`.
5. **Host the two files** on any static host — Netlify, Vercel, Cloudflare Pages, GitHub Pages, or your own server. Keep `index.html` and `create.html` in the same folder.

## Making a poll

Open `create.html`, type a question and 3–6 answers, click **Build poll link**.
You get a share link and an `<iframe>` embed. Paste the embed into your platform's
HTML/embed block; on platforms that unfurl URLs (Substack, Notion, etc.) the bare
share link often works on its own.

The **Poll ID** is what ties votes together. Keep it to keep counting the same poll;
generate a new one to start a fresh tally.

## How the daily reset works

- Each vote is stored with `poll_day` = the current date in `America/Los_Angeles`, computed **on the server** (so a visitor's clock can't shift it). DST is handled automatically.
- Results only count votes from today's Pacific date, so at 00:00 PT the tally returns to zero.
- An hourly job deletes rows older than 25 hours — your "retain for 25 hours" window, with a one-hour buffer past midnight.

## Security model

- The votes table has row-level security on with **no policies**, so it can't be touched directly through the API.
- All access goes through two functions — `cast_vote` and `get_results` — that visitors can only *execute*. They can record and read a vote but can't read the raw table, change the schema, or see voter tokens.
- The anon/publishable key is meant to be public; this is the intended Supabase pattern.

## Known limits (typical for anonymous embedded polls)

- **One vote per browser**, tracked by a token in `localStorage`. Some browsers partition or block storage for third-party iframes; if so, a returning reader may be able to vote again, and "your vote" won't be remembered across reloads. Voting still works either way.
- There's no identity check, so it isn't fraud-proof against a determined actor generating tokens — appropriate for reader-sentiment polls, not for anything binding. If you later need stronger dedup, the natural place to add it is inside `cast_vote` (e.g. a per-IP rate check).
- Changing an answer's text while keeping the same Poll ID can re-map which option a past vote pointed at. Use a new Poll ID when you change the answers.
