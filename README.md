# Tokyo Refill Infrastructure Decision Tool

A map-based decision-support web application that identifies refill deserts across Tokyo's 23 Special Wards, ranks candidate locations for new refill points, and simulates coverage improvements under different expansion scenarios.

Built as a focused v1 around one planning question: **where should new refill points be added in Tokyo to reduce single-use bottle dependence most effectively?**

**Live demo:** `https://<your-github-username>.github.io/<repo-name>/` (see deployment section below)

---

## Quick Start (Local)

Because the app loads JSON data via `fetch()`, browsers block it from `file://` URLs for security (CORS). You need a tiny local HTTP server. Pick one:

### Option A — Python (no install if you have Python 3)

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>
python3 -m http.server 8000
```

Then open `http://localhost:8000` in your browser.

### Option B — Node.js (if you prefer)

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>
npx serve .
```

Then open the URL printed in the terminal (usually `http://localhost:3000`).

### Option C — VS Code Live Server

Install the "Live Server" extension, open the project folder, right-click `index.html` → "Open with Live Server."

If you open `index.html` directly by double-clicking it, the app will detect the CORS error and show an inline error message explaining how to fix it.

---

## Deploy to GitHub Pages

The repo ships with a ready-to-go GitHub Actions workflow (`.github/workflows/deploy.yml`) that auto-publishes every push to `main`.

**One-time setup on GitHub:**

1. Push the repo to GitHub (see _Pushing to GitHub_ below if starting fresh).
2. Go to the repo on GitHub → **Settings** → **Pages**.
3. Under **Build and deployment** → **Source**, pick **"GitHub Actions"**.
4. Push any commit to `main` (or go to **Actions** → **Deploy to GitHub Pages** → **Run workflow** manually).
5. Within a minute, the live URL appears at the top of the Pages settings page.

Your URL will be `https://<username>.github.io/<repo-name>/`. Share that with judges.

### Alternative: "Deploy from branch" (no Actions)

If you prefer the older Pages flow:

1. **Settings** → **Pages** → **Source**: "Deploy from a branch"
2. **Branch**: `main`, **Folder**: `/ (root)`
3. Save. Wait ~1 minute. Your URL appears.

The `.nojekyll` file in the repo root prevents GitHub from running Jekyll on the content, which would otherwise ignore any files or folders starting with underscores.

---

## Pushing to GitHub (if starting from a fresh clone)

From this folder:

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main

# Create a new empty repo on GitHub first (no README, no .gitignore, no license —
# we already have those). Then connect it:
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```

After the first push, GitHub Actions will kick off automatically and deploy to Pages.

---

## What the App Does

When you open it, the app runs four staged computations:

1. **Loads data** from `./data/*.json` (4 files, ~30 KB total).
2. **Builds a hex grid** over the Tokyo 23-ward bounding box (~1.1 km cells, filtered to populated areas).
3. **Computes per-hex metrics**: nearest-refill distance, gap score, demand score, combined priority.
4. **Scores every candidate site** on the same dimensions.
5. **Runs a baseline scenario**: greedy selection of the top 10 candidates, before/after coverage computed against all 2.8k+ hexes.

Then the interface is live:

- **Header metric cards** show live totals (refill points mapped, deserts detected, underserved wards, top priority ward).
- **Left panel** controls layers, heatmap view mode (gap / demand / combined), scoring weight sliders, and candidate-type filter.
- **Center map** shows everything spatially: refill points, heatmap, candidates, top-ranked recommendations, ward labels.
- **Right panel** ranks the top 15 recommended sites and the top 10 underserved wards. Clicking a recommendation opens a detail panel with plain-language rationale.
- **Footer scenario panel** lets you adjust N (0–30 new points) and the scoring weights, then re-run. You get before/after coverage @ 500 m, mean distance, deserts resolved, high-demand areas newly served, and a dual-line coverage curve.

---

## Architecture

Static web app. No backend. No build step. No framework.

```
Browser
  ├── Leaflet 1.9.4           (map rendering, layer management)
  ├── Turf.js 6.5.0            (hex grid generation, geodesic distance)
  ├── Chart.js 4.4.1           (before/after coverage curves)
  └── Custom JS (~1000 lines)  (scoring, ranking, simulation, UI)

Data (loaded via fetch on page load):
  ├── data/refill_points.json       (112 existing mymizu-style points)
  ├── data/candidate_sites.json     (64 potential new locations)
  ├── data/tokyo_wards.json         (23 wards + real stats)
  └── data/demand_anchors.json      (48 transit/commercial hubs)

Basemap: CartoDB Voyager (no labels) via tile CDN.
```

All three JS libraries load from public CDNs (unpkg, jsdelivr). No npm install required. First page load needs internet; hex grid then computes locally and stays cached in memory.

### Why split the data?

The data is in `./data/` as separate JSON files so you can edit any of them (add more refill points, adjust demand anchor weights, swap in a different candidate pool) without touching the HTML. Reload the page and the app picks up the new data.

In particular, to plug in the **live mymizu feed**, you'd replace `data/refill_points.json` with a fetched response from the mymizu API, matching the schema:

```json
{
  "meta": { "source": "...", "count": N },
  "points": [
    { "id": "...", "name": "...", "type": "...", "ward": "...", "lat": 35.6, "lng": 139.7 }
  ]
}
```

---

## Data Sources & Attribution

The primary dataset `data/refill_points.json` is a **curated demo set** of 112 refill points modeled on the real mymizu distribution across Tokyo. In production, this should be replaced with the live mymizu feed:

- **mymizu Refill Map** — github.com/mymizu/mymizu-web — the open dataset behind the mymizu app. Free API key available on request. 200,000+ refill spots worldwide with detailed Japan coverage.
- **mymizu / Social Innovation Japan** — mymizu.co/home-en — the NPO behind mymizu. Attribution required for any use of their dataset.

Supporting datasets used to build the candidate pool, demand layer, and ward overlay:

- **OpenStreetMap** — openstreetmap.org — stations, public buildings, libraries, parks, universities, commercial anchors. The 48 demand anchors in `data/demand_anchors.json` are real station/landmark coordinates selected for activity weight.
- **PLATEAU Project (MLIT)** — mlit.go.jp/plateau/en — Japan's official 3D city model. In production, building footprints and floor-area attributes from the 23-ward CityGML dataset would replace the coarse anchor-based demand proxy.
- **PLATEAU Data Portal (G-Spatial)** — geospatial.jp — direct download of Tokyo building geometry.
- **Tokyo Metropolitan Government Open Data** — portal.data.metro.tokyo.lg.jp — ward boundaries, population, daytime population. `data/tokyo_wards.json` uses 2024 TMG estimates.
- **Ministry of the Environment Japan** — env.go.jp — national plastic waste and municipal solid waste survey data. Used for framing, not for calculation in this v1.

The 64 candidate sites in `data/candidate_sites.json` are real Tokyo locations (stations, ward offices, libraries, parks, universities) selected as plausible refill hosts, weighted toward underserved outer wards.

---

## Scoring Logic

Every hex cell and every candidate site gets three scores on a 0–100 scale.

### Gap score

Measures how poorly a location is currently served by the existing refill network.

```
nearest_m   = geodesic distance (Turf) to the closest existing refill point
gap_score   = min(100, nearest_m / 12)      for candidates
            = min(100, nearest_m / 15)      for hex cells
```

A candidate 1.2 km from any refill point scores 100 (maximum gap). A candidate 240 m away scores 20.

### Demand score

Measures how much latent activity is present — proxying for "refill importance." Each of the 48 demand anchors contributes exponentially decaying weight:

```
demand_raw = Σ  anchor.weight × exp( -distance_km(hex, anchor) / 0.9 )
              over all anchors
demand_score = 100 × (demand_raw / max_demand_across_all_hexes)
```

The 0.9 km decay scale roughly matches walkable refill service radius. A location right next to Shinjuku Station (weight 100) with no competing anchors gets a score near 100; a location 2 km from the nearest mid-sized station (weight 40) gets something in the 15–25 range.

### Combined / priority score

A weighted sum the user controls via three sliders on the left panel:

```
combined = w_gap × gap  +  w_demand × demand  +  w_spacing × spacing
                          (sliders default: 0.50, 0.30, 0.20)
```

For hex cells, spacing is ignored. For candidates, spacing is computed dynamically during greedy selection (below).

### Desert threshold

A hex is flagged `isDesert = true` when its nearest-refill distance exceeds **800 m** — roughly 10 minutes' walk. This is the cutoff the underserved-ward ranking uses.

### Coverage threshold

A hex is "covered" when its nearest-refill distance is ≤ **500 m**. The scenario simulator reports before/after coverage at this radius.

---

## Ranking Method (Candidate Selection)

Naive top-N-by-score fails because the highest-scoring candidates cluster in the same underserved pockets — picking all 10 would put them on the same block.

Instead, ranking uses **greedy selection with a rolling spacing score**:

```
deployed = [ ... all existing refill points ... ]
ranked   = []

repeat N times:
  for each remaining candidate c:
    c.spacing = min(100, nearest_distance(c, deployed) / 15)     # meters / 15
    c.final   = w_gap × c.gap + w_demand × c.demand + w_spacing × c.spacing
  pick the candidate with highest c.final
  ranked.append(picked)
  deployed.append(picked)        # subsequent candidates now see it
```

Once a candidate is selected, every remaining candidate's spacing score is recomputed against it. The second pick gets its spacing penalty applied relative to the first pick; the third relative to both; and so on. This spreads recommendations across the city without requiring hard distance constraints.

---

## Simulation Method

"Run Scenario" executes:

```
1. Take the top-N ranked candidates (using current weights + filter).
2. Build deployed set = existing refill points ∪ topN candidates.
3. For every hex in the grid:
      new_nearest_m = distance to closest point in deployed set
4. Aggregate:
      covered_before = % hexes with nearest_m ≤ 500 before
      covered_after  = % hexes with nearest_m ≤ 500 after
      mean_before    = mean nearest_m before
      mean_after     = mean nearest_m after
      deserts_resolved  = count of hexes that were desert (>800m) and are now not
      hi_demand_served  = count of hexes with demand > 60 that moved from uncovered to covered
5. Render before/after cards + Chart.js dual-line coverage curve across distance thresholds 200–1500 m.
```

The chart lets the user see not just coverage at 500 m but the full curve — so a scenario that improves 500 m coverage modestly but dramatically reduces deep-desert hexes becomes visible.

---

## File Structure

```
tokyo-refill-tool/
├── index.html                      # The application
├── README.md                       # This file
├── LICENSE                         # MIT
├── .gitignore
├── .nojekyll                       # Tells GitHub Pages to skip Jekyll
├── .github/
│   └── workflows/
│       └── deploy.yml              # Auto-deploy to GitHub Pages on push to main
└── data/
    ├── refill_points.json          # 112 existing refill points
    ├── candidate_sites.json        # 64 candidate new locations
    ├── tokyo_wards.json            # 23 wards + real stats
    └── demand_anchors.json         # 48 demand-proxy anchors
```

---

## Demo Flow (for presentations)

A 3-minute walkthrough:

1. **Open the app.** Point out the 112 refill points clustered heavily in Shibuya/Shinjuku/Minato. The header shows live metrics including top priority ward.
2. **Switch the heatmap to "Gap."** The burnt-orange cells in the outer wards show the refill deserts. Switch to "Demand." The glow over central Tokyo shows where people actually are. Switch to "Combined." The tool now highlights where high demand overlaps with low access — the real intervention priorities.
3. **Show the top recommendations.** Click the #1 site in the right panel. The detail pane explains why: *"Sits inside a refill desert with very high local activity. Nearest existing refill is 920 m away."*
4. **Adjust the weights.** Slide Gap down, Demand up. The rankings reshuffle — recommendations lean toward high-traffic central locations even when existing refill is nearby. Slide Demand down, Gap up. Rankings shift to outer wards.
5. **Run a scenario.** Set N = 15. Click Run Simulation. Coverage jumps. The dual-line chart shows improvement across all walking distances. Point out "deserts resolved" and "high-demand areas newly served" as the two plain-language outcomes for stakeholders.

---

## Extending for Production

The v1 is deliberately narrow. To harden it for real municipal use:

**Data pipeline.** Subscribe to the mymizu API for live refill coverage — just swap `data/refill_points.json` with a fetched response. Extract building polygons from PLATEAU CityGML and aggregate floor area per hex to replace the anchor-based demand proxy. Pull ward boundary polygons from Tokyo Metropolitan Government Open Data and replace centroid dots with proper GeoJSON polygons.

**Better demand proxy.** Instead of 48 hand-weighted anchors, use: (a) OSM amenity count per hex, (b) JR/Metro station ridership (MLIT open data), (c) daytime population density from TMG, (d) building floor area from PLATEAU. Normalize each, combine with configurable weights.

**Real cost constraints.** Add a budget parameter. Each candidate site has an install cost (park = low, private commercial = high). Replace greedy-by-score with a budget-constrained knapsack over total impact.

**Equity layer.** Add a ward-level socioeconomic index. Surface scenarios that explicitly improve coverage in lower-income wards.

**Persistence.** Let users save scenarios to a URL hash so they can share specific configurations with colleagues.

---

## Tech Choices (rationale)

| Choice | Reason |
|---|---|
| Pure static site | Hackathon portability. No server, no build step. GitHub Pages ready. |
| Leaflet over Mapbox/Google | No API key. MIT-licensed. Battle-tested for choropleth and heatmap overlays. |
| Turf.js over a backend geo service | Runs in browser. Handles hex grid, centroid, geodesic distance out of the box. |
| Hex grid over wards | Wards are huge and uneven (Chiyoda 11 km², Ota 62 km²). Hexes give a uniform 1.1 km resolution that planners expect. |
| Greedy selection over ILP | 64 candidates × 30 picks is fast in greedy. Greedy is explainable to non-technical stakeholders. ILP would be a production upgrade. |
| CartoDB Voyager no-labels basemap | Neutral, muted palette that lets the data layers sit on top without visual conflict. |

---

## Troubleshooting

**The page opens but shows "Could not load data files".**
You opened `index.html` directly with `file://`. Browsers block cross-file reads for security. Use one of the local server options in _Quick Start_ above, or deploy to GitHub Pages.

**GitHub Pages URL shows 404 or just a readme.**
Check **Settings → Pages** — Source must be "GitHub Actions" (or "Deploy from branch: main / root"). Give the Actions workflow 1–2 minutes to finish.

**Heatmap doesn't update when I move sliders.**
Hard refresh the page (Cmd+Shift+R on Mac, Ctrl+Shift+R on Windows). The initial scenario computation runs once on load; sliders re-render immediately after.

---

## License

MIT. See `LICENSE`. Data attributions: mymizu / Social Innovation Japan, OpenStreetMap contributors, CARTO, MLIT PLATEAU, Tokyo Metropolitan Government.
