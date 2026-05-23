## Regime-Aware Adaptive Design of Experiments

This repository contains a prototype workflow for **regime-aware adaptive design of experiments**. The central idea is that many real-world process-improvement problems are not stationary optimization problems. Processes may shift across seasons, raw-material lots, feedstock sources, equipment states, operator practices, biological states, or accumulated process history. In those settings, an adaptive DoE workflow that assumes one stable response surface may learn the wrong thing from the right data.

This prototype explores whether adaptive DoE can be improved by explicitly detecting and responding to **operational regimes**.

Rather than treating all historical data as equally relevant, the workflow asks:

- Has the process shifted?
- What regime does the current process appear to occupy?
- Which historical observations still transfer?
- Which model should guide the next batch of experiments?
- Should the system exploit the current local regime, preserve global structure, or deliberately probe uncertainty?

The guiding principle is:

> Stationarity should be treated as a hypothesis to be tested, not as the default assumption of adaptive experimental design.

### Novelty and scope

This repository presents a formative proof-of-concept for a **Regime-Aware Adaptive Design of Experiment workflow** intended for dynamic and non-stationary process environments.

This work **does not claim to introduce a fundamentally new theoretical optimization framework or algorithmic class**. Related ideas already exist across fields including:

- non-stationary Bayesian optimization
- contextual optimization
- dynamic optimization
- adaptive control systems
- transfer learning
- changepoint and regime-detection methods

The contribution of this repository is an **applied synthesis and reproducible prototype** showing how these ideas can be organized into a practical workflow for adaptive experimentation.

The workflow is framed around a process-learning problem: how should an adaptive experimental system decide **what to remember, what to discount, and what to relearn** when the system being optimized is changing?

This repository should be interpreted as:

- a proof-of-concept
- a hypothesis-generating framework
- an open technical note
- a foundation for future experimentation and benchmarking

It should not currently be interpreted as:

- a validated production framework
- a benchmarked state-of-the-art optimization method
- a claim of algorithmic novelty

The examples provided use synthetic datasets and simplified assumptions intended to demonstrate behavior and stimulate further development.

### Core hypothesis

The working hypothesis is that an adaptive DoE system can make better recommendations under drift if it uses inferred regime structure to balance:

- retention of useful historical information
- discounting of outdated information
- rapid adaptation to emerging process regimes
- interpretability for experimental decision making

In a stationary process, global learning is often desirable because every experiment contributes to the same underlying response surface. In a nonstationary process, however, older experiments may become partially misleading. A global model may average across incompatible process states, while a purely local model may adapt quickly but lose transferable structure.

This prototype tests whether regime-aware modeling can improve adaptive DoE by comparing several strategies:

- `random`: random candidate selection baseline
- `global_naive`: one global surrogate trained on all data
- `local_only`: surrogate trained only on the current detected regime
- `global_residual`: global backbone model plus regime-specific residual correction
- `gr_mc`: global-residual model with Monte Carlo scoring
- `gr_mc_composite`: global-residual model with composite acquisition
- `full_v2`: expanded acquisition plus adaptive batch policy
- `local_gr_weighted`: weighted ensemble of local-only and global-residual predictions
- `local_gr_committee`: committee-style local/global-residual ensemble

### Prototype structure

The prototype combines three main capabilities.

#### 1. Regime detection

The workflow uses rolling model diagnostics and changepoint detection to infer when the process has shifted. The diagnostic channels include rolling model fit, residual behavior, feature-importance entropy, and feature-importance stability. These rolling diagnostics are then passed through PELT changepoint detection to identify regime boundaries.

The regime-detection tuning notebook performed a grid search over:

- diagnostic channel subsets
- PELT penalty
- rolling window size
- PELT cost model
- minimum segment size

The best detector configuration used:

- all diagnostic channels
- penalty: `8.0`
- window size: `20`
- PELT model: `l2`
- minimum segment size: `window`

Across 10 seeds, this configuration achieved:

- composite score: `0.829 ± 0.064`
- ARI: `0.778 ± 0.083`
- changepoint location error: `4.4 ± 1.9` runs
- detected changepoints: `4.5 ± 0.5`

This suggests that the detector is useful but imperfect. The current implementation should therefore be understood as a prototype regime signal rather than a definitive regime labeler.

#### 2. Stable feature identification

The prototype includes noise-referenced feature stability selection. The goal is to distinguish features that are consistently informative across regimes from features that appear important only under specific transient conditions.

This is important because regime-aware DoE should not simply forget all historical information. Some relationships may remain globally useful, while others may be local to a particular operational state.

The broader goal is to separate:

- globally stable effects
- regime-specific effects
- spurious effects
- transient effects caused by drift or noise

#### 3. Adaptive experimental recommendation

The adaptive loop proposes new experimental batches using surrogate models and candidate-scoring policies. The core question is whether experimental recommendations improve when the surrogate is aware of regime structure.

The prototype evaluates several modeling policies:

- train on all data
- train only on the current regime
- train a global model and correct it with regime-specific residuals
- score candidates using uncertainty and robustness terms
- combine local and global-residual models through ensemble logic

### Surrogate and acquisition tuning

The surrogate/acquisition tuning notebook evaluates the adaptive DoE strategies across a hard nonstationary simulation with multiple regime shifts. The hard-mode schedule uses five regimes, with transitions at runs `30`, `60`, `90`, and `120`. The adaptive experiment starts from 30 initial runs and then proceeds through 24 adaptive iterations with batch size 5.

The main result is that **local regime-aware modeling is currently the strongest simple policy**.

Across 10 seeds, the main methods produced the following mean batch-best performance:

| Method | Final cumulative best | Mean batch-best |
|---|---:|---:|
| random | `18.17 ± 2.21` | `8.81 ± 0.67` |
| global_naive | `19.03 ± 2.54` | `9.19 ± 0.89` |
| local_only | `19.58 ± 2.34` | `9.84 ± 1.02` |
| global_residual | `19.59 ± 2.46` | `9.09 ± 0.53` |
| gr_mc | `18.60 ± 2.54` | `9.13 ± 0.72` |
| gr_mc_composite | `19.31 ± 2.57` | `9.16 ± 0.68` |
| full_v2 | `18.85 ± 3.46` | `8.97 ± 0.88` |

The strongest evidence was not final cumulative best, which often saturates once a good point is found. The more informative metric was **mean batch-best yield**, which measures the quality of the recommendations being made throughout the shifting process.

On mean batch-best yield:

- `local_only` significantly outperformed `random`
- `local_only` significantly outperformed `global_naive`
- `global_residual` did not outperform `global_naive`
- `global_residual` was significantly worse than `local_only` for batch-best performance

This suggests that, under hard regime shifts, **current-regime relevance matters more than broad historical memory**.

### Ensemble tests

Additional tests explored two local/global-residual ensemble approaches:

- `local_gr_weighted`
- `local_gr_committee`

These ensemble approaches improved over some global baselines but did not outperform `local_only`.

| Method | Final cumulative best | Mean batch-best | Total information gain |
|---|---:|---:|---:|
| global_naive | `19.026 ± 2.545` | `9.187 ± 0.891` | `1.100 ± 0.885` |
| local_only | `19.577 ± 2.341` | `9.844 ± 1.019` | `1.177 ± 0.739` |
| global_residual | `19.589 ± 2.464` | `9.095 ± 0.533` | `1.298 ± 0.802` |
| local_gr_weighted | `18.681 ± 2.312` | `9.603 ± 0.733` | `1.309 ± 0.646` |
| local_gr_committee | `18.988 ± 2.418` | `9.450 ± 0.762` | `1.233 ± 0.575` |

The weighted ensemble significantly improved over `global_residual` and `global_naive` on batch-best yield, but it did not beat `local_only`.

This is an important prototype result: global memory appears potentially useful, but only when it is introduced carefully. Under hard drift, naive global transfer can dilute the relevance of the current regime.

### Interpretation

The current results support the basic premise of the project:

> In nonstationary process-improvement problems, adaptive DoE should be regime-sensitive.

The most reliable result so far is that a simple local regime-aware surrogate can outperform a stationary global surrogate on batch quality. The more complex methods are not yet clearly superior, but they point toward a promising next stage: adaptive weighting between local and global knowledge based on regime confidence and recent predictive validity.

The prototype does not yet prove that regime-aware adaptive DoE always finds a better final optimum. Instead, it suggests something more operationally useful:

> Regime-aware adaptive DoE can make better recommendations during unstable process evolution.

That distinction matters. In real process-development settings, the value of an adaptive DoE workflow is not only whether it eventually finds one high-performing condition. It is also whether it keeps recommending good, informative, and context-appropriate experiments as the process changes.

### Current limitations

Several limitations remain.

1. **Regime labels are still imperfect.**  
   The detector performs well enough to be useful, but detected regimes should not yet be treated as ground truth.

2. **Final cumulative best is not sufficiently sensitive.**  
   This metric can saturate early and obscure the quality of later recommendations.

3. **Local-only is currently hard to beat.**  
   More complex methods need better criteria for when global memory is useful versus harmful.

4. **The benchmark is still simulated.**  
   The current oracle is useful for controlled testing, but real process data will introduce messier drift, confounding, noise, missingness, and delayed feedback.

5. **Regime confidence is not yet fully integrated.**  
   The next version should move beyond hard regime assignment and use soft regime confidence or predictive-validity weighting.

6. **The current prototype does not yet prove value beyond recency filtering.**  
   A key next step is to compare regime-aware modeling against simpler baselines that use only recent data.
