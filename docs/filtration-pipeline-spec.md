# Data Filtration Pipeline — Implementation Specification
## Fukushima EMDB Air Dose Rate → SOM-Ready Parquet

**Version:** 1.0
**Date:** 2026-03-25
**Target file:** `filter_emdb.py`

---

## 1. Input Data Contract

### 1.1 Source Layout

```
dataset/
  {class_dir}/          # e.g. 02_drive_survey, 03_walk_survey, ...
    {item_dir}/         # e.g. item_087, item_090, ...
      {year}.zip        # e.g. 2011.zip, 2022.zip
        *.csv           # one or more CSV files per ZIP
```

The glob pattern to discover all ZIPs is:
```
dataset/**/*.zip
```

`source_item` is extracted as the `item_NNN` portion of the parent directory name (strip the `item_` prefix if desired, or keep as-is — be consistent).
`source_class` is extracted as the class directory name (e.g., `02_drive_survey`).

### 1.2 CSV Encoding Rules

**Rule 1 — Header detection:**
Read the first non-empty line of each CSV. If it starts with `測定カテゴリー` (UTF-8), treat as Japanese-header file. If it starts with `Category` (ASCII), treat as English-header file.

**Rule 2 — Encoding trial sequence:**
Attempt decoding in this order: `utf-8-sig`, `utf-8`, `shift_jis`, `cp932`. Use the first encoding that produces a parseable header row without `UnicodeDecodeError`. Log which encoding succeeded per file.

**Rule 3 — Item 335 (English-header, mojibake data):**
Item 335 has ASCII column names but the data cells contain shift_jis bytes decoded as latin-1, producing mojibake in string columns. The numeric columns (indices 4, 5, 12, 13, 14, 17, 36) are unaffected. Apply the following strategy:
- Parse with `encoding='cp932'` first (ignoring errors='replace' only as a last resort).
- String columns that are non-recoverable (category, prefecture, city, sampling_point_state, note) should be read and accepted as-is — they are NOT used as filter criteria except `sampling_point_state` and `note` for the `is_decontaminated` flag.
- For `is_decontaminated` on item 335, also search for the byte sequences `\xe9\x99\xa4\xe6\x9f\x93` (UTF-8 for 除染) decoded as latin-1 or cp932 equivalents. If string matching is unreliable, set `is_decontaminated = False` for all item 335 rows and log a warning. This is acceptable because item 335 covers only 0–34.5 km and its decontamination flag is a non-destructive annotation, not a filter.

**Rule 4 — Header normalization:**
Both header variants map to the same canonical column names by position index (not by name matching), since the column count is always 37. Use positional access exclusively. Do not rely on header string matching for column selection.

### 1.3 CSV Schema (37 columns, positional)

| Index | Canonical Name | Pandas dtype | Nullable | Notes |
|-------|---------------|-------------|---------|-------|
| 0 | `category` | `str` | No | Always `空間線量率` / `Spatial dose rate` in kept rows |
| 1 | `measurement_type` | `str` | No | Walk, Car, AirBorne, 定点, etc. |
| 2 | `organization` | `str` | Yes | NRA, CAO, 福島県, etc. |
| 3 | `mesh_code` | `str` | Yes | Representative mesh code (not primary key) |
| 4 | `latitude` | `float64` | No | Decimal degrees WGS-84 |
| 5 | `longitude` | `float64` | No | Decimal degrees WGS-84 |
| 6 | `prefecture` | `str` | Yes | |
| 7 | `city` | `str` | Yes | |
| 8 | `point_id` | `str` | Yes | Measurement point number — not used |
| 9 | `sampling_start_date` | `str` | Yes | Always empty; do not parse |
| 10 | `sampling_finish_date` | `str` | Yes | Always empty; do not parse |
| 11 | `correction_date` | `str` → `date` | No | Primary time axis. See Section 3.1 |
| 12 | `value` | `float64` | No | Dose rate in µSv/h |
| 13 | `statistical_error` | `float64` | Yes | Absolute error in same unit as value |
| 14 | `detection_lower_limit` | `float64` | Yes | Empty string → NaN |
| 15 | `unit` | `str` | Yes | Expected: `μSv/h` |
| 16 | `sample_mass_kg` | `str` | Yes | Not used |
| 17 | `measurement_height_cm` | `float64` | Yes | Standard is 100 cm |
| 18 | `lower_depth_cm` | `str` | Yes | Not used |
| 19–24 | *(unused)* | `str` | Yes |濁度, SS濃度, 水深, 含水率, 試料の状態, 測定方法 |
| 25 | `sampling_point_state` | `str` | Yes | Used for decontamination flag |
| 26–28 | *(unused)* | `str` | Yes | 試料の種類 large/middle/small |
| 29 | `note` | `str` | Yes | Used for decontamination flag |
| 30 | `project_id` | `str` | Yes | |
| 31 | `mesh_id_basic` | `str` | Yes | Basic grid square |
| 32 | `mesh_id_250m` | `str` | No | Primary spatial key for all mesh-level operations |
| 33 | `mesh_id_100m` | `str` | Yes | |
| 34 | `mesh_id_50m` | `str` | Yes | |
| 35 | `mesh_id_20m` | `str` | Yes | |
| 36 | `distance_from_fdnpp_km` | `float64` | Yes | Pre-computed; Haversine fallback if NaN |

**Columns dropped before writing Parquet:** indices 8, 9, 10, 16, 18, 19, 20, 21, 22, 23, 24, 26, 27, 28, 31, 33, 34, 35. These are retained in memory during processing but excluded from output.

---

## 2. Output Data Contract

### 2.1 Parquet Schema

File: `output/filtered_emdb.parquet`

| Column | Type | Nullable | Description |
|--------|------|---------|-------------|
| `category` | `string` (dict-encoded) | No | Always `空間線量率` |
| `measurement_type` | `string` (dict-encoded) | No | Walk / Car / AirBorne / 定点 |
| `organization` | `string` (dict-encoded) | Yes | |
| `mesh_code` | `string` | Yes | |
| `latitude` | `float32` | No | Truncated from float64 at write time |
| `longitude` | `float32` | No | |
| `prefecture` | `string` (dict-encoded) | Yes | |
| `city` | `string` | Yes | |
| `correction_date` | `date32` | No | Arrow date type, no time component |
| `value` | `float32` | No | µSv/h |
| `statistical_error` | `float32` | Yes | |
| `detection_lower_limit` | `float32` | Yes | |
| `unit` | `string` (dict-encoded) | Yes | |
| `measurement_height_cm` | `float32` | Yes | |
| `sampling_point_state` | `string` | Yes | |
| `note` | `string` | Yes | |
| `project_id` | `string` (dict-encoded) | Yes | |
| `mesh_id_250m` | `string` | No | |
| `distance_from_fdnpp_km` | `float32` | No | Haversine value if original was NaN |
| `year` | `int16` | No | Extracted from correction_date |
| `is_decontaminated` | `bool` | No | Default False |
| `height_anomalous` | `bool` | No | |
| `source_item` | `string` (dict-encoded) | No | e.g., `item_090` |
| `source_class` | `string` (dict-encoded) | No | e.g., `03_walk_survey` |

**Parquet write settings:**
- Engine: `pyarrow`
- Compression: `snappy`
- Row group size: 500,000 rows
- No partitioning (single file output)

### 2.2 Parquet column order

Preserve the order listed in Section 2.1. Do not sort alphabetically.

---

## 3. Pre-processing (Before Filter Stages)

### 3.1 Date Parsing

Column index 11 (`correction_date`) contains dates in one of two formats:
- `YYYY/MM/DD` (e.g., `2022/10/12`)
- `YYYY/M/D` (e.g., `2023/2/28`)

Use `pandas.to_datetime(col, format='mixed')` or equivalently `dateutil.parser.parse`. Do NOT use a fixed `format=` string as it will fail on single-digit months/days.

Rows where `correction_date` cannot be parsed (empty string, non-date text, NaN) must be:
1. Logged as parse failures with item, file, and row index.
2. Dropped before any filter stage runs.
3. Counted in `filter_summary.json` under `stage_0_parse_failures`.

### 3.2 Distance Column Computation

After loading each file, apply:
```
If distance_from_fdnpp_km is NaN or empty:
    Compute Haversine distance from (latitude, longitude) to FDNPP = (37.421389, 141.032778)
    Set distance_from_fdnpp_km to computed value
    Set distance_was_computed = True (internal flag, not written to Parquet)
    Log: "Haversine fallback used for N rows in {item}/{file}"
```

FDNPP coordinates (WGS-84): latitude 37.421389°N, longitude 141.032778°E.

Haversine implementation must use Earth radius = 6371.0 km. Do not use geopy to avoid an optional dependency; implement directly or use `numpy`.

Rows where both `distance_from_fdnpp_km` is NaN/empty AND `latitude`/`longitude` are NaN/empty must be dropped before Stage 1 and counted in `stage_0_missing_coords`.

### 3.3 Numeric Coercion

Apply `pd.to_numeric(..., errors='coerce')` to columns: `value` (12), `statistical_error` (13), `detection_lower_limit` (14), `measurement_height_cm` (17), `distance_from_fdnpp_km` (36), `latitude` (4), `longitude` (5). Coercion failures produce NaN; handle per column as specified in each stage.

### 3.4 Category Pre-filter

Before running filter stages, drop rows where `category` (column 0) is not `空間線量率`. In practice this field should always be `空間線量率` within these dataset directories, but non-air-dose records occasionally appear in `78_other`. Log counts per item. These rows are NOT counted in the main pipeline stages — they are removed in pre-processing and reported separately under `stage_0_non_air_dose`.

### 3.5 Null mesh_id_250m

Rows where `mesh_id_250m` is null or empty string after coercion must be dropped before Stage 3 (which uses it as a grouping key). Count them under `stage_0_missing_mesh_id`.

### 3.6 Computed columns added at load time

After loading and before Stage 1:
```python
df['year'] = df['correction_date'].dt.year.astype('Int16')
df['source_item'] = item_dir_name          # str, from path
df['source_class'] = class_dir_name        # str, from path
df['height_anomalous'] = (
    df['measurement_height_cm'].notna() &
    (df['measurement_height_cm'] != 100.0)
)
df['is_decontaminated'] = (
    df['sampling_point_state'].str.contains('除染|decontamination', case=False, na=False) |
    df['note'].str.contains('除染|decontamination', case=False, na=False)
)
```

---

## 4. Filter Stage Specifications

The pipeline applies six filter stages sequentially to the combined DataFrame assembled from all valid CSV files. "Combined" means all ZIPs from all items concatenated into one DataFrame before any mesh-level group operations (Stages 3, 4, 5 operate on the full combined pool).

### Stage 1 — Distance Cutoff

**Input precondition:** All rows have a valid `distance_from_fdnpp_km` (NaN rows already dropped in pre-processing).

**Logic:**
```python
df = df[df['distance_from_fdnpp_km'] <= distance_cutoff]
```

Default `distance_cutoff` = 80.0 km (inclusive).

**Edge case:** The distance column in some files stores integer strings (e.g., `"85"` without decimal). After numeric coercion this becomes `85.0`. No special handling needed beyond coercion.

**Output postcondition:** All rows have `distance_from_fdnpp_km` in [0, distance_cutoff].

**Expected row reduction:** ~15–25% of total rows dropped (based on Item 090 empirical data showing ~20% of rows are beyond 80 km in later years; earlier years extend further).

---

### Stage 2 — Detection Limit Exclusion

**Input precondition:** Stage 1 output.

**Logic:**
```python
mask_below_limit = (
    df['detection_lower_limit'].notna() &
    (df['value'] <= df['detection_lower_limit'])
)
df = df[~mask_below_limit]
```

This is intentionally `<=` not `<`. A value exactly equal to the detection lower limit is considered a non-detect and must be excluded.

**Edge case — value is NaN:** Rows where `value` is NaN after coercion are also dropped at this stage:
```python
df = df[df['value'].notna()]
```
Count NaN-value drops separately in the summary under `stage_2_nan_value`.

**Edge case — detection_lower_limit is NaN:** When `detection_lower_limit` is NaN, the condition `detection_lower_limit.notna()` is False, so the row is kept regardless of `value`. This is correct behavior — missing detection limit means the record is presumed a valid detection.

**Output postcondition:** All rows have `value > 0` (effectively, since all dose-rate values should be positive and values at or below detection limit are excluded). No row has NaN `value`.

**Expected row reduction:** Typically small (< 5%) for air dose rate data; the detection limit column is sparse.

---

### Stage 3 — Initial Dose Rate Threshold (Mesh-Level)

**Purpose:** Remove entire mesh cells whose earliest recorded dose rate is at or below `dose_threshold` (default 0.10 µSv/h). This eliminates background-noise locations with no contamination signal.

**Input precondition:** Stage 2 output. All rows have valid `mesh_id_250m`, valid `correction_date`, valid `value`.

**Logic:**
```python
# Find earliest correction_date per mesh cell
earliest_date_per_mesh = df.groupby('mesh_id_250m')['correction_date'].min()

# Get the value at that earliest date (if multiple rows share the earliest date, take their mean)
earliest_values = (
    df
    .merge(earliest_date_per_mesh.rename('earliest_date'), on='mesh_id_250m')
    .query('correction_date == earliest_date')
    .groupby('mesh_id_250m')['value']
    .mean()
)

# Identify mesh cells that PASS the threshold
passing_meshes = earliest_values[earliest_values > dose_threshold].index

# Keep only rows belonging to passing mesh cells
df = df[df['mesh_id_250m'].isin(passing_meshes)]
```

**Boundary condition:** `> dose_threshold`, not `>=`. A value exactly at 0.10 µSv/h fails the threshold and the entire mesh cell is removed.

**Edge case — same mesh_id_250m appears in multiple source_items:** This is handled naturally. The group operation pools all records for a given mesh_id_250m regardless of source_item. The initial dose rate is the global minimum date across all sources contributing to that mesh cell. This is the correct behavior — the physical location has one true early reading.

**Edge case — mesh_id_250m with only one row that gets removed by Stage 2:** The mesh simply does not appear in Stage 3 input. No special handling needed.

**Output postcondition:** For every remaining mesh_id_250m, the earliest-date mean value is > dose_threshold.

**Expected row reduction:** Significant. Empirical data shows median dose rates at 60–80 km range are 0.06–0.10 µSv/h, so a meaningful fraction of outer-zone mesh cells will be eliminated. Estimated 30–50% of Stage 2 rows dropped, concentrated in the 50–80 km band.

---

### Stage 4 — Temporal Completeness (Mesh-Level)

**Purpose:** Remove mesh cells with fewer than `min_temporal_points` distinct measurement dates. Mesh cells with too few time points cannot support temporal feature extraction for the SOM.

**Input precondition:** Stage 3 output.

**Logic:**
```python
date_counts = df.groupby('mesh_id_250m')['correction_date'].nunique()
qualifying_meshes = date_counts[date_counts >= min_temporal_points].index
df = df[df['mesh_id_250m'].isin(qualifying_meshes)]
```

Default `min_temporal_points` = 4.

`nunique()` counts distinct calendar dates, not distinct (date, value) pairs. Two rows measured on the same date count as 1 time point.

**Edge case — cross-item date diversity:** A mesh cell covered by two source items in different years may collectively have enough dates even if each item alone does not. The pooled count is used. This is intentional — if the same physical 250m cell has 2 dates from drive survey and 2 dates from walk survey, that is 4 distinct dates and passes Stage 4. The Stage 5 measurement-type consistency check will then enforce which type is kept.

**Output postcondition:** For every remaining mesh_id_250m, the count of distinct `correction_date` values is >= `min_temporal_points`.

**Expected row reduction:** Moderate. Many peripheral mesh cells have only 1–2 survey years. Estimated 10–25% additional row reduction.

---

### Stage 5 — Measurement Type Consistency (Mesh-Level)

**Purpose:** Enforce one measurement_type per mesh_id_250m to prevent systematic mixing of survey geometries (car vs. walk vs. airborne).

**Input precondition:** Stage 4 output.

**Logic:**

Step 1 — Identify mesh cells with mixed measurement types:
```python
type_counts_per_mesh = df.groupby('mesh_id_250m')['measurement_type'].nunique()
mixed_meshes = type_counts_per_mesh[type_counts_per_mesh > 1].index
```

Step 2 — For each mixed mesh, determine the winning type:

For majority-wins: count rows per (mesh_id_250m, measurement_type), take the type with the highest count.

For tie-breaking, apply the following priority order (highest to lowest):
1. `Walk`
2. `Car`
3. `AirBorne`
4. `定点`
5. Any other type (alphabetical among remaining, for determinism)

```python
PRIORITY = {'Walk': 0, 'Car': 1, 'AirBorne': 2, '定点': 3}

def resolve_type(group):
    counts = group['measurement_type'].value_counts()
    max_count = counts.max()
    candidates = counts[counts == max_count].index.tolist()
    if len(candidates) == 1:
        return candidates[0]
    # Tie: apply priority
    return min(candidates, key=lambda t: PRIORITY.get(t, 99))
```

Step 3 — Build a mapping mesh_id_250m → winning_type:
```python
winning_type = df.groupby('mesh_id_250m').apply(resolve_type)
```

Step 4 — Filter rows:
```python
df = df[df['measurement_type'] == df['mesh_id_250m'].map(winning_type)]
```

**Note:** For non-mixed mesh cells (only one type), Step 4 is a no-op (all rows pass).

**Logging requirement:** For each mixed mesh where rows are dropped, log: mesh_id_250m, types present, row counts per type, winning type, rows dropped.

**Output postcondition:** For every remaining mesh_id_250m, all rows have the same `measurement_type`.

**Expected row reduction:** Small in absolute terms but meaningful for correctness. Mixed mesh cells are uncommon but possible where survey campaigns overlap spatially.

---

### Stage 6 — Height Flagging (Non-Destructive)

**Purpose:** Flag rows where measurement height deviates from the standard 100 cm. No rows are dropped.

**Logic:**
```python
df['height_anomalous'] = (
    df['measurement_height_cm'].notna() &
    (df['measurement_height_cm'] != 100.0)
)
```

This column was already computed at load time (Section 3.6). Stage 6 is a verification step confirming the flag is consistent with final data. No rows are removed. Log the count of height_anomalous = True rows and their breakdown by source_item.

**Output postcondition:** `height_anomalous` is correctly set for all rows. Row count unchanged from Stage 5.

---

## 5. Edge Case Catalog

### 5.1 Empty ZIPs or ZIPs with No CSV Files

```python
with zipfile.ZipFile(zip_path) as zf:
    csv_names = [n for n in zf.namelist() if n.lower().endswith('.csv')]
    if not csv_names:
        log.warning(f"No CSV files in {zip_path} — skipped")
        summary['skipped_zips'].append(str(zip_path))
        continue
```

An empty ZIP (zero members) raises `BadZipFile` in some Python versions. Wrap the `ZipFile` constructor in a try/except `zipfile.BadZipFile` and log as corrupted.

### 5.2 CSV Files with Zero Data Rows

A CSV that has only a header and no data rows produces an empty DataFrame. After concatenation, this contributes nothing. No error; log as `empty_csv` per file.

### 5.3 CSV Files with Wrong Column Count

If a CSV has a column count != 37 after parsing:
1. Log the file path, actual column count, and first header value.
2. Skip the file (do not attempt positional alignment to a 37-column schema).
3. Count under `stage_0_wrong_column_count`.

### 5.4 NaN mesh_id_250m After Load

Some rows may have an empty string or whitespace for `mesh_id_250m`. Apply:
```python
df['mesh_id_250m'] = df['mesh_id_250m'].str.strip().replace('', pd.NA)
```
Drop rows where `mesh_id_250m` is NA after this normalization (counted in `stage_0_missing_mesh_id`).

### 5.5 Duplicate Rows

Exact duplicate rows (all 37 columns identical) may exist where multiple CSV files within the same ZIP cover overlapping measurement periods. Deduplicate before Stage 1:
```python
df = df.drop_duplicates(subset=[
    'mesh_id_250m', 'correction_date', 'value', 'measurement_type', 'source_item'
])
```
Count and log deduplicated rows under `stage_0_duplicates`.

### 5.6 Mixed Mesh Cells Spanning Multiple Source Items

The same `mesh_id_250m` can appear in both `item_087` (car survey) and `item_090` (walk survey). This is a valid scenario handled by Stage 5 (measurement-type consistency). After Stage 5, all rows in that mesh cell are from the winning measurement type. The `source_item` column retains the original item provenance so downstream analysis can trace the origin.

### 5.7 Distance Stored as Integer String

Column 36 sometimes contains `"85"` (no decimal) or `"6"` (single digit). `pd.to_numeric(..., errors='coerce')` handles this correctly, producing `85.0` and `6.0`. No special handling needed.

### 5.8 Date Format Variants

Known variants observed in samples:
- `2011/06/06` — standard
- `2023/2/28` — single-digit month and day
- `2022/06/22` — standard

Use `pandas.to_datetime(series, format='mixed', dayfirst=False)`. The `format='mixed'` parameter (Pandas >= 2.0) or `infer_datetime_format=True` (Pandas < 2.0) will handle both. After parsing, store as `datetime.date` objects (`.dt.date`) before writing to Parquet as Arrow `date32`.

### 5.9 Files Where `correction_date` Column is Entirely Empty

If all values in column 11 are empty strings or NaN for an entire file, the file is effectively undated and cannot be used for temporal filtering. Drop the file entirely in pre-processing and log under `stage_0_undated_files`.

### 5.10 value Column Contains Non-Numeric Text

Some files may have placeholder text like `"N/D"` or `"-"` in the value column. `pd.to_numeric(..., errors='coerce')` converts these to NaN. They are then dropped in Stage 2 under `stage_2_nan_value`.

---

## 6. Sensitivity Analysis

When `--sensitivity` flag is passed, the pipeline runs twice with `dose_threshold` values of 0.10 and 0.15 µSv/h.

### 6.1 Output Files Per Run

For each threshold `T` in {0.10, 0.15}:

```
output/
  sensitivity_T010/
    filtered_emdb.parquet
    filter_summary.json
  sensitivity_T015/
    filtered_emdb.parquet
    filter_summary.json
  sensitivity_comparison.json
```

Directory names use the threshold multiplied by 100 to avoid decimal in path: `T010` for 0.10, `T015` for 0.15.

### 6.2 Comparison Metrics (sensitivity_comparison.json)

```json
{
  "generated_at": "ISO-8601 timestamp",
  "thresholds_compared": [0.10, 0.15],
  "comparison": {
    "total_rows": {"T010": N, "T015": N},
    "total_meshes": {"T010": N, "T015": N},
    "meshes_retained_by_T010_only": N,
    "meshes_retained_by_T015_only": 0,
    "meshes_retained_by_both": N,
    "row_overlap_fraction": 0.0,
    "mesh_overlap_fraction": 0.0,
    "per_class_rows": {
      "02_drive_survey": {"T010": N, "T015": N},
      "03_walk_survey": {"T010": N, "T015": N}
    },
    "distance_band_retention": {
      "0-20km":  {"T010": N, "T015": N},
      "20-40km": {"T010": N, "T015": N},
      "40-60km": {"T010": N, "T015": N},
      "60-80km": {"T010": N, "T015": N}
    },
    "year_range": {
      "min_year": {"T010": N, "T015": N},
      "max_year": {"T010": N, "T015": N}
    }
  }
}
```

`meshes_retained_by_T015_only` should always be 0 because a higher threshold is strictly more restrictive. Assert this and raise a warning if violated.

---

## 7. filter_summary.json Schema

```json
{
  "generated_at": "2026-03-25T14:32:00Z",
  "parameters": {
    "dataset_dir": "/path/to/dataset",
    "output_dir": "/path/to/output",
    "distance_cutoff_km": 80.0,
    "dose_threshold_usvh": 0.10,
    "min_temporal_points": 4
  },
  "file_discovery": {
    "total_zips_found": 463,
    "total_zips_skipped": 0,
    "total_zips_corrupted": 0,
    "total_csv_files_found": 0,
    "total_csv_files_empty": 0,
    "total_csv_files_wrong_columns": 0,
    "total_csv_files_undated": 0
  },
  "stage_0_preprocessing": {
    "rows_loaded_raw": 0,
    "rows_dropped_parse_failures": 0,
    "rows_dropped_non_air_dose": 0,
    "rows_dropped_missing_coords": 0,
    "rows_dropped_missing_mesh_id": 0,
    "rows_dropped_duplicates": 0,
    "rows_after_preprocessing": 0,
    "haversine_fallback_rows": 0
  },
  "stage_1_distance": {
    "rows_in": 0,
    "rows_dropped": 0,
    "rows_out": 0,
    "pct_dropped": 0.0
  },
  "stage_2_detection_limit": {
    "rows_in": 0,
    "rows_dropped_below_limit": 0,
    "rows_dropped_nan_value": 0,
    "rows_out": 0,
    "pct_dropped": 0.0
  },
  "stage_3_dose_threshold": {
    "rows_in": 0,
    "meshes_in": 0,
    "meshes_dropped": 0,
    "meshes_out": 0,
    "rows_out": 0,
    "pct_rows_dropped": 0.0,
    "pct_meshes_dropped": 0.0
  },
  "stage_4_temporal_completeness": {
    "rows_in": 0,
    "meshes_in": 0,
    "meshes_dropped": 0,
    "meshes_out": 0,
    "rows_out": 0,
    "pct_rows_dropped": 0.0,
    "pct_meshes_dropped": 0.0
  },
  "stage_5_type_consistency": {
    "rows_in": 0,
    "mixed_meshes_found": 0,
    "rows_dropped": 0,
    "rows_out": 0,
    "pct_dropped": 0.0
  },
  "stage_6_height_flag": {
    "rows_in": 0,
    "height_anomalous_true": 0,
    "rows_out": 0
  },
  "final_output": {
    "total_rows": 0,
    "total_meshes": 0,
    "total_items": 0,
    "year_range": [0, 0],
    "distance_range_km": [0.0, 0.0],
    "value_range_usvh": [0.0, 0.0],
    "is_decontaminated_true": 0,
    "height_anomalous_true": 0,
    "measurement_type_counts": {},
    "organization_counts": {},
    "source_class_counts": {}
  },
  "per_item_breakdown": {
    "item_087": {
      "source_class": "02_drive_survey",
      "zips_processed": 14,
      "rows_loaded": 0,
      "rows_after_distance": 0,
      "rows_in_final": 0,
      "meshes_in_final": 0
    }
  }
}
```

**Note on per_item_breakdown:** The per-item row counts for stages 3–5 are approximate because those stages operate on the combined DataFrame and cannot be cleanly attributed to individual items for mesh-level operations. Report `rows_after_distance` (Stage 1 output, attributable by source_item) and `rows_in_final` (final output count by source_item). Do not attempt to attribute mesh-level drop counts per item.

---

## 8. Logging Specification (filter.log)

Use Python `logging` module. Format:
```
%(asctime)s [%(levelname)s] %(message)s
```

Log to both file (`filter.log`) and stdout simultaneously using two handlers.

Log levels:
- `INFO`: Pipeline start/end, stage transitions, row counts at each stage boundary, parameter summary.
- `WARNING`: Haversine fallback used, encoding fallback used, item 335 decontamination flag unreliable, mixed-type meshes resolved, empty ZIPs, empty CSVs.
- `ERROR`: File cannot be opened, ZIP is corrupted, column count mismatch, all records for an item fail pre-processing.
- `DEBUG`: Per-file row counts, per-file encoding used (activate with `--verbose` flag).

Required log lines at each stage:
```
INFO  Stage 1 (distance ≤ 80.0 km):     in=N  dropped=N  out=N
INFO  Stage 2 (detection limit):         in=N  dropped=N  out=N
INFO  Stage 3 (dose threshold > 0.10):   in=N  meshes_dropped=N  rows_out=N
INFO  Stage 4 (temporal ≥ 4 dates):      in=N  meshes_dropped=N  rows_out=N
INFO  Stage 5 (type consistency):        in=N  dropped=N  out=N
INFO  Stage 6 (height flag):             anomalous=N  (non-destructive)
INFO  Final output: N rows, N meshes written to output/filtered_emdb.parquet
```

---

## 9. CLI Argument Handling

```python
import argparse

parser = argparse.ArgumentParser(description='Filter JAEA EMDB air dose rate data.')
parser.add_argument('--dataset-dir', default='dataset/', type=str)
parser.add_argument('--output-dir', default='output/', type=str)
parser.add_argument('--distance-cutoff', default=80.0, type=float)
parser.add_argument('--dose-threshold', default=0.10, type=float)
parser.add_argument('--min-temporal-points', default=4, type=int)
parser.add_argument('--csv', action='store_true',
                    help='Also write output/filtered_emdb.csv')
parser.add_argument('--dry-run', action='store_true',
                    help='Discover files and report counts only; do not load or filter data.')
parser.add_argument('--sensitivity', action='store_true',
                    help='Run at thresholds 0.10 and 0.15 µSv/h and write comparison.')
parser.add_argument('--verbose', action='store_true',
                    help='Enable DEBUG-level logging.')
```

`--dry-run` behavior: Walk the dataset directory, count ZIPs and estimated CSV files (by opening each ZIP and listing members without extracting), print a table of item → ZIP count → estimated size, then exit without loading any data. Write no output files. Duration should be under 30 seconds for 463 ZIPs.

`--csv` behavior: After writing Parquet, additionally write `output/filtered_emdb.csv` using `df.to_csv(..., index=False, encoding='utf-8-sig')`. The `utf-8-sig` BOM ensures correct rendering in Excel. Note: CSV output for large datasets may be slow; this is expected.

`--sensitivity` behavior: Overrides `--dose-threshold`. Runs the full pipeline twice (0.10 and 0.15), writes to subdirectories, then writes comparison JSON. The main `output/` directory receives no files in sensitivity mode — all output goes to subdirectories.

If `--sensitivity` is combined with `--dose-threshold`, raise `argparse.ArgumentError`: "Cannot combine --sensitivity with --dose-threshold."

---

## 10. Python Environment and Dependencies

Required packages (to be added to `requirements.txt`):
```
pandas>=2.0.0
pyarrow>=12.0.0
numpy>=1.24.0
```

No additional geospatial libraries (geopandas, shapely, geopy) are required. Haversine is implemented in pure NumPy.

Python version: 3.9+.

The implementation must NOT use `dask` or `spark` — the dataset fits in memory (estimated 2–8 GB peak RAM for full load before filtering; filter aggressively early, specifically Stage 1 distance filter is applied per-file at load time before concatenation to reduce peak memory).

---

## 11. Memory Management Strategy

Because the dataset is large, apply Stage 1 (distance filter) at the per-file level during loading, before appending to the combined DataFrame:

```python
chunks = []
for zip_path in all_zips:
    df_chunk = load_csv_from_zip(zip_path)          # pre-processing per file
    df_chunk = df_chunk[df_chunk['distance_from_fdnpp_km'] <= distance_cutoff]  # Stage 1 inline
    chunks.append(df_chunk)
df_combined = pd.concat(chunks, ignore_index=True)
```

This avoids holding 100% of raw data in memory simultaneously. Stages 2–6 then operate on the already-distance-filtered combined frame.

Do not store the raw (pre-filter) DataFrame after distance filtering is applied to each chunk.

---

## 12. Implementation Order Recommendation

For the Python developer, implement in this order:

1. CLI argument parsing and logging setup.
2. ZIP discovery (`--dry-run` mode becomes testable immediately).
3. Single-file CSV loader with encoding detection, column coercion, and computed columns.
4. Stage 1 inline (distance filter per chunk at load time).
5. Concatenation and Stages 2–6 in sequence.
6. Parquet writer and filter_summary.json writer.
7. `--csv` flag.
8. `--sensitivity` flag (wraps steps 4–7 in a loop).
9. Integration test: run on a single item directory, verify row counts, verify Parquet schema.
