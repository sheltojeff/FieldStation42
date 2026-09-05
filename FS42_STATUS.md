# FieldStation42 — Project Status

**Generated against:** commit `5bd2007` ("Fix AutoBumpAgent keys and add robustness to main loop"), the tip of both `main` and `origin/main`.
**Last commit date:** 2026-02-01. **Working tree:** clean.
**Repo:** `sheltojeff/FieldStation42`, a fork of `shane-mason/FieldStation42`. No upstream remote is configured — `origin` points at the fork only.

---

## 0. Read this first — four things that aren't obvious

**1. Your last commit broke autobump in production.** `5bd2007` renamed the dict keys returned by `AutoBumpAgent.gen_bumps()` and updated the tests, but did not update the only production caller. Details in §5 — this is the most actionable item in this document, and it is a small fix.

**2. The Raspberry Pi is running code that does not exist in this repository.** A traceback from the Pi references three files that are absent from git entirely:

| File on the Pi | In this repo? |
|---|---|
| `fs42/station_io.py` (`load_and_process_all_stations`, `_process_single_config`) | **No — missing** |
| `fs42/fs42_server/api/ppv.py` | **No — missing** |
| `fs42/fs42_server/api/tmdb_helper.py` | **No — missing** |

The Pi's `station_manager.py:189` calls `self.station_io.load_and_process_all_stations()`; this repo's `station_manager.py:164` still loads configs inline via `glob.glob("confs/*.json")`. The Pi's `api/__init__.py` imports a `ppv_router` at line 9; this repo's version imports seven routers and no PPV.

So there is **uncommitted local work on the Pi**: a config-loading refactor plus a pay-per-view feature with TMDB integration. None of it is pushed, and the SD card is the only copy. See §6.

**3. There are no commits since March.** Zero. The last commit of any kind is 2026-02-01. See §7.

**4. This clone is shallow.** `.git/shallow` pins history at `394d967` (2025-08-24); only 51 commits are visible. Anything earlier is not in this clone. `git fetch --unshallow` to recover it.

---

## 1. Current architecture

About 10,700 lines of Python. Two long-lived processes, one shared SQLite database, and two file-based "sockets" that are really just polled files.

The central idea: `station_42.py` pre-builds a broadcast schedule from a media catalog; `field_player.py` plays it back against the wall clock, so tuning to a channel joins programming already in progress.

### Entry points

- **`station_42.py`** (606 lines) — build/admin tool. ~20 flags: `-r` rebuild catalogs, `-x` delete schedules, `-d/-w/-m` add day/week/month, `-u` print schedule, `-p` print catalog, `-q/-a` rebuild/scan sequences, `-b/-t` black-detect / chapter-detect, `--reset_chapters/--reset_breaks`, `-g` Textual TUI, `-s` server, `--limit_memory` (sets `RLIMIT_AS`). Ordering matters in `main()`: schedules are deleted before catalogs rebuild, then the fluid file cache is trimmed. With `-s` **or no arguments at all** it falls through to serving the web console on :4242.
- **`field_player.py`** (335 lines) — the playback daemon. Unless `--no_server`, it forks the FastAPI server as a daemon `multiprocessing.Process` with a shutdown queue and a command queue.
- **`fs42/hot_start.sh`** — four lines: `cd ~/FieldStation42`, activate `env/`, background-and-disown `field_player.py` and `fs42/command_input.py`, both redirected to `/dev/null`. This is why boot failures are invisible.
- **`fs42/command_input.py`** — serial channel changer on `/dev/ttyAMA0` @ 9600. Reads JSON from a Pico and writes it into `runtime/channel.socket`; channel 99 kills the player, 98 kills and halts the box. When idle it echoes the current channel back over UART to drive an external display.
- **`page_stream/`** — shell only, no Python. Renders a webpage as a channel: Xvfb → kiosk Chromium → `ffmpeg -f x11grab` → HLS → `python3 -m http.server`, with a generated `confs/web_<id>.json`.
- **`docker/`** + root `Makefile` — `python:3.11-slim-bullseye` with tk/mpv/ffmpeg/X11/pulse, bind-mounting `catalog/`, `runtime/`, `confs/` from outside the image. `make station_42 RUN_MODE=docker ARGS=-r`, with `OS_ENV=wsl|linux` selecting DISPLAY/pulse paths.

### The player loop

`main_loop()` in `field_player.py` is a `while True` dispatching on `network_type`: `"guide"` → `show_guide()`, `"web"` → `show_web()`, everything else → `play_slot(network_name, now)`, which computes weekday, hour, and a seek offset from the current minute and second. That offset is the whole illusion.

Each iteration yields a `PlayerOutcome` carrying a `PlayerState` (`station_player.py:60`): `FAILED`, `EXITED`, `SUCCESS`, `CHANNEL_CHANGE`, `EXIT_COMMAND`. `CHANNEL_CHANGE` (lines 160-225) parses a JSON payload for `direct`/`up`/`down`, skipping `hidden` stations. `FAILED` (lines 227-252) shows `standby_image` after two seconds stuck, sleeps one second, re-checks input, and retries forever.

`StationPlayer` (`station_player.py`, 527 lines) is the heart:

- **mpv** over `python-mpv-jsonipc` on `/tmp/mpvsocket`, `fs=True, idle=True, force_window=True`. `start_mpv` can be disabled in `main_config.json` to attach to an already-running mpv.
- `play_file()` (L157) — existence check → status socket update → **autobump interception** (a path prefixed `:autobump:=` routes to the web renderer instead of mpv) → `panscan`/`keepaspect` → `_apply_vfx()` → `mpv.play()` → `wait_for_property("duration")`.
- `_play_from_point()` (L419) — the inner loop. Iterates the block plan, seeks `entry.skip + initial_skip`, then busy-waits against a **wall-clock `target_end_time`** rather than mpv's position. That is what keeps channels time-accurate. Applies a 0.5s fade when a clip is cut short.
- `play_slot()` → `LiquidManager().get_play_point()`. On `ScheduleNotFound` it calls `schedule_panic()`, which generates one more day of schedule on the fly and reloads.
- `scramble_effects` (L75-85) — ffmpeg `lavfi=[geq=...]` filters for premium-channel scrambling.

`reception.py` holds `ReceptionStatus`, a borg singleton with a `chaos` float driving noise/scroll filters, plus the three transition functions (`short`/`long`/`none`) passed into `main_loop`; the long one plays `runtime/static.mp4` between channels.

### The two "sockets" (plain files, polled)

| File | Written by | Read by |
|---|---|---|
| `runtime/channel.socket` | `command_input.py`, `remote/commands.py`, `api/player.py`, `pi/cable_box.py` | `field_player.input_check()` — reads, then truncates |
| `runtime/play_status.socket` | `station_player.update_status_socket()` | `api/player.py`, `fs42/osd/*`, `command_input.py`, remote UI |

`update_status_socket` is the single publish point for status, network name, channel number, timestamp, title, duration, and file path. Paths default from `StationManager.server_conf`; `runtime/` is gitignored.

### Content and scheduling — one system, three layers

There is exactly one active scheduling system. The naming suggests otherwise, but **Catalog, Liquid, and Fluid are complementary layers, not competing generations.** The genuinely old system was pickled `.bin` files (`catalog_path` / `schedule_path` keys still linger in configs and the JSON schema but nothing reads them) plus an `fs42/series.py` that no longer exists.

**Database:** `runtime/fs42_fluid.db`, path from `server_conf["db_path"]`. Six tables, created lazily by four different IO classes:

| Table | Created in | Holds |
|---|---|---|
| `catalog_entries` | `catalog_io.py:25` | per-station media index: path, title, duration, tag, play count, JSON hints |
| `liquid_blocks` | `liquid_io.py:27` | the built schedule: station, type, start/end, sequence key, break info, plan JSON |
| `named_sequence`, `sequence_entries` | `sequence_io.py:19,30` | series playback position |
| `file_meta`, `break_points`, `chapter_points` | `fluid_statements.py:181-207` | duration/mtime cache, black-frame breaks, ffprobe chapters |

The layering is consistent: `*_io.py` = raw SQL, `*_api.py` = static facade, everything else calls the API.

- **Catalog** (`catalog.py` 536 lines) — `ShowCatalog.build_catalog()` dispatches by network type; `_build_standard()` walks every day/hour slot to discover tags and scans `content_dir/<tag>`. `find_candidate()` filters by duration plus hints, then picks randomly among the lowest-play-count entries. `make_reel_block()` builds commercial pods.
- **Liquid** (schedule) — `LiquidSchedule._fluid()` is the core scheduler: for each wall-clock mark it reads the slot, resolves a tag, picks content or the next sequence episode, rounds duration up to `schedule_increment`, emits a block; gaps become `LiquidOffAirBlock`. `LiquidBlock.make_plan()` prefers chapter markers over black-detect breaks and clips breaks to at most one per two minutes. `LiquidManager` is a borg singleton caching all schedules in memory, with `get_play_point()` walking a block's plan to compute `(index, offset)`.
- **Fluid** (media cache) — `FluidBuilder.scan_file_cache()` avoids re-probing unchanged files; `scan_breaks()` / `scan_chapters()` populate the break and chapter tables; `trim_file_cache()` prunes deleted files.
- **Sequences** — `NamedSequence` sorts episodes by path and derives start/end from percentages. Nice detail: `LiquidManager.reset_sequences()` rewinds to the episode of the first future block when you delete a schedule, so you don't lose your place.
- **Hints** (`schedule_hint.py`) — directory names drive scheduling constraints: `DayPartHint`, `BumpHint`, `MonthHint`, `QuarterHint` (`Q1`-`Q4`), `RangeHint` (`December 1 - December 25`, with correct year-boundary wrap). A subdirectory named `November` or `prime` inside a tag folder automatically restricts that content.
- **`ReelCutter`** interleaves commercial pods into a feature using `break_strategy` (`standard` / `end` / `center`) and chapter positions.

### Configuration

`StationManager` (borg singleton) is the config authority. `load_main_config()` reads optional `confs/main_config.json` overrides — socket paths, `db_path`, server host/port, time formats, `normalize_titles`, `day_parts`. `load_json_stations()` globs `confs/*.json`, runs each through `ConfigProcessor.preprocess`, applies defaults (`network_type=standard`, `schedule_increment=30`, `break_strategy=standard`, `break_duration=120`, `hidden=False`), verifies that sign-off/off-air/standby/BRB media exist on disk, normalizes `clip_shows`, and tags each station `_has_catalog` / `_has_schedule`.

`ConfigProcessor` runs two transforms before anything else sees the config:
- `_process_templates()` (L17) — expands `day_templates` by inlining a named template wherever a day's value is a string.
- `_process_strategy()` (L43) — expands `slot_overrides`, merging a named override's keys into a slot, restricted to a whitelist (`start_bump`, `end_bump`, `bump_dir`, `commercial_dir`, `break_strategy`, `sequence`, `schedule_increment`, `random_tags`, `video_scramble_fx`, `marathon`).

**`.gitignore:13` ignores `*confs/*.json`, so your real station configs are not in version control.** Only `confs/examples/` is tracked.

### Web and display layer

| Path | What it is | Wired in? |
|---|---|---|
| `fs42/fs42_server/` | **The** FastAPI app. Seven routers: `summary`, `catalogs`, `schedules`, `stations`, `themes`, `build` (background-threaded rebuilds with `task_id` polling), `player` (545 lines — status socket reads, channel writes, CPU temp via vcgencmd/thermal_zone/lm-sensors, volume via pactl → amixer → wpctl) | **Active**, from both `field_player.py` and `station_42.py -s` |
| `fs42/fs42_server/static/` | The whole web console: index, catalog, schedule, guide, diagnostics, remote, about, four themes, plus `bump/` — an HTML/CSS/JS station-ID bump page driven by query params | **Active** |
| `fs42/ux/` | Textual TUI — `StationApp` → welcome → catalog/schedule screens, each with a `.tcss` stylesheet | **Active**, via `-g` |
| `fs42/webrender/` | PySide6 `QWebEngineView` full-screen renderer for `network_type: "web"` and for autobumps | **Active**, imported defensively (PySide6 optional) |
| `fs42/overlay/ticker.py` | Frameless PySide6 scrolling news ticker with a QSharedMemory single-instance guard | **Active**, via `POST /player/ticker` → command queue |
| `fs42/guide_tk.py` (392 lines) | Tk rendering of the guide channel; `GuideWindowConf` has ~30 styling knobs validated at startup | **Active**, forked by `show_guide()` |
| `fs42/osd/` | Separate GLFW/PyOpenGL process polling the status socket. `logo_display.py` does per-channel static/animated-GIF logos; `content_classifier.py` classifies current content as commercial/bump/show to hide the logo during ads | **Standalone** — nothing in `field_player.py` starts it; launch `python3 fs42/osd/main.py` yourself |
| `fs42/pi/` | `cable_box.py` (Adafruit matrix keypad + TM1637 display, writes the socket) and `remote_controller.py` (evdev IR/Flirc, talks to the **HTTP API** rather than the socket) | **Standalone**, run manually |
| `fs42/pico/aerial_listener.py` | **CircuitPython**, not CPython — runs on an RP2040, sends JSON over UART | Separate hardware |
| `fs42/guide_render/` | Only `static/left.png` and `static/right.png`. `guide_builder.py:77` defaults `template_dir="fs42/guide_render/templates/"`, **which does not exist** | Vestigial |
| `fs42/remote/` | A second, standalone FastAPI remote app. Nothing imports or launches it; the real remote UI is `fs42_server/static/remote.html` at `/remote` | **Orphaned** |
| `fs42/diagchannel/` | Tk diagnostic channel; `diag_channel_runner` is referenced only by its own `__main__` | **Dead** |

Also stale: `page_stream/example_launcher/example_start_fs42.sh` references `fs42/change_channel.py`, which no longer exists.

### Tests

`test/` holds four files. **51 tests, all passing** at `5bd2007` — verified by running pytest. Pure unit tests, no fixtures, no DB, no I/O.

| File | Tests | Covers |
|---|---|---|
| `test_title_parser.py` | 20 | release-group prefixes, `S06E07`, `s6-e7`, `06x7`, `(1999)`, `Episode 4` |
| `test_autobump_agent.py` | 20 | query param mapping/encoding, missing-title `ValueError`, `gen_bumps` key contract |
| `test_schedule_hint.py` | 11 | `MonthHint`, `QuarterHint`, `RangeHint` including year-boundary wrap |
| `test_series.py` | 0 | Entirely commented out — tests a `fs42.series` module that no longer exists |

**Untested:** the player loop, `LiquidSchedule._fluid`, `ReelCutter`, every `*_io.py` / SQL layer, `ConfigProcessor`, `SlotReader`, `StationManager`, and every API router. That gap is exactly where the §5 regression slipped through.

---

## 2. On "v1" vs "v2"

**There is no v1/v2 versioning in this project, and I found no evidence there ever was.** Specifically: no `__version__` or any version string; **zero git tags**; no mention of "v1", "v2", "legacy", "deprecated", "migration", or "rewrite" in the README, `docs/`, or any of the 51 commit messages. Upstream calls the whole thing "Alpha software."

So "what's done in v2 / what's still v1-only" has no literal answer. What *does* exist is a set of **modernization tracks where a newer mechanism was added alongside an older one that still works**. If that is what you meant, here is the real split.

### Newer mechanism, in place and working

| Area | Newer | Older, still present |
|---|---|---|
| **Config format** | `day_templates` — named reusable day templates plus one-line day references | Explicit per-weekday blocks. Both parse; `_process_templates()` inlines the template at load time and downstream code never knows the difference. |
| **Admin UI** | FastAPI web console on :4242 with a full REST API | Textual terminal UI, still the default for `station_42.py -g` |
| **Metadata store** | SQLite (the fluid/liquid/catalog tables) | Pickled `.bin` files — dead, but `catalog_path`/`schedule_path` keys still linger in configs and the schema |
| **Duration probing** | `ffprobe` (#437, 2025-10-19) | moviepy fallback, still in `media_processor.py` |
| **Config validation** | `station_config_schema.json` + `docs/STATION_CONFIG_README.md` (2025-10-19) | Ad-hoc `if key in dict` checks in `station_manager.py` — **these are what actually run**; see §4 |
| **Bumps** | AutoBump agent — bumpers generated as web pages, injected via a `:autobump:=` path sentinel | Static bump directories |
| **Guide channel** | Web guide (`fs42_server/static/guide.html`) | Tk guide (`guide_tk.py`, 392 lines), still what `show_guide()` actually forks |

### Old-only, no newer equivalent

- **Tk surfaces** — `guide_tk.py` is still the live guide renderer; `diagchannel/` has no replacement.
- **Config loading has no isolation and no schema enforcement** — see §4.
- **Hardware input** (`fs42/pi/`, `fs42/pico/`) is unchanged, undeclared in requirements, and has no API-side equivalent.
- **`hot_start.sh`** is still a bare shell script — no systemd unit, no logging, no restart-on-failure.
- **Guide channels cannot be scheduled** — `liquid_schedule.py:260` and `catalog.py:113` raise `NotImplementedError`. In practice these are correctly guarded (`guide` is in both `no_catalog` and `no_schedule`), so they are landmines rather than live bugs. Worth knowing that the guard lives in a different file from the raise.

---

## 3. Immediate operational issues on the Pi

Live, not historical — this is why the player is not starting:

1. **`confs/music42_settings.json` is malformed** — missing the top-level `"station_conf"` wrapper, producing `KeyError: 'station_conf'`. Because config loading has no per-file isolation (§4), this one file stops *every* station from loading.
2. **`requests` is not installed** in the Pi's virtualenv, and the PPV feature's `tmdb_helper.py` imports it at module scope, killing the API subprocess. Note that `tmdb_helper.py` is part of the **uncommitted** work — the dependency was never added to `install/requirements.txt`, and could not have been, since the code isn't in the repo.

---

## 4. Known bugs and rough edges

All verified by reading the code at `5bd2007`.

### Critical

**Autobump is broken for every user who enables it.** `catalog.py:419-420` is the only production consumer of `AutoBumpAgent.gen_bumps()`, and it still asks for the pre-rename keys:

```python
autos = AutoBumpAgent.gen_bumps(self.config)
start_candidate = autos.get("message_bump", None)   # now always None
end_candidate   = autos.get("next_bump", None)      # now always None
```

`gen_bumps` returns `start_block` / `end_block` (`autobump_agent.py:16,26`). Because `.get()` defaults to `None` rather than raising, this fails silently until:

```python
remaining -= start_candidate.duration   # AttributeError: 'NoneType' has no attribute 'duration'
```

All three strategies crash — `"both"` (the default) leaves both `None`; `"start"` and `"end"` each set only one. The shipped example `confs/examples/indie42.json` has an `autobump` block with no `strategy`, so copying it into `confs/` and building a schedule reproduces this immediately. Full story in §5.

### High

- **The new catch-all turns that crash into an infinite 1-second spin.** `field_player.py:147-153` converts any playback exception into `PlayerState.FAILED`, which routes to the stuck-handler that sleeps one second and retries forever. For a transient failure that is right; for a deterministic one the condition never changes, so the app logs the same error once per second forever instead of failing visibly. Two aggravating details: it logs `f"...{e}"` with **no traceback** (the rest of the codebase uses `logger.exception`), and it wraps only the `play_slot` branch — `show_guide` and `show_web` are still unprotected.
- **`wait_for_property("duration")` still hangs forever.** `station_player.py:216-219`. The comment admits it: *"we can use a simple check or just hope for the best."* `python-mpv-jsonipc` 1.2.1 blocks with no timeout, so on a truncated or corrupt file no exception is ever raised and the `except` is unreachable for the case it claims to handle. The player thread parks inside `play_file`, the wall-clock loop is never reached, `input_check_fn()` is never polled — so **you cannot even change channel away from the broken file.** Only Ctrl-C recovers. A real fix needs a watchdog calling `mpv.terminate()`, or a bounded poll of `mpv.duration`.
- **One malformed config file kills startup for all stations.** `station_manager.py:164-251`. There is a per-file `try`, but every error path ends in `exit(-1)`: missing `station_conf` (caught at 244, exits at **249**), a nonexistent media path (**189**), a malformed clip show (**217**). Stations already validated into `station_buffer` are discarded. `glob.glob` returns arbitrary order, so *which* file gets named in the error varies between runs. This is the direct cause of your current outage.
  - Worse, `station_manager.py:251` sorts by `channel_number` **outside** the try. A config missing `channel_number` passes the entire loop unvalidated and dies here with a bare `KeyError: 'channel_number'` — no filename, no station name, no log line.
  - `station_manager.py:253-256` builds the name and number indexes by plain assignment, so two stations sharing a `channel_number` silently overwrite each other with no warning.
  - `exit(-1)` lives in a library module that the API server, the TUI, and the CLI all construct — a config error hard-kills whichever process touched it, with no HTTP error and no chance for a caller to handle it.
- **Two infinite busy-loops in the channel-change path.** `field_player.py:199-206` (down) and `214-220` (up) are `while not found` loops that exit only on a non-hidden station. If every station is `hidden` — a plausible state, since `hidden` is a documented per-station option — the process pins a core at 100% with no logging, timeout, or sleep.

### Medium

- **`schedule_panic` can loop forever.** `station_player.py:388-403`. It calls `add_days(1)`, appending to the *end* of the schedule, then returns `FAILED` → 1-second retry. If the query is out of bounds in the *past* — exactly what happens on a Pi with no RTC that boots before NTP syncs — extending the future never satisfies it. The result appends a day of schedule to SQLite every second, forever, while the screen stays black. It also constructs `LiquidSchedule(station_by_name(name))` with no `None` check.
- **`play_file`'s return value is discarded.** `station_player.py:441`. It returns `False` for a missing file, but execution falls straight through to `seek()` and the wall-clock wait. If a drive unmounts after the catalog was built, the channel shows black for the *entire scheduled duration* — potentially 30 minutes — with one unheeded log line.
- **Two `join()` calls with no timeout**, both on the user-initiated channel-change path: `station_player.py:312` (guide) and `381` (web). A wedged Tk or Qt child blocks the player forever. Note this is an oversight, not a choice — the duration-expiry path at 364-368 does it correctly with `join(timeout=3)`, `is_alive()`, `terminate()`.
- **Wrong bump pool.** `catalog.py:423` — with `strategy: "start"`, the closing bump is drawn from `prebump` instead of `postbump` (compare the correct non-autobump path at line 409). A user with curated `prebump/` and `postbump/` directories gets "we'll be right back" bumps playing on the way *out* of the break.
- **Dead ternary.** `station_manager.py:222-223`: `if conf["schedule_increment"]: fill_target = 0.95 if conf["schedule_increment"] else 0.73`. The ternary tests the same condition as the enclosing `if`, so `0.73` is unreachable — and the adjacent comment ("0.73 is the calculated average for content vs breaks") shows it was meant to apply somewhere. Clip-show durations are silently wrong.
- **Undeclared dependencies.** `fs42/pi/remote_controller.py` imports `evdev` and `requests`; **neither appears in `requirements.in` or `requirements.txt`**, and `install.sh` only installs from the latter, so the remote-controller feature `ImportError`s on a fresh install. (`pi/README_remote_controller.md` does say to `pip install evdev requests` manually — but then says to run under `sudo`, which won't see a user venv.) Separately, `PIL` is imported directly by `guide_tk.py` and `osd/render.py` but is only present transitively via moviepy.
- **The JSON schema is decorative.** `station_config_schema.json` declares `required: ["network_name", "channel_number"]`, but **nothing references it** — there is no `jsonschema` import anywhere and no such dependency. All validation is the ad-hoc checks above. Wiring this in, per-file, would fix an entire class of bug at once.

### Low

- **CI cannot fail.** `.github/workflows/unit_tests.yml` sets `continue-on-error: true` with `# TODO: This should be set to false or removed soon! (once tests no longer fail)`. Tests all pass now, so **this can be turned off today** — it is the cheapest quality win available, and it is what let the §5 regression through.
- **Four bare `except:` clauses** in `api/player.py` (470, 487, 518, 545) in the audio helpers. Bare `except` catches `KeyboardInterrupt` and `SystemExit`. Line 545 returns the literal `"MUTE"` on any error — reporting the system as muted when it merely failed to ask.
- **`fs42_server.py:42-50`** — `except Exception: pass` around `shutdown_queue.get_nowait()`. The expected exception is `queue.Empty`; catching everything means a broken queue degrades into a loop that can never see the shutdown message, so the API child outlives its parent.
- **`station_player.py:130-138`** — `except Exception: pass` around web-process teardown, then `self.web_process = None` drops the only handle: a guaranteed orphan with no log line.
- **`build.py:36,139`** — `station_by_name()` result used with no `None` check, so a typo'd name surfaces in the UI as `Error: 'NoneType' object is not subscriptable` instead of "no such station". `build.py:13-16` — task dicts are never evicted, so a long-running server leaks one entry per rebuild.
- **`command_input.py:8`** opens the serial port at **module import time**, so importing it on any non-Pi machine raises `SerialException`. Nothing imports it today, which is the only reason this doesn't bite.
- **`guide_builder.py:121`** — guide layout hardcoded to 3 time blocks (*"this isn't an extendable approach"*); **`:132`** — no 12/24-hour option. **`osd/render.py:148`** — *"TODO: figure out which monitor to use"*.
- **`hot_start.sh` discards all output** — the README itself warns it "swallows output so you'll never know what's going wrong." Every boot failure is silent; a log file instead of `/dev/null` would have made the current outage self-diagnosing.

**Checked and *not* a bug:** `PlayerOutcome(PlayerState.FAILED)` from the last commit is valid — `PlayerState.FAILED` exists (`station_player.py:61`) and both names are imported.

---

## 5. What you were working on last

### Committed: `5bd2007`, 2026-02-01 — your only commit in the visible history

Three changes:

1. **`autobump_agent.py`** — `gen_bumps()` return keys renamed `message_bump`/`next_bump` → `start_block`/`end_block`.
2. **`field_player.py`** — wrapped `play_slot()` in try/except so a playback exception degrades to `FAILED`.
3. **`station_player.py`** — try/except around `wait_for_property("duration")`.

**What prompted it:** `test/test_autobump_agent.py` was added on 2025-09-25 in the "Autobump implementation" commit already asserting `start_block` / `end_block`, while the implementation returned the other names. Verified by running the suite at the parent commit:

```
3 failed, 48 passed    (KeyError: 'end_block')
```

and at `5bd2007`:

```
51 passed in 0.10s
```

So a real mismatch had sat there since September 2025, invisible because CI is `continue-on-error`.

**But the fix went the wrong direction.** The rename made the *tests* pass while breaking the *only production caller*, `catalog.py:419-420`, which was never updated (§4, Critical). The suite is green and the feature is broken — and change #2, the new catch-all, is what hides it in the field: instead of an `AttributeError` traceback you get a black screen and a log line repeating once per second.

**Suggested fix** — small, and the highest-value thing in this document:

```python
# fs42/catalog.py:419-420
start_candidate = autos.get("start_block", None)
end_candidate   = autos.get("end_block", None)
```

Then guard the `remaining -=` lines at 427-428 against `None`, and while you are in there, change line 423's `ShowCatalog.prebump` to `postbump`. Add a test that exercises `make_reel_block()` with an autobump config — that is the coverage gap that allowed this.

### Uncommitted: the work actually on the Pi

This is the part that matters most. Reconstructed from the Pi's traceback (§0):

- **`fs42/station_io.py`** — a new module (320+ lines) extracting config loading out of `StationManager` into a `StationIO` class with `load_and_process_all_stations()` and `_process_single_config()`, with `station_manager.py` rewired to delegate to it. From the traceback it logs errors and continues rather than calling `exit(-1)` — i.e. it appears to be your fix for exactly the fragility described in §4. (Inferred from the traceback shape; I cannot read the file.)
- **`fs42/fs42_server/api/ppv.py`** and **`fs42/fs42_server/api/tmdb_helper.py`** — a pay-per-view channel with TMDB metadata lookup, registered as an eighth router.

**None of this is committed, pushed, or backed up.** It exists on one SD card in a device that currently will not boot into the player. Recover it first:

```bash
cd ~/FieldStation42
git status
git add fs42/station_io.py fs42/fs42_server/api/ppv.py fs42/fs42_server/api/tmdb_helper.py
git add -u                       # picks up station_manager.py, api/__init__.py
git commit -m "WIP: station_io refactor and PPV/TMDB channel"
git push -u origin main
```

Add `requests` to `install/requirements.txt` in the same commit. And reconsider whether `confs/*.json` should stay gitignored — right now your station lineup has no backup either.

---

## 6. Git log summary

### "Since March": nothing

**There are zero commits after 2026-02-01.** Whichever March you meant, two facts shape the answer:

- Nothing has been committed in months; the repository has been dormant since February.
- The clone is shallow at 2025-08-24, so **March 2025 is not in this clone at all** — I cannot summarize it without `git fetch --unshallow`.

What follows is therefore the complete visible history: 51 commits, 2025-08-24 → 2026-02-01.

| Month | Commits |
|---|---|
| 2025-08 | 10 |
| 2025-09 | 18 |
| 2025-10 | 22 |
| 2025-11 – 2026-01 | 0 |
| 2026-02 | 1 |

Authorship is overwhelmingly upstream: **Shane C Mason** (47), plus **Tim** (#437), **daseinkapital** (#390), and **sheltojeff** (1). This fork tracks upstream closely and carries essentially no local divergence *in git* — which makes the uncommitted Pi work (§5) all the more anomalous.

**August 2025 — configuration and web guide.** News ticker ("Stealth monkey", #389); memory limiter for the CLI (#390); configuration templates and the first "new stye config examples" on 2025-08-30 (the `day_templates` format); channel numbers in the web guide; title parser improvements; status reporting fixed to show block title rather than current file title.

**September 2025 — autobump and guide polish.** The headline is **AutoBump** (2025-09-25): "Autobump implementation", "improve closing auto-bump", an example config using it, and autobump positioning (09-29) — the feature you later touched. Alongside: guide rendering fixes (scroll speed as a parameter, reset animation, column sizing), `normalize_title` in `main_config.json`, batch load improvements, standby image fix, PySide import protection, several `sequence_api.py` iterations, a more permissive number parser.

**October 2025 — playback correctness, the busiest month.** Heavy bump-scheduling work: "Correct pre and post bump scheduling", "Correct start_bump and end_bump", "…on clip shows", "Correct issues in clip playback - especially autobumps". **Chapter marker support** landed (10-04) and was made opt-out (10-17). Audio: volume mute fix, volume capped at 100%. Remote debounce. `black_detect` fixed. **Station config JSON schema and detailed documentation** (10-19) plus **ffprobe duration probing** (#437, Tim). The month closes on robustness — "Dont fail on schedule not found" and "Small code cleanups and robustness improvements" (both 10-21), the last upstream activity in this clone.

**November 2025 – January 2026 — dormant.**

**February 2026 — your commit.** `5bd2007`, detailed in §5.

---

## 7. Suggested next steps

1. **Commit and push the Pi's uncommitted work** (§5). Everything else here is reversible; this is not.
2. Fix `confs/music42_settings.json` and `pip install requests` to get the player running again (§3).
3. **Fix `catalog.py:419-420`** — three lines, and it un-breaks autobump (§4, Critical).
4. Flip `continue-on-error` to `false` in CI. The suite is green, so the safety net now costs nothing — and it is what would have caught #3.
5. Give `hot_start.sh` a log file instead of `/dev/null`, or move to a systemd unit with `Restart=on-failure` and journald logging.
6. Land the `station_io.py` per-file config isolation properly, and wire up `station_config_schema.json` so validation is declarative rather than ad-hoc. Add a test that a malformed config skips that station instead of stopping the system.
7. `git fetch --unshallow`, and add `shane-mason/FieldStation42` as an `upstream` remote — this fork is roughly ten months behind whatever has happened upstream since October 2025.
