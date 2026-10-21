# MLM – Technical Documentation

Analysis of the MLM (Myanonamouse Library Manager) code base, version 0.5.0 (`server/Cargo.toml`).
Source analysed: branch `docs/ui-section`, about 23,000 lines of Rust (roughly 8,000 of which are DB schema history in `mlm_db/src/v*.rs`), a small amount of vanilla JS/CSS, and Askama HTML templates.

> Provenance (from `README.md`): this is a community-maintained continuation of StirlingMouse's MLM, whose original repository was taken down. The current maintainer says they are not a Rust developer and offer only bare-minimum fixes, with no guarantees.

---

## 1. What MLM does

MLM sits between **MyAnonamouse (MaM)**, a private torrent tracker for books, and **qBittorrent**. It has two optional halves:

1. **Auto-downloader.** It searches MaM, decides which torrents are worth having, and adds them to qBittorrent. It respects your ratio, unsat slots and wedge supply.
2. **Library organizer.** It watches qBittorrent for finished torrents and hardlinks (or copies/symlinks) the files into a clean `Author/Series/Title` tree. This tree is suitable for Audiobookshelf (ABS) and similar tools. It replaces lower-preference formats with better ones.

Either half can be used alone. A built-in web UI (default port 3157) shows state and lets you act manually. It ships as a Docker image and a Windows tray application.

### Feature list

| Area | Feature |
|---|---|
| **Autograbbers** (`[[autograb]]`) | Search sources: `bookmarks`, `freeleech`, `new`, `mine`, `uploader(<id>)`. Filters: query text with `search_in` fields, categories, media type, language, flags (tri-state), size range, uploader exclusions, upload date range, seeders/leechers/snatched ranges. Sorting: `low_seeders`, `low_snatches`, `oldest_first`, `random`. Per-grabber `search_interval`, `unsat_buffer`, `wedge_buffer`, `max_active_downloads`, `max_pages`, `category`, `dry_run`, `mark_removed`. |
| **Cost policy** | `free` (only freeleech/VIP), `wedge` (must use a wedge), `try_wedge` (wedge if possible), `ratio` (alias `all`: pay with ratio), `metadata_only` (update the DB, download nothing), `metadata_only_add` (also create DB rows for torrents you already have). |
| **Snatchlist import** (`[[snatchlist]]`) | Pulls your MaM snatch lists (types come from `SnatchlistType` in `mlm_mam/src/enums.rs`) and records them as metadata-only library entries. Only metadata costs are allowed there; language and date filters are rejected. |
| **Lists** | Goodreads RSS shelves (`[[goodreads_list]]`) and Notion databases (`[[notion_list]]`). Each list has ordered `grab` blocks (first match wins), `prefer_format`, and its own buffers and interval. |
| **Safety** | `min_ratio` (default 2), `unsat_buffer` (default 10), `wedge_buffer` (default 0), `ignore_torrents` list, `add_torrents_stopped`, rate-limit and HTTP 406 "account blocked" handling. |
| **Format preference** | Ordered `audio_types`, `ebook_types`, `music_types`, `radio_types`. MLM downloads only formats on the list and keeps the best one. If a better format of the same book arrives later, the older torrent is "replaced": its library files are removed and its qBittorrent entry is re-tagged or re-categorised. |
| **Tagging** | `[[tag]]` rules assign a qBittorrent category and tags when a torrent is added. Rules use the same filter set as autograbbers, and categories are first-match-wins while tags merge. |
| **Library organizer** (`[[library]]`) | A library is selected by qBittorrent `category` or `download_dir` (not both), optionally narrowed by `allow_tags`/`deny_tags`. Link methods: `hardlink` (default), `hardlink_or_copy`, `hardlink_or_symlink`, `copy`, `symlink`, `no_link`. Per-library format override. `path_mapping` translates qBittorrent save paths to paths visible to MLM. |
| **Layout** | `Author/Series/Series #N - Title {Narrator}/` for series and `Author/Title {Narrator}/` otherwise. Editions are appended after a comma. `exclude_narrator_in_library_dir` drops the `{Narrator}` part (compatible with "booktree"). Multi-disc folders are normalised to `Disc N`. A `metadata.json` in ABS format is written next to the files. |
| **Audiobookshelf** | Optional `[audiobookshelf]` section. A periodic matcher links library torrents to ABS items (`abs_id`), and a cleaned or replaced torrent can delete its ABS item. |
| **Metadata** | Parsed from MaM search results: authors, narrators, series with numbers, edition, language, flags, categories, filetypes, size, ISBN. Manual edits are possible in the UI (`MetadataSource::Manual`). |
| **Web UI** | Home with run-now buttons, torrents table, torrent detail and edit, events log, MaM search and download, lists, errors, selected, replaced, duplicate, and a read-only config page. Plus a small JSON API. |
| **Windows** | System tray icon (open UI, open config, open logs, quit), native error windows for config and port failures. Packaged with `cargo-packager`. |
| **Ops** | Env-var configuration, daily rolling log files, `--update-search-title` maintenance flag, DB export function (currently commented out in `main`). |

---

## 2. Tech stack

| Layer | Choice |
|---|---|
| Language | Rust, edition 2024; Docker build image `rust:1.91` |
| Async runtime | `tokio` (full), `tokio-stream`, `tokio-util`, `futures` |
| Web server | `axum` 0.8, `axum-extra` (forms), `tower`, `tower-http` (`ServeDir` for assets) |
| Templating | `askama` 0.14 (compile-time HTML templates in `server/templates/`) |
| Front-end | Hand-written JS and CSS in `server/assets/` (about 600 lines, no framework or bundler). A custom elements file plus `index.js` that intercepts `inline` forms. Type-checked via `tsc` with `checkJs` (`package.json` and `tsconfig.json` only exist for that). |
| Database | `native_db` (embedded, ACID, redb-based, pinned git rev) with `native_model` versioned models. One file, `data.db`. |
| HTTP clients | `reqwest` + `reqwest_cookie_store` for MaM and ABS; `qbit` crate (fork of `qbittorrent-webui-api`) for qBittorrent |
| Torrent files | `lava_torrent` (parse `.torrent`, compute info-hash) |
| Parsing | `serde`/`serde_json`, `serde-nested-json`, `quick-xml` (Goodreads RSS), `scraper` (Goodreads HTML), `regex`, `nom`, `htmlentity`, `unidecode`, `sublime_fuzzy`, `matchr` (fuzzy scoring) |
| Config | `figment` (TOML file plus `MLM_CONF_*` environment variables), `serde` with `deny_unknown_fields` |
| Observability | `tracing`, `tracing-subscriber` (pretty stderr + optional daily rolling file), `tracing-appender`, `tracing-panic` |
| Misc | `time`, `uuid`, `sanitize-filename` (fork), `openssl` (vendored), `dirs`, `file-id` |
| Windows only | `tray-item`, `winsafe` (GUI), `open`, `embed-resource` |
| Packaging | Multi-stage `Dockerfile` (dependency-caching layer, `debian:trixie-slim` runtime); `cargo-packager` metadata for Windows |
| CI | GitHub Actions: Docker (build, push to GHCR, cosign sign, nightly cron + `main` + `v*.*.*` tags), Windows (test and package, `main`/`windows` branches and tags), Docs (mdBook to GitHub Pages) |
| Docs | mdBook (`docs/`) |

Several dependencies are pinned to git revisions of third-party forks (`native_db`, `lava_torrent`, `qbit`, `sanitize-filename`). That is a supply-chain and maintenance consideration.

---

## 3. Repository layout

Cargo workspace with four crates:

```
Cargo.toml              workspace: server, mlm_db, mlm_parse, mlm_mam
mlm_parse/              Pure text helpers: title normalisation, name cleaning, edition parsing
mlm_db/                 Data model, migrations, helper impls (Size, Language, Flags, categories, series)
mlm_mam/                MaM HTTP client and API data types
server/                 The `mlm` binary: scheduler, grabbing, linking, web UI, Windows shell
  src/                  main.rs, config*, autograbber, torrent_downloader, linker, cleaner,
                        snatchlist, audiobookshelf, qbittorrent, stats, logging, exporter,
                        lists/, web/, windows/
  templates/            Askama templates (base, pages/, partials/)
  assets/               JS, CSS, icons (category icons under icons/cats/)
docs/                   mdBook sources (src/), book.toml, docker-compose.yml (local nginx)
.github/workflows/      docker-publish, windows-publish, docs
Dockerfile
```

Dependency direction: `server` → `mlm_mam` → `mlm_db` → `mlm_parse`.

### Per-crate size and purpose

- **`mlm_parse`** (about 290 lines, 3 tests). `normalize_title` produces the `title_search` key that is used to detect "the same book" across torrents. `parse_edition` extracts edition information from titles and tags. `clean_name` tidies author and narrator names.
- **`mlm_db`** (`lib.rs` plus `v01`–`v17` plus `impls/`). See section 5.
- **`mlm_mam`** (`api.rs`, `search.rs`, `user_data.rs`, `user_torrent.rs`, `meta.rs`, `enums.rs`, `serde.rs`). See section 6.
- **`server`**, the largest crate. `config_impl.rs` (about 1,200 lines) is the biggest file, mostly torrent-filter matching and its 50 tests.

---

## 4. Runtime architecture

`server/src/main.rs` starts everything and then runs the web server on the main task.

### Startup sequence

1. Set up logging. `MLM_LOG_DIR` sets the log directory (empty means stderr only), and `MLM_LOG` is the filter (default `mlm=debug`).
2. Locate config (`MLM_CONFIG_FILE`) and DB (`MLM_DB_FILE`). Platform defaults come from the `dirs` crate. In debug builds they are `./config.toml` and `./data.db`. A minimal config (`mam_id = ""`) is created if none exists.
3. Load config with Figment: TOML first, then `MLM_CONF_*` env vars override. Unknown keys are an error (`deny_unknown_fields`).
4. Open `native_db`, run migrations, optionally handle `--update-search-title` and exit.
5. Create the MaM client. With no `mam_id`, MLM still starts so the UI is reachable, but no background jobs run.
6. Spawn background tasks (below), build `Context`, and run the web server.

### Background tasks

Every task is a `tokio::spawn` loop that waits on either a timer (`sleep(60 * interval)`) or a `tokio::sync::watch` channel. The channels are how the UI's "run now" buttons trigger jobs (`Triggers` in `stats.rs`). An interval of 0 means "only run when triggered".

| Task | Interval setting | Work |
|---|---|---|
| Downloader | triggered | `grab_selected_torrents`: turn selected torrents into qBittorrent downloads |
| One task per `[[autograb]]` | `search_interval` (default 30 min) | `run_autograbber` |
| One task per `[[snatchlist]]` | `search_interval` | `run_snatchlist_search` |
| One task per list | `import_interval` (default 135 min; alias `goodreads_interval`) | `run_list_import` |
| Linker + cleaner, one task per `[[qbittorrent]]` | `link_interval` (default 10 min) | `link_torrents_to_library`, then `run_library_cleaner` |
| ABS matcher | `[audiobookshelf].interval` (default 10 min) | `match_torrents_to_abs` |

Each run records its start time and result in the shared `Stats` struct, which the Home page shows and live-updates over `/stats-updates`.

### Concurrency control

- **One writer at a time.** `DatabaseExt::rw_async()` takes a global `tokio::sync::Mutex` before opening a read-write transaction. `rw_try()` fails instead of waiting and is used for opportunistic writes such as saving the `mam_id` cookie.
- **One autograbber at a time** (`AUTOGRABBER_MUTEX`), even though each grabber has its own task. This serialises MaM traffic.
- **Politeness delays.** 400 ms between MaM search pages, 1 s between torrent downloads, and a 60 s cache on MaM user info.

### Pipeline (end to end)

```
 MaM search / list import / snatchlist
            │  select_torrents()  (filters, format preference, duplicate and library checks)
            ▼
   SelectedTorrent rows  ──►  downloader trigger
            │  grab_selected_torrents()  (unsat + ratio budget, wedge decision)
            ▼
   .torrent fetched from MaM  ──►  added to qBittorrent (category + tags)
            │  Torrent row created (id = info-hash)
            ▼
   qBittorrent finishes download
            │  link_torrents_to_library()  (every link_interval or on demand)
            ▼
   hardlinks/copies in library_dir + metadata.json   ──►  Linked event
            │  run_library_cleaner()
            ▼
   better format for same book?  ──►  old one cleaned, qbit re-tagged, ABS item deleted
            │  match_torrents_to_abs()
            ▼
   Torrent.abs_id set
```

---

## 5. Data model (`mlm_db`)

### Storage and versioning

`native_db` stores typed records. `native_model` gives every model an id and a `version`. `mlm_db/src/lib.rs` defines every historic version (`v01`–`v17`) in the `MODELS` table so old data can be read, and each new version implements `From<previous>`. `migrate()` runs `rw.migrate::<T>()` for each current type at startup. The type aliases at the top of `lib.rs` (`pub type Torrent = v17::Torrent`) point the rest of the code at the current version. To change the schema, add `v18.rs`, define the models and conversions, register them in `MODELS`, and update the aliases.

### Current records (v17 unless noted)

| Record | Key | Purpose |
|---|---|---|
| `Torrent` | primary `id` (info-hash, or a MaM-id string for metadata-only rows); unique secondary `mam_id`; secondary `title_search`, `created_at` | A torrent MLM knows about, with its `TorrentMeta`, library path and files, selected formats, category, linker name, `abs_id`, `replaced_with`, `library_mismatch`, `client_status` |
| `SelectedTorrent` | primary `mam_id`; unique optional `hash`; secondary `title_search` | A queued/in-flight download: dl link, cost, category, tags, buffers, grabber name, `created_at`/`started_at`/`removed_at` |
| `DuplicateTorrent` | primary `mam_id` | Torrents skipped because an equal or better one already exists |
| `ErroredTorrent` | primary `ErroredTorrentId` (`Grabber`/`Linker`/`Cleaner`/...) | Last error per stage, with metadata for display |
| `Event` | UUID, secondary `torrent_id`, `mam_id`, `created_at` | Audit log: `Grabbed`, `Linked`, `Cleaned`, `Updated` (field diffs), `RemovedFromMam` |
| `List` / `ListItem` (v05/v04) | | Goodreads/Notion list state and per-book search status (`TorrentStatus`) |
| `Config` (v01) | key/value | Persists the rotating `mam_id` cookie |

`TorrentMeta` (shared) holds `mam_id`, `vip_status`, `media_type` (Audiobook, Ebook, Musicology, Radio, Manga, ComicBook, PeriodicalEbook, PeriodicalAudiobook), categories, language, flag bits, filetypes, file count, size, title, edition, authors, narrators, series with entry numbers, metadata source (`Mam` or `Manual`) and upload time.

`title_search` is the central dedup key. Selecting and cleaning both do a range scan on it and then call `TorrentMeta::matches` to decide whether two entries are the same book.

---

## 6. MaM client (`mlm_mam`)

`api.rs` wraps `reqwest::Client` with a cookie jar:

- **Auth.** The `mam_id` cookie comes from config, or from the DB if a rotated one was stored (preferred, with fallback to config if the stored one fails). After any response, `store_cookies()` writes back the refreshed cookie, using `rw_try` so it never blocks. This is what lets the session survive MaM's cookie rotation.
- **Endpoints used.** `jsonLoad.php?snatch_summary=true` (ratio, up/down, wedges, unsat count and limit), `tor/js/loadSearchJSONbasic.php` (JSON POST search), `tor/download.php/<dl_hash>?tid=<id>[&fl]` (`&fl` spends a wedge), and `loadUserDetailsTorrents.php` (snatch list).
- **Errors.** HTTP 429 maps to `RateLimitError`, and HTTP 406 maps to `AccountBlockedError`. The downloader aborts its run when it sees the latter, to avoid hammering the site while blocked.
- **User agent.** `MLM/<version>`; 20 s request timeout.
- **Search types.** `search.rs` defines `SearchQuery`, `Tor`, `SearchFields`, `MaMTorrent` and `SearchError`. `MaMTorrent::as_meta()` converts a result into `TorrentMeta` and returns `MetaError::UnknownMediaType` for categories MLM can't classify. `enums.rs` has `SearchIn`, `SearchKind`, `SearchTarget`, `Categories` and `SnatchlistType`.

### Search syntax

MLM does not parse queries itself. `query` and the web Search box go to MaM's engine, which supports `@field` prefixes (title, author, narrator, series, description, filenames, fileTypes, tags), `"phrases"`, `|` for or, grouping and wildcards. For Goodreads, MLM builds `@title <cleaned title> @author ("A"|"B")`, stripping quotes, slashes and bracketed suffixes, expanding `and`/`&` to `(and|&)`, and using the pre-colon title as an alternative.

---

## 7. Component details

### 7.1 Autograbber (`autograbber.rs`)

`run_autograbber`:
1. Takes the global autograber lock and fetches MaM user info.
2. `max_torrents = unsat_limit − unsat_count − unsat_buffer`. If the grabber has `max_active_downloads`, it also subtracts selected torrents already tagged with this grabber.
3. If budget remains (or the cost is metadata-only), call `search_and_select_torrents`, then poke the downloader.

`search_torrents` translates config into MaM search fields (target/kind, sort, flags as a show/hide bitfield, language ids, sizes, dates), pages through results (up to `max_pages`, with 400 ms spacing; default 50 pages for bookmarks, freeleech and mine, 0 for others), and applies `TorrentFilter::matches` client-side as a second filter. With `mark_removed`, any gap in the returned id range for a torrent already in the DB is confirmed with a direct lookup. If MaM no longer has it, `client_status` becomes `RemovedFromMam` and an event is written.

`select_torrents` is the decision core, for each candidate in order:
1. Skip `ignore_torrents`. Skip unknown media types (logged).
2. In a write transaction: if already selected, update the stored meta/buffer if it changed and move on. If already in the library, update metadata when it changed (or add a linker-less row for `metadata_only_add`).
3. `metadata_only_add` creates a `Torrent` row; `metadata_only` stops here.
4. Require at least one wanted filetype (`config.preferred_types(media_type)`). Otherwise skip.
5. Compare with other selected torrents of the same `title_search`. If an equal or better preference exists, record a `DuplicateTorrent` and skip. If the new one is better, un-select the old one (and record it as a duplicate).
6. Do the same against the library. A worse candidate becomes a duplicate. A better one proceeds, which later makes the cleaner replace the old one.
7. Compute category and tags from `[[tag]]` rules, work out `TorrentCost` (VIP, personal freeleech and global freeleech win over the configured cost), insert a `SelectedTorrent`, and stop after `max_torrents`.

`dry_run` runs all the logic without writing.

### 7.2 Downloader (`torrent_downloader.rs`)

`grab_selected_torrents` computes the remaining ratio budget as `(uploaded − downloaded − bytes_in_flight) / min_ratio` and the free unsat slots, then loops over un-started selections:

- Skip if no unsat slots remain beyond the torrent's buffer, or if the size would exhaust the ratio budget.
- `grab_torrent` re-reads user info and decides the wedge. For `UseWedge`, it fails if `wedges <= wedge_buffer`. For `TryWedge`, it skips the wedge when stock is low or the torrent is already free/VIP. It then fetches the `.torrent` (refreshing a stale dl link once), computes the info-hash, and checks whether qBittorrent (any configured instance) already has it.
  - If it does and it is complete: insert a `Torrent` row and drop the selection. If it is in progress: mark `started_at`.
  - If the library already has it linked: drop the selection.
  - Otherwise, for non-wedge free costs, re-verify with MaM that the torrent is still free, bump the local unsat count, add it to qBittorrent with category and tags (`add_torrents_stopped` supported), create the `Torrent` row, mark `started_at`, and write a `Grabbed` event.
- A stale dl link removes the selection (it stays on the Errors page and can be restored). `AccountBlockedError` stops the whole run. Failures are stored via `update_errored_torrent`. There is a 1 s delay between grabs and a bounded retry for rate limits.
- Only the first `[[qbittorrent]]` instance receives new downloads. Linking is done for all of them.

### 7.3 Library linker (`linker.rs`)

`link_torrents_to_library` lists every torrent from qBittorrent and, for each fully downloaded one (`progress >= 1.0`):
- Removes the matching `SelectedTorrent`.
- For torrents MLM already tracks: syncs the linker name and category, detects "torrent not registered with this tracker" (marks `RemovedFromMam`), and for already-linked torrents recomputes the expected library path to flag `LibraryMismatch` (`NoLibrary`, `NewLibraryDir`, `NewPath`). Mismatches are shown in the UI and fixable there. It tolerates a flipped `exclude_narrator_in_library_dir` without flagging.
- For new ones: `find_library` picks the first library whose `category` or `download_dir` matches, honouring `deny_tags`/`allow_tags` (qBittorrent tags are a comma-separated string). `match_torrent` selects the best audio and best ebook extension from the library's or global ordered lists (one of each per torrent), queries MaM by hash for metadata, and handles torrents with unknown media types by applying the qBittorrent `on_invalid_torrent` category/tags.
- `link_torrent` builds the destination directory (`library_dir`), creates it, links only files with the selected extensions (using `map_path` for path mapping and `DISK_PATTERN` for "CD/Disc N" folders), writes `metadata.json` (ABS format), upserts the `Torrent` row with `library_path` and `library_files`, and writes a `Linked` event. Windows long-path handling and detailed "file not found" diagnostics are included.
- Hard-link failures from a different filesystem or a missing source get descriptive errors, and `AlreadyExists` is tolerated.

Also exposed: `refresh_metadata` and `refresh_metadata_relink` (re-fetch from MaM and move files if the computed path changed), used by the UI.

### 7.4 Library cleaner (`cleaner.rs`)

Scans linked torrents ordered by `title_search`, groups them into batches of the same book (`Torrent::matches`), and in each batch of two or more keeps the one with the best format rank. Ties are broken by total size of the linked files. Every other torrent is marked `replaced_with`, optionally re-categorised/tagged in qBittorrent (`on_cleaned`), has its library files removed (empty folders, `cover.jpg` and `metadata.json` too), and its ABS item deleted if the keeper lives at a different path. A `Cleaned` event is recorded. Replaced entries appear on the Replaced page and can be re-linked.

### 7.5 Lists (`lists/`)

- **Goodreads.** `run_goodreads_import` reads the RSS feed (`quick-xml`) and scrapes extra details (cover, authors, series) with `scraper`. For each book it searches the local library (`matchr` fuzzy scoring with `search_library`), then queries MaM with each `grab` block in order (`search_grab`). `select_torrent` picks the best hit per media type. `prefer_format` can suppress the ebook or audiobook if both exist. `check_cost` and `not_wanted` update the per-item status (`TorrentStatus`), and finally `select_torrents` is called with the list's buffers. Progress is stored as `ListItem`s, which the Lists pages display.
- **Notion.** `run_notion_import` queries a Notion data source over the REST API with a token, using a `mam_fields` mapping, and feeds the same machinery.

### 7.6 Snatchlist (`snatchlist.rs`)

Pages through the user's snatch list (250 rows per page, 400 ms pause) and creates or updates metadata-only `Torrent` rows so that existing snatches count as "in the library" for dedup purposes.

### 7.7 Audiobookshelf (`audiobookshelf.rs`)

A thin REST client. It looks up libraries and search results, matches a `Torrent` by ASIN/ISBN/title/author, stores the `abs_id`, can `update_book`/`delete_book`, and generates the `metadata.json` content (`create_metadata`) that the linker writes beside each book.

### 7.8 Logging and errors (`logging.rs`, `qbittorrent.rs`)

`update_errored_torrent` upserts or clears an `ErroredTorrent` for each pipeline stage, depending on the stage's result. `write_event` appends to the event log. `qbittorrent.rs` has a `CategoryCache` and `ensure_category_exists`, so categories are created in qBittorrent on demand.

---

## 8. Web layer (`server/src/web/`)

Axum router with the shared `Context { config, db, mam, stats, triggers }`. Pages return Askama templates. Mutations are plain HTML form POSTs to the same URL with an `action` field, and `assets/index.js` upgrades forms marked `inline` into `fetch` calls with optimistic UI templates.

| Route | Purpose |
|---|---|
| `/` | Home: last run and result of each background job, per-job "run now" (`run_linker`, `run_search`, `run_import`, `run_downloader`, `run_abs_matcher`), live updates via `/stats-updates` |
| `/torrents`, `/torrents/{id}`, `/torrents/{id}/edit`, `/torrents/{id}/{filename}` | Filterable, sortable, paginated table with bulk actions (`clean`, `refresh`, and more). Built-in "find problems" filters: bad title, bad authors, bad series, leading characters, subtitle, missing ebook, initials, series with holes, series authors. Detail page, manual metadata edit, torrent file download |
| `/events` | Audit log with field diffs |
| `/search` | MaM search and download, with wedge-over-size highlighting |
| `/lists`, `/lists/{id}` | Goodreads/Notion lists and per-book status |
| `/errors` | Per-stage errors with retry and remove |
| `/selected` | Queue of torrents about to be downloaded (remove, update) |
| `/replaced`, `/duplicate` | Superseded and skipped torrents, with actions |
| `/config` | Read-only config view (passwords hidden), plus "apply tag rule to existing torrents" |
| `/api/search`, `/api/torrents/{id}` | JSON for the UI and integrations. The second takes either a MaM id or an info-hash |
| `/assets/*` | Static files. `resolve_assets_dir()` checks `MLM_ASSETS_DIR`, then the Windows data dirs, then next to the executable, then the source tree |

`tables.rs` holds the shared filter, sort and column machinery (`ItemFilter`, `Key`, and so on) that the table pages reuse. There is **no authentication** on any route, so the UI should be bound to a trusted network or put behind a reverse proxy.

---

## 9. Configuration reference (summary)

Top-level keys and defaults (`config.rs`): `mam_id`, `web_host` ("0.0.0.0"), `web_port` (3157), `min_ratio` (2.0), `unsat_buffer` (10), `wedge_buffer` (0), `add_torrents_stopped` (false), `exclude_narrator_in_library_dir` (false), `search_interval` (30), `link_interval` (10), `import_interval` (135, alias `goodreads_interval`), `ignore_torrents`, `audio_types` (m4b, m4a, mp4, mp3, ogg), `ebook_types` (cbz, epub, pdf, mobi, azw3, azw, cbr), `music_types` (pdf, mp3), `radio_types` (mp3).

Tables: `[search] wedge_over`, `[audiobookshelf] url/token/interval`, `[[qbittorrent]] url/username/password/on_cleaned/on_invalid_torrent/path_mapping`, `[[autograb]]`, `[[snatchlist]]`, `[[goodreads_list]]`, `[[notion_list]]`, `[[tag]]`, `[[library]]`.

Environment variables: `MLM_CONFIG_FILE`, `MLM_DB_FILE`, `MLM_LOG_DIR`, `MLM_LOG`, `MLM_ASSETS_DIR`, and `MLM_CONF_<key>` overrides for config values.

Note: some documentation defaults are out of date relative to the code. `docs/src/full.md` shows `search_interval = 30`, `goodreads_interval = 60` and `wedge_buffer = 0`, but the code's import interval default is 135 minutes. Autograb `type` is documented loosely, and the code also accepts `mine` and `uploader`.

---

## 10. Build, deployment and CI

- **Docker.** The multi-stage `Dockerfile` pre-builds dependencies from stub crates for layer caching, then builds the real sources and touches the entry files to force a rebuild. The runtime image is `debian:trixie-slim` with CA certificates, `/mlm`, and `/server/assets`. It sets `MLM_CONFIG_FILE=/config/config.toml`, `MLM_DB_FILE=/data/data.db`, and an empty `MLM_LOG_DIR`. Typical compose mounts `/config`, `/data` and the media tree, with identical in-container paths for the qBittorrent downloads and library so hardlinks work.
- **Windows.** `windows_subsystem = "windows"` hides the console. A tray icon and error windows replace it. The installer is built via `cargo-packager`.
- **CI.** Docker (GHCR push, cosign signing, nightly), Windows (tests then package), Docs (mdBook to GitHub Pages).
- **Docs locally.** `docs/docker-compose.yml` (added this session, untracked) serves `docs/book` with nginx. Run `mdbook build` in `docs/` to refresh.

---

## 11. Tests and quality

About 70 unit tests exist, concentrated in:
- `server/src/config_impl.rs` (50): filter matching.
- `server/src/linker.rs` (7).
- `mlm_db` (7): series and size handling.
- `mlm_parse` (3): edition parsing.
- `mlm_mam` (4): URL building, enums, user-data parsing.

There are no integration tests against qBittorrent or MaM, no web-handler tests, and CI runs `cargo test -p mlm` on Windows.

---

## 12. Observations and risks

1. **No UI authentication.** Anyone who can reach port 3157 can trigger downloads, delete library files, and see the library.
2. **Secrets in plain text.** `mam_id`, qBittorrent credentials and the ABS/Notion tokens live in the TOML file. The MaM cookie is also stored in the DB.
3. **Forked, git-pinned dependencies** (`native_db`, `lava_torrent`, `qbit`, `sanitize-filename`) and a vendored OpenSSL raise the maintenance and supply-chain burden. The original upstream repo is gone.
4. **Schema sprawl.** 17 schema versions are kept in `mlm_db` and all stay registered in `MODELS`, so old versions can only be removed once no DB file predates them.
5. **MaM coupling.** The code relies on undocumented MaM endpoints and on scraping Goodreads HTML. Both can break without warning. The recent fixes (dl-link refresh, `&fl` wedge flag, 406 handling) show this fragility.
6. **Repetitive scheduler code.** `main.rs` repeats the same timer/trigger/stats loop for each job type. A helper would shorten it and reduce copy-paste bugs.
7. **Single qBittorrent target for downloads.** Only the first `[[qbittorrent]]` receives new torrents.
8. **Docs drift.** Defaults and some options in the mdBook differ from the code (see section 9).
9. **Fixed buffers.** Grabbing relies on cached MaM user info (60 s) plus local bookkeeping (`add_unsats`), so concurrent manual downloads on the site can briefly exceed the limits.
