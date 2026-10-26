# Amsterdam Airbnb: Revenue and Pricing Intelligence

**Question:** How can hosts earn more and keep guests happy? Where is the money, what drives it, and which listings and hosts should be acted on first?

An end-to-end analytics project on Airbnb's public Amsterdam data (Inside Airbnb, June 2026 snapshot): 10,369 listings and 545,125 guest reviews. It covers data profiling and cleaning in Python, a cloud data pipeline (AWS S3 to Snowflake), business SQL, a hypothesis test, a machine-learning fair-price model, and a three-page Power BI dashboard.

> **Honest scope:** this is a public snapshot, not Airbnb's internal data. Revenue and occupancy are **modelled estimates** (estimated revenue = price x estimated booked nights). They describe what hosts gross, not Airbnb's own income. Findings are associations, not proven causes.

---

## Headline findings

| # | Finding | Evidence |
|---|---|---|
| 1 | About 4 in 10 listings are not bookable | 3,992 of 10,369 listings have no price (38.5%). 90% of those (3,591) have a fully blocked calendar for the next 365 days |
| 2 | Superhosts are about 17% of listings but earn about half of estimated revenue | 1,714 Superhost listings hold 58.94M of 115.08M (51.2%). Median revenue 34,412 vs 6,394 for regular hosts, at a lower median price (228 vs 314) |
| 3 | The Superhost advantage is large and holds inside both room types | Mann-Whitney effect size 0.82. Entire homes 35,402 vs 6,205. Private rooms 33,915 vs 9,380. Ratings are almost identical (4.86 vs 4.85) |
| 4 | Regular hosts leave far more supply idle | 44% of regular-host listings are unpriced vs 14% for Superhosts |
| 5 | One neighbourhood dominates revenue, through bookings rather than price | Centrum-West median revenue 27,115 at median price 298, vs Centrum-Oost 14,649 at 288 |
| 6 | Demand is a 7-month plateau | 2023 to 2025: April to October hold about 68% of reviews. January and February are about 42% below the April to October average |
| 7 | Two independent methods agree on a core group of underpriced busy listings | SQL peer rule 588 of 6,313 (9.3%); ML rule 278 of 5,401 (5.1%); **230 flagged by both**; median gap among ML candidates -34.2% |

## Recommendations

1. **Price-test the 230 listings flagged by both methods**, starting with high-rated Superhost listings (regular-host candidates have lower ratings, 4.59 vs 4.86, so their low prices may be justified).
2. **Help regular hosts reach Superhost-level reliability** through a pilot (association, not proven cause).
3. **Reactivation outreach for blocked listings** (about 35% of all listings are blocked for the full year).
4. **Seasonal pricing:** hold or raise prices April to October; offer longer-stay deals December to February.
5. **In weaker neighbourhoods, focus on bookings** (listing quality, reviews) rather than price cuts.

Every recommendation should be validated with a randomised test before any uplift is claimed.

---

## Architecture

```
Inside Airbnb CSVs (listings, reviews)
        |
        v
Python (profile, clean, flag)  -->  data/processed/*.csv
        |
        v
AWS S3   raw/  and  clean/   (read access through an IAM role, no keys in code)
        |
        v
Snowflake   STAGING (tables)  -->  ANALYTICS (views)
        |                              |
        v                              v
Business SQL                      Power BI (3 pages)
        
Python ML (random forest fair price) --> ml_scored_listings.csv --> Power BI
```

| Layer | Tool | Role |
|---|---|---|
| Prepare | Python (pandas, scikit-learn, scipy, matplotlib) | Profiling, cleaning, flags, statistics, EDA, ML |
| Storage | AWS S3 | `raw/` (original files) and `clean/` (validated files) |
| Warehouse | Snowflake | `STAGING` tables loaded with `COPY INTO`; `ANALYTICS` views |
| BI | Power BI | Market Overview, Pricing and Revenue, Host and Underpriced Listings |

## Data

| Table | Rows | Columns | Use |
|---|---|---|---|
| `listings` | 10,369 | 90 raw, 37 kept, 43 after flags | Property, host, price, availability, ratings, estimated occupancy and revenue |
| `reviews` | 545,162 raw, 545,125 usable | 6 raw, 8 after flags | Guest reviews and a demand signal |

Source: [Inside Airbnb](https://insideairbnb.com/get-the-data/), June 2026 snapshot, licensed CC BY 4.0. The calendar file was intentionally not used (millions of rows; listings already summarise availability and estimated occupancy).

Raw data is not stored in this repository. Download the two detailed files from Inside Airbnb and place them in `data/raw/`.

## Data quality log (summary)

| Issue | Rule applied |
|---|---|
| 13 columns 100% empty | Dropped |
| `price` stored as text with a currency symbol | Converted to `price_num` |
| 38.5% of price and revenue missing | Kept as null; `is_priced` flag; verified that missing price means a blocked calendar |
| 64 extreme prices (above the 99th percentile) | Flagged `price_outlier`, kept; medians used |
| Estimated occupancy capped at 255 (886 listings) | Flagged `occupancy_capped`; stated as a limitation |
| `maximum_nights` placeholder 2,147,483,647 | Capped at 365 |
| `has_availability` contradicts `availability_365` | Field not used |
| `bathrooms` and `beds` about half missing | Not used as model inputs |
| 37 reviews without text; 9,045 reviews for listings not in the snapshot | Dropped / flagged `in_listings` |
| Backslashes in 36 review comments broke the Snowflake load | Removed upstream; escape handling disabled; reloaded |

Estimated revenue was verified to equal price x estimated occupancy on all 6,377 priced listings, so revenue and occupancy are excluded from the price model to avoid leakage.

## Statistics

Mann-Whitney U test on estimated revenue, Superhost (1,479 priced listings) vs regular (4,760): p-value essentially zero, **effect size 0.82** (a random Superhost listing out-earns a random regular listing about 82% of the time). Revenue is skewed, so a rank-based test was used, and the effect size is reported because significance alone says little with large samples.

## Machine learning: fair-price model

- **Target:** log of nightly price. **Inputs:** bedrooms, accommodates, room type, neighbourhood, latitude, longitude.
- **Left out on purpose:** occupancy, revenue and review counts (they depend on price, so they would leak the answer).
- **Data:** 5,401 listings (912 dropped for missing bedrooms); 4,320 train, 1,081 test.

| Model | MAE (price units) | R-squared |
|---|---|---|
| Baseline (always the average) | 127 | about 0 |
| Linear regression | 96 | 0.40 |
| **Random forest** | **92** | **0.47** |

Top features: bedrooms 0.294, longitude 0.254, latitude 0.221, entire home 0.102, accommodates 0.074. Every listing was scored with 5-fold cross-validated predictions. A listing is an "ML underpriced candidate" if it is at least 20% below its fair price and has at least 200 estimated booked nights. The model is a screening tool: its average error is about a third of the median price.

## Power BI dashboard

| Page | Question | Screenshot |
|---|---|---|
| Market Overview | How big is the market, where is the revenue, when is demand high? | `powerbi/page1_market_overview.png` |
| Pricing and Revenue | What drives revenue: host type, room type, price band, idle supply? | `powerbi/page2_pricing_revenue.png` |
| Host and Underpriced Listings | Which hosts and listings should be acted on first? | `powerbi/page3_host_underpriced.png` |

Model: `Neighbourhoods` (1) to (many) `Listings`; `Listings` (1) to (1) `ML Scores` through the listing ID (stored as text); `Monthly Reviews` stands alone. Dashboard KPIs were reconciled with Snowflake (10,369 listings, 6,377 priced, median price 291, Superhost share 16.8%).

## Validation across tools

| Check | Result |
|---|---|
| Row counts, Python vs Snowflake | 10,369 and 545,125 match |
| Nine KPIs re-run in Snowflake | All match Python |
| Dashboard cards vs warehouse | Match |
| Underpriced flag in the view vs the original query | 588 both |

## Problems solved (what I learned)

| Problem | Cause | Fix |
|---|---|---|
| `COPY INTO` failed at row 338,548 | A review consisting of a single backslash was read as an escape | Removed backslashes, disabled escape handling |
| Load returned 0 rows | File uploaded to the wrong S3 prefix | `LIST @stage` to find the path, moved the file |
| Dashboard cards did not match the warehouse | 19-digit listing IDs lost precision as numbers | Treated IDs as text in Snowflake and Power BI |
| Top 10 chart showed a neighbourhood with a tiny sample | Two filters ran at the same time | Measure that returns blank under 30 priced listings |
| DAX row-level label failed as a measure | A measure cannot read a single row's value | Moved labels into the Snowflake view |

## Repository structure

```
Airbnb_Pricing_Analytics/
  data/raw/             (not committed)
  data/processed/       (not committed)
  python/               01_data_profiling, 02_statistics, 03_eda notebooks
  ml/                   04_price_model notebook
  sql/                  setup, S3 integration, stage, load, validation, business queries, views
  powerbi/              .pbix file and dashboard screenshots
  documentation/        data quality log, data dictionary, business requirements
  README.md
```

## How to reproduce

1. Download the detailed `listings` and `reviews` files from Inside Airbnb into `data/raw/`.
2. Run the profiling and cleaning notebook; save the clean files to `data/processed/`.
3. Create an S3 bucket with `raw/` and `clean/` folders and upload the files (block public access).
4. In Snowflake, run the SQL scripts in order: setup, storage integration (with your own IAM role), stage, load, validation, views.
5. Run the statistics, EDA and ML notebooks.
6. Open the Power BI file and point it to your Snowflake account.

Never commit access keys, passwords or API keys. Use environment variables or a hidden prompt.

## Limitations

- One snapshot in time; no price history, so effects of price changes cannot be measured.
- Revenue and occupancy are modelled estimates; occupancy is capped at 255 nights.
- 14% of listings were dropped from the model for missing bedrooms.
- R-squared 0.47: useful for screening, not for setting exact prices.
- Associations, not proven causes (for example, Superhost status).
- A GenAI step (theme extraction from recent reviews, with reviewer names removed and the model describing only validated numbers) is planned but not part of this release.

## Author

Shashank | [GitHub](#) | [LinkedIn](#)

*Data: Inside Airbnb (CC BY 4.0). This project is not affiliated with or endorsed by Airbnb.*
