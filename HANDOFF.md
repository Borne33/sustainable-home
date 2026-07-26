# The Sustainable Home — Developer Handoff

This file gives a new chat everything needed to keep editing this app. Hand it to the new chat
along with `index.html`. Written 2026-07-06.

---

## 1. What this is

A single-file web app: **`index.html`** (no build step, no framework, no dependencies except the
Supabase JS client loaded from a CDN). It's a household sustainability tracker with these views:

- **Consumables** — shopping/replenishment list (one-time / regular / recurring items, each with a
  store, tag, and 0–10 eco rating). This is the home view.
- **Sustainability News** — a feed of curated articles you can add to your list or ideas.
- **Needs** — bigger sustainability projects/ideas with scoring, cost, and ROI.
- **Stores** — where you shop, each with an eco rating.
- **Definitions** — static reference page explaining the eco tiers.

Everything is stored in `localStorage` and, when signed in, synced to **Supabase** (Postgres).

---

## 2. Locations, repo, deploy, backend

| Thing | Value |
|---|---|
| Local working copy | `/Users/alexbornemann/Downloads/Sustainable home/index.html` |
| GitHub repo | `git@github.com:Borne33/sustainable-home.git` (owner **Borne33**, branch **main**) |
| Live site | GitHub Pages — expected `https://borne33.github.io/sustainable-home/` (confirm under repo **Settings → Pages**) |
| Deploy mechanism | Pages serves `index.html` at the repo root. The file **must** be named `index.html` (a prior 404 was caused by a file named `the-sustainable-home.html`). |
| Supabase project | **The Sustainable Home**, ref `qwjpnbclqgeyansbucyh`, region us-east-2 |
| Supabase URL / key in code | `SB_URL` + `SB_KEY` near the top of the script (~line 191). `SB_KEY` is a **publishable** key — safe to be in client HTML. |

**Auth to push is already set up on this Mac** via an SSH deploy key
(`~/.ssh/id_ed25519_github`, configured in `~/.ssh/config` for `github.com`). No tokens needed.

### Push workflow (copy-paste)
The working copy is in a Dropbox CloudStorage folder that is **not** a git repo, so pushes are done
by cloning to `/tmp`, copying the file in, and pushing:

```bash
cd /tmp && rm -rf sh_push && git clone --depth 1 git@github.com:Borne33/sustainable-home.git sh_push
cp "/Users/alexbornemann/Downloads/Sustainable home/index.html" /tmp/sh_push/index.html
cd /tmp/sh_push
git config user.name "Borne33"; git config user.email "abornemann33@gmail.com"
git add index.html
git commit -m "Your message

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
git push origin HEAD
cd /tmp && rm -rf sh_push
```
Pages rebuilds ~1 min after push; hard-refresh (Cmd+Shift+R) to clear the cached page.
**Only push when the user asks.**

---

## 3. How to verify changes (IMPORTANT — no preview server, no Node)

Two environment constraints on this machine:
- **`node` is not installed.**
- The **preview/dev server cannot start** from this folder — it's under Dropbox CloudStorage and
  the macOS sandbox denies `getcwd`, so `python3 -m http.server` fails to launch via the harness.

Instead, verify with macOS's built-in **JavaScriptCore** through `osascript`. This is reliable and
was used to validate every change this session.

### 3a. Syntax check
```bash
cd "/Users/alexbornemann/Downloads/Sustainable home"
SCRATCH="$TMPDIR"   # or any writable dir outside CloudStorage, e.g. /tmp
python3 - "$SCRATCH/inline.js" <<'PY'
import re,sys
html=open("index.html","r",encoding="utf-8").read()
blocks=re.findall(r'<script(?![^>]*\bsrc=)[^>]*>(.*?)</script>', html, re.S|re.I)
open(sys.argv[1],"w",encoding="utf-8").write("\n;\n".join(blocks))
PY
osascript -l JavaScript <<OSA
var app=Application.currentApplication(); app.includeStandardAdditions=true;
var src=app.read(Path("$SCRATCH/inline.js"));
try { new Function(src); "SYNTAX OK"; } catch(e){ "SYNTAX ERROR: "+e.message; }
OSA
```

### 3b. Behavior check (stubbed DOM harness)
The whole inline script can be **executed** in JavaScriptCore by stubbing the few browser globals it
touches, then simulating clicks and asserting on `state`. Pattern used this session:

- Stub `document.getElementById('app')` to return an object with `addEventListener` (capture the
  `click`/`input`/`change` handlers into a map) and a settable `innerHTML`. Stub `localStorage`,
  `location`, `window`, `setInterval`, `setTimeout`. Leave `supabase` **undefined** so `sb` is null,
  `cloud.enabled` is false, and no network/auth code runs.
- Concatenate: `stubs.js` + extracted inline script + `test.js`, then `eval(src)` inside `osascript`.
- Simulate a click with:
  ```js
  function fireClick(attrs){
    var el={getAttribute:function(n){return (n in attrs)?attrs[n]:null;}};
    var ev={target:{classList:{contains:function(){return false;}},closest:function(){return el;}}};
    __handlers.click(ev);
  }
  fireClick({'data-act':'bulkmode'});
  fireClick({'data-act':'bulktoggle','data-id':'a'});
  ```
- Assert on `state.items`, `state.bulkSel`, `state.overlay`, etc. This exercises the real render
  functions too (they run during each `render()` call).

This caught nothing broken this session (25/25 assertions passed for the bulk-edit feature) but it's
the way to prove logic without a browser.

---

## 4. Architecture / conventions

- **One file.** All CSS is in one `<style>` block in `<head>`; all JS is in one inline `<script>`
  before `</body>`. There is also one external `<script src>` for the Supabase UMD client.
- **Style is ES5-ish**: `var`, `function` expressions, string concatenation to build HTML. Match it.
  (JavaScriptCore supports ES6 for *test* code, but keep the app itself in the existing style.)
- **Rendering:** state lives in the global `state` object (and `cloud` for auth). `render()`
  (~line 516) rebuilds `app.innerHTML` from scratch every time by dispatching on `state.view` and
  concatenating strings from helper functions. There is no diffing — after mutating `state`, call
  `save()` (persist + debounced cloud sync) and/or `render()`.
- **Event handling = delegation + `data-act`.** Three listeners are attached once to `#app`:
  `click` (~586), `input` (~653), `change` (~679). Each does
  `e.target.closest('[data-act]')`, reads `data-act` / `data-id` / `data-type` / `data-v`, and runs
  a `switch`. **To add an interaction:** render a button/input with `data-act="myaction"` (+ `data-id`
  if it targets an item), then add a `case 'myaction':` to the right listener. Click = buttons;
  input = text fields (fires per keystroke); change = `<select>` and commit-style inputs.
- **Escaping:** use `esc()` for text content and `escA()` for values inside HTML attributes. Always
  escape user/data strings when building HTML.
- **Icons:** `ico('name')` returns an inline SVG; icon paths live in the `P` object (~line 180–235).
  Add a new one with `Object.assign(P,{myicon:'<path .../>'});`.
- **Overlays/sheets:** bottom-sheet modals are built in `overlayHTML()` (~439) keyed on
  `state.overlay` and wrapped by `ovlWrap(title, body)`. Set `state.overlay='x'` to open,
  `state.overlay=null` (or `closeovl`) to close.

### Key helpers
`draftForm()` (add-item form), `rowDisplay()`/`rowEdit()`/`row()` (consumable rows),
`segType()` (type segmented control), `freqField()`, `ecoOptions()`/`storeOptions()`/`freqOptions()`,
`groups()` (filter+sort items into unchecked/purchased/archive via `matches()`+`comparator()`),
`menuNav()` (main menu order/labels, ~794).

---

## 5. Data model + Supabase sync

Client item shape (camelCase): `{id, name, type, freqUnit, freqDays, tag, store, eco, checked, checkedAt, createdAt}`
where `type` ∈ `onetime|regular|recurring`, `eco` is `''` or 0–10.

Sync is per-table with mapper pairs (~line 970+): `mapItemRow`/`rowItem` (table
`consumables_items`), `mapStoreRow`/`rowStore` (`stores`), `mapNeedRow`/`rowNeed` (`needs_ideas`),
`mapNewsRow`/`rowNews` (`news_articles`), plus a `profiles` row for sort prefs. DB columns are
snake_case (e.g. `freq_unit`, `checked_at`). Rows are scoped by `user_id` and RLS should restrict
each user to their own rows. `save()` (line 285) writes localStorage **and** calls `cloudSync()`, a
1.2s-debounced `pushAll()`. `pullAll()` loads on sign-in; if the account is empty it pushes local
data up instead.

**If you add a field to items**, update: the client shape wherever items are created
(`addItem`, news "add to list"), `mapItemRow`, `rowItem`, and the Postgres table column (via a
Supabase migration). Otherwise it won't round-trip through cloud sync.

### News crawler backend (Supabase)
Automated news pulling is live server-side:
- **Table `news_sources`** (`id text` pk, `user_id uuid`, `url`, `category` default `house`,
  `created_at`, `last_crawled`), RLS = own-rows-only. Client mappers `mapSourceRow`/`rowSource`;
  synced via `syncTable('news_sources', …)` (mirror). Managed in the **News → Sources** overlay.
- **Edge function `crawl-news`** (Supabase Functions, Deno, `verify_jwt=false`): iterates all
  `news_sources`, fetches each URL, does RSS/Atom **feed discovery** (`<link rel=alternate>`), parses
  with `deno.land/x/rss`, strips HTML, dedupes by `(user_id,url)`, inserts only items newer than
  `last_crawled` (first run = last 30 days), then bumps `last_crawled`. Uses the injected
  `SUPABASE_SERVICE_ROLE_KEY`. **Auth:** requires header `x-crawl-secret` = the constant baked into
  the function (`sh_crawl_9f3k2mZ7qWpL4xR8`). Rotate by editing the function + the cron job.
  - **How many (constants at top of function):** `PER_SOURCE_CAP=10` per source, `USER_BUDGET=40`
    total per user per crawl, distributed **fair-share round-robin** across that user's sources
    (freshest first). Budget is grouped **per user**, not global.
  - **Category assignment (`classify()`):** keyword-stem scoring over the article, where the feed's
    own `<category>` tags weigh most (×3), then title (×2), then body (×1); highest of the 7
    categories wins, else falls back to the source's `category` (default `house`). Keyword lists are
    in the `CATKW` map — edit there to tune. No LLM/API key involved (a Haiku fallback for ambiguous
    items was discussed as a future upgrade).
- **Schedule:** `pg_cron` job `crawl-news-daily` at `0 11 * * *` UTC calls the function via
  `net.http_post` (pg_net) with the secret header. Manage via `select * from cron.job;` /
  `cron.unschedule(...)`.
- **Sync interaction (important):** `news_articles` push is **upsert-only** (`upsertRows`, not the
  mirror `syncTable`) so crawler rows aren't deleted by the client; client "Remove" (`newsdel`) does a
  **targeted DB delete**; `refreshNewsFromCloud()` merges newly crawled rows into local state when the
  user opens the News page (`nav`→news).
- **Not client-triggerable:** the secret must stay server-side, so there's no "crawl now" button —
  new sources are picked up on the next daily run. Manual test: `curl -X POST <url>/functions/v1/crawl-news -H 'x-crawl-secret: …'`.

---

## 6. Auth (email + password)

Login is email + password (converted from magic-link this session). Relevant code:
`renderLogin()` (~978), `doEmailPass()` (~993, handles both sign-in `signInWithPassword` and sign-up
`signUp`), `doForgotPass()` (~1011, `resetPasswordForEmail`), `initAuth()` (~1022). `cloud` state
holds `authEmail`, `authPass`, `authMode` (`signin`/`signup`), `authMsg`, `authErr`.

---

## 7. What changed this session (most recent first)

-1. **Needs library expansion (2026-07-26 session).**
   - Added **398 new seed Needs ideas** (House 98, Energy 75, Water 69, Air 50, Waste 58,
     Transport 48) on top of the original 10/category. Each: `status:'idea'`, cost tier, ROI
     (nullable), a 4-part `q:[lifecycle,production,use,endOfLife]`, and 2 sources.
   - **Two places** they live: (a) appended to the `NEEDS_SEED` const in `index.html` (for fresh
     installs), and (b) **inserted directly into Supabase `needs_ideas`** for the primary account
     (`user_id f83f8760-e217-40f2-af13-a98ea88b7d81`) so they show immediately.
   - **DB row ids** use a stable scheme `s2` + 2-char category + 3-digit index: house `s2ho###`,
     energy `s2en###`, water `s2wa###`, air `s2ai###`, **waste `s2ws###`** (NOT `s2wa` — water and
     waste both start "wa", so waste is `ws` to avoid a PK collision), transport `s2tr###`. Find them
     with `where id like 's2%'`. `NEEDS_SEED` JS entries have no id (runtime `uid()`).
   - **Sync caveat (important):** `needs_ideas` push is a MIRROR (`syncTable`) — it deletes DB rows
     not present in local state. After a bulk DB insert, an existing open client must **reload**
     (triggers `pullAll`) before editing, or its stale local list will wipe the new rows on the next
     push. Generator script + chunked SQL were kept in the session scratchpad only.
   - The scheduled news crawl (`pg_cron` `crawl-news-daily`) was turned OFF by the user this session
     (it wasn't seeding Needs usefully); re-enable via `cron.schedule(...)` if desired.
   - The working copy moved to **`/Users/alexbornemann/Sustainable home/index.html`** (was under
     `~/Downloads/Sustainable home/`). Push workflow (§2) is otherwise unchanged.

0. **Eco Score model + News filter + archive/delete (2026-07-12 session).**
   - **Eco Score replaces all prior ratings.** The 0–10 `eco` on items/stores and the 6 `QUAL`
     bars on needs are gone. Everything now uses a **4-part score** stored as `q:[lifecycle,
     production, use, endOfLife]` (each 0–10). **Overall = sum × 2.5 → 0–100.** Helpers:
     `ECO` (criteria), `ecoScore(q)`, `scoreCls(v)` (≥75 boldgreen / ≥50 green / ≥25 amber / <25 red),
     `scoreBadge(q)`, `qGrid()` (the shared 4-dropdown form + live total), `fixQ4()`/`qFromEco()`
     (legacy migration), `estimateArticleQ()` (app-suggested scores for articles). News cards show
     the badge; `newsEdit` and a new **intake popup** let you edit all four.
   - **DB (already applied):** added `eco_q jsonb` to `consumables_items`, `stores`,
     `news_articles`; `needs_ideas.q` is now length-4. New table **`app_config`** (`user_id` pk,
     `config jsonb`, RLS own-rows) holds the editable per-user config, synced in `pushAll`/`pullAll`.
     Mappers write `eco:null` + `eco_q` and read `eco_q` (falling back to legacy `eco`).
   - **Editable config on Definitions** (`state.config` = `{threshold, rubric{}, cats[]}`, defaults
     in `defaultConfig()`): Eco Score rubric text, the 7 category labels/definitions/**keywords**,
     and the **news Eco-Score threshold** — all user-editable and synced. The keywords + threshold
     **drive the crawler**.
   - **News intake popup** (`overlay==='newsadd'`, `openNewsAdd()`/`applyNewsAdd()`): opens in place
     on the News page, pre-fills from the article + suggested score, has a Consumables/Needs toggle,
     Done marks the article `added` and stays on News. (Old `newsact` navigate-away flow removed.)
   - **Archive/delete** (`newsLifecycle()`, client-side): read + non-added + non-highlighted articles
     archive after **30 days** (collapsible "Archive · 30 Days Old" section, `state.newsArchiveOpen`)
     and delete after **90 days** (local + DB). Age = `now - seenAt` (first-pulled). Archived items
     pop back out if re-read/added/highlighted (flags recomputed each pass).
   - **Crawler (edge fn `crawl-news`, v4, already deployed):** `PER_SOURCE_CAP=10→4`; reads each
     user's `app_config` for keywords + threshold; `classify()` returns "" (→ **drops** the article)
     when it fits no category; estimates the Eco Score and **drops anything < threshold**; writes
     `eco_q`. Mirrors the client's `estimateArticleQ`/keyword lists — keep them in sync.
   - **UI fixes:** bottom bar ~25% taller (`.botrow` min-height 48→60, `.botbar` padding, `.wrap`
     padding-bottom 150→188); red swipe-delete flash fixed (`.swipedel{opacity:0}` + `.sw-active`
     toggled during an active swipe); login-page flash on load fixed (`cloud.booting` splash until
     `initAuth` resolves; boot no longer renders login first).
   - Not yet pushed to GitHub Pages — the live site still runs the old file until you push §2.

1. **Bulk edit on Consumables** — a toggle button left of Search enters bulk mode; round checks
   become squares (`.cb.sq`); a selection bar (`bulkBar()`) with select-all + Edit appears above the
   list; the `bulkedit` overlay edits type/store/tag/eco across selected items, each field
   defaulting to "leave unchanged". State: `state.bulkMode`, `state.bulkSel` (id→true map),
   `state.bulkDraft`. Actions: `bulkmode`, `bulktoggle`, `bulkselall`, `bulkopen`, `bulktagon`,
   `bulkapply` (click) and `bulktype`/`bulkstore`/`bulkeco` (change), `bulktag` (input).
2. **Main menu reorder + capitalization** — order is now Consumables, Sustainability **News**,
   Needs, Stores, Definitions, then Sign out (in `menuNav()`).
3. **Magic link → email + password** auth (see §6).
4. **Renamed the deployed file to `index.html`** to fix the Pages 404.

---

## 8. Outstanding items the user must set in the Supabase dashboard (can't be done from code)

- **"Confirm email" toggle** (Authentication → Providers → Email): ON = new users must click an
  email link before they can sign in; OFF = instant sign-in. The sign-up code shows a "check your
  email to confirm" message that assumes ON.
- **Redirect URLs** (Authentication → URL Configuration): the live site URL must be listed or the
  **Forgot password** reset link won't return users to the app.
- **Leaked Password Protection** (advisor flagged it WARN/disabled) — worth enabling now that
  passwords are used.

The Supabase MCP tools available in-session can read tables, run SQL, and read advisors, but **cannot
read or change these GoTrue auth toggles** — they're dashboard-only.

---

## 9. Recipe: adding a new feature (the pattern that works here)

1. Read the relevant render function(s) and the three event listeners to match style.
2. Add any new `state` fields near the top of the `state` object.
3. Build UI in the appropriate render/`overlayHTML` function using `data-act` (+ `data-id`), `ico()`,
   `esc()`/`escA()`.
4. Add `case` handlers to the click/input/change listeners; mutate `state`, then `save()` and/or
   `render()`.
5. Add any CSS to the `<style>` block (reuse existing classes like `.f`, `.done`, `.cb`, `.item`).
6. Verify with the JavaScriptCore syntax check **and** a stubbed-DOM behavior harness (§3).
7. Push with the §2 workflow **only when the user asks**.
