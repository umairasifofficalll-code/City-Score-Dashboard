# Where Should I Live? A Digital Nomad City Scorecard

An end-to-end data pipeline and interactive Power BI dashboard that ranks 63 cities worldwide for remote workers. Raw data goes in, a validated and scored dataset comes out, and viewers set their own priorities (cost, internet, safety, English level) so the ranking updates instantly.

![Dashboard](dashboard.png)

> **Data note:** the dataset is **simulated** for portfolio practice. Values are roughly realistic but are not real measurements, so the rankings are for demonstration only. The pipeline is built so real sources (for example Numbeo for costs and Speedtest for internet speed) can be dropped in with the same column names.

## Business question
Which city is best for a remote worker, and how does the answer change depending on what matters most to them?

## Pipeline overview
```
5 raw CSV files
      |
      v
1. INGEST      load cities, costs, quality, visas, monthly rent
      |
      v
2. CLEAN       per-table fixes + cleaning log (duplicates, names, missing values)
      |
      v
3. TRANSFORM   aggregate monthly rent to 1 row per city, join all tables, build features
      |
      v
4. VALIDATE    join validation + row-count and range checks (stops on failure)
      |
      v
5. MODEL       min-max scoring, weighted rank, K-Means city types
      |
      v
6. SERVE       city_scores.csv -> Power BI dashboard (refresh to update)
```
The whole flow is repeatable: rerun the two scripts, click **Refresh** in Power BI, and the dashboard updates. Every cleaning fix is recorded in `cleaning_log.csv`.

**Data-quality safeguards built into the pipeline**
- `validate="one_to_one"` / `"many_to_one"` on every join, so a join that would create extra rows raises an error
- Assertion that the master table has exactly 63 unique cities
- Range checks (scores must be 0-100) and a check that scoring weights add up to 1
- Blank visa values verified to appear only for countries with no remote-work visa

## What I built
1. **Cleaned 5 raw tables** with Python (pandas)
2. **Joined them** into one master table of 63 cities
3. **Scored every city** on a 0-100 scale with adjustable weights
4. **Grouped cities into 4 types** using K-Means clustering
5. **Built a Power BI dashboard** with weight sliders, a ranked chart, a world map, a cost vs. score scatter plot and a rent trend

## Data
| File | Rows | Content |
|---|---|---|
| cities.csv | 63 | city, country, region |
| costs.csv | 65 | rent, groceries, coworking, utilities, transport |
| quality.csv | 63 | internet speed, safety, air quality, English level, healthcare |
| visas.csv | 52 | country-level remote-work visa, minimum income, maximum stay |
| rent_monthly.csv | 2,268 | 36 months of rent per city |

## Data cleaning
| Table | Issue | Fix |
|---|---|---|
| cities | extra spaces and inconsistent capitalization (5 rows) | `strip()` and `title()` |
| cities | different names for one country, e.g. USA / US (2 rows) | mapped to one standard name |
| costs | exact duplicate rows (2) | `drop_duplicates()` |
| costs | missing coworking price (3) | filled with the region median |
| quality | missing internet speed (3) | filled with the region median |
| rent_monthly | 36 rows per city would multiply the join | reduced to one value per city (12-month rent growth) |

Blank visa income and stay values are intentional: they mean the country has no remote-work visa, so they were left empty rather than filled.

Joins use `validate=` so pandas stops with an error if a join creates extra rows, and the master table is asserted to have exactly 63 rows.

## Method
- **Scaling:** each metric is min-max scaled to 0-100 (cost is inverted so cheaper scores higher)
- **Weighted score:** `score = sum(weight x metric score)`; the dashboard recalculates it live with DAX when sliders move
- **Clustering:** K-Means (k = 4) on the four scaled scores, with each cluster named by its quality rank and average monthly cost
- **Rent growth:** 12-month change per city from the monthly rent series

## Dashboard features
- Weight sliders for cost, internet, safety and English
- Top 10 cities by weighted score
- World map of scores by country
- Monthly cost vs. score, coloured by city type
- Rent trend by month, with region and city filters

## Tech stack
Python (pandas, scikit-learn) | Power BI (DAX, parameters) | CSV

Scope note: this is a file-based pipeline run on demand. It does not use a database or an orchestrator such as Airflow.

## Repository structure
```
├── data/            raw CSV files
├── clean/           cleaned tables, city_master.csv, city_scores.csv
├── 01_cleaning.ipynb
├── 02_score_and_cluster.py
├── dashboard.pbix
├── dashboard.png
└── README.md
```

## How to run
```bash
pip install pandas scikit-learn
# 1. run 01_cleaning.ipynb  -> creates clean/city_master.csv
python 02_score_and_cluster.py   # -> creates clean/city_scores.csv
# 2. open dashboard.pbix in Power BI Desktop and click Refresh
```

## What I learned
- Clean each table before joining, since duplicates multiply and messy keys break joins
- A table with many rows per entity (monthly rent) must be aggregated before it joins a one-row-per-city table
- Blank values can be meaningful, so check why a value is missing before dropping or filling it

## Possible next steps
- Replace the simulated data with real sources
- Add local salary data to measure "months of work to afford rent"
- Add a visa filter ("show only cities I qualify for at my income")

**Author:** Umair Asif
