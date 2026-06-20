# DiskAuditor

A local, offline disk usage auditor for macOS with a **pre-scan cache
architecture**: scanning and browsing are fully decoupled, so the UI is
always instant — it only ever reads a cached index, never the filesystem
live.

Think `du`/`ncdu`, but the slow part (walking the filesystem) runs once in
the background, and the browser UI is just a fast viewer over the result.

## How it works

Two independent pieces:

- **`scanner.py`** — walks a directory tree and writes a JSON cache
  (`cache.json` + `scan_meta.json`). Stdlib-only, runs standalone, runs via
  `launchd` daily, or is triggered from the UI.
- **`app.py`** — a small Flask server that loads the cache into memory and
  serves the browser UI. It never touches the filesystem at request time —
  browsing is always instant, even on large trees.

Stays on a single filesystem (like `du -x` / `find -xdev`), so APFS
firmlinks and Time Machine snapshot volumes don't inflate totals when
scanning from `/`. Symlinks are never followed. Permission-denied,
cross-device, and symlinked items show up in the UI with a 🔒 instead of
being silently skipped.

Full behavioral spec, including the exact cache schema, API contract, and
an acceptance test checklist: see [`SPEC.md`](./SPEC.md).

## Requirements

- macOS
- Python 3.9+
- No database, no auth, no external services — fully offline, fully local

## Quick start

```bash
git clone https://github.com/<you>/diskauditor.git
cd diskauditor
./run.sh
```

`run.sh` creates a virtualenv (`.venv/`), installs `requirements.txt`,
frees up port 5001 if something else is using it, and starts `app.py`.

Then open **http://localhost:5001**.

On first run, if there's no cache yet, the app automatically kicks off a
background scan of your home directory — you'll see a live progress banner
while it runs. Nothing needs to be configured up front.

## Manual scanning

You don't need the UI to build or refresh a cache — `scanner.py` runs
standalone:

```bash
python scanner.py                  # defaults to your home directory
python scanner.py /Users/you/Movies   # scan a specific path
```

This writes `cache.json` and `scan_meta.json` next to `scanner.py`. If the
scanned path is a subfolder already inside the existing cache, it does a
fast **merge** (only that subtree is re-walked); otherwise it does a full
**replace** and that path becomes the new cache root. See `SPEC.md` for the
exact rule.

## Using the UI

- **Go** — jump to any path already in the cache. Instant, no rescan.
- **Scan** — kick off a background scan of the typed path, with a live
  progress banner.
- **Reload Cache** — re-read `cache.json` from disk without scanning (handy
  after the daily scheduled scan runs).
- Click any column header to sort; click a folder row to navigate into it.
- A cache older than 7 days shows an amber "stale" warning.

## Running the daily scan automatically (launchd)

`com.diskauditor.scanner.plist` schedules `scanner.py` to run daily at
2:00 AM against your home directory, independent of whether the Flask app
is running. Install/uninstall instructions are in a comment block at the
top of that file — copy it to `~/Library/LaunchAgents/`, then:

```bash
launchctl load ~/Library/LaunchAgents/com.diskauditor.scanner.plist
```

Logs go to `/tmp/diskauditor.out` and `/tmp/diskauditor.err`.

## API

The Flask app exposes a small JSON API the UI is built on (useful if you
want to script against the cache yourself):

| Endpoint | Description |
|---|---|
| `GET /api/config` | Returns the home directory |
| `GET /api/meta` | Scan metadata (root, last scan time, totals) |
| `GET /api/tree?path=<absolute_path>` | A node plus its direct children |
| `POST /api/rescan` | Starts a background scan (`{"path": "..."}` optional) |
| `GET /api/rescan/status` | Whether a scan is running, and its progress |

Full request/response shapes and error codes are documented in `SPEC.md`.

## Project structure

```
diskauditor/
├── scanner.py                       # filesystem walker, writes the cache
├── app.py                           # Flask server + embedded UI
├── requirements.txt                 # single dependency: flask>=3.0.0
├── run.sh                           # venv setup + launch
├── com.diskauditor.scanner.plist    # daily launchd schedule
├── SPEC.md                          # full build spec + acceptance tests
└── README.md
```

## Limitations

- macOS-oriented (the cross-device / firmlink handling assumes APFS
  semantics); should still run on Linux but isn't tuned for it.
- The UI only ever reflects the last scan — it won't notice filesystem
  changes until you rescan or the daily job runs.
- Merge-mode rescans assume the directory structure *above* the scanned
  path hasn't changed since the last full scan (see `SPEC.md` for the exact
  rule and how to force a full rescan).

## License

Add your preferred license here (e.g. MIT) before publishing.
