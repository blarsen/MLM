# MLM – Functional Specification

MLM ("MyAnonamouse Library Manager") is a self-hosted service that connects the private book tracker **MyAnonamouse (MaM)** with a **BitTorrent client (qBittorrent)** and, optionally, **Audiobookshelf (ABS)**. This document describes what the application does and how it behaves. It is written so that the application can be rebuilt on any technology stack. It deliberately contains no implementation details beyond the external interfaces that must be honoured.

Conventions: "MUST/SHOULD" mark behaviour that has to be reproduced. Setting names are given in `code font` and are part of the user-facing contract. Reproducing them verbatim keeps existing user configuration files working.

---

## 1. Purpose and scope

MLM has two independent, optional halves:

1. **Auto-downloader.** It finds torrents on MaM that match user-defined rules, decides whether they are worth having, and adds them to qBittorrent. It protects the user's account by respecting ratio, unsatisfied-snatch ("unsat") slots and freeleech-wedge supply.
2. **Library organizer.** It watches qBittorrent for finished torrents and links the wanted files into a clean `Author/Series/Title` folder tree suitable for Audiobookshelf and similar tools. It writes metadata beside the files. It replaces lower-preference formats of the same book with better ones.

Either half can be used on its own. Torrents added manually to qBittorrent are organized just like automatically grabbed ones.

A built-in web UI shows state, lets the user trigger jobs, search MaM, inspect and edit metadata, and resolve problems.

### Limitations that are part of the product

- Only qBittorrent is supported as a torrent client. Only the first configured client receives new downloads. All configured clients are scanned for linking.
- The unit of management is a *torrent*. A collection torrent holding many books is treated as a single book.
- Only torrents from MaM can be managed. Metadata is only as good as MaM's.
- The web UI has no authentication. It is intended for a trusted network or behind a reverse proxy.

---

## 2. Concepts and glossary

| Term | Meaning |
|---|---|
| **MaM id** | Numeric tracker id of a torrent. |
| **Info-hash** | The torrent's content hash, used as the id of every torrent MLM manages in the client. |
| **Unsat slot** | MaM limits how many torrents the user may hold "unsatisfied". MLM reads the current count and the limit. |
| **Wedge** | A consumable that makes a torrent freeleech when applied at download time. |
| **Freeleech / VIP** | Torrents that do not count against download credit. Kinds: *global freeleech*, *personal freeleech*, *VIP* (permanent or with an expiry date). |
| **Media type** | Audiobook, Ebook, Musicology, Radio, Manga, Comic Book / Graphic Novel, Periodical Ebook, Periodical Audiobook. |
| **Library** | A configured rule mapping finished qBittorrent torrents to an output folder. |
| **Linking** | Making the wanted files of a finished torrent appear in the library folder (hardlink by default). |
| **Cleaning / replacing** | Retiring a library entry because a better version of the same book is now in the library. |
| **Metadata-only entry** | A known torrent with no files managed by MLM. It counts as "I already have this" for duplicate detection. |
| **Selected torrent** | A torrent that has passed the filters and is queued for download. |

---

## 3. Data MLM keeps

MLM needs a persistent store that survives restarts and supports schema upgrades without losing data. Everything below must persist.

### 3.1 Torrent metadata (shared structure)

Every known torrent carries these fields:

- MaM id; VIP status (not VIP / permanent / temporary until a date; may be unknown).
- Media type; fiction/non-fiction main category (optional).
- **Legacy category** (one value from the tracker's fixed category list, see §12) and **genre categories** (a set from the tracker's newer genre list).
- Language (one value). Flags: *crude language, violence, some explicit, explicit, abridged, lgbt*.
- File types present (list of extensions as the tracker reports them), file count, total size.
- Title; optional edition (a label such as "2nd Edition" plus a number); authors (ordered list); narrators (ordered list); series (list of name plus entry numbers, §6.3).
- Metadata source: **MaM** (fetched) or **Manual** (edited by the user).
- Upload time.

### 3.2 Records

| Record | Contents |
|---|---|
| **Torrent** (a book MLM knows) | id (info-hash; for metadata-only entries a string derived from the MaM id); MaM id; normalized title key (§6.1); metadata; created time; library path and list of linked files (relative paths); which audio and ebook extension were chosen; qBittorrent category last seen; name of the library rule that linked it; ABS id; "replaced with" (id and time) if retired; library mismatch state (§8.5); client status (*not in client* / *removed from MaM*); optional list-item reference (Goodreads book id). |
| **Selected torrent** | MaM id; download link; optional info-hash; cost decision (§7.4); category and tags to apply; unsat and wedge buffers in force; grabber name; metadata; normalized title; created, started and removed times. |
| **Duplicate torrent** | MaM id, metadata and the id of what it duplicates. Created when a candidate is skipped because an equal or better one exists. |
| **Errored torrent** | One row per (stage, torrent). Stages: Grabber (by MaM id), Linker (by hash), Cleaner (by hash). Holds the last error text, display metadata and time. A successful run of the stage clears it. |
| **Event** | Immutable audit entries (§3.3). |
| **List** and **list item** | State of each imported Goodreads list and, per book, search status for the audio and ebook side (§7.8). |
| **Config store** | Key/value store. Used to keep the most recent rotated MaM session cookie. |

### 3.3 Events

Each event has a time, optional torrent id, optional MaM id and one of these payloads:

- **Grabbed**: grabber name, cost, whether a wedge was used.
- **Linked**: linker (library) name, library path.
- **Cleaned**: library path, files removed.
- **Updated**: list of field changes (field name, from, to).
- **Removed from MaM**.

---

## 4. Configuration

MLM is configured by one structured text file (the original uses TOML). Unknown keys MUST be rejected with a clear error so typos are caught. Any top-level setting can also be overridden by an environment variable named `MLM_CONF_<setting>`. A missing config file is created with just an empty `mam_id`, so that the UI still starts.

If `mam_id` is empty, the application starts and serves the UI but runs no background jobs.

### 4.1 Top-level settings

| Setting | Default | Meaning |
|---|---|---|
| `mam_id` | (required) | MaM session id (cookie value) created in the user's MaM security preferences. |
| `web_host` / `web_port` | `0.0.0.0` / `3157` | Where the UI listens. |
| `min_ratio` | 2 | MLM never downloads a torrent that would take the account below this ratio. |
| `unsat_buffer` | 10 | Unsat slots always left free. |
| `wedge_buffer` | 0 | Wedges always left unused. |
| `add_torrents_stopped` | false | Add torrents to qBittorrent in stopped state. |
| `exclude_narrator_in_library_dir` | false | Leave `{Narrator}` out of folder names (§8.4). |
| `search_interval` | 30 (min) | Default interval for autograbbers and snatchlist jobs. |
| `link_interval` | 10 (min) | How often finished torrents are scanned. |
| `import_interval` | 135 (min) (alias `goodreads_interval`) | Default interval for list imports. |
| `ignore_torrents` | empty | MaM ids that must never be selected. |
| `audio_types` | m4b, m4a, mp4, mp3, ogg | Ordered preference (best first) for Audiobook and Periodical Audiobook. Formats not listed are neither downloaded nor linked. |
| `ebook_types` | cbz, epub, pdf, mobi, azw3, azw, cbr | Same for Ebook, Manga, Comic Book and Periodical Ebook. |
| `music_types` | pdf, mp3 | Same for Musicology. |
| `radio_types` | mp3 | Same for Radio. |

An interval of 0 means "run only when triggered from the UI".

### 4.2 Sections

- `[search]`: `wedge_over` (size). Used by the UI search page (§9.7).
- `[audiobookshelf]`: `url`, `token`, `interval` (default 10 min). Optional.
- `[[qbittorrent]]` (repeatable): `url`, `username`, `password`, `path_mapping` (map of path prefix as seen by qBittorrent to path prefix as seen by MLM), `on_cleaned` and `on_invalid_torrent` (each: optional `category`, list of `tags`).
- `[[autograb]]` (repeatable): §7.1.
- `[[snatchlist]]` (repeatable): §7.7.
- `[[goodreads_list]]` and `[[notion_list]]` (repeatable): §7.8.
- `[[tag]]` (repeatable): §7.5.
- `[[library]]` (repeatable): §8.1.

### 4.3 The shared torrent filter

`autograb`, `snatchlist`, each `grab` block of a list, and each `tag` block share one set of optional filter fields. A torrent matches only if it satisfies **every** field that is set. Unset fields impose nothing.

| Field | Semantics |
|---|---|
| `name` | Label for the block. It appears in logs, events and in the UI. |
| `media_type` | List of media types. The torrent must be one of them. |
| `categories` | Per main type (`audio`, `ebook`, `musicology`, `radio`): `true` = all categories of that type, `false` = none (type excluded), or a list of category names (§12). Default: audio and ebook all, musicology and radio none. |
| `languages` | List of language names (§12). Torrent language must be in the list. A torrent whose language cannot be determined fails. |
| `flags` | Per flag: `true` = flag required, `false` = flag forbidden, unset = ignored. Several may be set, so a block may both require and forbid. |
| `min_size` / `max_size` | Human sizes such as `15 MiB`, `1.2 GiB`, `100 KiB` (binary units; KB/MB/GB-style spellings also accepted). Inclusive. |
| `exclude_uploader` | Uploader names to reject. |
| `uploaded_after` / `uploaded_before` | Dates (`YYYY-MM-DD`). Inclusive of the whole day. |
| `min_/max_ seeders`, `leechers`, `snatched` | Inclusive numeric bounds. |

Matching also exists against *already stored* metadata (used when a tag rule is applied to existing library items, §9.9). In that mode, if a filter field needs information MLM did not store (size, uploader, dates, peer counts, or language/category/flags when absent), MLM falls back to asking MaM for the torrent's current data.

---

## 5. External systems

### 5.1 MaM

MLM must be able to:

- **Read account state**: ratio, uploaded and downloaded totals, wedge count, unsat count and unsat limit. Cache for about 60 seconds.
- **Search torrents** with: free text and the fields to search in (author, description, filenames, filetypes, narrator, series, tags, title), search target (bookmarks, new torrents, own uploads "mine", torrents by uploader id), kind (e.g. freeleech, free-or-VIP), language ids, main categories and category ids, upload date range, size range, seeders/leechers/snatched ranges, flag show/hide filter, sort order (default newest first, plus oldest first, random, lowest seeders, lowest snatches), page offset (up to 100 per page), and "return download link".
- **Look up one torrent** by MaM id or by info-hash.
- **Download the `.torrent` file**. Optionally with the "use wedge" option.
- **Read the user's snatch lists** (kinds: unsat, inactive-unsat, seeding-unsat, seeding-satisfied, inactive-satisfied, active uploads), 250 rows per page.

Behavioural requirements:

- Authentication is by session cookie. MaM rotates the cookie. MLM MUST persist the newest cookie after each response (without blocking other work) and prefer the stored one on the next start, falling back to the configured one if the stored one fails.
- HTTP 429 means rate limited. HTTP 406 means the account is temporarily blocked: any download run MUST abort on it.
- Politeness: 400 ms between search pages, 1 s between torrent downloads, 20 s request timeout, identifiable user agent `MLM/<version>`.
- Only one autograbber and one list import may talk to MaM at a time.

MaM search results carry: id, title, authors, narrators, series (name plus number), description, ISBN or ASIN (`ASIN:`-prefixed), media type, legacy category, genre categories, language, flags (bit field), file types (space separated), file count, size, tags text, uploader name and id, added time, seeders, leechers, times completed, free / personal-freeleech / VIP (with expiry) markers, and a download hash.

### 5.2 qBittorrent

Required operations: log in; list torrents (hash, name, save path, category, tags (comma separated), progress, state); list a torrent's files; add a `.torrent` with category, tags and optional stopped state; set category; add and remove tags; create a category on demand (and reuse a cache so it is created only once); start; stop; delete torrent with data; check whether a hash already exists.

A torrent counts as finished when progress is 100%.

### 5.3 Audiobookshelf

Required operations (API-key authenticated): list libraries with their folders; search a library by author; fetch an author's items; update a book's metadata; delete a book.

### 5.4 Goodreads and Notion

- Goodreads: fetch the shelf's RSS feed, plus scrape extra book details (cover, authors, series) from the book's page. A browser-like user agent is required.
- Notion: query a data source with a bearer token.

---

## 6. Metadata handling

### 6.1 Normalized title (`title_search`)

Used as the "same book" key. Steps: transliterate to ASCII, lowercase, replace ` & ` with ` and `, remove a leading "the ", "a " or "an ", remove all characters that are not letters, digits, underscore or space, and remove the words "volume" and "vol.".

### 6.2 "Same book" test

Two torrents are the same book if all hold:

1. Media types are equal, except that Ebook and Comic Book are interchangeable.
2. Languages are equal.
3. Editions agree: both have none, or both have one with the same number. If the number is 0, the labels must match too. (If exactly one has an edition they are *different*.)
4. They share at least one author.
5. Both have no narrators, or they share at least one narrator.

Candidates are first pre-selected by equal normalized title, then confirmed with this test.

### 6.3 Series

A series is a name plus a list of entries. An entry is a number (`3`, `1.5`, `.5`), a range (`1-3`), or a part (`2p1` = book 2 part 1). Entries are comma-separated. Unparseable numbers yield a series with no entries (warn, don't fail). Display form: numbers as written, ranges `a-b`, parts `NpM`. Entries sort by their first number, ties: single < part < range.

### 6.4 Cleaning incoming MaM metadata

Applied to every record built from MaM data:

- HTML entities in names and series are decoded.
- **Names** (authors, narrators): in each space-separated word longer than 3 characters, an all-caps word is lower-cased after its first letter; an all-lowercase word gets its first letter capitalized.
- Series names: remove trailing "(Digital)" / "(Light Novel)".
- If the media type is Ebook but the legacy category is an audio category, treat it as an Audiobook.
- **Edition extraction**: look in the title, then the tags text, for phrases such as "2nd Edition", "(First edition)", "[Second Canadian Edition]", "- 25th Anniversary Edition:", "2012 Edition", "3rd ed.". Result: title with the phrase removed (any text after the phrase becomes `: subtitle`), edition label with each word capitalized (`ed.` → `Edition`), and the number (ordinal words/numerals, or 0 if none such as "UK Edition"). A phrase in the tags only sets the edition and leaves the title.
- Unless the type is periodical, with exactly one author: strip a trailing " by <Author>" / " - <Author>" or leading "<Author> - " from the title.
- Remove noise from titles: trailing ": A Novel" / ": A Memoir", "(digital)", "[epub]", "(unabridged)", "(light novel)" and similar format markers, and "- 123.epub" suffixes. Trim.
- Records MaM returns with an unknown media type, category, language, size or date are rejected with a descriptive error, and logged/shown rather than silently dropped.

### 6.5 Manual edits

The user can override title, authors, narrators, series, language, category and flags on a torrent in the UI. The source becomes *Manual*. The media type follows the chosen category. Authors and narrators are entered one per line; series as `Name #entries` lines.

---

## 7. Auto-downloader

### 7.1 Autograbber (`[[autograb]]`)

An autograbber is a periodic MaM search. Settings: `type`, `cost`, `query`, `search_in`, `sort_by`, `max_pages`, the shared filter (§4.3), `search_interval`, `unsat_buffer`, `wedge_buffer`, `max_active_downloads`, `dry_run`, `mark_removed`, `category`, `name`.

- `type`: `bookmarks`, `freeleech` (global freeleech list only), `new`, `mine`, or `{ uploader = <id> }`.
- `cost`: see §7.4. Default `free`.
- `sort_by`: `oldest_first`, `random`, `low_seeders`, `low_snatches`. Default newest first.
- `max_pages`: pages of 100 results. Default 50 for bookmarks, freeleech and mine; 1 for others.
- `category`: qBittorrent category forced on everything this grabber downloads, overriding tag rules.
- `max_active_downloads`: cap on selected-but-not-yet-finished torrents attributed to this grabber (requires `name`).
- `dry_run`: run everything but write nothing and download nothing; only log.
- `mark_removed`: for torrents MLM already stores, if the search results show an id gap where a stored torrent should be, ask MaM directly. If MaM no longer has it, set its client status *removed from MaM* and write an event.

Run procedure:

1. Take the global autograbber lock. Read account state.
2. Budget = unsat limit − unsat count − buffer (grabber's `unsat_buffer` or the global one). If `max_active_downloads` is set, budget = min(budget, that cap − currently active for this grabber). If the budget is 0 and the cost is not metadata-only, do nothing.
3. Search MaM, translating the filter into search parameters (flags into a show/hide set, languages into ids, sizes, dates, categories). Page through results with spacing. Then apply the full filter again locally to each result.
4. Run selection (§7.2) up to the budget.
5. Trigger the downloader (§7.3).

### 7.2 Selection

For each candidate torrent, in order:

1. Skip if its MaM id is in `ignore_torrents`. Skip and log if its metadata can't be built.
2. If already selected: refresh its stored metadata and buffers if changed, then continue.
3. If already in the library: refresh the library metadata if it changed (recording an *Updated* event with the field changes); then continue. For cost `metadata_only_add`, if absent, create a metadata-only entry.
4. If cost is `metadata_only` or `metadata_only_add`: stop here.
5. Require at least one file type from the preferred list for the media type. Otherwise skip.
6. **Duplicates among selected**: if another selected torrent is the same book (§6.2): if that one has an equal or better format rank (§8.6), record the candidate as a *duplicate* and skip. If the candidate is better, un-select the old one (recording it as a duplicate) and continue.
7. **Duplicates in the library**: same comparison against library torrents. A worse or equal candidate becomes a duplicate. A better candidate proceeds; the cleaner later retires the old entry.
8. Determine category and tags: category = the grabber's `category`, else the first matching `[[tag]]` rule with a category; tags = union of tags from all matching rules (§7.5).
9. Determine cost (§7.4) and store a Selected torrent with the buffers in force and the grabber name.
10. Stop when the budget is used up.

### 7.3 Downloader

Runs whenever selections were added, or on UI request.

Budgets: the ratio budget is `(uploaded − downloaded − bytes_in_flight) / min_ratio`. Free unsat slots = limit − count. Unstarted selected torrents are processed in order:

- Skip if no unsat slot remains beyond that torrent's own buffer, or if its size would exceed the remaining ratio budget (free costs do not use ratio budget).
- Wedge decision: `UseWedge` fails if wedges ≤ `wedge_buffer`. `TryWedge` skips the wedge if stock is at/below the buffer or the torrent is already free. Otherwise it uses one.
- Fetch the `.torrent` file (if the stored link is stale, refresh it once; if still stale remove the selection and leave it visible as an error). Compute the info-hash and check whether any configured client already has it:
  - Finished in client: create the Torrent record, drop the selection.
  - Downloading in client: mark the selection as started.
  - Already linked in the library: drop the selection.
- Otherwise, for non-wedge free costs, re-verify with MaM that the torrent is still free. Then add it to the first qBittorrent with the chosen category and tags (creating the category if needed). Honour `add_torrents_stopped`. Create the Torrent record (id = hash), mark the selection started, and write a *Grabbed* event.
- Local unsat count is incremented so later items in the same run see it.
- 1 s pause between downloads. Errors are stored per selected torrent (stage Grabber). A 406 aborts the whole run.

### 7.4 Cost policy

Config `cost` values: `free` (default), `wedge`, `try_wedge`, `ratio` (alias `all`), `metadata_only`, `metadata_only_add`.

Selected-torrent cost is derived as: VIP → *Vip*; personal freeleech → *PersonalFreeleech*; global freeleech → *GlobalFreeleech* (these three always win over the configured cost, so nothing is wedged needlessly); otherwise `wedge` → *UseWedge*, `try_wedge` → *TryWedge*, else *Ratio*. With configured cost `free`, a torrent that is not free by one of those routes is not selected.

### 7.5 Tag rules (`[[tag]]`)

Each block = shared filter + optional `category` + optional `tags`. For a torrent: the category is taken from the **first** matching rule that defines one; tags are the **union** over all matching rules. These are applied to qBittorrent when the torrent is added.

### 7.6 Format preference

Per media type there is an ordered list of extensions (§4.1). A torrent's *rank* is the position of the first listed extension present among its file types (lower is better; none present = worst). MLM only downloads torrents with a listed extension. Between two versions of the same book, the lower rank wins.

### 7.7 Snatchlist import (`[[snatchlist]]`)

Settings: `type` (unsat, inact_unsat, seed_unsat, seed_sat, inact_sat, uploads_active), `cost` (only `metadata_only` / `metadata_only_add` allowed), `max_pages`, shared filter (language and date filters are NOT allowed and cause an error), `search_interval`, `dry_run`.

It pages the user's snatch list and creates or updates metadata-only entries so that torrents the user already has count as owned for duplicate detection.

### 7.8 Lists

**Goodreads list** (`[[goodreads_list]]`): `url` (RSS of a shelf), `name`, `prefer_format` (`audio` or `ebook`), one or more `grab` blocks (each: `cost` + shared filter), `search_interval`, `unsat_buffer`, `wedge_buffer`, `dry_run`. List id is `<goodreads-user-id>:<shelf>`.

Per run: for each book on the shelf (series info is extracted from titles like "Title (Series, #3)"):

1. Create/update the list item. Skip books that already have both formats found, or the preferred format found.
2. Search the local library with fuzzy title+author matching; mark sides as *existing* if owned.
3. For each `grab` block in order, search MaM with `@title <title> @author ("A"|"B")`. Quotes, slashes and bracketed suffixes are stripped from the title. "and"/"&" is expanded to `(and|&)`. The part before a colon is also tried alone. Apply that block's filter. The first block that yields a hit per media side wins.
4. Choose the best hit per side (audio, ebook).
5. If both sides were found and `prefer_format` is set, mark the non-preferred side *not wanted*.
6. Update each side's status (*Selected, Wanted, Not wanted, Existing*) and run selection (§7.2) with the block's cost.
7. 400 ms between books.

Which sides a list *can* want comes from its grab blocks' category settings (a list whose blocks all exclude ebooks never wants ebooks).

**Notion list** (`[[notion_list]]`): `data_source`, `token`, `name`, `mam_fields` (names of URL properties), `grab` blocks. Each row's URL properties are read, the trailing number of each URL is taken as MaM id. Ids already known are skipped. Others are fetched from MaM and pushed through the first matching grab block's selection.

---

## 8. Library organizer

### 8.1 Libraries (`[[library]]`)

A library maps finished torrents to an output folder: `library_dir` plus **either** `category` (match qBittorrent category) **or** `download_dir` (match torrents whose save path is under it). Not both. Optional: `name`, `method`, `allow_tags`, `deny_tags`, `audio_types`, `ebook_types`.

The first library, in config order, that matches wins. A torrent is rejected from a library if it has **any** `deny_tags`. If `allow_tags` is non-empty it must have **any** one of them.

`method`: `hardlink` (default), `hardlink_or_copy`, `hardlink_or_symlink`, `copy`, `symlink`, `no_link` (record metadata only, no files).

Per-library `audio_types` / `ebook_types` override the global preference lists.

### 8.2 Link run

Runs every `link_interval`, after each download run, and on UI request, for every configured qBittorrent. For each torrent that is 100% complete:

1. Remove any matching Selected torrent.
2. If MLM already tracks it: sync category and linker name; if the client reports the tracker no longer knows the torrent, set *removed from MaM*; if already linked, recompute the expected folder and flag a mismatch (§8.5).
3. Otherwise find its library (§8.1). If none, ignore it.
4. Query MaM by info-hash for metadata. If the media type is unknown, apply the client's `on_invalid_torrent` category/tags and record an error.
5. **Select formats**: the best audio extension and the best ebook extension present among the files (using the preference lists; case-insensitive). At most one of each. Files with other extensions are ignored.
6. Link (§8.3). Record success or error per torrent (stage Linker). Errors are shown on the Errors page.

### 8.3 Linking

- A torrent without any author cannot be linked (error).
- Destination folder per §8.4. Create it.
- For every file with the chosen audio or ebook extension, link it into the destination. If the file lives in a sub-folder whose name contains `CD`, `Disc` or `Disk` followed by a number, put it in a `Disc N` sub-folder. Otherwise the file goes directly in the destination. Other folder structure is flattened.
- The source path is the qBittorrent save path translated through `path_mapping` (longest-prefix match; replaces the matched prefix).
- Method fallbacks per §8.1. "Already exists" is tolerated. Cross-device and missing-source failures produce clear diagnostic messages.
- Write `metadata.json` in the destination (§8.7).
- Save the Torrent record with library path, linked file list, chosen formats, category and linker. Write a *Linked* event.

### 8.4 Folder naming

Under `library_dir`, with each component sanitized to be a valid file name:

- With series: `Author/Series/Series #<entries> - Title` (the first series that has entries, else the first series; if that series has no entries, the last component is just the Title).
- Without series: `Author/Title`.
- Author = first author.
- If there is an edition: append `, <Edition>` to the last component.
- If there are narrators and `exclude_narrator_in_library_dir` is false: append ` {<first narrator>}`.
- If `exclude_narrator_in_library_dir` is true, narrators are omitted, but if the narrator-less folder already exists on disk (and the torrent has narrators), the narrator form is used instead to avoid collisions.

Examples: `Brandon Sanderson/Stormlight Archive/Stormlight Archive #1 - The Way of Kings {Michael Kramer}`; `Andy Weir/Project Hail Mary {Ray Porter}`; for ebooks the same without `{…}`.

### 8.5 Library mismatch

For already linked torrents, when the expected folder (given current metadata and config) differs from the stored one, flag it: *No library* (no library rule matches any more), *New library dir*, or *New path*. The UI displays the state and offers refresh-and-relink, which moves the files. A flipped `exclude_narrator_in_library_dir` alone does not count as a mismatch.

### 8.6 Cleaner (replacement of lesser formats)

Runs after each link run. Linked torrents are grouped by same-book test (§6.2). In each group of two or more:

1. Keep the torrent with the best format rank (§7.6). Ties: the one with the larger total size of its linked files.
2. Every other member is *replaced*: mark `replaced with = keeper` and time; in qBittorrent apply the client's `on_cleaned` category and tags (creating the category if needed); remove its linked files, then the folder if empty, plus `cover.jpg` and `metadata.json`; if ABS is configured and the keeper is at a different path, delete the old ABS item; clear its library path, ABS id and mismatch; write a *Cleaned* event.

The same operation can be invoked manually from the UI ("clean") with ABS deletion on. Replaced entries remain visible and can be relinked.

### 8.7 `metadata.json`

An Audiobookshelf-compatible file written next to the files with: `authors`, `narrators`, `series` (display strings `Name #entries`), `title` and `subtitle` (split at the first colon, unless the part before it is shorter than 4 characters), `description`, `isbn` and `asin` (from the tracker's ISBN field: a value starting `ASIN:` is the asin, otherwise the isbn), `tags` (`["LGBT"]` if the flag is set), `genres` (legacy category plus genre categories, de-duplicated), `language`, `explicit`, `abridged`.

### 8.8 Refresh

"Refresh" re-fetches a torrent's metadata from MaM and stores changes (as an *Updated* event, and updating ABS if linked). "Refresh and relink" additionally moves the files if the computed folder changed. A manual-source torrent keeps its manual metadata unless explicitly refreshed.

---

## 9. Audiobookshelf integration

If `[audiobookshelf]` is configured:

- **Matcher** (every `interval` minutes, or on demand): for each linked torrent without an ABS id, find the ABS library whose folders contain the torrent's library path, search it by first author, fetch that author's items and match the item whose path equals the library path. Store the ABS id.
- After linking or a metadata refresh, push updated metadata to ABS for matched items.
- On clean/replace, delete the ABS item (§8.6).
- The UI shows a link to the ABS item.

Without this section, ABS still benefits from `metadata.json`.

---

## 10. Web UI

Server-rendered pages with simple form posts (mutations are actions posted to the same page; some forms update in place without reloading). No authentication.

### 10.1 Home `/`

For every configured job (each autograbber, snatchlist, list import, linker, cleaner, downloader, ABS matcher) shows the label, the last start time and result (success or the error). Each has a **Run now** button (run linker, search, import, downloader, ABS matcher). The page live-updates when jobs finish.

### 10.2 Torrents `/torrents`

All known torrents, newest first. Includes:

- Free-text query (fuzzy match over title/authors/series), column filters by clicking values (media type, category, genre categories, flags, title, author, narrator, series, language, filetype, linker, qBittorrent category, metadata source, library mismatch, client status, etc.).
- Sort by any column, selectable columns, page size and paging.
- Bulk actions on selected rows: **clean**, **refresh**, **remove record**.
- **Metadata problem finders** (each lists torrents likely to have bad metadata): duplicated titles that differ only in normalization; same author spelled differently; same series spelled differently; leading "The/A" differences; titles containing subtitles; audiobooks with no ebook counterpart; bunched initials in names (e.g. "JK" instead of "J K"); series with holes in their numbering (among linked torrents); series with inconsistent author sets.

### 10.3 Torrent detail `/torrents/{id}`

Shows all metadata, library location and files, ABS link, MaM link, events for this torrent, qBittorrent state (progress, category, tags, files) when available. Actions: select/wedge download (for known but not-downloaded torrents), clean, refresh, refresh-and-relink, remove record, start/stop in qBittorrent, remove from qBittorrent with data, clear "replaced" marker, set qBittorrent category and tags (adds missing tags, removes unticked ones). Also download the `.torrent` file. Edit page `/torrents/{id}/edit` implements §6.5.

### 10.4 Other pages

- **Events** `/events`: audit log, newest first, with field diffs; filterable by type, grabber, linker, field.
- **Selected** `/selected`: queue of selected torrents with cost, category, tags, grabber, times; actions: remove, update (change category/tags/cost).
- **Errors** `/errors`: per stage error list (Grabber, Linker, Cleaner) with timestamp. Actions: remove entry; for grabber errors restore the selection.
- **Replaced** `/replaced`: retired torrents with what replaced them. Actions: refresh, refresh-and-relink, remove.
- **Duplicate** `/duplicate`: torrents skipped as duplicates with the entry they duplicate; actions: replace (force-select the duplicate and retire the other), remove.
- **Lists** `/lists`, `/lists/{id}`: imported lists and, per book, the audio/ebook status; filter by any/audio/ebook wanted; action "mark done".
- **Search** `/search`: manual MaM search with filters. Each result shows the cost (freeleech, VIP, snatched, etc. icons), and **download** or **download with wedge** buttons. Torrents larger than `[search] wedge_over` get the wedge button highlighted by default. These start a selection with category/tags from tag rules.
- **Config** `/config`: read-only view of the active configuration with secrets hidden, a link to run each autograbber's search on MaM, and for each tag rule a button "apply to existing torrents" (§10.5).

### 10.5 Applying a tag rule retroactively

For every stored torrent matching the rule (using stored metadata, falling back to MaM as in §4.3) set the rule's category and add its tags in the chosen qBittorrent instance.

### 10.6 JSON API

- `GET /api/search`/`POST /api/search`: search MaM; returns results with the current state (downloaded, selected, etc.).
- `GET /api/torrents/{id}`: accepts a MaM id or info-hash. Returns stored torrent + client state, or if unknown, the MaM record and cleaned metadata.

### 10.7 Static assets

Icons for every tracker category, flag, freeleech, VIP, snatched; a stylesheet; small scripts for in-place form submission and live updates. Favicon differs in development builds.

---

## 11. Platform behaviour

- **Docker**: config at `/config/config.toml`, database at `/data/data.db`, log to stderr only. Library and download paths must be identical inside the container for hardlinks to work.
- **Windows**: runs without a console window, as a tray application. Menu: open UI, open config, open logs, quit. Native error dialogs for invalid config and for failure to bind the port. Data in the user's application-data folders. Long-path-safe file operations.
- Environment variables: `MLM_CONFIG_FILE`, `MLM_DB_FILE`, `MLM_LOG_DIR` (empty = console only; otherwise daily rotating log files), `MLM_LOG` (log filter, default debug for MLM), `MLM_ASSETS_DIR`, `MLM_CONF_*` overrides.
- A maintenance flag recomputes the normalized title of all records (used after the algorithm in §6.1 changes).
- **Scheduling**: every job is a loop that waits for its interval or an explicit trigger. Only one database write happens at a time. A job that fails records the error and retries at the next interval. A failing item never stops its siblings.
- **Persistence upgrades**: a database written by any older version of MLM MUST open and be upgraded in place, without data loss.

---

## 12. Reference data

### 12.1 Media types (tracker id → name)
1 Audiobook, 2 Ebook, 3 Musicology, 4 Radio, 5 Manga, 6 Comic Book / Graphic Novel, 7 Periodical Ebook, 8 Periodical Audiobook. Older records carry only a main category: 13 audio, 14 ebook, 15 musicology, 16 radio; these map to media types 1–4. Fiction/Non-fiction main category: 1 = Fiction, 2 = Nonfiction.

### 12.2 Flags (bit in tracker flag byte)
bit 1 crude language, 2 violence, 3 some explicit, 4 explicit, 5 abridged, 6 lgbt. In configuration the flag names are `crude_language` (also `crude`, `language`), `violence`, `some_explicit`, `explicit`, `abridged`, `lgbt`.

### 12.3 Legacy categories (tracker id = name)

**Audiobook**: 39 Action/Adventure, 49 Art, 50 Biographical, 83 Business, 51 Computer/Internet, 97 Crafts, 40 Crime/Thriller, 41 Fantasy, 106 Food, 42 General Fiction, 52 General Non-fic, 98 Historical Fiction, 54 History, 55 Home/Garden, 43 Horror, 99 Humor, 84 Instructional, 44 Juvenile, 56 Language, 45 Literary Classics, 57 Math/Science/Tech, 85 Medical, 87 Mystery, 119 Nature, 88 Philosophy, 58 Pol/Soc/Relig, 59 Recreation, 46 Romance, 47 Science Fiction, 53 Self-Help, 89 Travel/Adventure, 100 True Crime, 108 Urban Fantasy, 48 Western, 111 Young Adult.

**Ebook**: 60 Action/Adventure, 71 Art, 72 Biographical, 90 Business, 61 Comics/Graphic Novels, 73 Computer/Internet, 101 Crafts, 62 Crime/Thriller, 63 Fantasy, 107 Food, 64 General Fiction, 74 General Non-fic, 102 Historical Fiction, 76 History, 77 Home/Garden, 65 Horror, 103 Humor, 115 Illusion/Magic, 91 Instructional, 66 Juvenile, 78 Language, 67 Literary Classics, 79 Magazines/Newspapers, 80 Math/Science/Tech, 92 Medical, 118 Mixed Collections, 94 Mystery, 120 Nature, 95 Philosophy, 81 Pol/Soc/Relig, 82 Recreation, 68 Romance, 69 Science Fiction, 75 Self-Help, 96 Travel/Adventure, 104 True Crime, 109 Urban Fantasy, 70 Western, 112 Young Adult.

**Musicology**: 19 Guitar/Bass Tabs, 20 Individual Sheet, 24 Individual Sheet MP3, 126 Instructional Book with Video, 22 Instructional Media - Music, 113 Lick Library - LTP/Jam With, 114 Lick Library - Techniques/QL, 17 Music - Complete Editions, 26 Music Book, 27 Music Book MP3, 30 Sheet Collection, 31 Sheet Collection MP3.

**Radio**: 127 Comedy, 130 Drama, 128 Factual/Documentary, 132 Reading.

Category names in configuration are matched case-insensitively against these names and tolerant spellings (e.g. `food`, `sci-fi` style variants are accepted for those names that have an obvious simplification).

### 12.4 Genre categories (newer taxonomy, tracker supplies ids 1…)
Action, Art, Biographical, Business, Comedy, Complete Editions (Music), Computer, Crafts, Crime, Dramatization, Education, Factual/News, Fantasy, Food, Guitar, Health, Historical, Home, Horror, Humor, Individual Sheet, Instructional, Juvenile, Language, LGBT, Lick Library LTP, Lick Library Techniques, Literary Classics, LitRPG, Math, Medicine, Music, Music Book, Mystery, Nature, Paranormal, Philosophy, Poetry, Politics, Reference, Religion, Romance, RPG, Science, Science Fiction, Self-Help, Sheet Collection, Sports, Technology, Thriller, Travel, Urban Fantasy, Western, Young Adult, Superheroes, Literary Fiction, Progression Fantasy, Contemporary Fiction, Drama/Plays. For display "Literary Fiction" is shown as "General Fiction" and "Contemporary Fiction" as "Contemporary".

### 12.5 Languages (tracker id → name)
1 English, 2 Chinese, 3 Gujarati, 4 Spanish, 5 Kannada, 6 Burmese, 7 Thai, 8 Hindi, 9 Marathi, 10 Telugu, 11 Tamil, 12 Javanese, 13 Vietnamese, 14 Punjabi, 15 Urdu, 16 Russian, 17 Afrikaans, 18 Bulgarian, 19 Catalan, 20 Czech, 21 Danish, 22 Dutch, 23 Finnish, 24 Scottish Gaelic, 25 Ukrainian, 26 Greek, 27 Hebrew, 28 Hungarian, 29 Tagalog, 30 Romanian, 31 Serbian, 32 Arabic, 33 Malay, 34 Portuguese, 35 Bengali, 36 French, 37 German, 38 Japanese, 39 Farsi, 40 Swedish, 41 Korean, 42 Turkish, 43 Italian, 44 Cantonese, 45 Polish, 46 Latin, 47 Other, 48 Norwegian, 49 Croatian, 50 Lithuanian, 51 Bosnian, 52 Brazilian Portuguese, 53 Indonesian, 54 Slovenian, 55 Castilian Spanish, 56 Irish, 57 Manx, 58 Malayalam, 59 Ancient Greek, 60 Sanskrit, 61 Estonian, 62 Latvian, 63 Icelandic. Names in config are case-insensitive and accept common aliases (e.g. "Brazilian", "BP", "Castilian", "Scottish", "Greek, Ancient").

### 12.6 Size syntax
A number plus a unit: B, KiB, MiB, GiB, TiB (powers of 1024; the KB/MB/GB/TB spellings are treated the same). Decimals allowed. Display uses the largest unit that keeps the value ≥ 1.

---

## 13. Safety rules (cross-cutting)

- Never start a download that would push ratio below `min_ratio`, use the last `unsat_buffer` unsat slots, or the last `wedge_buffer` wedges.
- Never touch files that MLM did not link: removal only deletes files recorded as linked, plus empty folders, `cover.jpg` and `metadata.json` in the torrent's own library folder.
- Never overwrite a different torrent's linked files silently. Conflicts surface as errors.
- A torrent MLM cannot classify is reported, not guessed.
- All failures are surfaced in logs and the Errors page, and never terminate the application.
