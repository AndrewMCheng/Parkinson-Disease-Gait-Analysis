# Wheel Running Gait Analysis

MATLAB pipeline that turns DeepLabCut pose estimates of a mouse on a running wheel into
gait metrics based on Sheppard et al. Input is DLC's filtered CSV output; output is one
row of body-length-normalized metrics per recording, grouped for comparison by treatment.

## Requirements

MATLAB R2023b or newer. Base MATLAB only — no add-on toolboxes.

## Contents

| File | Role |
| --- | --- |
| `wheel_running_gait_analysis_full.mlx` | **Current pipeline.** Manifest-driven; does everything below in one run. |
| `wheel_running_gait_analysis.mlx` | Legacy. Per-file analysis, one recording at a time. |
| `wheel_running_gait_analysis_normalizer.mlx` | Legacy. Divides length metrics by body length. |
| `wheel_running_gait_analysis_compiler.mlx` | Legacy. Box plots of a results CSV, grouped by Author's Note. |
| `Normalized Gait Analysis.csv` | The manifest — the input that drives `_full`. |
| `Normalized Gait Analysis Full.csv` | Output of `_full`. Overwritten on every run. |

`_full` was written by Claude Code by consolidating the other three, which are kept for
reference and because the compiler is still useful on its own. New work should use `_full`.

Video and DLC output are **not** in this repo.

## Data layout

Put one folder per animal ID beside the scripts:

```
<repo>/
  wheel_running_gait_analysis_full.mlx
  Normalized Gait Analysis.csv
  1564/
    1564_20250709_2025-07-09-134234-0000DLC_resnet50_..._filtered.csv
    ...
  1715/
  ...
```

`dataDir` resolves from the script's own location, so the repo works from any path with no
edits. Only `*_filtered.csv` files are read; the unfiltered DLC output is ignored.

Expected DLC format: 19 columns, three header rows (scorer / bodyparts / coords), data from
line 4, bodyparts in the order `nose, FR, FL, BR, BL, tail`, each with x / y / likelihood.
Recordings are 100 fps (`fps` in Configuration).

## Running the current pipeline

1. Drop the new animal's folder next to the script.
2. Add one row per recording to `Normalized Gait Analysis.csv`. Only the first three columns
   matter; leave the rest blank.
3. Open `wheel_running_gait_analysis_full.mlx` and Run.
4. Results land in `Normalized Gait Analysis Full.csv`, followed by box plots of every
   metric grouped by Author's Note.

To inspect a specific recording, set `diagnosticRows` in Configuration to its manifest row
numbers. That renders a gait raster (stand/swing per paw plus inferred wheel state) and a
paw/wheel speed trace over `diagnosticRange`.

### The manifest

| Column | Meaning |
| --- | --- |
| `FilePath` | Path to the filtered DLC CSV, relative to the repo root. |
| `ID` | Animal ID. |
| `AuthorsNote` | Treatment group — **also selects the analysis window**. |

Everything after column 3 is ignored on read and recomputed from scratch, so stale values
there are harmless.

`AuthorsNote` is not just a label:

| Value | Frames analyzed |
| --- | --- |
| `150 DPI` | 1 – 120,000 (first 20 min) |
| `LDOPA` | 300,000 – 420,000 (min 50 – 70) |
| anything else (`Training video`, `60 DPI`, …) | whole recording |

A long drug-day recording therefore appears **twice** in the manifest — once as `150 DPI`
and once as `LDOPA` — pointing at the same file. Both windows clamp to the file length
rather than erroring, so a file shorter than the window is silently analyzed in full.

Paths are resolved in three passes: used as-is if they exist, then joined to the repo root,
then as `<folder>/<file>` under the repo root. Old manifests with absolute paths from
another machine still work.

## How the analysis works

Per recording, after windowing:

1. **Vectors.** For each bodypart, `dx`/`dy` from `gradient` of x and y, giving
   `speed = hypot(dx, dy)` in pixels/frame and `angle = atan2(dy, dx)`.

2. **Gait classification** — direction only, magnitude ignored. `angle` within ±π/2 (moving
   in +x) is `swing`; otherwise `stand`. This is a proxy for swing/stance, not a contact
   detector.

3. **Wheel suppression.** For all six paw pairs, if relative speed difference and relative
   angle difference are both under `threshold_similarVector` (0.25), both paws are moving
   together — they are riding the wheel, not stepping. Any `swing` so flagged is
   reclassified `stand`.

4. **Paw speed floor.** Anything slower than `threshold_pawSpeed` (3 px/frame) is `stand`,
   which removes jitter that direction alone would count as a swing.

5. **Stride and step.** A stride is a contiguous `swing` run; stride length is the distance
   between its first and last frame. At the end of each swing, step length and width come
   from the opposite paw's position (y and x offsets respectively).

6. **Duty factor** is the percentage of frames spent in `stand`. **Temporal symmetry** is
   `(left - right) / (left + right)` on the front and back stand duty factors.

7. **Angular velocity** is the mean frame-to-frame change in the nose→tail axis angle.

8. **Lateral displacement** is the y-range of the nose or tail across each back-right stride
   cycle; **phase offset** is where in that cycle the maximum falls, as a fraction.

9. **Normalization.** The 14 length metrics are divided by that recording's mean nose–tail
   distance, measured over the same window, making them body lengths rather than pixels.
   Rates, ratios, percentages and phase offsets are left alone.

Thresholds live in Configuration: `threshold_pawSpeed`, `threshold_similarVector`, and
`threshold_wheelOn` (diagnostic plots only).

### Output columns

`FilePath`, `ID`, `AuthorsNote`, then 26 metrics:

- `StrideSpeed`
- `StrideLength*`, `StepLength*`, `StepWidth*` for FR / FL / BR / BL — *normalized*
- `TemporalSymmetryF`, `TemporalSymmetryB`
- `DutyFactorF`, `DutyFactorB` (percent)
- `AngularVelocity` (radians/frame)
- `Nose`/`TailLateralDisplacementBR` — *normalized* — and their phase offsets
- `StrideCountPerFrame*` for each paw

## Wheel speed is inferred, not measured

**No wheel encoder data is used anywhere in this pipeline.** `StrideSpeed` is derived
entirely from the video: it is the mean of `wheelSpeed`, which is paw speed on frames where
paws were flagged as moving together in step 3, signed negative when motion points backward
along the wheel.

Real wheel speed recordings exist for only about ten files. That is too few to analyze as
its own group or to validate the inference against, so it is deliberately left out rather
than applied inconsistently across the dataset. The `_full` script is titled
"WITHOUT WHEEL SPEED DATA" for this reason.

If enough encoder data is collected later, it would replace the inference in step 3 —
measured wheel speed would identify wheel-riding frames directly instead of the
similar-vector heuristic, and `StrideSpeed` would become a measurement. That is the single
biggest accuracy improvement available to this pipeline.

## Legacy scripts

The original three-stage workflow, superseded by `_full`:

1. `wheel_running_gait_analysis.mlx` — set `id`, `authorNote` and `filePath` at the top, Run,
   then un-comment the `writetable` append near the bottom to add one row to
   `Andrew Wheel Running Gait Analysis.csv`. Repeat per recording. It also carries the
   raster and speed plots that became `diagnosticRows` in `_full`.
2. `wheel_running_gait_analysis_normalizer.mlx` — reads that CSV, divides the 14 length
   columns by each file's mean nose–tail distance, writes `Normalized Gait Analysis.csv`.
3. `wheel_running_gait_analysis_compiler.mlx` — reads `Normalized Gait Analysis Full.csv`
   and box-plots the first 22 metrics grouped by Author's Note. Reads only the first 25
   columns, so the `StrideCountPerFrame` columns are not plotted.

Note that the manifest `_full` consumes is the normalizer's old output file, which is why it
still carries metric columns. They are ignored.

## Known issues

- **`StrideSpeed` sign is a convention, not a measurement.** Most values are negative
  because the wheel turns one way. Which direction counts as negative depends on camera
  orientation, so a recording filmed from the other side flips sign. Compare magnitudes, or
  wrap the `strideSpeed` assignment in `abs()`.
- **Duty factor and lateral displacement use unfiltered gait.** Steps 3 and 4 apply only to
  stride/step metrics. Duty factor, temporal symmetry and lateral displacement are computed
  from raw direction-based gait, so wheel-riding frames still count as swing there. This is
  inherited from the legacy script and preserved deliberately; be aware when comparing.
- **Swing/stance is direction-based** and has no likelihood filtering — DLC's confidence
  column is read but not used to reject low-confidence frames.
- The legacy `wheel_running_gait_analysis.mlx` computes
  `back_dutyFactor = (fl + fl) / 2` instead of `(bl + br) / 2`. `_full` is correct; the
  legacy value for that one metric is wrong.

# Happy Coding! or whatever
