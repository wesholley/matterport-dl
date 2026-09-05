# Instructions for Self-hosted Matterport

This folder contains a preserved, fully offline copy of one or more Matterport virtual tours, plus the tool used to capture and serve them. It was set up specifically so the tour keeps working even if the original listing is taken down or Matterport itself goes away — viewing it never contacts matterport.com.

Your current capture: **Microsoft Building 3**, model id `SZSV6vjcf4L`, aliased as `Microsoft_building_9`.

## Quick start

```bash
python3 run.py Microsoft_building_9 127.0.0.1 8080
```

Then open **http://127.0.0.1:8080** in a browser. Stop the server with `Ctrl+C`.

That's it — `run.py` sets up its own Python virtual environment automatically the first time you run it, so there's no separate install step for basic viewing.

## Requirements

- **Python 3.12 or newer.** Check with `python3 --version`.
- Nothing else to install manually for viewing — `run.py` handles its own venv and dependencies on first run.

## Viewing vs. downloading — why they're different

This matters because it affects what you need installed, and it's easy to get confused between the two:

| | Needs `curl_cffi` | Talks to matterport.com |
|---|---|---|
| **Viewing** an already-downloaded tour | No | No — `JSNetProxy.js` redirects every request the viewer makes to your local server instead |
| **Downloading or refreshing** a tour | Yes | Yes |

`curl_cffi` exists purely to impersonate a real Chrome browser's network fingerprint so Matterport's bot detection doesn't block the download. It's irrelevant once the tour is on disk. If you only ever plan to view what's already downloaded, you can ignore everything below about `curl_cffi` and just use the quick start above.

## Two requirements files, two purposes

**Important: these two files install the exact same set of packages** (`aiofiles`, `curl_cffi`, `Pillow`, `requests`, `tqdm`, plus `pyreadline3` on Windows) — including `curl_cffi`, even though (per the table above) *viewing* doesn't actually need it. Neither file is a stripped-down "viewer only" install. The only difference between them is **which versions of those packages get installed**:

- **`requirements.txt`** — installs whatever is newest on PyPI at the time you run it. This is the default choice for a **fresh download**: `curl_cffi`'s whole job is impersonating a current browser closely enough to get past Matterport's bot detection, and that detection evolves, so a newer `curl_cffi` is more likely to still work than an old pinned one.
- **`requirements-lock.txt`** — installs the exact versions that were tested together and confirmed working, frozen at the versions listed inside the file, instead of whatever happens to be newest. Prefer this when you want a *reproducible* environment rather than an up-to-date one — most relevant if you're coming back **years later just to view** an archive (where an old, possibly-stale `curl_cffi` pin doesn't matter, since viewing never uses it), but it's also a reasonable fallback for downloading if a `requirements.txt` install ever breaks and you want to fall back to a combination that's known to have worked at some point.

```bash
# latest versions (default choice for a fresh download)
pip install -r requirements.txt

# exact pinned versions (default choice for long-term archival viewing)
pip install -r requirements-lock.txt
```

Either way, only *downloading* actually needs the packages to be fully working — `curl_cffi` in particular. Viewing has no hard dependency on that pin holding up, even though the package still gets installed alongside everything else.

## Setting up and using the virtual environment manually

`run.py` creates and activates this venv for you automatically — most people can skip this section entirely and just use the Quick start command above. This section exists for when that automatic path doesn't work: a future Python release `run.py`'s own version check doesn't recognize, a corrupted or deleted `venv/` folder, or you just want explicit control over what's installed.

**Tested with:** Python 3.14.7, on macOS. Minimum supported: Python 3.12 (per `README - matterport-dl Tool Reference.md`). Check what you have with `python3 --version` before starting.

### First-time setup (macOS / Linux)

```bash
cd "/path/to/this/folder"          # the folder containing matterport-dl.py
python3 -m venv venv               # only needed if venv/ doesn't already exist
source venv/bin/activate           # your shell prompt should now show (venv)
python -m pip install --upgrade pip
pip install -r requirements-lock.txt   # known-good pinned versions - see above for when to use requirements.txt instead
```

### First-time setup (Windows, PowerShell)

```powershell
cd "C:\path\to\this\folder"
python -m venv venv
venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements-lock.txt
```

### Running matterport-dl.py directly, once the venv is active

```bash
python matterport-dl.py Microsoft_building_9 127.0.0.1 8080   # serve (viewing)
python matterport-dl.py SZSV6vjcf4L                            # download/refresh (needs curl_cffi installed and working)
```

Leave the venv when done with `deactivate` (same command on all platforms).

### The `venv/` folder already in this project — and why it won't survive a move

There's a working `venv/` folder in this project directory as of this writing, built at `/Users/wes/Downloads/matterport-dl-main`. **If you move or copy this project folder somewhere else (including onto network storage), that `venv/` folder will not reliably work anymore and should be deleted and recreated at the new location.** This isn't a maybe:

```
$ head -1 venv/bin/pip
#!/Users/wes/Downloads/matterport-dl-main/venv/bin/python3.14
```

`pip` (and other scripts inside `venv/bin/`) have that exact original absolute path baked into their first line. Once the folder lives somewhere else, that path no longer exists, and running `venv/bin/pip` directly fails outright. `run.py`'s own auto-setup mostly avoids this — it calls `venv/bin/python` directly rather than going through `pip`'s shebang, and that one happens to resolve through a relative symlink out to the system-wide Python install rather than back into the project folder, so it can *appear* to keep working right up until the moment something needs `pip` (e.g. installing a package that goes missing). Relying on that distinction is fragile and not worth it, especially since network storage may mean this folder gets opened from more than one machine, or a different OS, over time — a venv is inherently tied to the one machine/OS/Python-install it was built against, never something to store or share as-is.

**The reliable move:** after relocating this folder, delete the old `venv/` entirely and let it rebuild fresh in the new location:

```bash
rm -rf venv                          # macOS/Linux — do this at the NEW location, after moving
python3 -m venv venv
source venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements-lock.txt
```

(`rmdir /s venv` on Windows instead of `rm -rf venv`.) This costs a couple of minutes and removes any doubt. `run.py` will also do this automatically the first time it's run from the new location and doesn't find a working `venv/` — but if you've just moved the folder and the *old* `venv/` directory is still physically present, `run.py`'s check only confirms the directory exists, not that it still works, so it may try to reuse the now-broken one. Deleting it first guarantees a clean rebuild either way.

### Confirming you're actually inside the venv

Easy to get wrong silently — a missed `activate` step means you're running against your system Python instead, which may be missing everything.

```bash
which python      # macOS/Linux
where python       # Windows
```

This should print a path *inside* this folder's `venv/` (e.g. `.../venv/bin/python`), not a system path like `/usr/bin/python3`. Then:

```bash
pip list
```

should show (at minimum) `curl_cffi`, `aiofiles`, `tqdm`, `Pillow`, and `requests` — compare their versions against `requirements-lock.txt` if you want to confirm you match the last known-working set exactly.

## Serving a tour

```bash
python3 run.py <id_or_alias> <host> <port>
```

Example:

```bash
python3 run.py Microsoft_building_9 127.0.0.1 8080
```

You can serve by the original model id (`SZSV6vjcf4L`) or by a rename alias (`Microsoft_building_9`) — both work for serving.

## Downloading a new tour

```bash
python3 run.py <matterport_url_or_id>
```

This requires the real model id or the full `https://my.matterport.com/show/?m=...` URL — **not** an alias. Downloading re-derives the model id from what you pass in and validates it strictly, so aliases (which can be any name) aren't accepted there, only for serving.

## Refreshing/re-downloading an existing tour

Same command as above, using the original id or URL again. Already-downloaded files are skipped automatically; only missing or changed files are re-fetched. This is useful if Matterport updates their viewer app and something that used to render now shows a blank screen or console errors — a refresh will pull in whatever changed.

## Renaming a tour (adding a friendly alias)

Run `run.py` with no arguments to get the interactive menu, then use the `rename` command:

```
rename SZSV6vjcf4L
```

You'll be prompted for the new name. This creates a symlink in `downloads/` and records the alias in that tour's `run_args.json` — it does not rename or move the actual capture folder.

## Where everything lives

```
downloads/
  SZSV6vjcf4L/              the actual downloaded capture
  Microsoft_building_9 ->   alias symlink pointing at SZSV6vjcf4L
```

Each capture's `run_args.json` records the settings it was downloaded with (and its alias, if any) so future refreshes stay consistent.

## Troubleshooting

- **Blank screen / stuck on "LOADING"**: open the browser console (F12 → Console). A `ChunkLoadError` means an asset didn't fully download — try refreshing the capture (see above).
- **Stuck loading — either the initial "LOADING" splash never clears, or it loads into dollhouse view but walk/panorama mode won't transition past a small stuck preview circle**: both are the same underlying cause, just at different points in the load sequence. Check the Network tab for a `graph` request that never completes (or fails as "empty response"), or keeps repeating over and over: this means Matterport's live app is asking for a GraphQL query this tool doesn't yet know about. Matterport adds these occasionally — so far this has happened twice (`GetModelAssets`, needed for the initial load; `GetSweeps`, needed specifically for walk mode, since it carries the actual sweep/location data to move between) — and can keep recurring over the years as Matterport's app evolves. The fix has two parts, both already applied in this copy for the two queries found so far, but worth knowing if a *different* unrecognized query shows up in the future:
  1. `do_GraphRequest()` (in `matterport-dl.py`) now always sends *some* response instead of silently hanging on an unrecognized query — so worst case this now degrades to a stuck-but-not-hung state (repeated failed requests visible in the Network tab) instead of an infinite hang with no clue why.
  2. To fix it properly rather than just degrade gracefully: open the browser's Network tab, find the failing/repeating `api/mp/models/graph?operationName=...` request, copy its full query-string, and add it as a new entry to the `GRAPH_DATA_REQ` dict near the top of `matterport-dl.py` (replace the `modelId` value inside it with `[MATTERPORT_MODEL_ID]`, matching the existing entries — leave other variables as-is). Then run a refresh (see above) to actually fetch and cache real data for it, and restart the server.
- **A file 404s in `server.log`, specifically under `/1/oembed` or `/1/display/resize` with a `vimeo.com` or similar URL in it**: this is a mattertag (info marker) with an embedded video. Those oEmbed/thumbnail requests go to a third-party embed service (not Matterport, not something this tool ever downloads), so they'll always 404 offline. Harmless — the mattertag itself still works, just without the video preview thumbnail.
- **A file 404s in `server.log`** (other than the above): not necessarily a problem. A few optional feature chunks (VR/plugin-related, not core viewing) are known to be unavailable even directly from Matterport's own servers and don't affect the tour itself.
- **`ModuleNotFoundError: No module named 'curl_cffi'` while just trying to view a tour**: this shouldn't happen with the fixed version of `matterport-dl.py` in this folder — viewing no longer imports `curl_cffi` at all. If you see it, you may be running an older/different copy of the script.
- **`OSError: [Errno 48] Address already in use` when starting the server**: something is already listening on that port. Two causes:
  - You (or something else) already have a copy running — check with `lsof -nP -iTCP:<port> -sTCP:LISTEN` (macOS/Linux) and either use that one or stop it first.
  - **A stale, orphaned server process from a previous run that was stopped incorrectly.** `run.py` bootstraps into its venv by replacing itself in-place (`os.execv`), specifically so there's only ever one process for the whole run — stopping it (however you stop it: Ctrl+C, `kill <pid>`, a process manager, a NAS service manager) reliably kills the actual server, since there's no separate child process left behind to orphan. If you're ever running an older copy that doesn't have this fix, killing only the outer process can leave a real child still holding the port; find it with `lsof` (its PID differs from whatever you tried to kill) and kill that instead. This matters most for exactly the kind of always-on hosting a NAS is used for, where the server gets started and stopped by something other than a person watching a terminal.

## Provenance of the fixes in this copy

The downloader in this folder has fixes beyond the original open-source tool, made necessary by changes to Matterport's own app since the original `matterport-dl` project was last updated for them:

1. A parser fix for Matterport's current webpack build (chunk filenames no longer include a content hash), without which core files like `init.js` and the current locale's translation chunk never download — this was the original cause of a blank-screen issue.
2. Two bug fixes for the rename/alias feature (a broken symlink, and serving-by-alias throwing an error instead of working).
3. A fix for `do_GraphRequest()` silently hanging (no HTTP response at all) on a GraphQL query it didn't recognize, plus added recognition of two specific queries Matterport's live app makes that this tool didn't know about: `GetModelAssets` (needed on initial load) and `GetSweeps` (needed specifically to transition into walk/panorama mode — without it, the tour loads into dollhouse view but walk mode gets stuck on a small preview circle and never enters a panorama). See the troubleshooting entry above for what to do if a *different* unrecognized query shows up in the future.
4. The `curl_cffi`-independent viewing behavior and the pinned `requirements-lock.txt` described above.
5. `run.py` now replaces itself in-place (`os.execv`) rather than spawning the venv's Python as a child process it waits on — found when hosting from network storage, where the server had been stopped in a way that left the real process orphaned and still holding the port, so the next start failed with "Address already in use". See the troubleshooting entry above.

These are preserved on **https://github.com/wesholley/matterport-dl** (`main` branch has everything). The bug fixes (not the archival-specific changes) have also been submitted upstream as [rebane2001/matterport-dl#206](https://github.com/rebane2001/matterport-dl/pull/206) — fixes #3 and #5 above still need to be added there.
