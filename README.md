# Produced Water Opportunity Mapping

A geospatial ML pipeline that identifies high-value produced water
reuse deployment zones across Texas, New Mexico, California, and Alberta.

## What this does
Integrates 8+ public regulatory datasets (TX, NM, CA,
AB) into a reproducible spatial database, then
applies unsupervised clustering to surface statistically validated
county-level opportunity zones for produced water reuse.

## Key outputs
- Basin-scale supply-demand heatmap (interactive HTML)
- Disposal stress index per county
- Ranked opportunity zones with composite score

## Live map
**[View interactive opportunity map →](https://atosiroy.github.io/produced-water-opportunity-mapping)**

9 toggleable layers — PW supply · TDS salinity · Water demand 2025 · SWD wells


## Tech stack
Python · geopandas · scikit-learn · folium · pandas · contextily

## How to run
```bash
git clone git@github.com:atosiroy/produced-water-opportunity-mapping.git
cd produced-water-opportunity-mapping
pip install -r requirements.txt
jupyter notebook notebooks/01_data_pipeline.ipynb
```
## Data sources
See [docs/data_dictionary.md](docs/data_dictionary.md) for full list
of sources, access dates, and assumptions.

## Licence
MIT
