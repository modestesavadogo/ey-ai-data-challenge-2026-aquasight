# EY Open Science AI & Data Challenge 2026
## Water Quality Prediction — Technical Model Description

**Team:** Modeste SAVADOGO & Mamadou Tahirou DIALLO  
**Best Leaderboard R²:** 0.4829  
**Targets:** Total Alkalinity (TA) · Electrical Conductance (EC) · Dissolved Reactive Phosphorus (DRP)

---

## 1. Problem Framing and Core Difficulty

The challenge asks to predict three water quality indicators at South African monitoring
stations from environmental and remote-sensing features. The central difficulty is not
purely predictive — it is **spatial generalization**: submission stations are clustered
along the southern Cape coast, while training stations are distributed across the entire
country. This geographic gap means the model cannot rely on learning patterns from nearby
labeled examples. Instead, it must learn the physical processes that drive water chemistry —
climate regime, terrain, soil composition, land use — and transfer those relationships to
an unseen region. Standard cross-validation is misleadingly optimistic in this setting
because spatial autocorrelation between folds inflates apparent performance.

The three targets also differ substantially in statistical character: TA is roughly bimodal
and influenced by geology; EC spans three orders of magnitude and tracks aridity; DRP is
right-skewed and event-driven by agricultural runoff. A single multi-output model cannot
capture these divergent driver structures, motivating independent per-target models.

---

## 2. Data Assembly

We assembled a feature dataset from five environmental data sources through a dedicated
extraction notebook that queries external APIs and merges results on
`(Latitude, Longitude, Sample Date)`:

| Source | Type | Key features |
|---|---|---|
| Landsat 8/9 (Microsoft Planetary Computer) | Dynamic, per-date | `nir`, `green`, `swir16`, `swir22`, `NDMI`, `MNDWI`, `NBR`, moisture ratios |
| TerraClimate / ERA5-Land (GEE) | Dynamic, per-date | `precip_7d_sum`, `precip_30d_sum`, `pet`, `soil_moist_top/deep`, `runoff`, `evaporation` |
| SRTM / OpenLandMap / MODIS (GEE) | Static, per-location | `elevation`, `slope`, `aspect`, `soil_ph`, `soil_clay`, `soil_carbon`, `veg_index` |
| SoilGrids-ISRIC / MERIT Hydro / CSP (GEE) | Static, per-location | `soil_cec`, `soil_soc`, `twi`, `human_mod` |
| ESA WorldCover / WorldPop (GEE) | Mixed | `ag_pct`, `urban_pct`, `forest_pct`, `water_pct`, `pop_density_3km` |

Static features (terrain, soil, land cover) are extracted once per unique station location.
Dynamic features are extracted per observation row, with the sample date used to filter the
correct imagery window. All merges use `(Latitude, Longitude, Sample Date)` as the composite
key to guarantee row-level correctness.

---

## 3. Preprocessing

**Hierarchical spatio-temporal imputation.** Remote-sensing features are frequently missing
due to cloud cover or sensor gaps. We fit a 5-level cascade of group means on the training
set only, falling back from fine-grained to coarse levels as needed:

```
Level 1 — Year × Month × Station      (within-season station behaviour)
Level 2 — Month × Station             (long-term seasonal station profile)
Level 3 — Station                     (long-term station mean)
Level 4 — Month                       (national seasonal signal)
Level 5 — Global mean                 (last-resort fallback)
```

This preserves the rich spatial and temporal structure of the dataset. Fitting imputation
parameters on training data only and applying them to both sets prevents any leakage.

**Distance-to-sea.** We compute the geodesic distance (km) from each observation point to
the nearest Natural Earth 10m coastline segment. Rather than using the raw distance directly
(which caused spatial overfitting in early experiments), we apply a bounded decay transform
to derive a smooth "marine influence" proximity signal that generalises better to the
southern coast submission zone. (Exact transform parameters withheld — see note below.)

---

## 4. Feature Engineering

Feature engineering was the highest-impact step in the pipeline, responsible for the largest
single jump in leaderboard R². The key insight was to replace spatial coordinates with
physically motivated derived features that capture the underlying environmental processes.

**Cyclical temporal encoding.** Month and day-of-year are encoded as sine/cosine pairs to
preserve circular continuity (January is adjacent to December). This prevents the model
from treating month 12 and month 1 as maximally distant.

**Precipitation rolling windows.** Station-level rolling sums and means of precipitation
across multiple lag windows capture antecedent moisture conditions at multiple time scales —
critical for phosphorus and conductance which respond to cumulative wetness history, not
just instantaneous precipitation.

**Climate regime discretization.** Terrain wetness, average precipitation, and potential
evapotranspiration are each binned into quantile-based regime categories. These categorical
features allow tree-based models to partition the feature space along physically meaningful
climate regime boundaries.

**Interaction features.** A set of cross-product interaction terms combines the spectral
moisture indices, climate variables, terrain variables, and land-use percentages to encode
physical processes (e.g. moisture signals, climate-zone effects, diffuse pollution loading,
topographic amplification of runoff, and nutrient retention capacity) more directly than
raw coordinates alone.

**Runoff index.** A composite index combining short-term precipitation, terrain slope, and
terrain wetness approximates the intensity of surface runoff at each station, which is a
direct driver of phosphorus and conductance.

> **Note:** The exact feature list, engineering formulas, and column-level details are
> withheld in this public repository, as this pipeline underlies ongoing (unpublished)
> research we intend to continue. The categories above describe the general approach.

---

## 5. Modeling Journey

We explored four strategies in sequence, each motivated by lessons from the previous phase.

**Phase 1 — Standard K-Fold CV (baseline R² ≈ 0.38–0.42).** Random Forest and gradient
boosting models with standard cross-validation. Spatial autocorrelation between folds
inflated CV scores relative to the leaderboard, revealing the overfitting problem clearly.
Separating into per-target models improved this phase to ~0.42.

**Phase 2 — Geographic station selection (no gain).** We restricted training to stations
geographically nearest the submission zone using a BallTree haversine distance search.
Performance did not improve: reducing the training set volume outweighed any distributional
alignment gain.

**Phase 3 — Distance-weighted training with station-level dropout CV (R² ≈ 0.45).** Each
training sample was weighted by the inverse distance of its station to the nearest submission
station. A dropout cross-validation loop (`run_dropout_cv`) randomly removed entire station
IDs at each iteration and averaged predictions across 15–20 rounds to reduce variance.
Improvement was real but modest — the distributional gap between interior and coastal stations
cannot be fully bridged by weighting alone.

**Phase 4 — Per-target ensembles with rich feature engineering (R² = 0.4829).** The final
strategy used the full training dataset (maximum data volume), applied the feature engineering
described above to reduce spatial overfitting, and trained independent weighted ensembles per
target. This was the approach that achieved the best leaderboard result.

---

## 6. Final Model Architecture

Each target uses a different ensemble reflecting its driver structure:

| Target | Ensemble |
|---|---|
| **Total Alkalinity** | ExtraTrees + RandomForest + HistGradientBoosting (weighted blend) |
| **Electrical Conductance** | RandomForest + XGBoost + HistGradientBoosting (weighted blend) |
| **Dissolved Reactive Phosphorus** | RandomForest (standalone) |

ExtraTrees was most effective for TA because its high variance tolerance captures the
bimodal distribution structure. XGBoost's regularisation helped EC where the target spans
a wide range. DRP's high skewness and event-driven nature made ensembling less stable than
a single well-tuned Random Forest.

All predictions are clipped to zero (concentrations cannot be negative).
Per-target feature sets were selected by recursive feature addition validated on the
leaderboard, combined with domain-knowledge pruning.

> **Note:** Exact blend weights, hyperparameters, and per-target feature counts are withheld
> in this public repository — see the note in Section 4.

---

## 7. Key Findings and Lessons Learned

The geographic gap between training and submission stations is the defining challenge.
No spatial selection or weighting strategy bridged this gap effectively — the real gains
came from features that capture the physical processes driving water chemistry regardless
of location. Precipitation antecedent conditions (rolling windows), terrain-driven runoff
(TWI × slope × precip), and climate regime encoding (PET discretization) were the highest-
impact features because they transfer well across geographic zones.

Standard cross-validation is not appropriate for this problem. Future work would benefit
from Leave-Location-Out CV or spatial blocking to get an honest performance estimate during
development, rather than discovering the gap only on the leaderboard.

With more time, we would explore: spatial meta-learner stacking (a second-level model aware
of station coordinates), Bayesian hyperparameter optimisation (Optuna), and semi-supervised
approaches that leverage the submission station feature distributions without their labels
to align the training and test distributions.

> **Best Leaderboard R² = 0.4829**
