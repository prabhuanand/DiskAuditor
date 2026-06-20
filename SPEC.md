# diskauditor — Build Spec

GitHub repo: `diskauditor`

Build a local macOS disk usage auditor with a pre-scan cache architecture.

## Architecture

Two separate components:

1. **scanner.py** — walks the file system, writes a JSON cache. Stdlib-only,
   no third-party dependencies. Runs via launchd daily OR on-demand from the
   terminal or UI.
2. **app.py** — Flask web server that reads the JSON cache and serves the
   browser UI. Never scans at request time.

The scanner and the UI are fully decoupled. The UI is always instant because
it reads from cache only.

---

## Scanner (scanner.py)

- Accepts a root path as CLI arg (default: `os.path.expanduser("~")`).
- Must be runnable standalone: `python scanner.py /Users/username`
- Stays on one filesystem — never crosses device boundaries (equivalent to
  `find -xdev` / `du -x`). This prevents APFS firmlinks and Time Machine
  snapshot volumes from inflating sizes on macOS when scanning from `/`.
- Uses `os.walk` with `topdown=True` so cross-device directories can be
  pruned from `dirnames` before recursion. Cross-device dirs get a stub node
  (`skipped=True`, `size_bytes=0`) so they still appear in the UI with a lock
  icon.
- **Does not follow symlinks, of any kind.** A symlinked file or symlinked
  directory is never stat'd through to its target. Every symlink — file or
  dir — is recorded as a stub node identical in shape to a cross-device stub:
  `skipped=True`, `size_bytes=0`, `child_count=0`, `child_names=[]`. It still
  appears in its parent's `child_names` so the UI can render it with a lock
  icon. (Symlinked dirs are also pruned from `dirnames` so `os.walk` never
  descends into them.)
- Handles `PermissionError` and `OSError` gracefully: mark item with
  `size=0` and `skipped=True`, continue.
- For every file and folder, records: `name`, `is_dir`, `size_bytes`
  (recursive total for dirs), `child_count` (total items under the dir,
  recursive — **includes stub/skipped entries**, each stub counts as 1 item
  contributing 0 bytes), `extension` (empty string `""` for directories),
  `skipped`, `child_names`.
- Print progress to stdout every 5000 files: `"Scanned 5000 files..."`

### Cache format (cache.json)

Flat dict: `{ "<absolute_path>": <node> }`

Each node contains:
`name, is_dir, size_bytes, child_count, extension, skipped, child_names`
(list of bare filenames/dirnames, **not** full paths).

The absolute path is the dict key — it is **not** stored inside the node, to
avoid redundancy. `child_names` stores bare names only; the app derives full
child paths as `parent_path + "/" + child_name` at query time.

Since dir sizes require a bottom-up pass but `topdown=True` is needed for
pruning, compute dir sizes in a post-processing step: sort all real
(non-stub) directory paths by descending path length (deepest first), then
accumulate `size_bytes` and `child_count` from children upward.

### Partial scans: merge vs. replace

The UI lets a user scan *any* path, not just the original root, so the
scanner must decide whether a new scan replaces the whole cache or merges
into it. Rule, evaluated against the existing `scan_meta.json` (if any):

- **No existing cache**, **or** the scanned path **equals** the current
  cache root, **or** the scanned path is **outside** the current root (not
  an ancestor or descendant) → **full replace.** Build a brand-new
  `cache.json` rooted at the scanned path; the new path becomes
  `scan_meta.json.root_path`.
- **Scanned path is a strict descendant of the current root** → **merge.**
  Re-walk only that subtree. Remove all existing index entries whose path is
  the scanned path or starts with `scanned_path + "/"`, then insert the
  freshly walked entries in their place. After inserting, also refresh the
  scanned path's **immediate parent's** `child_names` (in case the scanned
  folder itself was newly created or deleted since the last full scan).
  Re-run the bottom-up size/`child_count` accumulation for every ancestor
  directory from the scanned path up to the existing root, using the
  now-merged `child_names` lists. `scan_meta.json.root_path` is left
  unchanged; `last_scanned_at`, `duration_seconds`, and the totals are
  updated to reflect the merged result.
  - **Known limitation, documented in code comments**: merge mode assumes
    the directory structure *above* the scanned path's immediate parent has
    not changed since the last full scan (e.g. an ancestor folder being
    renamed or deleted won't be detected). A full rescan of the root clears
    this up.

### Atomic writes & locking

- `cache.json` and `scan_meta.json` are each written to a `.tmp` sibling
  file first, then moved into place with `os.replace()` — this guarantees
  `app.py` never reads a half-written file.
- A PID lockfile (`.scanner.lock`, in the same directory as `scanner.py`) is
  created at the start of a run and removed at the end. If `scanner.py` is
  invoked while a lockfile exists and the PID inside it is still alive, the
  new invocation prints a message and exits immediately (code 1) rather than
  racing the in-progress scan. A stale lockfile (PID no longer running) is
  ignored and overwritten.

Output files are written to the same directory as `scanner.py`:

- `cache.json` — flat path index (described above)
- `scan_meta.json` —
  `{ root_path, last_scanned_at (ISO timestamp), duration_seconds, total_files, total_dirs, total_size_bytes, skipped_count }`
  (`skipped_count` counts all skipped entries — permission errors,
  cross-device stubs, and symlink stubs combined.)
- `.scanner.lock` — transient PID lockfile, removed on successful or failed
  exit.

---

## Flask app (app.py)

- On startup: loads `cache.json` via `json.load()` directly into memory as
  the path index (no `build_index` traversal needed — the file *is* the
  index). Prints node count on load.
- If `cache.json` does not exist on startup: automatically starts
  `scanner.py` on the home directory as a background subprocess.
- All reads/writes of the in-memory index and meta dict, including the swap
  that happens when a rescan completes, are guarded by a single
  `threading.RLock` so a request can never observe a half-swapped state.

### Routes

- **GET `/api/config`** — returns `{ "home": os.path.expanduser("~") }`.
  Always available, no cache required.
- **GET `/api/meta`** — returns `scan_meta.json` contents.
  `404 { "error": "no cache available" }` if no cache.
- **GET `/api/tree?path=<absolute_path>`** — looks up `path` in the index,
  assembles the node plus its direct children (each child looked up via
  `child_names`). Grandchildren are not included (bounded response size).
  Response shape:
  ```json
  {
    "path": "<absolute_path>",
    "node": { "name": ..., "is_dir": ..., "size_bytes": ..., "child_count": ..., "extension": ..., "skipped": ... },
    "children": [
      { "path": "<absolute_path>/child", "name": ..., "is_dir": ..., "size_bytes": ..., "child_count": ..., "extension": ..., "skipped": ... },
      ...
    ]
  }
  ```
  `404 { "error": "path not found in cache" }` if `path` isn't in the index.
  `400 { "error": "missing path parameter" }` if `path` is omitted.
- **POST `/api/rescan`** — accepts an optional JSON body
  `{ "path": "<absolute_path>" }`; if omitted, defaults to the current
  `scan_meta.json.root_path`, or the home directory if there's no cache yet.
  - Validates the path is a real directory:
    `400 { "error": "<path> is not a valid directory" }` if not.
  - If a scan is already running: `409 { "error": "scan already running" }`.
  - Otherwise spawns `scanner.py` as a subprocess, captures stdout for
    progress, reloads the cache when it finishes, and returns
    `{ "status": "started" }`.
- **GET `/api/rescan/status`** — returns
  `{ "running": bool, "progress": "<latest stdout line>" }`.

Runs on `http://localhost:5001`.

---

## Browser UI (single page, vanilla JS + HTML/CSS, no frontend framework)

The entire HTML is embedded as a raw string inside `app.py` (no template
files, no Jinja2 processing).

### Layout

- **Top bar:**
  - Path input box (monospace, wide) — pre-filled with the current cache
    root path on load; pre-filled with the home directory when no cache
    exists.
  - **Go** button — instantly navigate to any path already in the cache; if
    not found, shows a contextual message explaining what the cache covers
    and how to scan it.
  - **Scan** button — runs `scanner.py` on the typed path as a background
    subprocess (merge or replace per the rule above, decided server-side);
    polls `/api/rescan/status` every 2 seconds; shows a progress banner with
    spinner and live file count; reloads the UI when done. If a scan is
    already running, the button shows the in-progress banner instead of
    starting a second one (handles the `409` response).
  - **Reload Cache** button — re-reads `cache.json` from disk without
    rescanning (used after launchd runs the daily scanner).
  - Small hint line under the buttons: *"Go — browse a path already in
    cache (instant) · Scan — index the typed path fresh (full rescan of the
    root, or a fast merge if it's a subfolder already in cache) · Reload
    Cache — load the latest cache written by the daily scheduled scanner"*
  - "Last scanned: `<timestamp>` — N files, X size" info line.
- **Breadcrumb bar:** shows the full absolute path of the current directory
  split into individual path segments. Segments above the scan root are
  dimmed and not clickable (not in cache). Segments at or below the scan
  root are blue and clickable for instant navigation. Current segment is
  white and non-clickable.
- **Main panel:** sortable table with columns: Name | Type | Size | % of
  Parent (inline bar) | Items.
  - Sortable columns: Name, Type, Size, Items. Clicking a header sorts
    descending; clicking the same header again toggles to ascending (and
    back). Default sort on load: Size descending.
  - Clicking anywhere on a folder row navigates into it (not just the name
    text).
  - Folders are bold blue; files are normal weight.
  - "% of Parent" bar: if the parent's `size_bytes` is `0`, render the bar
    at 0% rather than dividing by zero / showing `NaN`.
- **Status bar** at bottom: item count and total size of current directory.

### Behavior

- On load: fetch `/api/config` (to get home dir), check
  `/api/rescan/status` (in case an auto-scan is already running), then load
  `/api/meta` and `/api/tree`.
- If an auto-scan is running on load: show the progress banner and poll
  until complete, then render.
- If no cache: pre-fill the path input with the home directory, show a
  no-cache message with the exact terminal command to run
  (`python scanner.py <home>`), and explain that scanning from home makes
  all breadcrumb segments clickable.
- Path not found in cache: show a message stating what path was not found,
  what the cache covers, and prompt to use Scan.
- Stale cache (> 7 days): show an amber warning banner.
- Scan already running when the user clicks Scan: show the existing
  progress banner instead of erroring.
- All browsing is instant (no filesystem calls at request time).

### Visual design

- Dark theme (`#1a1a1a` background, `#242424` elevated surfaces).
- Folders bold blue (`#4a9eff`), files normal white.
- Items > 1 GB: row highlighted amber, bar fill amber.
- Inline proportional bar per row showing % of parent directory size.
- Human-readable sizes (B / KB / MB / GB / TB), raw bytes shown in tooltip.
- Skipped/inaccessible/cross-device/symlinked items: lock icon 🔒, size
  shown as `"—"`.

---

## launchd plist (com.diskauditor.scanner.plist)

- Runs `scanner.py` daily at 2:00 AM.
- Default scan path: user's home directory.
- Calls the system `python3` directly (not the project venv) — `scanner.py`
  is stdlib-only and has no dependency on Flask or the venv `run.sh` builds.
- Include full install and uninstall instructions in a comment block at the
  top of the file.
- Log stdout/stderr to `/tmp/diskauditor.out` and `/tmp/diskauditor.err`.

---

## run.sh (startup wrapper)

- Creates a Python venv (`.venv/`) if it doesn't exist.
- Installs dependencies from `requirements.txt` via public PyPI
  (`--index-url https://pypi.org/simple/`).
- Kills any existing process on port 5001 before starting (using `lsof`;
  does not use `set -e`, to avoid silent exits).
- Launches `app.py` via `exec`.
- Single command to start: `./run.sh`

---

## Error handling

- No cache on startup: auto-scan home directory in background; UI shows
  live progress.
- Stale cache (> 7 days): amber warning banner in UI.
- Permission-denied items: lock icon, size `"—"`.
- Symlinks (file or dir): lock icon, size `"—"` — never followed.
- Cross-device directories (firmlinks, mount points): stub node, shown as
  skipped.
- Path not in cache: contextual message with cache coverage info and
  prompt to Scan.
- Concurrent scan requests: `409` from the API; UI shows the existing
  progress banner rather than erroring.
- Corrupt/partial cache reads: prevented by atomic `.tmp` + `os.replace()`
  writes in the scanner, so this should not occur in normal operation.

## API error contract (summary)

All API errors are JSON: `{ "error": "<message>" }`.

| Status | Used for |
|---|---|
| 400 | Bad/missing input (invalid path, missing `path` param) |
| 404 | Resource not found (no cache, path not in cache) |
| 409 | Conflict (scan already running) |

## Acceptance Test Checklist

A concrete way to verify a build is correct, rather than "looks right."
Uses one synthetic test tree for the scanner checks, plus a set of API and
UI checks layered on top of it.

### 1. Build the test tree

```bash
mkdir -p testroot/sub1/sub2 testroot/sub1/empty
echo "hello world" > testroot/a.txt              # 12 bytes
echo "12345" > testroot/sub1/b.txt               # 6 bytes
dd if=/dev/zero of=testroot/sub1/sub2/big.bin bs=1024 count=10  # 10240 bytes
ln -s /etc testroot/sub1/linktoetc               # symlink to a dir, outside the tree
```

Resulting layout:

```
testroot/
├── a.txt                (12 bytes)
└── sub1/
    ├── b.txt             (6 bytes)
    ├── empty/            (empty dir)
    ├── linktoetc -> /etc (symlink — must be stubbed, never followed)
    └── sub2/
        └── big.bin       (10240 bytes)
```

### 2. Scanner — full scan correctness

Run: `python scanner.py testroot`

Expected `scan_meta.json`:

| Field | Expected value |
|---|---|
| `root_path` | absolute path to `testroot` |
| `total_files` | `3` |
| `total_dirs` | `4` (`testroot`, `sub1`, `sub1/empty`, `sub1/sub2`) |
| `total_size_bytes` | `10258` (= 12 + 6 + 10240) |
| `skipped_count` | `1` (the symlink) |

Expected `cache.json` entries (spot-check, not exhaustive):

| Path | `is_dir` | `size_bytes` | `child_count` | `skipped` |
|---|---|---|---|---|
| `testroot` | `true` | `10258` | `7` | `false` |
| `testroot/sub1` | `true` | `10246` | `5` | `false` |
| `testroot/sub1/sub2` | `true` | `10240` | `1` | `false` |
| `testroot/sub1/empty` | `true` | `0` | `0` | `false` |
| `testroot/sub1/linktoetc` | `true` | `0` | `0` | `true` |
| `testroot/a.txt` | `false` | `12` | `0` | `false` |
| `testroot/sub1/b.txt` | `false` | `6` | `0` | `false` |
| `testroot/sub1/sub2/big.bin` | `false` | `10240` | `0` | `false` |

`testroot`'s `child_count` of `7` = every nested file/dir/stub counted
recursively (`a.txt`, `sub1`, `b.txt`, `empty`, `linktoetc`, `sub2`,
`big.bin`).

Pass/fail: every value above must match exactly. `testroot/sub1/linktoetc`
must never have its target (`/etc`) walked — confirm no `/etc/*` paths leak
into `cache.json`.

### 3. Scanner — cross-device stubbing

If a second mounted volume is available (e.g. an external drive or a
`hdiutil`-created disk image on macOS), place it inside the scanned tree and
confirm its mount point appears as a stub node (`skipped=true`,
`size_bytes=0`) and is **not** descended into — no entries from that volume
appear in `cache.json`. If no second volume is available for testing,
verify this by code inspection: the `st_dev` comparison against `root_dev`
must run before any recursion into a subdirectory.

### 4. Scanner — permission-denied handling

```bash
mkdir testroot/locked && touch testroot/locked/secret.txt
chmod 000 testroot/locked
python scanner.py testroot
chmod 755 testroot/locked   # restore so cleanup works
```

Expected: `testroot/locked` appears in `cache.json` with `skipped=true`,
`size_bytes=0`, and the scan completes without raising — no traceback, no
non-zero exit code from a permission error.

### 5. Scanner — merge vs. replace

1. Run `python scanner.py testroot` (full scan, establishes the root).
2. Add a new file: `echo "new" > testroot/sub1/sub2/new.txt`.
3. Run `python scanner.py testroot/sub1` (a strict descendant of the
   current root).
4. Expected: this is a **merge**. `scan_meta.json.root_path` is unchanged
   (still `testroot`). `testroot/sub1/sub2/new.txt` now appears in
   `cache.json`. `testroot/a.txt` (outside the merged subtree, a sibling of
   `sub1`) is **still present and untouched** in `cache.json` — proving the
   merge didn't wipe the rest of the cache. `testroot`'s `size_bytes` has
   increased to reflect the new file, proving the bottom-up re-aggregation
   ran up to the root.
5. Now run `python scanner.py /tmp` (outside the current root entirely).
   Expected: this is a **full replace** — `scan_meta.json.root_path`
   becomes `/tmp`, and none of the old `testroot/*` entries remain in
   `cache.json`.

### 6. Scanner — atomicity & locking

1. Confirm no `cache.json.tmp` / `scan_meta.json.tmp` files are left behind
   after a normal run (they should be renamed away via `os.replace()`).
2. Start a scan on a large directory in the background, then immediately
   invoke `python scanner.py testroot` again in a second terminal while the
   first is still running. Expected: the second invocation detects the live
   `.scanner.lock` PID and exits immediately with a message, rather than
   running concurrently.
3. Confirm `.scanner.lock` does not exist after either run completes
   (success or failure).

### 7. Flask API — behavioral checks

With `app.py` running and `cache.json` from step 2 in place:

| Request | Expected response |
|---|---|
| `GET /api/config` | `200`, `{"home": "<expanded ~>"}` |
| `GET /api/meta` | `200`, matches `scan_meta.json` contents |
| `GET /api/tree?path=<testroot>` | `200`, `node.size_bytes == 10258`, `children` includes `a.txt` and `sub1` |
| `GET /api/tree?path=/nonexistent` | `404`, `{"error": "path not found in cache"}` |
| `GET /api/tree` (no `path`) | `400`, `{"error": "missing path parameter"}` |
| `POST /api/rescan` `{"path": "<testroot>"}` | `200`, `{"status": "started"}` |
| `POST /api/rescan` again immediately, before the first finishes | `409`, `{"error": "scan already running"}` |
| `GET /api/rescan/status` while running | `200`, `{"running": true, "progress": "<last stdout line>"}` |
| `GET /api/rescan/status` after completion | `200`, `{"running": false, ...}` |
| `GET /api/meta` with `cache.json` deleted and not yet rebuilt | `404`, `{"error": "no cache available"}` |

### 8. UI — manual checklist

- [ ] Fresh install, no `cache.json`: loading `/` shows the no-cache
      message with the exact `python scanner.py <home>` command, and the
      path input is pre-filled with the home directory.
- [ ] After a scan completes, the table renders sorted by Size descending
      by default.
- [ ] Clicking the "Size" header again flips to ascending; clicking "Name"
      sorts alphabetically.
- [ ] `testroot/sub1/linktoetc` (or any symlink/cross-device stub) renders
      with a 🔒 icon and `"—"` for size, not `0 B`.
- [ ] An empty directory's row shows its "% of Parent" bar at 0%, not a
      broken/`NaN` bar.
- [ ] Breadcrumb segments above the scan root are dimmed/non-clickable;
      segments at or below the root are blue and clickable.
- [ ] Typing an un-cached path and clicking **Go** shows the "not found in
      cache" message rather than a blank table or a crash.
- [ ] Clicking **Scan** while a scan is already in progress shows the
      existing progress banner rather than starting a duplicate or erroring
      in the UI.
- [ ] Manually editing `scan_meta.json.last_scanned_at` to a date more than
      7 days in the past and reloading the UI shows the amber stale-cache
      banner.
- [ ] Any item over 1 GB renders its row and bar in amber.

---

## Constraints

- No database, no auth, fully offline, fully local.
- macOS compatible.
- No frontend framework — vanilla JS + HTML/CSS only.
- Single third-party dependency: `flask>=3.0.0` (used only by `app.py`;
  `scanner.py` remains stdlib-only).
- `requirements.txt` included.
