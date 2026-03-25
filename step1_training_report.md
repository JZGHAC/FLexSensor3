# Step 1 report — data ingest, label audit, and training set definition

## Objective
Start the fatigue-onset training workflow by verifying the raw files, reading the 5-minute subject-condition labels from the `Record` sheet, and converting them into session-level training intervals.

## Files ingested
- Raw bilateral CSVs: 8 files extracted from 4 uploaded ZIP archives
- Subject-condition workbook: `Cycling fatigue test record.xlsx` (`Record` tab)

## Raw data audit
| session   | left_file                      | right_file                      |   left_samples |   right_samples |   shorter_duration_min |   sample_mismatch |   left_angle1_mean |   right_angle1_mean |   left_angle2_mean |   right_angle2_mean |
|:----------|:-------------------------------|:--------------------------------|---------------:|----------------:|-----------------------:|------------------:|-------------------:|--------------------:|-------------------:|--------------------:|
| XY1       | 2026-3-3-17-40-22 Left XY.csv  | 2026-3-3-17-40-31 Right XY.csv  |          87817 |           87550 |                29.1833 |               267 |           -32.9235 |            -35.5276 |           -90.0376 |             96.4634 |
| XY2       | 2026-3-4-23-2-32 Left XY.csv   | 2026-3-4-23-3-0 Right XY.csv    |          90846 |           91408 |                30.282  |               562 |           -28.4109 |            -35.7593 |           -99.887  |             99.9162 |
| LM        | 2026-3-18-11-14-39-Left-LM.csv | 2026-3-18-11-14-39-Right-LM.csv |          85471 |           87070 |                28.4903 |              1599 |           -19.3862 |            -32.0204 |          -126.858  |            108.52   |
| KH        | 2026-3-18-11-52-32-Left-KH.csv | 2026-3-18-11-52-32-Right-KH.csv |          91786 |           90525 |                30.175  |              1261 |           -68.2287 |           -114.907  |          -108.587  |             25.3823 |

### Key observations
- All 4 sessions have both left and right files present.
- No missing values were found in `angle1` or `angle2` columns in the extracted CSVs.
- Session durations from the shorter side range from 28.49 to 30.28 min.
- Left/right sample counts are not identical, so later preprocessing should align by sample index and cross-correlation before bilateral feature extraction.
- `angle2` signs differ strongly by side for most riders, which supports applying side-specific sign harmonization before computing symmetry features.

## Record-tab supervision extracted
|   Time_min |   XY1_RPE |   XY1_Cadence |   XY1_HR |   XY2_RPE |   XY2_Cadence |   XY2_HR |   LM_RPE |   LM_Cadence |   LM_HR |   KH_RPE |   KH_Cadence |   KH_HR |
|-----------:|----------:|--------------:|---------:|----------:|--------------:|---------:|---------:|-------------:|--------:|---------:|-------------:|--------:|
|          5 |         4 |            78 |      130 |         4 |            77 |      114 |        4 |           77 |     125 |        4 |           81 |     138 |
|         10 |         5 |            79 |      134 |         4 |            79 |      120 |        4 |           77 |     132 |        4 |           80 |     136 |
|         15 |         6 |            80 |      135 |         6 |            78 |      119 |        5 |           80 |     140 |        4 |           80 |     137 |
|         20 |         6 |            80 |      135 |         6 |            81 |      124 |        6 |           79 |     140 |        4 |           80 |     138 |
|         25 |         7 |            81 |      140 |         7 |            80 |      122 |        7 |           77 |     146 |        6 |           78 |     138 |
|         30 |         8 |            82 |      140 |         8 |            78 |      124 |        8 |           80 |     151 |        6 |           78 |     141 |

## Labeling protocol used for training
- 0–180 s: `IGNORE_WARMUP`
- 180–480 s: `IGNORE_BASELINE`
- 480–510 s: `IGNORE_POSTBASELINE_BUFFER`
- After 8.5 min, training labels are assigned from the 5-minute RPE log:
  - `Fresh`: RPE <= 5
  - `Transition`: first sustained RPE = 6 block
  - `Fatigued`: RPE >= 7
- Session-specific labeled intervals:
  - XY1: Fresh 8.0–12.5 min, Transition 12.5–22.5 min, Fatigued 22.5–30.0 min
  - XY2: Fresh 8.0–12.5 min, Transition 12.5–22.5 min, Fatigued 22.5–30.0 min
  - LM: Fresh 8.0–17.5 min, Transition 17.5–22.5 min, Fatigued 22.5–30.0 min
  - KH: Fresh 8.0–22.5 min, Transition 22.5–30.0 min, no fatigued block

## 30 s / 10 s rolling-window label counts
| session   |   duration_min |   n_windows |   Fresh |   Transition |   Fatigued |   IGNORE_WARMUP |   IGNORE_BASELINE |   IGNORE_POSTBASELINE_BUFFER |
|:----------|---------------:|------------:|--------:|-------------:|-----------:|----------------:|------------------:|-----------------------------:|
| XY1       |        29.1833 |         173 |      24 |           60 |         39 |              17 |                30 |                            3 |
| XY2       |        30.282  |         179 |      24 |           60 |         45 |              17 |                30 |                            3 |
| LM        |        28.4903 |         168 |      54 |           30 |         34 |              17 |                30 |                            3 |
| KH        |        30.175  |         179 |      84 |           45 |          0 |              17 |                30 |                            3 |

## Assessment
This is enough to proceed to preprocessing and feature generation. The main limitation is label sparsity: RPE is only sampled every 5 minutes, so onset is interval-censored rather than exact. That is acceptable for the planned three-state model (`Fresh`, `Transition`, `Fatigued`) with temporal smoothing.

## Recommended Step 2
Preprocess and align the bilateral raw signals for each session:
1. Low-pass filter `angle1` and `angle2`
2. Harmonize sagittal sign conventions
3. Estimate left/right lag using cross-correlation
4. Detect pedal cycles from the sagittal trace
5. Build cycle-level and 30 s rolling-window features
