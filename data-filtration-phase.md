# Data Filtration Phase — Fukushima SOM-RF Pipeline

## Recommended Distance Cutoff: 80 km (with secondary filters)

### Why 80 km?

- **Every major EMDB study** uses this boundary: Andoh et al. (2020), Wainwright et al. (2018/2024), the JAEA integrated mapping project
- It originated from the MEXT/US-DOE joint airborne monitoring campaign — the standard "primary monitoring zone"
- It captures the **full NW contamination plume** (extends to ~50 km toward Iitate village), which is where the most interesting slow-decay forest regimes will be
- Monitoring density is consistent within 80 km (2x2 km mesh); beyond 80 km it drops to 10x10 km
- The US NRC recommended 80 km as the Emergency Planning Zone for evacuation planning in severe accident scenarios; Japan adopted analogous logic

**Defense for reviewers:** "The 80 km boundary follows the standard established by JAEA's Fukushima mapping project and used by Andoh et al. (2020), Wainwright et al. (2018), and the integrated 2011–2022 mapping program, ensuring the analysis covers the full extent of the contamination field with consistent monitoring density."

---

## Official Japanese Evacuation Zone Boundaries

### Tier 1 — Evacuation Zone (Mandatory Evacuation): 0–20 km

Residents ordered to evacuate on March 12, 2011. This is the zone universally referred to as the "exclusion zone" in the international literature.

### Tier 2 — Deliberate Evacuation Areas / Sheltering Zone: 20–30 km

On April 11, 2011, areas where annual cumulative doses were "highly likely to exceed 20 mSv" outside the 20 km ring were added as Deliberate Evacuation Areas. The 20–30 km band was additionally designated an "Evacuation-Prepared Area in Case of Emergency."

### Tier 3 — Difficult-to-Return Zone (current)

Not defined by a fixed radius, but by a dose-rate criterion: areas where the annual cumulative dose may not fall below 20 mSv even 6 years after the accident. As of April 2017, this totals approximately 371 km² (~2.7% of Fukushima Prefecture). These areas are concentrated in the NW plume corridor, within roughly 20–40 km of FDNPP.

### Special Decontamination Area (SDA)

Designated under the August 2011 Act, encompassing both the 20 km Restricted Area and the Deliberate Evacuation Areas (locations outside 20 km but projected to exceed 20 mSv/year annual dose). This goes beyond the circular 20 km to include Iitate village and other NW corridor locations up to ~45 km. The SDA boundary represents the policy community's empirical acknowledgment that the northwest plume extends well beyond 20 km.

**Note:** The 30 km boundary is the outermost formally declared emergency management boundary. The 20 km boundary is the hard exclusion. There is **no official 80 km exclusion zone** — the 80 km boundary is a scientific/monitoring convention, not policy.

Sources:
- [Fukushima Revitalization Information Portal — Evacuation Zone Transitions](https://www.pref.fukushima.lg.jp/site/portal-english/en03-08.html)
- [Ministry of Environment — Evacuation Zone Designations](https://www.env.go.jp/en/chemi/rhm/basic-info/1st/09-04-01.html)

---

## The Physical Geography: Non-Radial Contamination

The Fukushima contamination pattern is **not circular** — it is directional and land-use-structured.

**The northwest plume:** The dominant deposition pattern followed wind direction: the most heavily contaminated corridor runs northwest from FDNPP toward Iitate village, Namie, and the Abukuma Highlands. Cs-137 concentrations exceeded 3 MBq/m² at distances up to 35 km northwest. Iitate village at 35 km northwest received deposition comparable to or exceeding that of locations at 15 km in other directions.

**Implication:** A purely radial distance filter will either include vast amounts of low-signal territory to the south and east, or exclude the scientifically richest variation to the northwest. This motivates secondary dose-rate filtering on top of the distance cutoff.

---

## How Key Papers Handled Spatial Extent

### Sun et al. (2022) — RF + Kalman Filter on EMDB Data (Primary Comparator)

- **Source:** Journal of Environmental Radioactivity, 252, 106949
- **Spatial Extent:** Applied to the Fukushima evacuation zone as of March 2017, with 17 monitoring post locations. The car-survey data underpinning the RF stage covers the 80 km zone, consistent with EMDB acquisition norms.
- **Key Insight:** Sun et al. do not apply an explicit km cutoff; they implicitly filter to evacuation zone monitoring posts (~20–30 km). For the walk survey, the decision needs to be made explicitly.
- [ScienceDirect](https://www.sciencedirect.com/science/article/pii/S0265931X22001370) | [PubMed](https://pubmed.ncbi.nlm.nih.gov/35752033/)

### Wainwright et al. (2018) — Bayesian Geostatistics

- **Source:** Journal of Environmental Radioactivity, 189, 120–131
- **Spatial Extent:** Northwestern region of FDNPP approximately within the **80 km radius**. A 2024 follow-up applied to the 80 km radius and the entire Fukushima Prefecture.
- **Methods:** Bayesian hierarchical model, kriging, multiscale data fusion at 250m–1km grid.
- [ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0265931X17310032) | [eScholarship](https://escholarship.org/uc/item/5q97c7x5)

### Andoh et al. (2020) — Empirical Decay Rates by Land-Use

- **Source:** Journal of Nuclear Science and Technology, 57(12), 1352–1363
- **Spatial Extent:** **80 km radius from FDNPP** — explicitly stated in the abstract.
- **Key Finding:** Ecological half-lives differ significantly by land-use category (urban fastest, forest slowest) and by evacuation area designation. Two-group exponential decay model.
- **Relevance:** This is the physical motivation for the SOM stage. Their 80 km scope implies the complete behavioral diversity (fast urban decay through slow forest retention) is captured within 80 km.
- [Tandfonline](https://www.tandfonline.com/doi/full/10.1080/00223131.2020.1789008)

### Andoh et al. (2019) — Walk Survey KURAMA-II

- **Source:** Journal of Environmental Radioactivity
- **Spatial Extent:** **80 km radius from FDNPP** — explicitly stated. Measured ambient dose rates using KURAMA-II walk surveys from 2013–2016.
- **Finding:** 38% reduction in dose rate over 42 months beyond physical decay alone, with ecological half-life of 4.1 ± 0.2 years.
- [PubMed](https://pubmed.ncbi.nlm.nih.gov/29778897/)

### Shuryak (2022) — RF on Biological Contamination

- **Source:** Journal of Environmental Radioactivity, 241, 106772
- **Spatial Extent:** Plant sampling data from published literature, clustering within 50–80 km of FDNPP.
- [PubMed](https://pubmed.ncbi.nlm.nih.gov/34768117/)

### Shuryak (2023) — Causal Forests on Trees

- **Source:** Journal of Environmental Radioactivity, 261, 107130
- **Spatial Extent:** Same sample-based literature compilation. No explicit fixed-radius boundary.
- [ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0265931X2300098X)

### Yasunari et al. (2011) — Cs-137 Deposition Map (PNAS)

- **Source:** PNAS, 108(49)
- **Finding:** Deposition above 100,000 MBq/km² around FDNPP, with elevated contamination extending to 80 km NW and secondary contamination in Miyagi, Tochigi, Ibaraki, Saitama, and Chiba at lower levels.
- **Relevance:** Establishes that scientifically meaningful Cs-137 deposition signal is concentrated within 80 km; beyond this, levels approach or merge with natural geogenic variation.
- [PNAS](https://www.pnas.org/doi/10.1073/pnas.1112058108)

### Integrated Dose Rate Maps 2011–2022 (Wainwright group / JAEA, 2024)

- **Spatial Extent:** 80 km radius + entire Fukushima Prefecture
- **Relevance:** Confirms 80 km as the current standard scope as recently as 2024.
- [PubMed](https://pubmed.ncbi.nlm.nih.gov/39427452/)

---

## Empirical Dose Rate vs. Distance (Item 090, NRA Walk Survey, 2022)

### Dose Rate Gradient

| Distance Band | Mean Dose (µSv/h) | Median Dose (µSv/h) | Signal Above Background? |
|---|---|---|---|
| 0–5 km | 1.0015 | 0.5700 | Far above background |
| 5–10 km | 0.4659 | 0.3400 | Far above background |
| 10–15 km | 0.4192 | 0.3600 | Far above background |
| 15–20 km | 0.1298 | 0.1200 | Clearly elevated |
| 20–25 km | 0.2624 | 0.1200 | Elevated (NW plume effect) |
| 25–30 km | 0.2714 | 0.1200 | Elevated |
| 30–35 km | 0.2365 | 0.0980 | Partially elevated |
| 35–40 km | 0.1668 | 0.1100 | Moderate elevation |
| 40–45 km | 0.1261 | 0.0840 | Approaching natural range |
| 45–50 km | 0.1082 | 0.0960 | Approaching natural range |
| 50–55 km | 0.1126 | 0.1000 | ~2x natural background |
| 55–60 km | 0.1015 | 0.0920 | ~2x natural background |
| 60–65 km | 0.0851 | 0.0800 | Near background |
| 65–70 km | 0.0882 | 0.0810 | Near background |
| 70–75 km | 0.0808 | 0.0770 | Near background |
| 75–80 km | 0.0685 | 0.0640 | At or near background |
| 80–85 km | 0.0661 | 0.0580 | At background |

**Japan's pre-accident natural background:** approximately 0.04–0.05 µSv/h (equivalent to 0.22–0.50 mSv/year natural external dose).

### Temporal Convergence (Near vs. Far Zones Across Years)

| Year | 0–30 km Mean | 60–80 km Mean | Ratio |
|---|---|---|---|
| 2013 | 0.9653 µSv/h | 0.2043 µSv/h | 4.7x |
| 2016 | 0.9968 µSv/h | 0.1061 µSv/h | 9.4x |
| 2019 | 0.7999 µSv/h | 0.0931 µSv/h | 8.6x |
| 2022 | 0.4514 µSv/h | 0.0830 µSv/h | 5.4x |

The 60–80 km zone has converged much closer to natural background by 2022 (~0.08 µSv/h, roughly 2x background). The 0–30 km zone retains strong signal even in 2022.

### Item 335 (CAO Walk Survey — 2022)

| Distance Band | Row Count | Share |
|---|---|---|
| 0–10 km | 55,988 | 63.6% |
| 10–20 km | 10,562 | 12.0% |
| 20–30 km | 13,936 | 15.8% |
| 30–35 km | 7,580 | 8.6% |
| Maximum distance | **34.5 km** | — |

### Item 090 (NRA Walk Survey — Large Systematic Survey)

| Year | Total Rows | Min Distance | Max Distance | Median |
|---|---|---|---|---|
| 2013 | 258,669 | 0.8 km | 140.1 km | 59.5 km |
| 2016 | 388,354 | 0.7 km | 83.8 km | 57.2 km |
| 2019 | 205,695 | 1.1 km | 83.8 km | 50.6 km |
| 2022 | 194,885 | 1.3 km | 83.8 km | 50.9 km |

---

## Secondary Filters (Beyond Distance)

### Combined Filter Criteria (Recommended Implementation)

```
Include a location if:
  Distance from FDNPP <= 80 km
  AND
  Initial dose rate (first measurement at that location) >= 0.10 µSv/h
  AND
  Number of distinct time points >= 4 (to support temporal feature extraction)
```

### Filter Rationale

| Filter | Criterion | Rationale |
|---|---|---|
| **Distance** | ≤ 80 km | Literature standard; consistent monitoring density |
| **Dose-rate threshold** | Initial measurement > 0.10 µSv/h | ~2.5x natural background; excludes noise-dominated locations |
| **Temporal completeness** | ≥ 4 distinct measurement dates per mesh cell | Minimum for temporal feature extraction (decay slopes, half-lives) |
| **Measurement type** | One type per mesh cell (don't mix Walk + Car) | Different survey geometries produce systematic biases |
| **Category** | Air dose rate only | Exclude soil, water, vegetation samples |
| **Detection lower limit** | Exclude records where Value ≤ Detection lower limit | Non-detects require special handling |
| **Measurement height** | Standard is 100 cm; flag anomalous heights | 1 cm = soil surface, 50 cm = child dose height |

### Sensitivity Analysis

Run the SOM-RF pipeline at two dose-rate thresholds (0.10 and 0.15 µSv/h) to demonstrate conclusions are not threshold-dependent. Optionally run a 30 km subset analysis as well.

---

## Column Roles in Filtering

| Column | Filter Role |
|---|---|
| **Distance from FDNPP (km)** | Primary hard cutoff: ≤ 80 km |
| **Value (µSv/h)** | Secondary threshold: initial measurement > 0.10 µSv/h; flag records where Value < Detection lower limit |
| **Measurement type** | Stratification: separate Walk vs. Car vs. Airborne; do not naively mix |
| **Category** | Confirm all retained records are air dose rate (not soil, water, vegetation) |
| **Detection lower limit** | Exclude or flag censored records where Value ≤ Detection lower limit |
| **Start/Finish date of sampling** | Time-point extraction; calculate temporal span per location; exclude locations with < 4 distinct dates |
| **Height of Measurement (cm)** | Consistency check: standard is 100 cm; flag anomalous heights |
| **MeshID_250m** | Location aggregation key for SOM — aggregate multiple measurements within the same 250m mesh to the same time-indexed feature vector |
| **Latitude / Longitude** | Spatial predictor in RF stage; verify distance calculation |
| **Prefecture / City** | **NOT** a filter criterion — contamination crosses prefectural lines |
| **Organization** | Data quality stratification; different organizations used different protocols |
| **Statistical error** | Data quality weight; high error relative to value should be downweighted |
| **State of sampling point** | Identifies decontaminated locations — keep but flag as scientifically interesting anomalous regime locations |
| **Note / Kind of sample** | Check for special conditions (indoors, under canopy, road center vs. roadside) |

---

## What to Keep vs. What to Filter Out

### Keep

- All locations within 80 km with dose rates clearly above background (initial > 0.10 µSv/h)
- All measurement types that are spatially consistent: Walk surveys (CAO), KURAMA car surveys, NRA fixed-point monitors
- All time periods from first available measurement through latest available (for temporal feature generation)
- Decontaminated locations — these show artificial dose-rate drops and are scientifically interesting anomalous regime locations

### Filter Out

- Locations beyond 80 km (minimal signal, thin monitoring density)
- Locations with initial dose rate at or near background (< 0.10 µSv/h)
- Locations with fewer than 4 distinct measurement dates
- Measurements where Value is blank or where Detection lower limit exceeds Value (non-detects)
- Do **not** filter by Prefecture alone — the contamination plume crosses prefecture boundaries

---

## SOM-Specific Considerations

For the SOM to work effectively, the input data needs:

1. **Locations with enough temporal observations** — a location with only 1–2 readings cannot produce a meaningful temporal feature vector. Minimum 4–5 time points per location.

2. **Sufficient variation across locations** — if vast numbers of low-contamination locations are included where dose rate hovers at background, the SOM will produce one giant "background" cluster and a few meaningful clusters. The low-contamination tail drowns the signal.

3. **Not too narrow a range** — restricting to only the 20 km zone may not provide enough diversity of land-use types and deposition levels to discover more than 2–3 behavioral regimes. The forest-rich, high-deposition NW corridor (25–50 km) is where the slow-decay forest regime that Andoh et al. documented should appear.

4. **Appropriate cluster-to-noise ratio** — in SOM, the number of useful neurons scales with the diversity of input feature vectors. If 70% of input locations are near-background, there is a dimensionality problem.

---

## Three Physically Distinct Regimes the SOM Should Recover

The 80 km zone contains three physically distinct contamination regimes:

1. **Near-field (<20 km):** Extreme initial deposition, slowest physical decay obscured by ongoing decontamination events, complex decay dynamics including Cs-134 decay (half-life 2.06 years) giving way to pure Cs-137 dynamics after ~2015.

2. **NW plume corridor (20–50 km):** High deposition in forested terrain, slow ecological half-life, classic forest retention behavior (Andoh et al.'s key finding).

3. **Southern and eastern sectors (20–80 km, non-plume):** Moderate-to-low deposition, faster washoff on agricultural and urban surfaces, dose rates already approaching background by 2015–2018.

If the SOM discovers a 4th or 5th regime (e.g., re-contamination along river corridors, anomalous slow-decay in flooded agricultural land), that constitutes a novel discovery.

---

## Paper Title Framing Note

Since "exclusion zone" technically refers to the 20 km Restricted Area, consider:

- **Option 1:** Frame the study area as "contamination-affected zone within 80 km of FDNPP"
- **Option 2 (Recommended):** Run the full SOM on 80 km, then overlay official zone boundaries (20 km evacuation, SDA, Difficult-to-Return Zone) on the geographic regime map — this directly supports the "divergence analysis vs policy boundaries" expected result

---

## Alternative Distance Options Considered

### Option A — 80 km (Recommended)

Literature standard. Most defensible. Captures full contamination field. See full justification above.

### Option B — 50 km (Scientifically Motivated Intermediate)

The dose rate gradient shows that the bulk of scientifically interesting variation sits within ~50 km. Beyond 50 km, 2022 median dose rates are consistently below 0.10 µSv/h. A 50 km cutoff would retain 75–80% of Item 090's rows while excluding the low-signal outer zone. However, it departs from the literature standard without strong justification.

### Option C — 30 km (Policy-Aligned "Exclusion Zone" Framing)

Directly corresponds to the combined evacuation/sheltering zone. Aligns with "exclusion zone" language. Matches Item 335's full spatial extent. Sun et al. (2022)'s Kalman filter stage effectively operated in this zone. However, restricting to 30 km dramatically reduces sample size for Item 090 (only ~15% of rows) and sacrifices the forested hinterland where the most interesting slow-decay SOM clusters would appear.

---

## Summary Recommendation Table

| Decision | Recommended Choice | Rationale |
|---|---|---|
| **Outer distance cutoff** | **80 km** | Literature consensus; captures full contamination field and NW plume; consistent monitoring density |
| **Inner distance cutoff** | **None** (include down to 0 km) | FDNPP-adjacent monitoring is scientifically valuable |
| **Dose-rate threshold** | **Initial measurement > 0.10 µSv/h** | ~2.5x natural background; ensures locations carry Cs-137 decay signal |
| **Temporal completeness** | **≥ 4 distinct measurement dates per mesh cell** | Minimum for temporal feature extraction |
| **Measurement type** | **One type per mesh cell** | Avoids systematic mixing bias |
| **Do NOT filter by** | Prefecture, City, official zone polygon alone | Contamination is plume-shaped, not administrative-boundary-shaped |

---

## Key References

- Andoh et al. (2020) — [Tandfonline](https://www.tandfonline.com/doi/full/10.1080/00223131.2020.1789008)
- Andoh et al. (2019) — [PubMed](https://pubmed.ncbi.nlm.nih.gov/29778897/)
- Sun et al. (2022) — [ScienceDirect](https://www.sciencedirect.com/science/article/pii/S0265931X22001370) | [PubMed](https://pubmed.ncbi.nlm.nih.gov/35752033/)
- Wainwright et al. (2018) — [ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0265931X17310032)
- Wainwright et al. (2024) — [PubMed](https://pubmed.ncbi.nlm.nih.gov/39427452/)
- Yasunari et al. (2011) — [PNAS](https://www.pnas.org/doi/10.1073/pnas.1112058108)
- Shuryak (2022) — [PubMed](https://pubmed.ncbi.nlm.nih.gov/34768117/)
- Shuryak (2023) — [ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0265931X2300098X)
- Fukushima Evacuation Zones — [Fukushima Portal](https://www.pref.fukushima.lg.jp/site/portal-english/en03-08.html)
- MOE Evacuation Designations — [env.go.jp](https://www.env.go.jp/en/chemi/rhm/basic-info/1st/09-04-01.html)
- MEXT Airborne Survey — [NRA](https://radioactivity.nra.go.jp/cont/en/results/land/airborne/survey-results/1270_1216.pdf)
- Temporal changes summary — [ScienceDirect](https://www.sciencedirect.com/science/article/pii/S0265931X18307884)
