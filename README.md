# PlumeFit

PlumeFit implements a template-based-fitting algorithm which functions as a post-processing method for vehicle emission data analysis. The algorithm was developed specifically for evaluating data gained trough roadside measurements using point sampling.  In point sampling, a small part of the diluted exhaust is drawn through an inlet next to the road and analyzed using various instruments, depending on the pollutants of interest. The measurement itself yields concentration time series that need to be evaluated via post-processing procedures to attribute the measured emissions to the relevant vehicle or vehicle group.

## Overview

 PlumeFit builds upon a two-step approach to evaluate the measurement data:

1. **Template extraction**

   Template extraction focuses on gaining prior knowledge required for signal separation using templates of isolated exhaust plumes. A template refers to the area-normalized, average plume shape of a vehicle group within a fixed time window around the peak. A vehicle group is a set of passes that share one template. Groups are formed from vehicle attributes available for each pass, such as vehicle category, fuel type, or exhaust position.

2. **Fitting**

   The second step fits these templates to every pass in the measurement time series. Overlaps are resolved as much as possible, and emission analysis is provided on per vehicle level.

**Input:** time-aligned instrument data and light barrier triggers, plus a vehicle mapping for vehicle groups.

**Output:** one .csv per day and channel with the fitted plume area of every pass.

## Background

PlumeFit was developed as part of my bachelor's thesis at TU Graz (Graz University of Technology) in 2026.

## Quick Start

Tested with Python 3.13 and 3.14.

```shell
pip install -r requirements.txt

python -m implementations.impl_templates.extraction_pipeline.extraction_template
python -m implementations.impl_templates.extraction_pipeline.grouping_template
python -m implementations.impl_templates.extraction_pipeline.copy_needed_data_to_fitting
python -m implementations.impl_templates.fitting_pipeline.fitting_template

# optional: figure of one pass (day, pass_index)
python -m implementations.impl_templates.fitting_pipeline.plot_pass 2024-05-13 12
```

The included data is **synthetic**. It shows the format and lets the templates run.

## Using your own data

1. **Provide the inputs**, see `data_needed_for_extraction/README.md`.
2. **Adjust the `SETTINGS` block** at the top of each implementation template script.

| Script                   | Adjust                                                       |
| ------------------------ | ------------------------------------------------------------ |
| `extraction_template.py` | file names, channel pairs (`PAIRS`), QA values, windows      |
| `grouping_template.py`   | vehicle groups (`GROUPS`)                                    |
| `fitting_template.py`    | days, template assignment, fit parameters, QA checks, diagnostics |

3. **Tune the extraction** with the console overview it prints: QA shares, template tail at the window end, peak positions. Each table names the parameters it helps to set.

Requirements of the data:

- (nearly) constant sampling interval, the same clock for instruments and light barrier

- the light barrier triggers before the plume is measured (the algorithm only check for peaks after the LB-Trigger)

For the algorithm to work properly there should be enough (around 30) passes per vehicle category (and channel) in interest that are isolated from other vehicle passes. By default the algorithm counts a spacing of 30 seconds or more to be sufficient. This is needed for template creation. 

## Output

```markdown
results/<run>/
├── run_config.json              # all settings of the run
├── peak_windows.csv             # peak window per channel in samples and s
├── passes/<day>/<channel>.csv   # the result, see below
└── diagnostics/<day>.zip        # <channel>_segments.csv.gz, <channel>_curves.csv.gz
```

### The fit in short

This section provides some background knowledge regarding the fit.

Every channel of a day is prepared as

$$r(t) = x(t) - \hat{b}(t) - m$$

with the measured signal $`x(t)`$, the background $`\hat{b}(t)`$ (centered rolling percentile) and the median $`m`$ of $`r`$ on
vehicle-free samples. In each segment $`r`$ is then described as

$$r(t) = \sum_k a_k s_k(t - \hat{\tau}_k) + \mathrm{baseline}(t) + e(t), \qquad \Delta t \sum_i s_k(t_i) = 1, \qquad a_k \ge 0$$

| Symbol                         | Meaning                                                      |
| ------------------------------ | ------------------------------------------------------------ |
| $`s_k`$                        | template with **unit area**, one column per pass or per merged group of passes |
| $`a_k`$                        | fitted amplitude. Because $`s_k`$ has unit area, $`a_k`$ **is the plume area** (channel unit · s) |
| $`\tau^0_j`$, $`\hat{\tau}_j`$ | initial and refined peak position of pass $`j`$ in samples   |
| $`\Delta t`$                   | sampling interval in s                                       |
| $`A`$                          | design matrix of the segment: template columns and baseline columns |
| $`n`$, $`p`$                   | number of valid samples and of columns of the segment        |
| $`\hat{\sigma}`$               | noise level of the channel on this day, column `sigma`       |

The coefficients are found by bounded least squares (BVLS), the baseline coefficients stay free:

$$\min_x \lVert r - A x \rVert^2 \quad \text{subject to} \quad a_k \ge 0$$

