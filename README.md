# Wheel Running Gait Analysis

MATLAB pipeline that turns DeepLabCut pose estimates of a mouse on a running wheel into
gait metrics based on Sheppard et al. Input is DLC's filtered CSV output; output is one
row of body-length-normalized metrics per recording, plotted by treatment and genotype.

## Requirements

MATLAB R2023b or newer. Base MATLAB only — no add-on toolboxes.

## Contents

| File | Role |
| --- | --- |
| `wheel_running_gait_analysis_full.mlx` | **Current pipeline.** Manifest-driven; does everything below in one run. |
| `wheel_running_gait_analysis.mlx` | Legacy. Per-file analysis, one recording at a time. |
| `wheel_running_gait_analysis_normalizer.mlx` | Legacy. Divides length metrics by body length. |
| `wheel_running_gait_analysis_compiler.mlx` | Legacy. Box plots of a results CSV, grouped by Author's Note. |
| `filepaths.csv` | The manifest — the input that drives `_full`. |
| `results.csv` | Output of `_full`. Overwritten on every run. |
| `NP recording file list.xlsx` | Animal metadata. `_full` reads the `Genotype` sheet to group the plots. |

`_full` was written by Claude Code by consolidating the other three, which are kept for
reference. New work should use `_full`.

Video and DLC output are **not** in this repo.

## Data layout

Put one folder per animal ID beside the scripts:

```
<repo>/
  wheel_running_gait_analysis_full.mlx
  filepaths.csv
  NP recording file list.xlsx
  1564/
    1564_20250709_2025-07-09-134234-0000DLC_resnet50_..._filtered.csv
    ...
  1715/
  ...
```

Paths in the manifest are relative to MATLAB's **Current Folder, which must be this folder** —
the script does not resolve paths from its own location. Only the files the manifest names are
read; by convention those are the `*_filtered.csv` outputs, and the unfiltered DLC CSVs sitting
alongside them are ignored.

Expected DLC format: 19 columns, three header rows (scorer / bodyparts / coords), data from
line 4, bodyparts in the order `nose, FR, FL, BR, BL, tail`, each with x / y / likelihood.
Recordings are 100 fps (`fps` in Configuration).

## Running the current pipeline

1. Drop the new animal's folder next to the script.
2. Add one row per recording to `filepaths.csv`.
3. Open `wheel_running_gait_analysis_full.mlx` and Run.
4. Results land in `results.csv`, followed by box plots of every metric, one figure per
   genotype, grouped by Author's Note.

A full run reanalyzes every file from scratch and takes roughly 15 minutes on the current
dataset.

### Diagnostic figures

Two settings in Configuration turn on per-file figures. They are off by default (`[]`) and
purely for inspection — they change no output.

```matlab
diagnosticRows = [19 22];        % manifest row numbers to render figures for
diagnosticRange = 1000:2000;     % frames shown in those figures
```

`diagnosticRows` holds **manifest row numbers**, i.e. line numbers in `filepaths.csv` counting
the header as row 0 — not animal IDs. Each listed row produces two figures, titled with the
row number, ID and Author's Note so they stay identifiable when several are open at once.

**Gait raster** — five bands over the frame range:

| Band | Meaning |
| --- | --- |
| `FR` `FL` `BR` `BL` | Gray = stand, black = swing |
| `Wheel` | Red = wheel inferred to be turning, white = still |

The wheel band is on when any paw pair matched the similar-vector test *and* the slowest paw
exceeds `threshold_wheelOn`, the second condition being there because a mouse can hold a paw
up while the wheel keeps spinning. Read the two together: black swing bars sitting over a red
wheel band are wheel-riding frames, which is exactly what step 3 reclassifies as stand.

Note the raster draws the **unfiltered** gait, so you see the filter's input rather than its
result. That is the point — it shows you what is about to be rejected.

**Speed trace** — the four paw speeds plus the per-frame max of the signed wheel speed. Useful
for sanity-checking `threshold_pawSpeed` and `threshold_wheelOn` against a real recording. The
y-axis is labelled `Pixels/Sec` but the values are pixels per *frame*, as everywhere else in
the pipeline.

Two gotchas:

- `diagnosticRange` indexes frames **after** windowing, because `applyFrameWindow` runs first.
  On an `LDOPA` row, frame 1000 of the figure is frame 300,999 of the recording.
- The range is clipped to the available frames, and if nothing is left the figure is skipped
  rather than erroring — so an out-of-range `diagnosticRange` silently produces no plot.

### The manifest

`filepaths.csv` needs exactly two columns:

| Column | Meaning |
| --- | --- |
| `FilePath` | Path to the filtered DLC CSV, relative to the Current Folder. |
| `AuthorsNote` | Treatment group — **also selects the analysis window**. |

```csv
FilePath,AuthorsNote
1715\1715_20260312_...100000_filtered.csv,60 DPI
1715\1715_20260617_...100000_filtered.csv,150 DPI
1715\1715_20260617_...100000_filtered.csv,LDOPA
```

Columns are matched **by name**, so their order does not matter and any extra columns are
ignored — older manifests that still carry the 25 metric columns work unchanged.

The animal `ID` is **derived from the filename**, not read from the file: it is the leading
digits of the base name (`1715\1715_20260617_...` → `1715`). A file that does not start with
its tag produces a warning and will not match a genotype.

`AuthorsNote` is not just a label. It selects the frame window in `applyFrameWindow`:

| Value | Frames analyzed | Time |
| --- | --- | --- |
| `150 DPI` | 1 – 120,000 | first 20 min |
| `LDOPA` | 300,000 – 420,000 | min 50 – 70 |
| anything else (`Training video`, `60 DPI`, …) | whole recording | — |

A long drug-day recording therefore appears **twice** in the manifest — once as `150 DPI`
and once as `LDOPA` — pointing at the same file. Both bounds clamp to the file length rather
than erroring, so a recording shorter than the window is analyzed in full (`150 DPI`) or
collapses to a single frame and is skipped with a warning (`LDOPA`).

Paths are resolved in two passes: used as-is if the file exists, otherwise retried as
`<folder>/<file>` under the Current Folder, which rescues old manifests holding absolute
paths from another machine.

## Grouping by genotype

After writing `results.csv`, `_full` reads the `Genotype` sheet of
`NP recording file list.xlsx` (`Tag`, `Sex`, `Genotype`, `DOB`, `2 month`, `Injections`) and
inner-joins it onto the results, matching `ID` to `Tag`. It then produces **one figure per
genotype**, each a tiled layout of all 26 metrics box-charted against `AuthorsNote`.

Figure titles read `KO (n = 3; total = 47)`, where `n` is the drug-trial cohort — the count of
`150 DPI` rows, which should equal the count of `LDOPA` rows since both come from the same
sessions — and `total` is every row for that genotype, training videos included. A warning
fires if those two counts ever diverge.

The sheet is read from `A2` with no end row, so animals appended to the bottom are picked up
automatically — just add them to the `Genotype` sheet and re-run. (It currently holds 23.)

The join is an inner join, so a recording whose ID is absent from the `Genotype` sheet is
dropped from the plots silently. The script prints how many of the result rows survived the
join; if that number is lower than expected, an ID is missing from the sheet.

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
   from the opposite paw's position.

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

## Output columns

`results.csv` is `FilePath`, `ID`, `AuthorsNote`, then the 26 metrics below. "Normalized"
means the value was divided by the recording's mean nose–tail distance, so its unit is body
lengths rather than pixels. Every per-stride metric is the **mean across all strides** in the
recording; a recording with no detected strides yields `NaN`.

**Whole-animal**

| Column | Unit | Meaning |
| --- | --- | --- |
| `StrideSpeed` | px/frame | Inferred wheel speed: the mean, over all frames and all four paws, of paw speed on frames flagged as wheel-riding in step 3 (zero elsewhere). Signed negative when motion points backward along the wheel, so most values are negative. **Not** normalized. |
| `AngularVelocity` | rad/frame | Mean frame-to-frame change in the angle of the nose→tail axis. Near zero for a mouse running straight; persistent non-zero means the body axis is rotating, i.e. turning or listing to one side. |

**Per paw** — `FR`, `FL`, `BR`, `BL`

| Column | Unit | Meaning |
| --- | --- | --- |
| `StrideLength*` | body lengths | Distance the paw travels during one swing, from its position at the first swing frame to its position at the last. *Normalized.* |
| `StepLength*` | body lengths | Fore-aft offset between this paw at swing start and the **contralateral** paw at swing end — the x axis, which is the direction of travel. How far forward the animal places one paw relative to the other. *Normalized.* |
| `StepWidth*` | body lengths | The same offset measured laterally, along y — how far apart the left and right paws are placed. A stance-width measure; widens with instability. *Normalized.* |
| `StrideCountPerFrame*` | strides/frame | Number of detected strides divided by the number of analyzed frames — a stride *rate*, not a count, so it is comparable across recordings of different length. Multiply by `fps` (100) for strides/second. |

**Per axle** — `F` (front) and `B` (back)

| Column | Unit | Meaning |
| --- | --- | --- |
| `DutyFactor*` | percent | Mean of the two paws' stand duty factors — the share of frames that paw spent in `stand`. Higher means more time planted. Values run high — 71–99.8% across the current dataset — because wheel-riding frames are classified as stand. |
| `TemporalSymmetry*` | −1 to 1 | `(left − right) / (left + right)` on the two stand duty factors. Zero means the two paws are loaded equally; the sign says which side is bearing longer. A gait asymmetry measure. |

**Body sway, referenced to the back-right stride cycle**

| Column | Unit | Meaning |
| --- | --- | --- |
| `NoseLateralDisplacementBR` | body lengths | Per BR stride cycle, the nose's y-range (max − min); averaged over cycles. How much the head swings side to side per stride. *Normalized.* |
| `TailLateralDisplacementBR` | body lengths | Same for the tail base. *Normalized.* |
| `NoseLateralDisplacementPhaseOffsetBR` | 0 to 1 | Where within the stride cycle the nose reaches its y-maximum, as a fraction of cycle length. Says *when* in the step the sway peaks. |
| `TailLateralDisplacementPhaseOffsetBR` | 0 to 1 | Same for the tail base. |

A stride cycle here is swing→stand on the back-right paw, using the same filtered gait as
every other metric.

## Wheel speed is inferred, not measured

**No wheel encoder data is used anywhere in this pipeline.** `StrideSpeed` is derived
entirely from the video, as described above.

Real wheel speed recordings exist for only about ten files (`wheel-speed-data/`). That is too
few to analyze as its own group or to validate the inference against, so it is deliberately
left out rather than applied inconsistently across the dataset.

If enough encoder data is collected later, it would replace the inference in step 3 —
measured wheel speed would identify wheel-riding frames directly instead of the
similar-vector heuristic, and `StrideSpeed` would become a measurement. That is the single
biggest accuracy improvement available to this pipeline.

## Legacy scripts

The original three-stage workflow, fully superseded by `_full`. These are kept for reference
only — they still point at the pre-rename filenames, they will not run as-is, and their
numbers are not guaranteed to match `_full`'s. Use `_full`.

1. `wheel_running_gait_analysis.mlx` — set `id`, `authorNote` and `filePath` at the top, Run,
   then un-comment the `writetable` append near the bottom to add one row to
   `Andrew Wheel Running Gait Analysis.csv`. Repeat per recording. It also carries the
   raster and speed plots that became `diagnosticRows` in `_full`.
2. `wheel_running_gait_analysis_normalizer.mlx` — reads that CSV, divides the 14 length
   columns by each file's mean nose–tail distance, writes its output.
3. `wheel_running_gait_analysis_compiler.mlx` — box-plots the first 22 metrics of a results
   CSV grouped by Author's Note, for one hardcoded genotype. Reads only the first 25 columns,
   so the `StrideCountPerFrame` columns are not plotted. `_full`'s grouping section grew out
   of this script.

## Known issues

- **`results.csv` files exported before 17 Aug 2026 have `StepLength` and `StepWidth`
  transposed.** `calculateStrideLength` used to take step width from the x offset and step
  length from the y offset, which is backwards — the body axis runs along image x (median
  nose→tail offset 401 px in x vs 45 px in y) and contralateral paws separate along image y
  (median BR↔BL offset 217 px in y vs 28 px in x). The code is fixed, but column names and
  order did not change, so an old export and a fresh one are **not** comparable on those two
  columns. Re-run the pipeline rather than mixing old and new rows.
- **`StrideSpeed` sign is a convention, not a measurement.** Most values are negative
  because the wheel turns one way. Which direction counts as negative depends on camera
  orientation, so a recording filmed from the other side flips sign. Compare magnitudes, or
  wrap the `strideSpeed` assignment in `abs()`.
- **Swing/stance is direction-based** and has no likelihood filtering — DLC's confidence
  column is read but not used to reject low-confidence frames.

## Author's Notes

Happy Coding! or whatever

Possible Next steps: 
- Process more data to reach a concrete conclusion because we currently have a small sample size (3 for KO and 2 for WT)
- Refine gait analysis. Currently, threshold_PawSpeed does most of the heavy lifting, and similarVectors is practically useless, as a result, wheel speed and stride speed are all heavily innacurate. However, I did notice in the threshold there is a 'noise' level below ~10 pixels per frame where there are a lot of similar paw speeds clustered together which is what similar Vectors attempts to replicate though I think some improvements can be made especially considering the innacurate wheel speed and stride speed.
- gait analysis (stride and swing and speed visualizations) to Neuropixel recordings



PS: The README is ai-generated so I apologize if it hallucinates (I can not be bothered to read it over). 
