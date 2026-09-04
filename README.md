# Underwriting-Time Gamma GLM for Single-Family Home Flood Damage

**Repository:** [github.com/DylanAttlesey/NFIP-Flood-Severity](https://github.com/DylanAttlesey/NFIP-Flood-Severity)

A log-link Gamma generalized linear model predicting the structural damage of a flood event in USD, fitted to anonymized National Flood Insurance Program claims and restricted to information available at underwriting time.

## Motivation

Given a set of facts about a property, it is an underwriter's job to determine the premium required to insure the property against unlikely, disastrous events while ensuring aggregate profit, or at least solvency. Generally, this risk can be modeled as the product of severity and frequency. This repository isolates the **severity** component.

The model uses only information available at underwriting time — with a few exclusions, noted below — to predict the damage done by a flood to the physical structure of a building.

## Data

| Source | Contribution |
|---|---|
| **OpenFEMA NFIP Redacted Claims v2** | Claim-level records, filtered to single-family residences. The target variable is inflation-adjusted `buildingDamageAmount` (assessed damage). |
| **IPUMS NHGIS** | ACS median home value at census block-group level, with imputation from tract and county medians where block-group values are unavailable. |
| **NOAA Coastal Classifications** | County-level flags for coastal shorelines and coastal watersheds. |
| **FRED `WPUIP2311001`** | PPI for construction inputs, used to trend damages to current cost level. |

## Results

The model achieves a **holdout bias of −1.25%** (the average of true losses is close to the average of predicted losses) and a **Gini index of 0.41** (the model performs significantly better than random at predicting which losses will be worse than others), but a **D² of 0.1021** — the model is quite poor at explaining the amount of variation in damage amounts.

<img width="1290" height="440" alt="NFIPSeverityLiftChart" src="https://github.com/user-attachments/assets/d94eb715-2e23-41f2-ab08-1d2bb69ed379" />

### Relativities

Log-link GLMs relate the expected value of a prediction to each predictor with a *relativity* — a coefficient describing how the base damage value is scaled by an increase in a continuous predictor or the presence of a categorical. A relativity above 1.0 indicates increased expected severity; below 1.0, a reduction.

| Feature | Coefficient | Relativity | p-value | 95% CI |
|---|---:|---:|---:|---|
| Intercept | 11.1056 | 66,543.43 | 0.0000 | [62719.32, 70600.70] |
| WATERSHED_FLAG | 0.1935 | 1.2135 | 0.0000 | [1.1864, 1.2412] |
| is_home_value_topcoded | 0.1781 | 1.1950 | 0.0000 | [1.1329, 1.2605] |
| floodZoneCurrent: Zone B & X (shaded) | 0.1744 | 1.1905 | 0.0000 | [1.1353, 1.2484] |
| floodZoneCurrent: A Zones | 0.1448 | 1.1558 | 0.0000 | [1.1383, 1.1735] |
| log(home_value_100k) | 0.1380 | 1.1480 | 0.0000 | [1.1379, 1.1581] |
| primaryResidenceIndicator: True | 0.1246 | 1.1327 | 0.0000 | [1.1156, 1.1500] |
| floodZoneCurrent: Unknown | 0.1157 | 1.1227 | 0.0000 | [1.1008, 1.1450] |
| is_elev_missing | 0.1095 | 1.1157 | 0.0000 | [1.1005, 1.1312] |
| obstructionType: Obstructed | 0.0287 | 1.0291 | 0.0209 | [1.0044, 1.0545] |
| is_county_imputed | 0.0261 | 1.0264 | 0.0037 | [1.0085, 1.0446] |
| basementEnclosureCrawlspaceType: Unknown | 0.0231 | 1.0234 | 0.0004 | [1.0103, 1.0367] |
| floodZoneCurrent: V Zones | 0.0004 | 1.0004 | 0.9850 | [0.9588, 1.0439] |
| elevationDifference | −0.0007 | 0.9993 | 0.0159 | [0.9987, 0.9999] |
| buildingAge | −0.0012 | 0.9988 | 0.0000 | [0.9985, 0.9990] |
| SHORELINE_FLAG | −0.0208 | 0.9795 | 0.0352 | [0.9607, 0.9986] |
| numberOfFloorsInTheInsuredBuilding | −0.0409 | 0.9599 | 0.0000 | [0.9521, 0.9679] |
| rentalPropertyIndicator: True | −0.0765 | 0.9264 | 0.0000 | [0.8980, 0.9557] |
| obstructionType: Unknown | −0.1159 | 0.8906 | 0.0000 | [0.8468, 0.9366] |
| basementEnclosureCrawlspaceType: Subgrade Crawlspace | −0.4819 | 0.6176 | 0.0000 | [0.5936, 0.6426] |
| is_mobile_home | −0.5317 | 0.5876 | 0.0000 | [0.5594, 0.6172] |
| basementEnclosureCrawlspaceType: Unfinished Basement | −0.6133 | 0.5416 | 0.0000 | [0.5295, 0.5539] |
| elevatedBuildingIndicator: True | −0.6820 | 0.5056 | 0.0000 | [0.4825, 0.5298] |
| basementEnclosureCrawlspaceType: Finished Basement | −0.8392 | 0.4321 | 0.0000 | [0.4221, 0.4422] |

**Note on `log(home_value_100k)`:** because home value enters on a log scale, this coefficient is an *elasticity*, not a relativity. The effect of doubling home value is 2<sup>0.1380</sup> = 1.10, a 10% increase in expected severity.

### Risk drivers

The most positively impactful predictor, unsurprisingly, is whether the home is located on a **coastal watershed** — a region where any sort of deluge flows ultimately toward the coast of a large body of water, such as an ocean or one of the Great Lakes. This relativity was 1.21, implying homes located on a coastal watershed have 21% greater expected severity.

The next highest was whether the given census block group had a **censored median value**, which happens when the median value is above $1 million — a 19.5% increase.

Interestingly, the censored-value coefficient lets us estimate the true median value of homes in these areas. Setting the censored and true-value expressions equal, all terms cancel except:

```
exp(β_topcoded) = g ^ β_logvalue
g = exp(β_topcoded / β_logvalue) = exp(0.1781 / 0.1380) = 3.6
```

So the median home values across these block groups may be near $3.6 million. Note that the $1,000,001 ceiling drops out entirely — only the ratio matters. Unfortunately, imputing this home value in the censored regions leaves the model mathematically unchanged, and so results in no improvement.

### Risk mitigators

The negative predictors were even more impactful:

- **Elevated houses** (on stilts) have 50% lower severity. This isn't terribly surprising at first glance, but it is important to note that this is a *severity* model — all claims correspond with reported building damage. It's feasible, a priori, that elevated homes would see only reduced *frequency* of flood damage, and that once flood waters were high enough to impact them they would show similar severity. We see this isn't the case.
- **Mobile homes** show similarly diminished severity, but reason suggests these would see significantly higher frequencies of claims, with the reduced severity due to their relative inexpensiveness.
- **Finished basements** were most reductive of all, at almost 60% lower expected severity. All reported basement types result in significantly reduced severity. I suspect this is because basements flood in much milder flood conditions than ground-level-or-above structures, so they may be much more likely to file from mild damage from mild floods, for which houses without a basement would not file. Unfortunately, the NFIP notes that adjusters regularly record this in feet instead of inches, so it is impossible to easily check if the floodwater heights are systematically lower for records pertaining to a home with a basement — though workarounds are certainly conceivable. 

### Surprisingly unimpactful

The relative elevation of homes over the 1% flood height line, the building age, and whether the county was directly on the coast. Only being on a coastal watershed seemed impactful as a binary.

### Holdout lift and calibration

To evaluate out-of-sample calibration, predictions on the 20% holdout set were ranked and bucketed into deciles. The ratio of actual to predicted damages highlights the model's accuracy at different severity tiers.

| Decile | N | Predicted Mean | Actual Mean | Actual / Predicted |
|---:|---:|---:|---:|---:|
| 1 | 5,496 | $29,547 | $33,500 | 1.134 |
| 2 | 5,495 | $42,279 | $36,636 | 0.867 |
| 3 | 5,496 | $50,030 | $44,289 | 0.885 |
| 4 | 5,495 | $57,428 | $48,352 | 0.842 |
| 5 | 5,496 | $70,509 | $66,860 | 0.948 |
| 6 | 5,495 | $82,429 | $82,673 | 1.003 |
| 7 | 5,495 | $88,615 | $94,752 | 1.069 |
| 8 | 5,496 | $93,992 | $102,533 | 1.091 |
| 9 | 5,495 | $99,876 | $107,907 | 1.080 |
| 10 | 5,496 | $111,145 | $117,536 | 1.058 |

## Key decisions

- **Temporal blinding.** Only variables observable at underwriting are used. Post-loss fields present in the claims file — `waterDepth`, `causeOfDamage`, `floodEvent` — would substantially improve fit but are unavailable when a policy is priced.
- **Gamma family with log link.** GLMs are favored by actuaries for their simplicity and interpretability. In this case, the gamma distribution was chosen as it is the canonical first-choice distribution for severity modeling, where greater losses see variance scale rapidly as a function of expected value. In the gamma's case, the variance scales with the square of the expected mean. The log link was chosen as it allows us to quantify the effects of risk drivers and mitigators as multiplicative scalers of a default severity, namely relativities. The one exception to this is the median home value, which was logged before input to allow a power relationship between home value and severity, which was found superior both conceptually and in terms of the Aikaike Information Criterion.  
- **Spatial and temporal imputation** of median home values from block group, tract, and county levels.
- **Assessed damage, not paid loss,** as the response — so the $250,000 NFIP building coverage limit does not censor the upper tail.

## Limitations

**This is a severity model, not a premium model.** It must be combined with a frequency model and administrative factors — deductibles, policy limits — to determine minimum profitable premiums.

**Regional rather than individual home values.** Private insurers generally will not insure homes against floods without an individual appraised value, but no large public dataset pairs individual home values with claim data. Values had to be imputed from multiple geographic levels: census block groups are designed to encompass demographically homogeneous areas and are therefore the best available, but quality declines rapidly when imputing from tract and then county level, with counties in particular hosting very large ranges of home values. Roughly 9 of 10 records correspond with floods in block groups with well-attested median home values.

**The model is underdispersed,** with a calibration slope of 1.19 — it overpredicts low-severity risks and underpredicts high-severity ones. This could be caused by the model simply not having access to the largest driver of variation among damages (flood height). It could also be due to variability in home values, which would drive variability in predictions, being flattened by the use of block group, tract, and county medians.

## Repository structure

```
NFIPDataWrangling_SingResSeverity.ipynb   Data pipeline — run first
Severity_GLM.ipynb                        Model fitting and diagnostics — run second
data/reference/
    county_classification.csv             NOAA coastal county classifications
```

**`NFIPDataWrangling_SingResSeverity.ipynb`** joins the NFIP claims file, NHGIS block-group median home values, and the NOAA coastal classifications; recodes sentinel and Census jam values; derives the modeling features; and runs a validation suite before writing `model_ready_nfip.parquet`.

**`Severity_GLM.ipynb`** trends damages to current cost level, fits the OLS baseline and the Gamma GLM, and produces the relativities, lift and calibration diagnostics, imputation sensitivity analysis, and comparison against alternative severity distributions.

## Running this

Both notebooks are written for Google Colab with data on Google Drive. To run locally, remove the `drive.mount` calls and point the path constants at a local directory.

**1. Obtain the source data** (not included — see the size note below):

| File | Source | Place at |
|---|---|---|
| `FimaNfipClaimsV2.parquet` | [OpenFEMA NFIP Redacted Claims v2](https://www.fema.gov/openfema-data-page/fima-nfip-redacted-claims-v2) | `BASE/` |
| `*_blck_grp.csv` | [IPUMS NHGIS](https://www.nhgis.org/) — ACS 5-year median home value, block-group level, one extract per vintage | `BASE/nhgis/` |
| `county_classification.csv` | included in this repository | `BASE/data/reference/` |

**2. Set the paths.** Edit `BASE` in the configuration cell of the wrangling notebook and `REPO` in the setup cell of `Severity_GLM.ipynb` to match your environment. Confirm `COUNTY_CSV` points at `data/reference/`.

**3. Run the wrangling notebook top to bottom.** It writes `data/processed/model_ready_nfip.parquet` and raises on any failed validation check.

**4. Run `Severity_GLM.ipynb` top to bottom.** It writes `model_comparison.csv`, `relativities_gamma.csv`, `imputation_sensitivity.csv`, and `model_spec.txt` to `outputs/`.

**Requirements:** `pandas`, `numpy`, `statsmodels`, `patsy`, `scipy`, `matplotlib`, `pandas-datareader`. The FRED series is fetched at runtime, so an internet connection is required.

**Note:** the NFIP claims file and NHGIS extracts are too large for version control and are excluded via `.gitignore`.

## Data sources and citations

**OpenFEMA — FIMA NFIP Redacted Claims (v2).** Federal Emergency Management Agency. https://www.fema.gov/openfema-data-page/fima-nfip-redacted-claims-v2 — This product uses the FEMA OpenFEMA API, but is not endorsed by FEMA. The Federal Government or FEMA cannot vouch for the data or analyses derived from these data after the data have been retrieved from the Agency's website.

**IPUMS NHGIS.** Steven Manson, Jonathan Schroeder, David Van Riper, Katherine Knowles, Tracy Kugler, Finn Roberts, and Steven Ruggles. *IPUMS National Historical Geographic Information System: Version [X.X]* [dataset]. Minneapolis, MN: IPUMS. https://doi.org/10.18128/D050.V[X.X] — *Replace [X.X] with the version you downloaded; NHGIS requires this citation form.*

**NOAA Office for Coastal Management.** *Defining Coastal Counties.* NOAA Digital Coast. https://coast.noaa.gov/digitalcoast/ — County classifications on 2010 boundaries; 452 coastal shoreline counties nested within 769 coastal watershed counties.

**U.S. Bureau of Labor Statistics.** Producer Price Index by Commodity: Special Indexes: Construction Materials [`WPUIP2311001`], retrieved from FRED, Federal Reserve Bank of St. Louis. https://fred.stlouisfed.org/series/WPUIP2311001

# References
**Goldburd, M., Khare, A., Tevet, D., and Guller, D.** *Generalized Linear
Models for Insurance Rating*, 2nd ed. (2025 revision). CAS Monograph Series
Number 5. Arlington, VA: Casualty Actuarial Society, 2025.
https://www.casact.org/monograph/cas-monograph-no-5

## License

MIT License

Copyright (c) 2026 Dylan Attlesey

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

*Note: the data sources above carry their own terms; this license covers the code in this repository only.*
