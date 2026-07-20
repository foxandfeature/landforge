# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Landforge has no application source code. The entire project is a single GitHub Actions
pipeline — `.github/workflows/build-tiles.yml` — that builds a global land-polygon [PMTiles](https://protomaps.com/)
archive from OpenStreetMap data and publishes it as a GitHub Release asset. There is no
package.json, build system, or test suite; "development" here means editing the workflow YAML
(mostly embedded bash and Python) and validating it by running the workflow.

Repo-tracked files:
- `.github/workflows/build-tiles.yml` — the entire pipeline, described below.
- `known_broken_regions.json` — Geofabrik region paths whose shapefile is currently listed but
  broken server-side (e.g. `europe/united-kingdom/england/london/enfield`). These are excluded
  from automatic `ALL_REGIONS` runs but can still be targeted by an explicit manual run against
  that exact region (with a warning). Remove an entry once Geofabrik fixes the underlying file.
- `regions.json` (being removed) — a formerly hand-maintained snapshot of Geofabrik's region
  list. It's superseded by fetching Geofabrik's live `index-v1-nogeom.json` at
  `prepare-region-queue` time (see below), so the candidate region list never drifts out of date.

## Running / testing changes

There's no local build or test command — everything runs as GitHub Actions. To validate a
change to the workflow:
- Trigger it manually via `workflow_dispatch` (Actions tab, or `gh workflow run build-tiles.yml
  -f region=<path>`) against a single small region (e.g. `europe/monaco`) rather than
  `ALL_REGIONS`, to get fast feedback.
- The `region` input takes a Geofabrik path (e.g. `europe/germany/niedersachsen`) or
  `ALL_REGIONS`. Only leaf regions that ship their own `-latest-free.shp.zip` are valid — bundle
  regions like "US Midwest" or "DACH" are rejected with a clear error (they're filtered out of
  `ALL_REGIONS` automatically too).
- `debug_run_id` + `debug_resume_at` let you re-run just the back half of the pipeline
  (`pmtile-generation`, or `regions-merge` to redo the combine step) against already-uploaded
  artifacts from a previous run, instead of re-downloading/re-clipping every region.
- YAML/shell syntax can be sanity-checked locally with `actionlint` if available, but there is
  no substitute for an actual workflow run for logic changes.

## Pipeline architecture

Jobs run in a `needs:`-chained graph, coordinated via three dedicated **orphan branches** that
exist purely as data/coordination stores and are never merged into `main`:

- `region-timings` — holds two files: `region_timings.json`, a rolling history (last 5 runs) of
  per-region ndjson-build time, and `tile_timings.json`, a rolling history of per-tile-grid-cell
  tiling time (see below for why these are two separate histories). Both updated at the end of
  every run; never cleaned up.
- `region-queue` — holds `queue.json`, the live shared work queue for `build-tiles-ndjson`'s
  per-region claims this run only. Force-pushed fresh at the start of every run and deleted at
  the end (`cleanup-region-queue`).
- `tile-queue` — a second, independent live work queue, for `build-tiles-highzoom`'s per-tile
  claims this run only. Force-pushed fresh after `build-tiles-ndjson` finishes (by
  `prepare-tile-queue`) and deleted at the end (`cleanup-tile-queue`).

A top-level `concurrency` group serializes entire workflow runs so two runs never race over
these branches or the shared caches.

Region processing is split into two phases, each with its own worker pool and its own dynamic
work queue: **`build-tiles-ndjson`** (download/clip/erase → NDJSON, sharded per Geofabrik region)
and **`build-tiles-highzoom`** (combined NDJSON → tiled `z8-14` shard, sharded per **tile-grid
cell** instead — see below for why). They're separate jobs — not just separate steps in one job —
because GitHub Actions can only gate a downstream job on an entire upstream job finishing, not on
an individual step, and `build-low-zoom-pmtiles` needs to run concurrently with
`build-tiles-highzoom` once both of their shared prerequisite (`combine-land-data`) is done, rather
than waiting on tiling to finish first.

**Why tiling shards by a coarse tile-grid cell (`TILE_GRID_ZOOM`, default z=4, 256 cells) instead
of by region:** two regions that are geographic neighbors can both contribute features to the same
z8-14 output tile (e.g. a shared coastline near a border). Sharding by region meant each region's
shard decided simplification/`--drop-smallest-as-needed` using only its own slice of that shared
tile - `tile-join` then had to merge two independently-simplified, potentially inconsistent
versions of it. Because z8-14 tiles nest exactly inside their `TILE_GRID_ZOOM` ancestor cell
(integer division of the tile coordinates), sharding by grid cell instead means no two work units
can ever produce the same z8-14 tile - each `build-tiles-highzoom` worker downloads the *whole*
combined world land stream once (published by `combine-land-data`) and just clips it fresh to
whichever cell it claims, so simplification for a tile always sees everything that could land in
it. This also means tiling can no longer use a fixed 1:1 worker assignment from phase 1 (tiling
cost per cell doesn't track any single region's ndjson-build cost), so it gets its own dynamic
queue (`tile-queue`, seeded by `prepare-tile-queue`) exactly like phase 1 has for regions.

Because every worker clips its own copy of the same combined stream rather than fetching specific
regions per tile, `prepare-tile-queue` doesn't need to know which regions overlap which cell either
- it just enumerates every one of the `TILE_GRID_ZOOM` grid's cells as a work unit. A cell that
turns out to be pure ocean isn't filtered out ahead of time; `build-tiles-highzoom` discovers that
itself when the per-cell clip comes back empty and skips tiling it on the spot (see
`tile_work_unit`'s `[ ! -s clipped.ndjson ]` check) - cheap enough that predicting it up front
isn't worth the extra machinery. An earlier design instead had `build-tiles-ndjson` publish each
region's bbox alongside its NDJSON so `prepare-tile-queue` could bbox-overlap-test every region
against every cell and drop non-overlapping ones; that's gone now that tiling clips from one
combined download instead of fetching per-region artifacts on demand.

1. **`prepare-region-queue`** — fetches Geofabrik's live region index (`index-v1-nogeom.json`)
   instead of using a repo-committed list, derives the leaf regions that have their own shapefile,
   filters out `known_broken_regions.json` (for `ALL_REGIONS`/scheduled runs only), estimates
   per-region time from `region_timings.json` history, sorts regions **largest-estimate-first**,
   and publishes that sorted list as `queue.json` on the `region-queue` branch. Also decides
   the worker pool size (`NUM_WORKERS`, up to 20, capped by region count).

2. **`setup-tools`** (parallel to `prepare-region-queue`) — builds/caches Tippecanoe from source
   (cache keyed on its latest upstream commit hash) and downloads/caches the global OSM land
   polygons (`land-polygons-split-4326`, cache keyed by month so it refreshes roughly in step with
   the monthly cron).

3. **`build-tiles-ndjson`** — a matrix of workers (`strategy.matrix.worker`, `fail-fast: false`).
   Each worker loops: **claim** the region at the front of `region-queue` (optimistic
   commit-and-push, retried on lost races), then run `build_region_ndjson`:
   - Download the region's `.poly` boundary and `.shp.zip` water layer from Geofabrik, with
     retry/backoff that distinguishes permanent 4xx failures from transient ones, plus a
     post-download `unzip -t` integrity check (a Geofabrik shapefile can 200 but be corrupt —
     this happened with `enfield`, hence `known_broken_regions.json`).
   - Convert the `.poly` boundary to WKT (`poly2wkt.py`, generated inline).
   - Filter the region's water layer to drop glaciers/wetlands (they're not open water and
     would erase land that should stay solid) — done before the clip so the water layer is
     already clean by the time it's used to erase.
   - `ogr2ogr -clipsrc` the global land polygons down to this region's boundary.
   - `mapshaper -erase` the filtered water out of the clipped land, writing **newline-delimited**
     GeoJSON (`format=geojson ndjson` is required — a plain FeatureCollection would break the
     later global concatenation and Tippecanoe's streaming parser).
   - gzip the NDJSON and verify the archive isn't corrupt. Every region a worker finishes stays in
     its own workspace as `<slug>.ndjson.gz` rather than being published individually; the whole
     batch is bundled into one `ndjson-worker-N` artifact at the end of the job instead, consumed
     by `combine-land-data` (and, indirectly through that combined stream, `build-tiles-highzoom`).
   - Every step is explicitly checked with `|| return 1` rather than relying on `set -e`,
     because this function is invoked as `if ! build_region_ndjson ...` — bash suppresses
     `errexit` for the entire duration of any command tested by `if`/`!`, including inside called
     functions. A silent unchecked failure previously let a corrupt/empty archive get uploaded
     as if the region had succeeded.
   - A failed region is logged and skipped; it does not fail the whole worker or block other
     regions.
   - Worker artifacts (`ndjson-worker-N`, `region-timings-ndjson-worker-N`) are uploaded with
     `if: always()` so partial progress survives even if the worker's loop ultimately exits
     non-zero.

4. **`prepare-tile-queue`** — enumerates every cell of the `TILE_GRID_ZOOM` grid (a pure function
   of that env var, independent of which regions succeeded in phase 1 - see above for why),
   estimates each cell's workload from `tile_timings.json` history (falling back to `0`, which
   just sorts it after every cell with real history), sorts **largest-estimate-first**, and
   publishes the flat list of `"z/x/y"` tile IDs as `queue.json` on `tile-queue`.

5. **`build-tiles-highzoom`** — a second matrix of workers, independent from `build-tiles-ndjson`'s.
   Each worker downloads the whole `world-land-ndjson` combined stream (published by
   `combine-land-data`) exactly once, then loops: **claim** a tile-grid cell from `tile-queue`,
   clip that shared local copy down to the cell's exact bounds (`ogr2ogr -f GeoJSONSeq -clipsrc`),
   and - unless the clip comes back empty (pure ocean, nothing to tile, logged and skipped) - tile
   it into that cell's own `z8-14` PMTiles shard (`-pS --simplification=10
   --drop-smallest-as-needed`). `-pS` exempts the deepest zoom (14) from `--simplification=10`.
   Because z8-14 tiles nest exactly inside their grid-cell ancestor, no two work units can ever
   emit the same z8-14 tile (see `build-low-zoom-pmtiles` below for why the low zoom range still
   needs a genuinely global view instead). Worker artifacts (`highzoom-worker-N`, bundling every
   cell's own `.pmtiles` shard this worker tiled, and `tile-timings-highzoom-worker-N`) are
   uploaded with `if: always()`.

6. **`cleanup-region-queue`** — deletes the `region-queue` branch once `build-tiles-ndjson` is
   done (`prepare-region-queue` recreates it fresh next run). **`cleanup-tile-queue`** does the
   same for `tile-queue` once `build-tiles-highzoom` is done.

7. **`update-region-timings`** and **`update-tile-timings`** — two separate jobs, each merging one
   worker-timing history into its own file on the shared `region-timings` branch, keeping the last
   5 samples each: `update-region-timings` merges every `build-tiles-ndjson` worker's phase-1
   timings into `region_timings.json` (purely ndjson-build time per region — the sole input to
   `prepare-region-queue`'s region-queue ordering), needing only `build-tiles-ndjson`;
   `update-tile-timings` merges every `build-tiles-highzoom` worker's phase-2 timings into
   `tile_timings.json` (per tile-grid cell — the input to `prepare-tile-queue`'s ordering), needing
   only `build-tiles-highzoom`. These are two separate histories, not summed into one: a tile's
   tiling cost isn't attributable to any single region once tiling shards by grid cell instead of
   by region. Splitting the job in two (not just the histories) lets `update-region-timings` run
   right after `build-tiles-ndjson` instead of waiting on the much longer tiling phase to finish
   too, since it has nothing to do with tiling. Both jobs push to the same branch with
   rebase-and-retry to handle races, including the case where the two land close together.

8. **`combine-land-data`** — downloads every `build-tiles-ndjson` worker's bundled
   `ndjson-worker-N` artifact, integrity-checks each region's archive (`gzip -t`) and concatenates
   them into one `world_land.ndjson` stream, published as the single `world-land-ndjson` artifact.
   A corrupted/truncated archive is skipped with a warning rather than aborting the whole merge,
   but the job still exits non-zero so it's visible, while still uploading the partial result
   (`if: always()`) so downstream jobs get whatever succeeded. Needs only `build-tiles-ndjson` (not
   `build-tiles-highzoom`); both `build-low-zoom-pmtiles` and `build-tiles-highzoom` need this
   job's combined stream in turn (the low-zoom pass for its global view, tiling workers to clip
   their own per-cell copies from), and run concurrently with each other once it's done.

9. **`build-low-zoom-pmtiles`** — runs Tippecanoe once, globally, over the combined stream, but
   only for `z0-7` (`--simplification=10 --drop-smallest-as-needed`, uniformly across the whole
   range — no `-pS` here since nothing in this shard is the "final" deepest zoom). This range
   can't be split per-worker like `z8-14` can: at low zoom many regions collapse into the same
   handful of shared tiles (every region on Earth maps to `0/0/0` at `z0`), so simplification and
   size-cap decisions need the whole picture in view or two regions sharing a tile could end up
   inconsistently simplified. It's still cheap as a single global pass because `z0-7` has orders
   of magnitude fewer tiles than the full `z0-14` range.

10. **`build-pmtiles`** — `tile-join`s the low-zoom shard from `build-low-zoom-pmtiles` together
    with every tile-grid cell's own `z8-14` shard from `build-tiles-highzoom` (one `.pmtiles` per
    successfully tiled cell, bundled per worker into `highzoom-worker-N`) into the final
    `world.pmtiles`. Needs both `build-tiles-highzoom` and `build-low-zoom-pmtiles`, which by this
    point have been running concurrently. Uses `-pk`/`--no-tile-size-limit` since tile-join's own
    writer re-checks the 500KB cap on every tile it copies into the joined output and, by default,
    silently drops (rather than keeps) one that's still oversized — worse than letting a rare tile
    stay a bit large, given each shard already enforced that cap itself via
    `--drop-smallest-as-needed` before this join. (This no longer has to guard against two
    different high-zoom shards colliding on the same tile — sharding by tile-grid cell rules that
    out structurally, see `build-tiles-highzoom` — but a single shard's own tile occasionally still
    exceeding the cap despite `--drop-smallest-as-needed` remains possible.) This two-tier zoom
    split (global low-zoom pass + per-cell high-zoom shards, joined at the end) replaced an earlier
    design that ran Tippecanoe once, globally, across all 15 zoom levels in this job — that
    single-runner build was hitting GitHub Actions' hard 6-hour per-job limit on hosted runners
    before it even finished.

11. **`publish-release`** — uploads `world.pmtiles` as an asset on a single rolling GitHub
    Release (tag `land-tiles-latest`), overwritten in place (`--clobber`) every run. Consumers
    should always fetch via `/releases/latest/download/world.pmtiles`, relying on GitHub's
    Range-request-capable CDN for `pmtiles.js`/MapLibre GL streaming.

## Conventions worth preserving when editing the workflow

- Region paths are Geofabrik paths (`continent/country/subdivision`), always derived from the
  `shp` URL in Geofabrik's index rather than hand-walking the `parent` chain — the `properties.id`
  field is only a leaf slug, not the full path.
- Any per-region failure must be isolated (log + skip + continue), never allowed to take down a
  whole worker or the run — the pipeline processes potentially hundreds of regions per run and
  a single bad Geofabrik file is expected, not exceptional.
- Downloads distinguish permanent failures (4xx → fail fast, don't retry) from transient ones
  (retry with exponential backoff) — follow this split for any new network call.
- Comments in the file carry real design rationale (why orphan branches, why NDJSON, why
  simplify only low zooms, etc.) — read them before changing the surrounding logic, and extend
  them rather than deleting them when the reasoning still applies.
