**Repository:** `oleava-technologies/produced-water-opportunity-mapping`
**Branch:** `main`

## How this repo is organized

The pipeline runs in six sequential stages. Each notebook takes files from disk, does one job, and writes files to disk. You can re-run any notebook independently as long as its inputs exist.

| # | Notebook | Job | Runtime |
|---|----------|-----|---------|
| 01 | Data ingestion | Fetch raw data from public APIs and files | ~10 min |
| 02 | Data cleaning | Harmonize, merge, engineer features | ~2 min |
| 03 | Supply analysis | K-means clustering, TDS characterization, SWD summary | ~1 min |
| 04 | Map construction | Build the deployed Folium map | ~3 min |
| 05 | Demand and drought | Integrate OSE and USDM layers | ~30 sec |
| 06 | Consistency check | County-level statistical validation | ~30 sec |

End-to-end runtime: ~17 minutes on standard Colab.

**Exploratory notebooks** live in Oleava personal storage `notebooks/exploratory/` work done during development that isn't part of the reproducible pipeline. See the exploratory section at the end of this document.

---

## 01_data_ingestion.ipynb

**Status:** Consolidated from four separate source notebooks (`02_data_pipeline`, `03_data_pipeline`, `OCD-data-extraction`, `NM_shape_extraction`). Two additional sections as well in demand side: OSE demand ingestion and USDM drought ingestion.

**Purpose:** Fetch all raw data from public sources: five ingestion sections, one per data source. This is the only notebook that talks to external APIs or extracts from PDFs. Everything downstream reads from `data/raw/`.

**Inputs:**
- None on disk. Reads from external APIs and public files.

**Outputs (to `data/raw/`):**
- `wells_data.csv` — WaterSTAR wells with quality/quantity attributes, one row per township
- `all_quarter_townships_quantity_data.csv` — monthly quantity time-series across 267 townships
- `nm_ocd_wells_raw.geojson` — all ~140K NM OCD wells (categorization happens in 02)
- `nm_counties.geojson` — 33 NM county polygons filtered from Census TIGER
- `nm_ose_demand_raw.pdf` or extracted CSV
- `nm_usdm_drought_2020_2025.csv` downloaded from source 

**Key steps in order:**
1. **Section A — WaterSTAR wells.** POST to `nmpw.waterstar.org/api/api/well`, paginate through pages 1–6, collect into a DataFrame, save as `wells_data.csv`
2. **Section B — WaterSTAR quantity time-series.** Read the 267 QTSIDs from `wells_data.csv`, loop through each ID calling the quantity API, concatenate results, save as `all_quarter_townships_quantity_data.csv`
3. **Section C — NM OCD wells (documented live API fetch).** Define `fetch_ocd` function, paginate through ~140K records from the ArcGIS FeatureServer, parse to GeoDataFrame with `EPSG:4326`, save as `nm_ocd_wells_raw.geojson`
4. **Section D — NM county shapefile.** Load `tl_2022_us_county.shp` from Census TIGER, filter to `STATEFP == '35'` (New Mexico), create `county_key` column (lowercase, stripped), save as `nm_counties.geojson`
5. **Section E — NM OSE demand.** Extract water demand tables from OSE PDF reports 
6. **Section F — USDM drought.** Fetch or parse the U.S. Drought Monitor CSV for NM counties, 2020–2025

**External calls:**
- WaterSTAR API: `https://nmpw.waterstar.org/api/api/well` (POST, paginated)
- NM OCD ArcGIS FeatureServer (paginated GET)
- Census TIGER shapefile download
- OSE PDF, USDM CSV endpoints

**Known issues / notes:**
- OCD API pagination takes about 8 minutes for the full 138,355 records; this is the longest single step in the entire pipeline
- The `wells_data.csv` produced here includes raw column names from the API; standardization happens in `02_data_cleaning.ipynb`


**Downstream consumers:** `02_data_cleaning.ipynb` (all sections)

**Provenance of the consolidated content:**
- Section A ← `02_data_pipeline.ipynb` (API fetch portion only, before cleaning)
- Section B ← `03_data_pipeline.ipynb` (entirety)
- Section C ← `OCD-data-extraction.ipynb` (fetch portion, before oil/SWD split)
- Section D ← `NM_shape_extraction.ipynb` (entirety)

---

## 02_data_cleaning.ipynb

**Status:** Consolidated from three source notebooks (`02_data_pipeline` cleaning portion, `TS_quantity` as canonical merge, `OCD_data_analysis` for categorization). Supersedes `merged_data.ipynb`, which is preserved under `exploratory/` for provenance.

**Purpose:** Take raw ingested data and produce clean, harmonized, feature-engineered files ready for analysis. All the "make the data usable" work happens here: column standardization, type coercion, feature engineering, merging.

**Inputs (from `data/raw/`):**
- `wells_data.csv` — WaterSTAR wells (from 01)
- `all_quarter_townships_quantity_data.csv` — quantity time-series (from 01)
- `nm_ocd_wells_raw.geojson` — all NM OCD wells (from 01)

**Outputs (to `data/processed/`):**
- `nm_clean.csv` — cleaned WaterSTAR wells, intermediate file
- `df_final.csv` — canonical merged dataset with all features, feeds clustering
- `nm_ocd_disposal_wells.geojson` — filtered to SWD wells
- `nm_ocd_oil_wells.geojson` — filtered to oil wells

**Key steps in order:**

**Section A — WaterSTAR wells cleaning:**
1. Load `wells_data.csv`
2. Apply `clean_columns` helper (lowercase, strip whitespace, replace spaces with underscores)
3. Rename columns using explicit dictionary to `geo_id`, `qty_5yr`, `qty_1yr`, `qty_365d`, `tds`, `qual_sample_count`, etc.
4. Apply `pd.to_numeric(errors='coerce')` to all numeric columns
5. Drop rows with NaN in `qty_5yr`, `qty_1yr`, `tds`
6. Save `nm_clean.csv`

**Section B — Time-series feature engineering and merge:**
7. Load `all_quarter_townships_quantity_data.csv`
8. Rename `'quarter township' → geo_id`, `'value' → volume`, convert `date` to datetime
9. Aggregate volume by `geo_id` and year → `df_yearly`
10. Compute `ts_trend` (linear slope of yearly volume) per township
11. Compute `ts_volatility` (std of yearly volume) per township
12. Compute `recent_growth` (last year / previous year) per township
13. Load `nm_clean.csv`, select core columns (`geo_id`, `qty_5yr`, `qty_1yr`, `qty_365d`, `tds`, `qual_sample_count`) → `df_core`
14. Apply `np.log1p` to `tds` and `qty_5yr` → `log_tds`, `log_qty_5yr`
15. Merge `df_core` with `trend_df`, `vol_df`, `growth_df` → `df_final`
16. Normalize `ts_trend_norm = ts_trend / qty_5yr`, `ts_volatility_norm = ts_volatility / qty_5yr`
17. Handle infinities in `recent_growth` (replace with NaN, then fill NaN with 0)
18. Save `df_final.csv`

**Section C — OCD wells categorization:**
19. Load `nm_ocd_wells_raw.geojson`
20. Filter to `type == 'Salt Water Disposal'` and count → SWD wells
21. Create `disposal_type` column on the SWD subset
22. Separately filter to oil wells based on `type` attribute
23. Save `nm_ocd_disposal_wells.geojson` (1,788 wells) and `nm_ocd_oil_wells.geojson` (81,371 wells)

**External calls:** None — reads only from `data/raw/`.

**Known issues / notes:**
- Log transformation uses `np.log1p` (natural log of 1+x); verify thesis §5.3.4 text is consistent with this choice
- The `clean_columns` function was defined in the source notebook but not always used consistently — the final rename via dictionary is what actually produced the canonical names
- Historical: an early version (`02_data_pipeline.ipynb`) computed abandoned features `supply_intensity` and `opportunity_score`; these are not preserved in the consolidated notebook
- Historical: an earlier merge notebook (`merged_data.ipynb`) did the same job as Section B; it was superseded by `TS_quantity.ipynb` and is preserved under `exploratory/`
- The Folium map from `OCD_data_analysis.ipynb` was moved to exploratory: map building belongs in `06_map_construction.ipynb`

**Downstream consumers:**
- `df_final.csv` → `03_supply_analysis.ipynb` (clustering, TDS characterization)
- `nm_ocd_disposal_wells.geojson` → `03_supply_analysis.ipynb` (SWD summary), `06_map_construction.ipynb`
- `nm_ocd_oil_wells.geojson` → `04_consistency_check.ipynb`, `06_map_construction.ipynb`
- `nm_clean.csv` → intermediate, not consumed downstream after the merge

**Provenance of the consolidated content:**
- Section A ← `02_data_pipeline.ipynb` (cleaning portion, after API fetch)
- Section B ← `TS_quantity.ipynb` (entirety; supersedes `merged_data.ipynb`)
- Section C ← `OCD_data_analysis.ipynb` (categorization and export portions)

---

## 03_supply_analysis.ipynb

**Status:** Can be found in three exploratory notebooks (`clusteringv2.ipynb`, `stats.ipynb` for TDS characterization, `OCD_data_analysis.ipynb` for SWD summary). This is the canonical clustering notebook — K-means is fit here exactly once. Anything downstream reads cluster labels from disk.

**Purpose:** Township-level supply-side analysis. Produces the K-means clustering, characterizes TDS distribution across townships, and summarizes SWD infrastructure by county. Outputs feed the consistency check, the demand/drought integration, and the map.

**Inputs (from `data/processed/`):**
- `df_final.csv` — merged township-level data (from 02)
- `nm_ocd_disposal_wells.geojson` — categorized SWD wells (from 02)
- `nm_counties.geojson` — county polygons (from 01, via 02 if used)

**Outputs:**
- `data/processed/nm_pw_clusters.csv` — **canonical cluster labels**, one row per township
- `data/processed/nm_swd_per_county.geojson` — county polygons enriched with SWD counts
- `data/processed/nm_swd_summary.csv` — SWD counts by county and disposal type
- `data/outputs/internal_cluster_metrics.csv` — silhouette, Davies-Bouldin, Calinski-Harabasz
- `data/outputs/silhouette_by_cluster.csv` — per-cluster silhouette values
- `data/outputs/tds_statistics.csv` — descriptive statistics for TDS
- `figures/cluster_profile_heatmap.png` — 4-cluster feature-value heatmap
- `figures/tds_distribution_histogram.png` — treatment-tech-colored TDS histogram

**Key steps in order:**

**Section A — K-means clustering:**
1. Load `df_final.csv`
2. Select feature columns: `[log_qty_5yr, log_tds, ts_trend_norm, ts_volatility_norm]`
3. Replace infinities with NaN, drop rows with NaN in features
4. Feature scaling
5. Fit K-means with `n_clusters=4, random_state=42`
6. Assign cluster labels back to the DataFrame
7. Save `nm_pw_clusters.csv` — **the canonical clustering, frozen from here on**
8. Generate cluster profile heatmap with thesis-consistent labels (Cluster 3 = Permian core, Cluster 1 = moderate Permian, Cluster 0 = gas basin, Cluster 2 = outliers)
9. Back-transform `log_qty_5yr` and `log_tds` to real units (`qty_5yr_real`, `tds_real`)
10. Compute and display real-number cluster means table
11. Compute cluster counts and percentages

**Section B — Internal cluster metrics:**
12. Compute aggregate silhouette score (expected ~0.396)
13. Compute Davies-Bouldin index (expected ~0.796)
14. Compute Calinski-Harabasz index (expected ~178.4)
15. Compute per-cluster silhouette table (mean, min, max per cluster)
16. Count negative silhouettes (expected 3/248 = 1.2%)
17. Save `internal_cluster_metrics.csv` and `silhouette_by_cluster.csv`

**Section C — TDS distribution characterization:**
18. Extract `tds_real` from township-level data
19. Compute descriptive statistics (min, 25th, median, mean, 75th, 90th, max)
20. Generate histogram color-coded by treatment technology bands (Brackish RO <10k, Seawater RO 10–35k, High-P RO 35–100k, Thermal >100k mg/L)
21. Add median and mean vertical reference lines
22. Save `tds_distribution_histogram.png` and `tds_statistics.csv`

**Section D — SWD county summary:**
23. Load `nm_ocd_disposal_wells.geojson`
24. Group and count disposal wells by `county` and `disposal_type`
25. Create pivot table: Injection Well vs SWD Well counts per county
26. Compute total disposal wells per county, sort descending
27. Spatial-join SWD counts back to county polygons → `nm_swd_per_county.geojson`
28. Save `nm_swd_summary.csv`

**External calls:** None — reads only from `data/processed/`.

**Known issues / notes:**
- Scaler choice: source notebook used StandardScaler; thesis §5.3.4 documents MinMaxScaler. Verify which matches the metrics reported (silhouette 0.396, DB 0.796, CH 178.4) and align both code and text
- Cluster label ordering: this notebook produces the canonical labeling used throughout the thesis and defense deck. Any downstream visualization must use the same labels
- PCA scatter plot and pair plot from `clusteringv2.ipynb` were moved to `exploratory/` — they don't add interpretive value beyond the heatmap
- The K-means fit at the end of the earlier "Notebook 3" (county aggregation) was a duplicate of this one and was cut in consolidation

**Downstream consumers:**
- `nm_pw_clusters.csv` → `04_consistency_check.ipynb`, `05_demand_and_drought.ipynb`, `06_map_construction.ipynb`
- `nm_swd_per_county.geojson`, `nm_swd_summary.csv` → `04_consistency_check.ipynb`, `06_map_construction.ipynb`
- `internal_cluster_metrics.csv`, `silhouette_by_cluster.csv` → thesis Annex C.1 (documentation)
- `tds_statistics.csv` → thesis §6.3 (documentation)

**Provenance of the consolidated content:**
- Section A ← `clusteringv2.ipynb` (entirety of K-means fit and heatmap portions)
- Section B ← new work added during thesis writing (metrics not in original notebooks)
- Section C ← `stats.ipynb` (TDS histogram and descriptive statistics portions)
- Section D ← `OCD_data_analysis.ipynb` (SWD summary and per-county export portions)

---

## 04_consistency_check.ipynb

**Status:** Consolidated from the county-aggregation and consistency-check portions of `stats.ipynb` (and/or the earlier "Notebook 3" — likely the same notebook under different names). Any duplicate K-means fit from those sources is cut here since clustering is done exactly once in `03_supply_analysis.ipynb`.

**Purpose:** The convergent consistency check at county level. Aggregates township clusters to counties, tests correlation against two independent regulatory datasets (oil well counts, SWD counts), and reports results with honest statistical framing.

**Inputs (from `data/processed/`):**
- `nm_pw_clusters.csv` — canonical cluster labels (from 03)
- `nm_swd_summary.csv` — county-level SWD counts (from 03)
- `nm_ocd_oil_wells.geojson` — categorized oil wells (from 02)

**Outputs:**
- `data/outputs/nm_cross_validation.csv` — county-level aggregated table with PW volume, SWD count, oil well count, TDS stats
- `data/outputs/consistency_check_stats.csv` — Spearman r, exact permutation p, Pearson r
- `data/outputs/leave_one_out_sensitivity.csv` — LOO sensitivity results
- `figures/consistency_check_scatter.png` — two-panel scatter with log axes

**Key steps in order:**
1. Load `nm_pw_clusters.csv` and aggregate to county level
2. Merge in SWD counts (from `nm_swd_summary.csv`) and oil well counts (spatial join from `nm_ocd_oil_wells.geojson` to counties)
3. Compute county-level totals: total PW volume, SWD count, oil well count
4. Compute per-county TDS statistics (median, max)
5. Save the aggregated table as `nm_cross_validation.csv` (n = 5 counties: Lea, Eddy, San Juan, Rio Arriba, McKinley)
6. Compute Spearman rank correlation: PW volume vs oil well count, PW volume vs SWD count
7. Compute exact permutation p-value using `scipy.stats.PermutationMethod(n_resamples=10000, random_state=42)` — the fix for scipy's default t-approximation which is unreliable at n=5
8. Compute Pearson correlation as secondary measure
9. Leave-one-out sensitivity: recompute r and p under each single-county exclusion
10. Generate two-panel scatter plot with log axes; distinguish gas-basin counties (San Juan, Rio Arriba) as diamonds and Permian counties (Eddy, Lea, McKinley) as circles
11. Save statistics and figures

**External calls:** None.

**Known issues / notes:**
- The framing here is critical and reflects a correction: this is a **diagnostic consistency check**, not external validation. Both external variables (oil wells, SWD counts) are coupled to PW volume, so the test cannot claim independence. The p-value of 0.083 (two-sided exact permutation) does not clear 0.05
- Original thesis language used "cross-validation" — replaced with "consistency check" 
- Figure caption must state honestly: right panel is not an independent replicate of the left panel; both use the same rank ordering of the same 5 counties
- Filename `nm_cross_validation.csv` is retained for backward compatibility with existing references, even though the "cross-validation" framing has been replaced

**Documentation:**
- Thesis §6.2 (consistency check), Annex C.2 (leave-one-out table)

---

## 05_demand_and_drought.ipynb

**Purpose (planned):** Integrate the NM OSE water demand data and USDM drought severity into the county-level summary. Produces the county-level table that feeds the map.

**Inputs (from `data/processed/` and `data/raw/`):**
- OSE demand data (from 01 Section E)
- USDM drought data (from 01 Section F)
- `nm_pw_clusters.csv` — cluster labels aggregated to county (from 03)
- `nm_counties.geojson` — county polygons (from 01)

**Outputs (planned):**
- `data/processed/nm_water_demand_2025_final.csv` — 2025 demand projections per county
- `data/processed/nm_drought_2020_2025.csv` — drought severity per county
- `data/processed/nm_county_summary_full.csv` — combined table with supply, demand, drought, cluster
- `figures/nm_drought_choropleth.png` — fixed 0–100% drought choropleth
- `figures/supply_demand_drought_scatter.png` — deployment opportunity matrix

**Key steps in order:**
1. Load OSE demand data, project to 2025
2. Compute agricultural demand and power sector demand per county
3. Load USDM drought data, aggregate weeks in drought per county 2020–2025
4. Compute `pct_weeks_drought` per county
5. Merge supply (from 03), consistency check (from 04), demand, drought into single county-level table
6. Generate drought choropleth with fixed 0–100% branca LinearColormap
7. Generate supply-demand-drought scatter plot
8. Save combined summary

**External calls:** None.

**Downstream consumers:**
- `nm_county_summary_full.csv` → `04_full_map.ipynb`

---

## 04_full_map.ipynb

**Purpose:** Build the interactive ten-layer Folium map, deployed to GitHub Pages.

**Inputs:**
- `data/processed/nm_county_summary_full.csv` — combined county-level table (from 05)
- `data/processed/nm_counties.geojson` — county polygons
- `data/processed/nm_ocd_disposal_wells.geojson` — SWD wells
- `data/processed/nm_ocd_oil_wells.geojson` — oil wells

**Outputs:**
- `main/index.html` — the deployed Folium map # docs/index to create github page

**Key steps:**
1. Load combined summary and geometries
2. Configure log-scale transforms for PW volume choropleth
3. Define drought colormap
4. Build ten layers: PW volume (choropleth log), SWD wells (points), oil wells (points), cluster labels, median TDS (choropleth), water demand 2025, agricultural demand, power sector demand, drought severity, county boundaries
5. Compose unified per-county tooltip integrating all metrics
6. Build custom HTML legend grouping layers into four conceptual categories (Supply, Demand, Drought, Infrastructure)
7. Save to `main/index.html`

**External calls:** None.

**Notes:**
- Deployed at oleava-technologies/produced-water-opportunity-mapping.github.io

**Downstream consumers:** End of pipeline. The map is the primary deliverable for Oleava stakeholders.

---

## Exploratory notebooks

Notebooks under `notebooks/exploratory/` were used during development but are NOT part of the reproducible pipeline. Their outputs are not consumed by any downstream notebook. They are preserved for provenance and reference.

### EXPLORATORY_pair_plot.ipynb
**Source:** `clusteringv2.ipynb` (pair plot section)
**Why exploratory:** The pair plot is dominated by outliers in three of the four dimensions (Cluster 2, the 4-township outlier bin, blows the axes wide open). In the ts_trend_norm and ts_volatility_norm columns, everything except Cluster 2 collapses to a single line at zero. Not defensible in defense.

### EXPLORATORY_pca_projection.ipynb
**Source:** `clusteringv2.ipynb` (PCA section)
**Why exploratory:** 2D PCA captures 82.6% of variance (PC1 57.4%, PC2 25.2%), which is defensible, but the projection shows a fan-shape rather than clean island clusters, and the two outlier points at PC1 ~15 stretch the axis. The cluster profile heatmap tells a cleaner story.

### EXPLORATORY_ocd_wells_overview_map.ipynb
**Source:** Folium visualization from `OCD-data-extraction.ipynb` and/or `OCD_data_analysis.ipynb`
**Why exploratory:** Ingestion notebooks shouldn't produce visualizations; the map notebook (06) handles all map building.

### EXPLORATORY_spatial_overview.ipynb
**Source:** `pw_shpfile.ipynb` (CRS reprojection and layer plots)
**Why exploratory:** CRS reprojection to EPSG:3857 for plotting is a concern of the map notebook, not a separate pipeline stage. 

### EXPLORATORY_time_series_distributions.ipynb
**Source:** `TS_quantity.ipynb` (histograms and scatter plots)
**Why exploratory:** Trend histogram, growth histogram, `log_qty_5yr` vs `log_tds` scatter — informed the feature engineering choices but are not part of the canonical pipeline.

### EXPLORATORY_SUPERSEDED_merged_data.ipynb
**Source:** `merged_data.ipynb`
**Why exploratory:** Earlier version of the quantity/quality merge. Superseded by `TS_quantity.ipynb` which does the same merge plus adds statistical exploration of features. 
---

## Naming conventions in this repo

- Sequential prefix (`01_`, `02_`, ...) — indicates the pipeline order
- Underscores, not hyphens or spaces
- Lowercase throughout
- Descriptive but concise
- `EXPLORATORY_` prefix for non-canonical notebooks
- `EXPLORATORY_SUPERSEDED_` for notebooks explicitly replaced by a newer canonical version

## File locations referenced

All paths are relative to the repo root:

- `data/raw/` — raw fetched data, treat as read-only after ingestion
- `data/processed/` — cleaned, harmonized, feature-engineered data
- `data/outputs/` — analytical results (statistics tables, sensitivity results)
- `figures/` — saved plots (PNG format)
- `map/` — the deployed Folium map HTML
- `notebooks/` — this folder
- `notebooks/exploratory/` — non-canonical exploration, stored in oleava private drive

## Where each figure was produced in the final report

| Figure | Source notebook | Section |
|---|---|---|
| Figure 3 — Convergent consistency check scatter | `04_consistency_check.ipynb` | 
| Figure 4 — TDS distribution histogram | `03_supply_analysis.ipynb` | 
| Figure 7 — NM drought choropleth | `05_demand_and_drought.ipynb` |
| Figure 8 — Interactive map screenshot | `06_map_construction.ipynb` |
| Figure 9 — Supply-demand-drought scatter | `05_demand_and_drought.ipynb` |
| Cluster profile heatmap | `03_supply_analysis.ipynb` | 


---