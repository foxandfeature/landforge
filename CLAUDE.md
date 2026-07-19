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
  list. It's superseded by fetching Geofabrik's live `index-v1-nogeom.json` at `prepare` time
  (see below), so the candidate region list never drifts out of date.

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

Jobs run in this order (`needs:` chain), coordinated via two dedicated **orphan branches** that
exist purely as data/coordination stores and are never merged into `main`:

- `region-timings` — holds `region_timings.json`, a rolling history (last 5 runs) of per-region
  processing time, used to estimate and sort work. Updated at the end of every run; never
  cleaned up.
- `region-queue` — holds `queue.json`, the live shared work queue for the current run only.
  Force-pushed fresh at the start of every run and deleted at the end (`cleanup-region-queue`).

A top-level `concurrency` group serializes entire workflow runs so two runs never race over
these branches or the shared caches.

1. **`prepare`** — fetches Geofabrik's live region index (`index-v1-nogeom.json`) instead of
   using a repo-committed list, derives the leaf regions that have their own shapefile,
   filters out `known_broken_regions.json` (for `ALL_REGIONS`/scheduled runs only), estimates
   per-region time from `region_timings.json` history, sorts regions **largest-estimate-first**,
   and publishes that sorted list as `queue.json` on the `region-queue` branch. Also decides
   the worker pool size (up to 20, capped by region count).

2. **`setup-tools`** (parallel to `prepare`) — builds/caches Tippecanoe from source (cache keyed
   on its latest upstream commit hash) and downloads/caches the global OSM land polygons
   (`land-polygons-split-4326`, cache keyed by month so it refreshes roughly in step with the
   monthly cron).

3. **`build-tiles`** — a matrix of workers (`strategy.matrix.worker`, `fail-fast: false`). Each
   worker loops: **claim** the region at the front of the shared queue (optimistic
   commit-and-push against `region-queue`, retried on lost races), then **process** it:
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
   - Tile that region's own `z8-14` PMTiles shard directly from the freshly-produced NDJSON
     (`-pS --simplification=10 --drop-smallest-as-needed`), right here, per region — not batched
     across a worker's regions or routed through a gzip round-trip first. Doing it in the same
     `$SECONDS`-timed span as everything else in `process_region` means a region that's slow to
     *tile* (not just slow to download/clip) still shows up in `region_timings.json`, so the
     dynamic largest-estimate-first queue naturally learns to schedule it earlier next run — the
     timing/scheduling machinery doesn't need to know or care that tiling happens here too. Safe
     to do per-region rather than needing a global view: at `z8+` a tile essentially never spans
     more than one region's own coastline, so there's no cross-region density decision that could
     come out inconsistent between regions. (See `build-pmtiles` below for why the low zoom range
     can't be split this way.)
   - gzip the raw NDJSON too (separately from the tiles), verify the archive isn't corrupt, and
     record elapsed time. This copy still feeds `combine-land-data` for the global low-zoom pass,
     which needs the raw, unclipped-detail geometry rather than anything derived from a region's
     own tiles.
   - Every step is explicitly checked with `|| return 1` rather than relying on `set -e`,
     because this function is invoked as `if ! process_region ...` — bash suppresses `errexit`
     for the entire duration of any command tested by `if`/`!`, including inside called
     functions. A silent unchecked failure previously let a corrupt/empty archive get uploaded
     as if the region had succeeded.
   - A failed region is logged and skipped; it does not fail the whole worker or block other
     regions.
   - Worker artifacts (`land-ndjson-worker-N`, `region-timings-worker-N`, `highzoom-worker-N` —
     the last bundling every region's own `.pmtiles` shard this worker produced) are uploaded
     with `if: always()` so partial progress survives even if the worker's loop ultimately exits
     non-zero.

4. **`cleanup-region-queue`** — deletes the `region-queue` branch once `build-tiles` is done
   (`prepare` recreates it fresh next run).

5. **`update-timings`** — merges every worker's per-region timings into `region_timings.json`
   on the `region-timings` branch (keeping the last 5 samples per region), pushing with
   rebase-and-retry to handle races.

6. **`combine-land-data`** — downloads every worker's `.ndjson.gz`, integrity-checks
   (`gzip -t`) and concatenates them into one `world_land.ndjson` stream. A corrupted/truncated
   artifact is skipped with a warning rather than aborting the whole merge, but the job still
   exits non-zero so it's visible, while still uploading the partial result (`if: always()`) so
   `build-low-zoom-pmtiles` gets whatever succeeded. This stream now only feeds the low-zoom
   shard below — the `z8-14` shards never pass through it, they're tiled directly from each
   region's own NDJSON inside `process_region`, in `build-tiles`.

7. **`build-low-zoom-pmtiles`** — runs Tippecanoe once, globally, over the combined stream, but
   only for `z0-7` (`--simplification=10 --drop-smallest-as-needed`, uniformly across the whole
   range — no `-pS` here since nothing in this shard is the "final" deepest zoom). This range
   can't be split per-worker like `z8-14` can: at low zoom many regions collapse into the same
   handful of shared tiles (every region on Earth maps to `0/0/0` at `z0`), so simplification and
   size-cap decisions need the whole picture in view or two regions sharing a tile could end up
   inconsistently simplified. It's still cheap as a single global pass because `z0-7` has orders
   of magnitude fewer tiles than the full `z0-14` range.

8. **`build-pmtiles`** — `tile-join`s the low-zoom shard from `build-low-zoom-pmtiles` together
   with every region's own `z8-14` shard from `build-tiles` (one `.pmtiles` per successfully
   processed region, bundled per worker into `highzoom-worker-N`) into the final `world.pmtiles`.
   Uses `-pk`/`--no-tile-size-limit` — tile-join's *own* default behavior for a tile that's still
   oversized after being merged from multiple shards (e.g. two neighboring regions' shards
   sharing a border tile) is to skip it outright, silently punching a hole in the map; `-pk` keeps
   that from happening, since each shard already enforced the 500KB cap itself before merging.
   This two-tier zoom split (global low-zoom pass + per-region high-zoom shards, joined at the
   end) replaced an earlier design that ran Tippecanoe once, globally, across all 15 zoom levels
   in this job — that single-runner build was hitting GitHub Actions' hard 6-hour per-job limit
   on hosted runners before it even finished.

9. **`publish-release`** — uploads `world.pmtiles` as an asset on a single rolling GitHub
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
