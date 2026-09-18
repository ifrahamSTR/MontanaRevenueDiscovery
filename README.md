# Montana STR Revenue Discovery

A statewide geographic discovery tool built from AirDNA/USPPD Montana data
(pulled via Snowflake). It is **not** a buy-box or market-selection report —
see the Charlotte project (`../../../../7AugBuyBox/Charlotte/webpage`) for that
kind of deliverable, or the sibling Idaho (`../../Idaho/webpage`) and Utah
(`../../Utah/webpage`) projects for the first two trials of this same
discovery-tool approach. This site exists to make geographic concentrations
of strong-revenue Montana listings visually discoverable, so an analyst can
decide where deeper market research is worth doing. The map does not
pre-select markets, pre-define clusters, or rank anything — every read is
left to the person looking at it.

This is the third state built on the Idaho site's architecture (see
"Reusing this for another state" in the Idaho README, and its
"Region discovery"/data-pipeline design) — same code, same method, entirely
new data, entirely new region-discovery result and regulatory research.

## Running locally

Plain HTML/CSS/JS, no build step, no framework. From this directory:

```
python3 -m http.server 8000
```

Then open `http://localhost:8000/index.html`. It will not work from `file://`
because of the `fetch()` call in `js/main.js`.

## Structure

Identical to the Idaho site's structure (see that project's README for full
detail on each file) — `index.html` (statewide shell), `region.html` (one
regional-page template, copied verbatim into every `regions/<slug>/`),
`js/config.js` (state-specific constants), `js/util.js`, `js/map.js`,
`js/charts.js`, `js/main.js` (statewide bootstrap), `js/region.js` (regional
bootstrap), `scripts/generate_map_data.py` (listings + region discovery),
`scripts/generate_region_pages.py` (region-page generator),
`data/listings.json` + `data/regions.json` (generated, don't hand-edit),
`data/regulations.json` (hand-curated STR regulatory research, keyed by
region slug — never touched by the generator scripts).

## Region discovery

Same DBSCAN-based method as Idaho and Utah, **re-tuned for Montana's
geography** — the opposite problem from Utah's: Montana's high-revenue
listings sit far more sparsely across a much larger state, so Utah's tight
`eps=8km`/`merge=25km` would fail to even form a cluster in several genuinely
real markets. Verified by inspecting cluster city composition at
`eps=10/12/15/20/25km`, then computing pairwise centroid distances between
the resulting clusters at `eps=15km` before choosing a merge threshold:

- `DBSCAN_EPS_KM = 15`, `DBSCAN_MIN_SAMPLES = 4`
- `MERGE_KM = 35` (vs. Idaho's 30, Utah's 25)
- `REGION_BUFFER_KM = 15` (unchanged)

The Paradise Valley (Emigrant/Livingston/Pray) and Gardiner clusters sit only
~33km apart — almost certainly one contiguous Yellowstone-River corridor
split by a gap in $90k+ listings — while every other pair of genuinely
distinct markets (e.g. Big Sky-to-Bozeman, Gardiner-to-West Yellowstone) sits
at 49km or farther. `MERGE_KM=35` is the smallest value that unions the
former without accidentally fusing the latter.

Produced **8 regions**, not a forced count — see `REGION_LABELS` in
`scripts/generate_map_data.py` for the resulting city→region-name mapping:
Bozeman, Flathead Valley & Glacier National Park, Big Sky, Paradise Valley &
Yellowstone North, West Yellowstone, Missoula, Red Lodge, and Billings.

## Data notes

- Source: `Snowflake/Montana/Iqbal_Montana_2026-09-18-0155.csv` — unlike
  Idaho's and Utah's exports, this filename correctly says "Montana."
  Confirmed `STATE_NAME == "Montana"`, 4,305 raw rows. Entire home/apt
  listings with 20+ reviews and 230+ active listing nights (LTM), already
  filtered upstream; 28 further excluded here per the CSV's own `EXCLUDE` QC
  flag — 4,277 in the working dataset. This is a much smaller dataset than
  Idaho's (5,736) or Utah's (11,386), and only 281 listings (6.6%) clear the
  $90k actual-revenue threshold — expect noticeably smaller, sparser regions
  than either prior state.
- `LOCATION_TYPE` has the same **three** categories as Idaho (not Utah's
  five): `Destination/Resort - Mountains/Lake`, `Mid-Size City`,
  `Small City/Rural` — `js/config.js`'s `locationTypeOrder` and the script's
  `LOCATION_TYPE_DISPLAY` were reverted to Idaho's set rather than kept at
  Utah's five-category version.
- `REVENUE_LTM` (actual) and `REVENUE_POTENTIAL_LTM` (AirDNA's optimized-
  calendar modeled ceiling) are both carried through and are switchable live
  on the map — never silently conflated.
- Statewide pattern check before reusing Idaho's/Utah's Findings-section
  narrative copy: ADR correlates with actual revenue at r=0.92 (occupancy
  r=0.10); $90k+ listings run roughly 2x the median bedrooms/accommodates of
  the rest of the market; the hot-tub revenue lift holds even controlling for
  bedroom count (22.6%→71.3% hit rate on 5BR+ homes, 1.9%→18.8% on smaller
  ones). One notable **difference from both Idaho and Utah**: Townhouse, not
  House or Cabin, converts at the highest rate here (12.8%, vs. House 10.7%,
  Cabin 5.6%, Condo 3.0%, Apartment 0.4%) — driven almost entirely by
  resort-town townhome developments in Whitefish, Big Sky, and Bozeman (all
  26 of the $90k+ townhouse hits sit in one of those three markets, or West
  Yellowstone). Cabin, Montana's other iconic property type, actually
  under-converts House here — the Findings copy was written fresh for this
  pattern rather than copied verbatim from either prior state.
- STR regulatory research (`data/regulations.json`) was gathered per-region
  — see the Overall regulation tier and sourcing in each region's own page.
  Montana, like Idaho, does not currently have a single sweeping statewide
  STR-preemption law on the scale of Idaho's HB 583; verify each region's own
  jurisdiction mix rather than assuming a shared statewide rule.

## Reusing this for a fourth state

See the Idaho project's own README ("Reusing this for another state") for
the general steps. This Montana build is itself a worked example of the
*opposite* geography problem from Utah: instead of retuning to keep close
markets from over-merging, Montana required loosening both `eps` and
`merge` just to let real, contiguous markets form clusters at all across a
sparse, large state — re-verify every tuning constant and every hand-written
narrative claim against the new state's own data rather than assuming any
prior state's numbers transfer.
