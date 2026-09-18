# Produced Water Opportunity Mapping

A geospatial ML pipeline that identifies high-value produced water
reuse deployment zones across New Mexico in the United States.

## What this does
Integrates 6 public regulatory datasets (N) into a reproducible spatial database, then
applies unsupervised clustering to surface statistically validated
county-level opportunity zones for produced water reuse.


## Live map
**[View interactive opportunity map →](https://oleava-technologies.github.io/produced-water-opportunity-mapping/)**

10 toggleable layers — PW supply · TDS salinity · Water demand 2025 · SWD wells


## Tech stack
Python · geopandas · scikit-learn · folium · pandas · contextily

## How to run
```bash
git clone git@github.com:oleava-technologie/produced-water-opportunity-mapping.git
cd produced-water-opportunity-mapping
pip install -r requirements.txt
jupyter notebook notebooks/01_data_ingestion.ipynb
```
## Data sources
See [docs/data_dictionary.md](docs/data_dictionary.md) for full list
of sources, access dates, and assumptions.

## Licence
MIT
