# Design: Selection Mode

**Status:** Built and verified on a test Dispatcharr v0.31.0 (PR branch `feature/selection-mode`).
This document describes the feature **as built**; §11 lists where it departs from the
original design and why.
**Target:** VOD2MLIB, as an opt-in feature intended for an upstream PR
**Date:** 2026-09-26

---

## 1. Problem

Classic VOD2MLIB writes a `.strm` for every movie and series in the Dispatcharr catalogue that
passes the category filter/exclude. The only controls are category prefixes and batch size.
A large IPTV catalogue therefore produces a library of tens of thousands of titles, most of
which the user never wants. Separately, one title often exists as several **copies** (provider
account × category, e.g. a 4K English copy and an HD German copy), and the plugin picks one
without showing the user what differs between them.

## 2. Goal

Let the user choose, title by title, what becomes part of their media library, and which copy
of it, from a browsable page that shows each copy's quality, audio languages and subtitles.

### Non-goals

- Multi-user or per-user libraries. One user on a LAN, one shared selection.
- Modifying Dispatcharr's own frontend. The feature stays inside the plugin.
- Replacing the media server's metadata. NFO generation stays as it is.
- Multiple copies of one movie side by side (e.g. 4K *and* HD). See §12, open 1.

## 3. Summary of behaviour

| Area | Behaviour |
|---|---|
| Opt-in | Setting **Selection mode**, default **OFF**. When OFF, the plugin behaves exactly as before (verified on a test instance: classic Generate unchanged, page stopped). |
| Default state | Every movie and series is **ignored** until selected. |
| Movies | The user selects one **copy** (one `M3UMovieRelation`, identified by account + stream id). |
| Series | The user selects **one copy of the series** (account + external series id); all episodes come from that copy. Seasons can be **excluded**; seasons that appear later are **included automatically**. |
| Disk changes | Only when the user presses **Apply**, after a **Review & Apply** summary. Classic Generate / Full rescan buttons are paused. |
| Deselect | Apply **deletes the files it recorded writing** for that title, and folders left empty. |
| Existing library | A banner offers **Scan library** (reads the `.strm` files, previews) then **Adopt** (records them as selected and applied, touching no files) or **Skip**. |
| Copy info | Quality, codec and bitrate from data Dispatcharr already stores; languages from name/category tags; real audio and subtitle tracks from an **ffprobe** through Dispatcharr's VOD proxy (per title, or "probe all selected"). |
| Ranking | Preferences (audio languages, subtitle languages, quality order) and an optional **per-title audio override** sort copies best first. Titles already picked keep their copy; a better one shows as a hint. |
| Page | Served by the plugin on its own port, from Dispatcharr's `daphne` process. Table and poster grid, search, provider / category / decade filters, tabs incl. **New since last visit** and **Flagged**. |
| Category filter/exclude | Limit what the page lists. They never deselect anything: a selected title in a newly excluded category stays selected and still appears under Selected / Pending / Flagged. |
| Cron | In selection mode the `[SCHEDULE]` task **maintains the applied selection** (URL refresh, new episodes, fallback, `no_copy`, restore, relink). It never applies un-Applied changes. The page's **Refresh selected now** runs the same code. |

## 4. Platform constraints (verified against Dispatcharr source and a live v0.31.0)

1. **Plugins cannot add pages or routes.** A plugin gets a settings form and action buttons only
   (`Plugins.md`). Hence the plugin's own web server.
2. **Plugins are trusted, in-process server code.** They can use the Django ORM and Celery;
   `stop(context)` is called when the plugin is disabled, deleted or reloaded.
3. **The `Plugin` object is created in every process that runs discovery** (uWSGI workers,
   daphne, prefork Celery workers; `apps/plugins/loader.py:361`). The page must therefore be
   started in exactly one process (§6.1).
4. **Plugins cannot add Django models or migrations.** Selection state lives in plugin-owned
   SQLite under `/data` (§7).
5. **Copy metadata exists only partly.** `M3UMovieRelation.custom_properties["detailed_info"]`
   holds the provider's `get_vod_info` block, but only once someone fetched it (1 of 250 movies
   on the test provider). `M3UEpisodeRelation.custom_properties["info"]` holds per-episode data
   once a series' episodes were fetched. Provider audio blocks list a single, often wrong, track.
   **Every field is optional**, and languages need a probe.
6. **ffprobe is in the image** (ffmpeg base; `apps/channels/tasks.py` uses it).
7. **Several episode relations can point at one `Episode`** (different copies). Episodes must be
   filtered by `M3UEpisodeRelation.series_relation`, and each copy's episode title read from its
   own `custom_properties.info.title` (the shared `Episode.name` is whichever copy was fetched
   last).
8. **Content has `created_at`**, enough for "new since last visit".
9. **Dispatcharr can delete and re-create every title.** An empty provider response during a
   refresh makes its orphan cleanup delete all movies; the next refresh re-imports them with new
   uuids. Selection rows are keyed by uuid, so they must be relinked (§9.6).
10. **DB connections:** Dispatcharr uses `django_db_geventpool` with 8 connections per process.
    Threads outside Django's request cycle must call `close_old_connections()` after each ORM
    use, or the pool empties and every later query fails (`LoopExit`).
11. **Install/update needs a restart:** daphne only loads plugins at startup, and a running
    process keeps the old `selection` modules cached.

## 5. Architecture

```
 Browser (LAN)                        Dispatcharr container
 ───────────────                      ─────────────────────────────────────────────────────
 Selection page  ──HTTP :9192──►  daphne process
 (static HTML/JS)                   ├─ watchdog (30 s): start/stop the page server
                                    ├─ page server (ThreadingHTTPServer) + JSON API
                                    │    ├─ reads ──► Django ORM (Movie, Series, *Relation)
                                    │    ├─ reads/writes ──► selection.db (SQLite, /data)
                                    │    └─ background threads: Apply, probe all,
                                    │       adoption scan, upkeep ("Refresh selected now")
                                    ▼
                               Plugin helpers (naming, NFO, writes) ──► /VODS (.strm + .nfo)

 Celery beat ──► dvr worker: vod2mlib.scheduled_rescan ──► selection mode ON?
                                                            └─ run_scheduled_upkeep (same code)
```

- **New code lives in the `selection/` subpackage.** `plugin.py` gains the settings, one action,
  an import shim, the watchdog start in `__init__`, `stop()`, a guard in `run()` that pauses bulk
  generation (and routes the scheduled run to upkeep), the `_schedule_snapshot` helper, and
  `_episode_target_names`: classic mode's episode naming moved into a helper, unchanged, so
  both modes write identical paths.
- **Files are written by the existing Plugin helpers** (`_movie_target_paths`,
  `_episode_target_names`, `_build_proxy_url`, `_write_if_different_preserve_times`,
  `_generate_nfo`, …), so selection mode names files exactly as classic mode does.
- **No new runtime dependencies:** stdlib `http.server`, `sqlite3`, `subprocess` (ffprobe) and
  plain JS.

### Package layout

```
vod2mlib/
  plugin.py              # settings, action, hooks (see above)
  plugin.json            # mirrors the fields/action
  selection/
    __init__.py
    runtime.py           # glue: settings from PluginConfig, boot/tick/shutdown, status, scheduled upkeep
    server.py            # SelectionService (list, select, jobs) + HTTP handler, auth, CSRF
    store.py             # SQLite schema v7, migrations, desired/applied state, sessions, lease
    catalogue.py         # DjangoCatalogue (ORM) + FakeCatalogue for tests and the dev server
    copyinfo.py          # CopyInfo from provider data, probe merge, language match, ranking
    probe.py             # ffprobe through Dispatcharr's VOD proxy
    apply.py             # Apply: write-then-prune per title, recorded files only
    upkeep.py            # scheduled upkeep, fallback / no_copy / restore, mass-loss guard
    relink.py            # move rows whose uuid vanished to the re-created title
    adopt.py             # scan .strm files, adopt an existing library
    devserver.py         # python -m selection.devserver [--no-login]: page on fake data
    static/index.html    # the page: one file, plain JS, no build step
tests/test_selection.py  # fake catalogue, tmp_path filesystem, real HTTP handler
```

## 6. The page server

### 6.1 Lifecycle

- **Owner process: daphne.** `runtime.is_page_host()` is true only in Dispatcharr's daphne
  (ASGI) process. It exists in every layout, runs plugin discovery at startup, and isn't
  gevent-patched, so page work can't stall streaming greenlets (a uWSGI worker, where the first
  build landed, can).
- **Watchdog.** `Plugin.__init__` → `runtime.boot()` starts a 30 s watchdog in the page host.
  Each tick reads live settings: selection mode ON + a password → start the server if it isn't
  running; OFF → stop it. Binding the port doubles as the lock against a second copy.
- **Owner record.** `start_server` writes pid, port and start time to `meta.owner`;
  `[SELECTION] Page status` reports it from any process without binding, plus a port-mapping
  hint and whether the Movies root is writable.
- **Stopping.** `stop()` and turning the mode off shut the server down.

### 6.2 Authentication

- **Selection page password** (required; the server doesn't start without one).
- Login issues a random token in an `HttpOnly; SameSite=Strict` cookie, valid 7 days. Only a
  sha256 of the token is stored (`page_session`), tied to a sha256 of the password it was issued
  under, so changing the password ends every session. Sessions survive restarts.
- State-changing requests need the header `X-VOD2MLIB: 1` (CSRF).
- The page isn't behind Dispatcharr's authentication; the design target is a single user on a
  LAN.

### 6.3 JSON API

| Method & path | Purpose |
|---|---|
| `GET /api/session`, `POST /api/login`, `POST /api/logout` | Session. |
| `GET /api/movies`, `GET /api/series` | Paged listing. Query: `q`, `state` (`all` / `new` / `selected` / `ignored` / `pending` / `flagged`), `page`, `page_size`, `account`, `category`, `year` (`2010s`, `before-1950`, `none`). Items carry ranked copies with CopyInfo and match marks, `selected`, `on_disk`, `chosen`, `better_copy`, `flag`, `new`, `audio_override`. The response also carries `new_total`, `accounts`, `categories`, `decades`. |
| `GET /api/series/{uuid}/seasons?account_id=&stream_id=` | Seasons and episode counts of a series copy (fetches its episodes from the provider). |
| `PUT /api/selection/{kind}/{uuid}` | Desired state: `{selected, account_id, stream_id, excluded_seasons, title}`. |
| `PUT /api/override/{kind}/{uuid}` | Per-title audio override `{"audio": "FR" \| null}`. |
| `GET /api/pending`, `POST /api/apply`, `GET /api/job` | Review, start Apply (background thread), progress/result. |
| `POST /api/probe/{kind}/{uuid}` | Probe every copy of a title (synchronous, a few seconds per copy). |
| `POST /api/probe-all`, `POST /api/probe-all/stop`, `GET /api/probe-job` | Probe all selected titles in the background. |
| `POST /api/upkeep`, `GET /api/upkeep-job` | Refresh selected now (the scheduled upkeep, run from the page). |
| `GET /api/adopt`, `POST /api/adopt/scan`, `POST /api/adopt/apply`, `POST /api/adopt/skip` | Adoption of an existing library. |
| `GET/PUT /api/prefs` | Preferences. |
| `POST /api/seen/{kind}` | Mark all seen (moves the "new" clock). |
| `POST /api/clear-flag/{kind}/{uuid}` | Dismiss a flag. |

### 6.4 Page

- **Header:** Movies | Series, Table | Grid (remembered per browser), search, provider /
  category / decade dropdowns (each hidden when there's no choice), tabs All / New · N /
  Selected / Ignored / Pending / Flagged, Preferences, Refresh selected now + "last upkeep N min
  ago", Probe all selected.
- **Table:** checkbox, title (raw provider name), state and flag badges, copy menu
  (`✓/?/✗ EN 1080p H264 8.2 Mb/s`, provider id added when labels collide), a detail line (audio,
  subtitles, default-track warning, "probed N min ago"), "better copy: switch", Audio override
  menu, Seasons checklist, Probe.
- **Grid:** posters (TMDB via Dispatcharr's logo), ✓ and state badges; a click selects with the
  chosen or best copy, or unselects.
- **Bottom bar:** pending count, **Review & Apply** (adds / removes / copy and season changes),
  progress of Apply, probe and upkeep jobs.
- `<meta name="darkreader-lock">` + `color-scheme`, so browser dark-mode extensions don't
  recolour the page's own dark theme. Works at phone width.

## 7. Storage (`selection.db`)

SQLite at `/data/vod2mlib/selection.db`, plugin-owned, surviving plugin upgrades and uninstall.
`meta.schema_version` (now 7); later columns are added in place with `ALTER TABLE`.

**Copy identity** is `(account_id, stream_id)` for movies and `(account_id,
external_series_id)` for series (stored in `*_stream_id`), never the relation id.

```sql
meta(key PRIMARY KEY, value)
  -- schema_version, adopted_at, prefs (JSON), seen_at_movie / seen_at_series,
  -- owner (page server pid/port/start), disk_lease, last_upkeep (JSON summary)

selection(kind, content_uuid,               -- PRIMARY KEY; kind 'movie' | 'series'
  title,
  desired_selected, desired_account_id, desired_stream_id, desired_excluded_seasons,
  applied_selected, applied_account_id, applied_stream_id, applied_excluded_seasons,
  flag, flag_detail,                        -- 'fallback' | 'no_copy' | 'duplicates'
  audio_override,                           -- per-title audio language, never pending
  tmdb_id,                                  -- remembered for relinking
  last_error, updated_at)

applied_file(path PRIMARY KEY, kind, content_uuid)   -- every file Apply/upkeep/adoption wrote
copy_probe(kind, account_id, stream_id, probed_at, result JSON, error)
page_session(token_hash PRIMARY KEY, password_tag, expires_at)
```

**Pending** = selected state differs, or the title is selected and its copy, excluded seasons
or a `duplicates` flag differ from what's applied. Excluded seasons are stored as sorted JSON
int arrays, so only exclusions are recorded and new seasons are included by default.

Keeping **desired** and **applied** state separate is what makes "nothing changes until Apply"
and the review summary straightforward. Recording **applied files** makes deletion exact, even
if naming settings change later.

## 8. Copy information and ranking (`copyinfo.py`)

### 8.1 CopyInfo

```
{quality: '2160'|'1080'|'720'|'SD'|None, height, video_codec, bitrate_kbps,
 langs: [audio], subtitle_langs: [...], source: 'provider'|'deep',
 probed_at, probe_error, match: 'match'|'unknown'|'mismatch', default_audio_ok}
```

Empty lists mean **unknown**. No provider calls are made to build it:

- **Quality:** video width/height (width first: 1920×800 is 1080p), else `4K` / `UHD` /
  `1080p` / `FHD` / `720p` tokens in the copy's own name or its category.
- **Codec, bitrate:** a movie's `detailed_info`, or one episode's `info.info` for a series copy
  (loaded for a page of copies in one `DISTINCT ON` query).
- **Language:** tags in the copy's name or category only (`EN - `, `|FR|`, `▪NL▪`, …). The
  provider's audio block is ignored; it lists one track and is often wrong.

### 8.2 Deep probe (`probe.py`)

`ffprobe` on `http://127.0.0.1:$DISPATCHARR_PORT/proxy/vod/{movie|episode}/{uuid}?stream_id=…`,
i.e. **through Dispatcharr's own VOD proxy**, so account connection limits hold. A series copy
is probed through its first episode. It records resolution, codec, every audio track's
language (and the default track) and every subtitle language, in `copy_probe`. A failed
re-probe keeps the last good result plus the error. Probes run one at a time (module lock); a
503 "no available provider" is retried 4× with 2 s waits, then reported as "no free
connection". **Probe all selected** covers every copy of every selected title, skips copies
probed successfully in the last 7 days, and stops after 3 "no free connection" copies in a row.

### 8.3 Ranking

`rank_key(copy, prefs)`, best first:

1. **Audio language match**: match, then unknown, then mismatch (audio known from a probe or a
   tag; a dimension with nothing ticked always matches).
2. **Quality**, in the user's order (4K first or 1080p first).
3. **Preferred subtitles** (a preference, not a requirement).
4. **Default audio track** in a preferred language.
5. **Bitrate**, higher first.
6. **Probed** over unprobed.
7. Stable tiebreaker: account id, stream id.

**Per-title audio override:** that title's audio preference becomes the chosen language only,
and the global subtitle languages become required (`title_prefs`); a probed copy with no
subtitle tracks counts as unknown, since subtitles may be burned in.

The same function drives the listing order, the default copy of a newly selected title, the
"better copy: switch" hint (set when the top copy beats the chosen one on anything but the
tiebreaker), adoption when files play several copies, and upkeep fallback.

## 9. Flows

### 9.1 Selecting

`PUT /api/selection/...` writes the desired columns only. The page sends the top-ranked copy
when a title is first selected. Unselecting clears season exclusions.

### 9.2 Apply (`apply.py`, background thread in daphne)

1. Takes the **disk lease** (`meta.disk_lease`, 6 h TTL) so Apply and upkeep never write at
   once, and checks the root folder is writable (a clear error otherwise).
2. For each pending title:
   - **Remove:** delete every recorded file, then folders left empty.
   - **Add / copy change / season change: write-then-prune.** Write the new file set (a series
     first refreshes the copy's episodes: one provider call, before anything is removed), then
     delete recorded files that aren't in it. Unchanged files keep their mtime, so the media
     server doesn't re-index.
   - Existing `.nfo` files the plugin didn't write are never overwritten or claimed.
3. `mark_applied` copies desired → applied per title as each succeeds, so a failure leaves the
   rest pending. It clears `duplicates` flags, and any flag once a title ends unselected.

### 9.3 Adoption (`adopt.py`)

**Scan library** walks the movie and series roots and parses every `.strm`
(`/proxy/vod/<movie|episode>/<uuid>[?stream_id=…]`), which names the title and copy whatever
naming produced the file. Episode → its `series_relation` → series copy; no `stream_id` → the
top-ranked copy. The preview lists titles to adopt, titles already known, duplicates, and files
not adopted (content Dispatcharr no longer has, or not a Dispatcharr link). **Adopt** records
each new title as selected and applied, with its `.strm`, sibling `.nfo` and `tvshow.nfo` in
`applied_file`; nothing on disk changes. A title found in several places or as several copies
gets `flag='duplicates'` and is pending until Apply tidies the extras. Unrecognised files are
never recorded, so never deleted. **Skip** hides the banner.

### 9.4 Scheduled upkeep (`upkeep.py`)

Runs from the Celery task (`vod2mlib.scheduled_rescan` reads **live** settings, not the
schedule snapshot, and in selection mode calls `runtime.run_scheduled_upkeep`) or from the
page. Under the disk lease, for every **applied** title:

- **Copy still there:** rewrite its files (URL refresh; no-op writes keep mtimes). Series also
  get new episodes, except in excluded seasons. Series upkeep is additive inside the current
  show folder (a provider briefly listing fewer episodes deletes nothing), but removes recorded
  files in a show folder no longer produced (a rename), after writing the new ones.
- **Copy gone, another exists:** switch to the best remaining copy by the page's preferences,
  write before removing, `flag='fallback'`. The desired copy follows only if it was the applied
  one, so an un-Applied change is kept.
- **No copy left:** delete the files, keep the title selected and applied, `flag='no_copy'`.
  A later run that finds a copy writes it back (`flag='fallback'`, "A copy is available
  again").
- Desired-but-not-applied changes are left alone. The summary goes to `meta.last_upkeep`.

### 9.5 Mass-loss guard

If at least **5** applied titles, and more than **20%** of them, lose every copy in one run,
none of them are deleted or flagged; the run reports "N lost every copy at once; nothing
deleted" with status `partial`. This covers a provider outage or Dispatcharr's mass delete
(§4.9).

### 9.6 Relink (`relink.py`)

At the start of upkeep and before each page listing (skipped if the disk lease is taken): a row
that is selected, on disk or overridden, and whose uuid no longer exists, moves to the title
that now holds its copy (account + stream / series id), else to the only title with its
remembered TMDB id. `store.move_title` moves the row and its `applied_file` entries (merging
into an existing row). The next upkeep then rewrites the `.strm` URLs at the same paths.

## 10. Settings and actions

In a `[SELECTION MODE]` section of `plugin.py` / `plugin.json`:

| id | type | default | notes |
|---|---|---|---|
| `selection_mode` | boolean | `false` | Master switch. Pauses Generate / Full rescan; routes the schedule to upkeep. |
| `selection_port` | number | `9192` | Page port inside the container; publish it in docker-compose. |
| `selection_password` | string (password) | `""` | Required. Left out of the schedule's settings snapshot. |

One action: **`[SELECTION] Page status`**: running or not, owner pid and uptime, the port to
publish, and whether the Movies root is writable.

Preferences, overrides, season exclusions and the "seen" clock are edited on the page and
stored in `selection.db`, not in plugin settings.

## 11. Changes from the original design

| Original design | As built | Why |
|---|---|---|
| Apply, probe-all and the scheduled run as **Celery jobs** on the `dvr` queue, with a `job` table | Apply, probe-all, adoption scan and "Refresh selected now" run as **background threads in daphne**; progress is kept in memory and polled. The cron stays a Celery task. | The page must live in daphne anyway (§6.1). Believed at the time that the `dvr` worker never loads plugins; the live cron check later showed it does run the plugin's task. Threads were simpler and are serialised by the disk lease. |
| Server bound by **whichever process wins the port** | Only **daphne** serves the page | The first build landed in a gevent uWSGI worker, where page work could stall streams. |
| **Quick check** (`refresh_movie_advanced_data`) as a second info source | Not built (deferred) | Provider detail was nearly always missing and its audio block wrong; the deep probe gives real tracks. |
| **Probe concurrency** setting (1–3) | One probe at a time | The test account allows one connection; parallel probes just get 503s. |
| Actions **Open page**, **Start / restart page**, **Status** | Only **Page status** | The watchdog starts and stops the page by itself. |
| Separate **`ranking.py`**, **`diff.py`**, **`jobs.py`** | Ranking in `copyinfo.py`; pending is a SQL `WHERE` in `store.py`; jobs in `server.py` | Smaller modules weren't needed. |
| Ranking by toggle match → quality → subtitle **preference order** → source | Audio match → quality → subtitles → default track → bitrate → probed | Checkboxes carry no order; subtitles became a preference after every movie without them was marked ✗. |
| Adoption by **recomputing target paths** | Adoption by **reading `.strm` URLs**, preview then adopt | The URL names the title and copy whatever naming produced the file. |
| Copy change = **remove + add** | **Write-then-prune** | Keeps unchanged files' mtimes, so the media server doesn't re-index. |
| Sessions **in memory** | Hashed sessions in `selection.db` | Every restart logged the page out. |
| (not planned) | Relink + mass-loss guard, per-title audio override, "better copy" hint, provider / category / decade filters, poster grid, dev server | Found necessary or requested while testing. |
| Prefer the provider's clean `name` once detail exists | Dropped | Detail exists for 1/250 movies and no series; where it exists it matches the regex result. |

## 12. Decisions and open questions

### Decided

1. **No copy left:** delete the files, keep the title selected, flag it `no_copy`, restore it
   automatically when a copy reappears (§9.4).
2. **Selected titles later hidden by a category exclude stay selected**, and still appear under
   Selected / Pending / Flagged.
3. **Upstream fit** is agreed with the VOD2MLIB maintainer before merging.
4. **Page host: daphne** (§6.1). Consequence: install or update needs a Dispatcharr restart.
5. **Languages from tags and probes only**; the provider's audio block is ignored (§8.1).
6. **Subtitles are a preference, not a requirement**; the ✓/?/✗ mark is audio only.
7. **Per-title audio override** is picked by hand (no reliable original-language data: no TMDB
   key, provider `language` often wrong); it is a preference, never pending.
8. **Already-selected titles keep their copy** when preferences or probes change; a better copy
   is a hint, one click to switch.
9. **No bitrate from the probe** for now. Possible later: `format=bit_rate` in the same call
   (often missing for streamed MKV).
10. **Relink** by copy id, then TMDB id (only if exactly one title has it), automatically.
    **Mass-loss guard** at ≥5 titles and >20%.
11. **Page shows the provider's raw names**; name cleaning only affects the files written.
12. **"New since last visit"** moves only with **Mark all seen**, per kind, and starts at the
    first listing (nothing is new right after install).
13. **Provider filter** lists titles with a copy on that account (every copy still shown);
    **category filter** by name with counts; **decade filter** with "Before 1950" and "No
    year" (years before 1900 count as none). With a provider picked, the same copy must match
    both. Counts follow the provider filter only.
14. **Schedule snapshot** leaves out `selection_password` (it is readable in Django admin).

### Open

1. **One copy per movie.** Several (e.g. `Title (1999) - 4K.strm` next to `- 1080p.strm`) is
   possible later; Jellyfin supports multi-version naming.
2. **Provider priority** as a ranking tiebreaker: not requested.
3. **Mirror accounts:** a second line of the same provider (same stream ids and categories)
   shows unprobed copies that duplicate probed ones; sharing probe results between them is a
   candidate small change.
4. **Quick check** (§11): deferred.

## 13. Testing

- **`tests/test_selection.py`** (192 tests, no Django): a `FakeCatalogue` built from a small
  spec; store and migrations in `tmp_path`; Apply, upkeep, relink and adoption against a
  temporary filesystem (add / remove / copy change / seasons / write-then-prune / only recorded
  files deleted / user files and NFOs preserved); the real HTTP handler on a local port (auth,
  sessions, CSRF, bad parameters); copy info, ranking and filters; the DB-connection regression
  with fake Django modules.
- **Regression:** the classic suite (`tests/test_helpers.py`) is unchanged and passes. All
  tests pass on Linux; on Windows, 20 path-separator assertions in `test_helpers.py` fail on
  `main` as well.
- **Live** (test Dispatcharr v0.31.0, two accounts on one provider, 23k series): every slice was
  verified on the test instance, including Apply → playback in mpv, restart survival, the DB pool leak,
  relink after Dispatcharr re-created all movies, the cron through Celery beat, mode OFF running
  classic Generate unchanged, and fallback / `no_copy` / restore by disabling an account.

## 14. Status

Built and verified: movies and series with copy choice; seasons; copy info; deep probe and
probe all; preferences, ranking and per-title override; scheduled upkeep with fallback,
`no_copy` and restore; adoption; persistent sessions; relink and mass-loss guard; new since last
visit; poster grid; provider, category and decade filters; and live checks of the cron,
mode OFF, and fallback / restore.

Selection mode writes files through classic mode's naming helpers, so its folder and file
names are exactly the ones classic mode would write. The version bump and CHANGELOG are the
maintainer's.
