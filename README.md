# DSAI3202 - Phase 1: ATP Tennis Data Pipeline

**Course:** DSAI3202 - Cloud Computing  

**Team Members:**
- Aneeha Sohail - 60105845
- Laiba Furqan - 60301575

---

## Project Description
This project builds a complete end-to-end data pipeline for ATP Tennis match data (2000-2026) on Microsoft Azure. The goal is to predict the winner of a tennis match using machine learning, based on player rankings, betting odds, and match context.

**ML Hypothesis:** Can we predict the winner of a tennis match based on player rankings, betting odds, surface type, and tournament context?

**Dataset:** ATP Tennis 2000-2026 (Kaggle)  
**Records:** 67,460 matches  
**Features:** 17 raw → 12 engineered features  
**Target Variable:** `winner_is_player1` (binary classification)

---

## Project Structure
```
DSAI3202-Tennis-Pipeline-Project/
├── notebooks/
│   ├── 01_data_ingestion.ipynb       # Data ingestion from Azure Storage
│   ├── 02_etl_pipeline.ipynb         # ETL cleaning and transformation
│   ├── 03_eda.ipynb                  # Exploratory data analysis
│   └── 04_feature_engineering.ipynb  # Feature extraction and selection
├── data/
│   └── raw/                          # Raw data zone
├── .gitignore
└── README.md
```

---

## Azure Architecture
| Resource | Name | Purpose |
|----------|------|---------|
| Storage Account | tennisdatalake60105845 | Data Lake (raw, processed, curated) |
| Databricks Workspace | amazon-dbx-60105845 | Compute & notebooks |
| Cluster | tennis-cluster | Standard_D4s_v3, single node |
| Catalog | amazon_dbx_60105845 | Unity Catalog for governance |

### Storage Zones
- `raw/` → original atp_tennis.csv (preserved, never modified)
- `processed/` → cleaned parquet files partitioned by year
- `curated/` → final feature set ready for ML

---

## II.1 Data Ingestion
**Notebook:** `01_data_ingestion.ipynb`

- **Source:** Kaggle - ATP Tennis 2000-2026 Daily Update dataset
- **URL:** https://www.kaggle.com/datasets/dissfya/atp-tennis-2000-2023daily-pull
- **Format:** CSV (8.98 MB)
- **Ingestion Mode:** Batch
- **Records:** 67,460 matches
- **Columns:** 17
- **Storage Path:** `abfss://raw@tennisdatalake60105845.dfs.core.windows.net/atp_tennis.csv`
- **Refresh Strategy:** Manual batch upload with Git versioning
- **Raw Data Preservation:** Original CSV stored in `raw/` container, never modified
- **Versioning:** Git tag `v1.0` marks Phase 1 completion

### Ingestion Process
1. Downloaded dataset from Kaggle
2. Created Azure Data Lake Storage Gen2 account (`tennisdatalake60105845`)
3. Created containers: `raw`, `processed`, `curated`
4. Uploaded CSV to `raw/` container via Azure Portal
5. Connected Databricks to storage using account key
6. Loaded data using PySpark and verified 67,460 records
7. Saved to `processed/` zone as Parquet for efficient querying

---

## II.2 ETL Process
**Notebook:** `02_etl_pipeline.ipynb`

All transformations are reproducible and implemented in PySpark on Azure Databricks.

### Transformations Applied
| Step | Issue | Fix |
|------|-------|-----|
| Type casting | `Odd_2` column had `-` string values | Used `try_cast` to convert to Double, invalid values → NULL |
| Feature creation | No target variable | Created `winner_is_player1` (1 if Player_1 wins, else 0) |
| Date extraction | Date column not fully utilized | Extracted `year` and `month` columns |
| Duplicates | Checked for duplicates | 0 duplicates found |
| Invalid ranks | Rank value of -1 means unknown | Replaced -1 with NULL |
| Column naming | "Best of" had space (invalid for Delta) | Renamed to `Best_of` |

### Output
- **Format:** Parquet (columnar, efficient)
- **Partitioned by:** year
- **Path:** `abfss://processed@tennisdatalake60105845.dfs.core.windows.net/tennis/cleaned/`
- **Records after cleaning:** 67,460

---

## II.3 Cataloging and Governance
**Notebook:** `02_etl_pipeline.ipynb` (Cell 5)

### Registered Tables
| Table | Catalog | Schema | Records |
|-------|---------|--------|---------|
| atp_tennis_clean | amazon_dbx_60105845 | default | 67,460 |
| atp_tennis_features | amazon_dbx_60105845 | default | 67,433 |

### Data Lineage
```
Kaggle CSV → raw/ (Azure Blob) → processed/ (Parquet) → curated/ (Delta Features)
```

### Schema Definition
| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| Tournament | String | Yes | Tournament name |
| Date | Date | Yes | Match date |
| Series | String | Yes | Tournament series (Grand Slam, ATP250, etc.) |
| Court | String | Yes | Indoor or Outdoor |
| Surface | String | Yes | Hard, Clay, or Grass |
| Round | String | Yes | Match round |
| Best_of | Integer | Yes | Best of 3 or 5 sets |
| Player_1 | String | Yes | First player name |
| Player_2 | String | Yes | Second player name |
| Winner | String | Yes | Match winner name |
| Rank_1 | Integer | Yes | Player 1 ATP ranking |
| Rank_2 | Integer | Yes | Player 2 ATP ranking |
| Pts_1 | Integer | Yes | Player 1 ranking points |
| Pts_2 | Integer | Yes | Player 2 ranking points |
| Odd_1 | Double | Yes | Betting odds for Player 1 |
| Odd_2 | Double | Yes | Betting odds for Player 2 |
| Score | String | Yes | Final match score |
| winner_is_player1 | Integer | No | Target variable (1=P1 wins, 0=P2 wins) |
| year | Integer | Yes | Year extracted from Date |
| month | Integer | Yes | Month extracted from Date |

### Assumptions
- Player listed as Player_1 is not always the home player
- Rank -1 means ranking was unavailable at the time
- Odd_2 `-` values mean odds were not recorded

---

## II.4 Exploratory Analysis
**Notebook:** `03_eda.ipynb`

### Key Statistics
| Metric | Value |
|--------|-------|
| Total matches | 67,460 |
| Date range | 2000 - 2026 |
| Most common surface | Hard (48%) |
| Most common series | ATP250 |
| Player 1 win rate | 50.0% (balanced) |
| Avg Player 1 Rank | 75.8 |
| Avg Player 2 Rank | 75.5 |
| Rank outliers (>500) | 667 matches |

### Correlations
- `rank_diff` and `winner_is_player1` show meaningful correlation
- Betting odds (`Odd_1`, `Odd_2`) are strong predictors of match outcome
- `pts_diff` correlates with `rank_diff` as expected

### Data Risks Identified
- ⚠️ 667 matches with rank > 500 (low-ranked players, noisier data)
- ⚠️ `Odd_2` had `-` string values → converted to NULL during ETL
- ⚠️ `Pts` columns had -1 values → replaced with NULL
- ⚠️ 27 records dropped due to nulls in key columns after feature selection

### Conclusion
Dataset is ready for feature engineering and ML modeling. The target variable is perfectly balanced (50/50), which means no class imbalance handling is needed.

---

## II.5 Feature Extraction and Selection
**Notebook:** `04_feature_engineering.ipynb`

### Feature Justification
| Feature | Description | Justification |
|---------|-------------|---------------|
| rank_diff | Rank_1 - Rank_2 | Higher ranked player (lower number) more likely to win |
| pts_diff | Pts_1 - Pts_2 | Points reflect recent form and consistency |
| odds_diff | Odd_1 - Odd_2 | Betting market is strong predictor of match outcome |
| p1_higher_ranked | 1 if Rank_1 < Rank_2 | Direct rank advantage signal |
| is_grand_slam | 1 if Series == Grand Slam | Top players perform better in Grand Slams |
| is_hard_court | 1 if Surface == Hard | Players specialize in certain surfaces |
| is_clay_court | 1 if Surface == Clay | Clay specialists (e.g. Nadal) dominate on clay |
| is_best_of_5 | 1 if Best_of == 5 | Longer matches favor physically stronger players |
| Rank_1 | Absolute ATP rank of Player 1 | Raw ranking signal |
| Rank_2 | Absolute ATP rank of Player 2 | Raw ranking signal |
| year | Year of match | Captures temporal trends in the sport |
| month | Month of match | Captures seasonal patterns (clay season, grass season) |

### Feature Selection Criteria
- Removed raw string columns (Tournament, Player names, Score) — not useful for ML directly
- Kept numerical and binary features only
- Dropped records with NULL values in selected features

### Output
- **Features:** 12 input features + 1 target
- **Records:** 67,433 (after dropping nulls)
- **Saved to:** `abfss://curated@tennisdatalake60105845.dfs.core.windows.net/tennis/features/`
- **Registered as:** `amazon_dbx_60105845.default.atp_tennis_features`

---

## Version Control
| Branch | Purpose |
|--------|---------|
| `main` | Production-ready code |
| `develop` | Development and experimentation |

| Tag | Description |
|-----|-------------|
| `v1.0` | Phase 1 complete - full pipeline deployed |

---

## How to Reproduce
1. Clone this repository
2. Set up Azure Databricks workspace
3. Create Azure Data Lake Storage Gen2 account
4. Upload `atp_tennis.csv` to `raw/` container
5. Run notebooks in order:
   - `01_data_ingestion.ipynb`
   - `02_etl_pipeline.ipynb`
   - `03_eda.ipynb`
   - `04_feature_engineering.ipynb`
6. Set your storage account key in each notebook where indicated
