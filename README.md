# SGRT — Security Geometry and Reachability Theory for QKD Links

**A runtime layer that tells a quantum communication link how far it is from losing security, how likely it is to lose it soon, and whether that can still be prevented.**

[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Notebook](https://img.shields.io/badge/notebook-QKD__SGRT.ipynb-orange.svg)](QKD_SGRT.ipynb)

This repository contains the complete, executable implementation accompanying the paper:

> **Security Geometry as a Runtime Layer for Quantum Communication Infrastructure: Predicting and Preventing Finite-Key Collapse**
> Y. Y. Ghadi, A. Ajayan, I. Alreshidi, S. Guizani, H. Hamam
> *Submitted to Future Generation Computer Systems — Special Collection on Advances in Quantum Computing: Methods, Algorithms, and Systems (Vol. IV).*

---

## Why this exists

Finite-key security proofs answer a **binary** question: given this block of data, can a secret key be extracted or not? A control plane that must admit a job, place a workload on a remote QPU, size a key buffer, or reroute traffic cannot act on a flag that flips at the moment security is already gone.

SGRT supplies the three quantities a runtime actually needs, computed from statistics a QKD stack already produces:

| Question | Quantity | Units |
|---|---|---|
| How far from insecurity? | Geodesic security distance $D_\mathrm{sec}$ to the finite-key critical manifold | statistical σ (invariant) |
| How soon, how likely? | Geometric time-to-boundary $\tau_g$ and calibrated first-passage probability $P_H$ | blocks / probability |
| Can it be saved? | Viability class (SAFE / RECOVERABLE / IRRECOVERABLE) and Security Resilience Reserve (SRR) | class / statistical σ |

The framework is built in three layers: an **exact density-matrix observation model** of BB84 (no QBER proxies), a **composable finite-key margin** whose zero set is the collapse manifold, and the **Fisher information geometry** of the observed counts, which flattens exactly under an arcsine transform so the geodesic distance is available in closed form.

## Headline results

All values are produced by the notebook and re-verified automatically on every run.

| Result | Value |
|---|---|
| Episode detection at a 5 % false-alarm budget (fused SGRT) | **40.2 %** vs 6.8 % (QBER threshold), 1.5 % (Page–Hinkley), 0.8 % (CUSUM) |
| Horizon-risk discrimination (AUPRC, $H=30$) | **0.421** vs 0.244 best classical baseline |
| Calibration of the collapse forecast (ECE, after isotonic recalibration) | **0.015** |
| Equal finite-key margin, unequal risk (lowest margin bin) | **68.8 %** vs 44.2 % collapse rate by geometric distance |
| Certification ≠ resilience | matched-margin pair: one RECOVERABLE (SRR 10.4 σ), one IRRECOVERABLE |
| Online monitor cost | **8.1 µs** per link-block (p99 28.4 µs), 448 B state per link |
| Fleet cost | **0.39 µs** per link-block amortized → 10⁴ links at 1 block/s = **0.39 % of one core** |
| Evaluation scale | 630 stochastic trajectories, 7 drift families, disjoint train/calibration/test seeds |

Runtime figures were measured on a single core of an Intel Xeon @ 2.8 GHz (Python 3.11, NumPy 2.4) and are hardware-dependent; treat them as an upper bound on cost.

## Repository layout

```
.
├── QKD_SGRT.ipynb          # self-contained executable notebook (regenerates everything)
├── src/
│   ├── sgrt_core.py        # Layers A–C: channel model, finite-key margin, Fisher geometry
│   ├── sgrt_dynamics.py    # stochastic degradation families, predictors, classical baselines
│   ├── sgrt_eval.py        # train/calibration/test protocol, AUROC/AUPRC, alarms, calibration
│   ├── sgrt_runtime.py     # deployment layer: SGRTMonitor (O(1)) and FleetMonitor (vectorized)
│   └── sgrt_experiments.py # Experiments I–IX: all figures, tables, manifest, self-verification
├── outputs/SGRT/
│   ├── figures/            # 11 figures (PDF + PNG) as they appear in the paper
│   ├── tables/             # 12 CSV tables
│   └── others/
│       └── results_manifest.json   # every reported number, machine-readable
├── requirements.txt
├── LICENSE
└── README.md
```

## Quickstart

**Run everything (≈ 9 minutes on one core):**

```bash
git clone https://github.com/<org>/sgrt-qkd.git
cd sgrt-qkd
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute --inplace QKD_SGRT.ipynb
```

Or open `QKD_SGRT.ipynb` in Jupyter/Colab and run all cells — it detects Colab and mounts Drive automatically.

Everything lands in `outputs/SGRT/`. Nothing is copied by hand: the figures and tables in the paper are exactly these files.

## Reproducibility guarantees

Three mechanisms make the results self-checking rather than self-reported:

1. **Physical sanity gates.** Execution aborts unless 12 checks pass — ideal channel gives zero error, pure depolarization gives $Q_Z = Q_X = d/2$, pure dephasing gives basis asymmetry, full intercept–resend gives the textbook 25 % disturbance, density matrices stay positive with unit trace, the closed-form observables match the density-matrix computation to machine precision, and the collapse boundary moves correctly with block size.
2. **Fixed, disjoint seeds.** Engine seed 42; train / calibration / test trajectory populations use seed blocks 100 / 200 / 300. Alarm thresholds and probability recalibration are fitted on the calibration split only and never see test data.
3. **Automatic claim verification.** The final cell re-checks **48 claims** made in the paper (37 numerical with explicit tolerances, 11 structural) against the freshly computed manifest, and raises an `AssertionError` if the text and the computation ever diverge.

An independent re-run on different hardware reproduced all shared scientific values bit-for-bit; only the timing measurements differ, as expected.

## Using the runtime monitor in your own stack

The monitor consumes only the two error counts your QKD stack already produces during parameter estimation, in constant time and constant memory:

```python
from sgrt_runtime import SGRTMonitor

mon = SGRTMonitor(N_sift=200_000, t_test=0.25, H=30)   # protocol configuration

for k_Z, k_X in link_telemetry_stream():               # error counts per block
    s = mon.update(k_Z, k_X)
    # s = {'margin': 0.2532,      finite-key secret fraction
    #      'D_sec': 75.15,        statistical sigma to the collapse manifold
    #      'consumption': 2.03,   sigma consumed per block
    #      'tau_g': 42.66,        estimated blocks to collapse at current drift
    #      'P_H': 0.0}            calibrated probability of collapse within H blocks
```

For many links from one process, `FleetMonitor(K)` evaluates the identical mathematics as array operations:

```python
from sgrt_runtime import FleetMonitor
import numpy as np

fleet = FleetMonitor(10_000)
state = fleet.update(kZ_array, kX_array)     # one scheduling tick, ~3.9 ms
```

These outputs are designed to be consumed, not just displayed: size key buffers by `P_H` over the replenishment horizon, treat `D_sec` as an admission-control margin before placing work on a remote QPU, use `tau_g` as lead time for draining or migrating, and weight routes by `SRR`.

## Figure and table map

| Artifact | Content |
|---|---|
| `fig_framework` | SGRT pipeline: telemetry → margin → geometry → risk → intervention |
| `fig_landscape_margin` | Finite-key margin and critical manifolds for $N = 10^4 \dots 10^7$ |
| `fig_distance_landscape` | Fisher geodesic distance vs naive Euclidean distance |
| `fig_example_trajectories` | Representative degradation runs with $D_\mathrm{sec}$ and $P_H$ |
| `fig_roc_pr`, `fig_episode_benchmark` | Benchmark against CUSUM, EWMA, Page–Hinkley, critical slowing down |
| `fig_calibration` | Reliability of the collapse forecast before/after recalibration |
| `fig_matched_margin` | Equal margin, unequal risk (hypothesis H1) |
| `fig_viability` | Viability classes and the Security Resilience Reserve (hypothesis H2) |
| `fig_decoy` | Decoy-state BB84 generalization |
| `fig_runtime` | Monitor latency distribution and fleet scaling |
| `tables/*.csv` | Populations, benchmarks, alarms, horizon robustness, finite-key scaling, H2 pair, decoy, parameters, runtime |

## Requirements

Python ≥ 3.10 with `numpy`, `scipy`, `pandas`, `matplotlib`, `scikit-learn`, `jupyter`. No GPU, no quantum hardware, and no network access are required.

## Citation

```bibtex
@article{Ghadi2026SGRT,
  author  = {Ghadi, Yazeed Yasin and Ajayan, Amal and Alreshidi, Ibrahim
             and Guizani, Sghaier and Hamam, Habib},
  title   = {Security Geometry as a Runtime Layer for Quantum Communication
             Infrastructure: Predicting and Preventing Finite-Key Collapse},
  journal = {Future Generation Computer Systems},
  note    = {Special Collection on Advances in Quantum Computing:
             Methods, Algorithms, and Systems (Vol. IV), submitted},
  year    = {2026}
}

@misc{SGRTrepo,
  author = {Ghadi, Yazeed Yasin and Ajayan, Amal and Alreshidi, Ibrahim
            and Guizani, Sghaier and Hamam, Habib},
  title  = {{SGRT}: Security Geometry and Reachability Theory for {QKD} Links},
  year   = {2026},
  howpublished = {\url{https://github.com/<org>/sgrt-qkd}}
}
```

## Scope and limitations

The study is simulation-based. The channel model is exact but is a three-parameter idealization; the degradation families are synthetic, though deliberately diverse (including abrupt shifts and near-critical mean-reverting operation). The viability analysis instantiates one bounded-rate control structure under worst-case constant drift rather than a full viability-kernel computation. The first-passage forecast uses a locally Brownian reduction along the distance coordinate, corrected empirically by recalibration. Runtime measurements characterize the monitoring computation in a single process; they exclude telemetry transport and control-plane messaging. Validation on recorded field telemetry, a latent-state filtering layer, and end-to-end policy evaluation are the natural next steps.

## License

MIT — see [LICENSE](LICENSE).

## Contact

Corresponding author: Amal Ajayan — a_amal@cb.students.amrita.edu
Habib Hamam — Habib.Hamam@umoncton.ca
