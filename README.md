# SeCAP: Delay-Bounded Adaptive MAC for IEEE 802.15.4e DSME Networks

<div align="center">

![Python](https://img.shields.io/badge/Python-3.9%2B-3776AB?style=flat-square&logo=python&logoColor=white)
![NumPy](https://img.shields.io/badge/NumPy-1.24%2B-013243?style=flat-square&logo=numpy&logoColor=white)
![Matplotlib](https://img.shields.io/badge/Matplotlib-3.7%2B-orange?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)
![Status](https://img.shields.io/badge/Status-Published-brightgreen?style=flat-square)
![Venue](https://img.shields.io/badge/Venue-IEEE-00629B?style=flat-square&logo=ieee&logoColor=white)

**Official simulation code for the paper:**

*"Delay-Bounded Adaptive MAC for IEEE 802.15.4e DSME Networks: Enhancing Resilience under Bursty and Dynamic IoT Traffic"*

**Sonali Anand** *(lead author & repository maintainer)*, Alekhya Gorrela, Raziur Rahman, Nikumani Choudhury, Anakhi Hazarika, Dipamani Choudhury, Tamoghna Ojha

BITS Pilani, Hyderabad Campus · IIT (ISM) Dhanbad

[![Email](https://img.shields.io/badge/Email-sonalianand2406%40gmail.com-EA4335?style=flat-square&logo=gmail)](mailto:sonalianand2406@gmail.com)li
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=flat-square&logo=linkedin)](https://linkedin.com/in/sonali-anand-aa175a189)

*Supported by DST-SERB Startup Research Grant SRG/2023/002016*

</div>

---

## Overview

IEEE 802.15.4e DSME uses **CAP Reduction (CR)** to maximise GTS (Contention-Free Period) slots for data, but this static approach causes severe queue buildup and delay spikes under bursty or event-driven IoT traffic.

We propose **SeCAP (Selective CAP Preservation)** — a lightweight, standard-compliant adaptive MAC mechanism that:

- Monitors the CAP queue length at each coordinator every multi-superframe
- Switches between CR mode (high throughput) and NCR mode (low latency) based on a theoretically-derived delay threshold
- Communicates mode changes to all devices via the existing CAP Reduction flag in the IEEE 802.15.4e beacon — **no protocol modifications required**

---

## Key Results

### Packet Delay vs. Offered Load (Table I)

| Offered Load | Static DSME | ACR | DCR | **SeCAP (ours)** |
|---|---|---|---|---|
| 0.1 pkts/s | 30 ms | 28 ms | 25 ms | **22 ms** |
| 0.5 pkts/s | 80 ms | 70 ms | 60 ms | **50 ms** |
| 0.7 pkts/s | 140 ms | 115 ms | 90 ms | **80 ms** |
| 0.9 pkts/s | 200 ms | 150 ms | 120 ms | **100 ms** |

> **~50% delay reduction** at high load vs. static DSME.

### Throughput–Delay Trade-off (Table III, 50 nodes, high load)

| Scheme | Throughput | Delay |
|---|---|---|
| Static DSME (CR) | 95% | 200 ms |
| ACR | 90% | 150 ms |
| DCR | 93% | 120 ms |
| **SeCAP (ours)** | **92%** | **80 ms** |

> SeCAP nearly matches full CR throughput while delivering near-NCR latency — the best trade-off of any evaluated scheme.

---

## Problem Formulation

### Why Static CAP Reduction Fails

In DSME's CR mode, only the **first superframe** of each multi-superframe contains a CAP. With `MO−SO = 2`, this means 3 out of 4 superframes have no contention access at all. Under bursty traffic, pending CSMA/CA frames accumulate rapidly and cannot be served, leading to unbounded delay growth.

### M/M/1 Queue Delay Model

Each coordinator's CAP queue is modelled as an M/M/1 system:

```
Davg = 1 / (µ − λ)          (Eq. 1 — average total delay)

Lq   = ρ² / (1 − ρ)          (Eq. 2 — average queue length, ρ = λ/µ)

E[Wq] = λ / (µ(µ − λ))       (Eq. 3 — average queuing delay)
```

The **delay threshold** that triggers CAP preservation is derived analytically:

```
λthr = µ − 1/Dthr             (Eq. 2 from Section II-B)
```

When `λ > λthr`, the queue operates beyond the tolerable delay bound and the algorithm switches to NCR mode, effectively doubling `µ` by providing a CAP in every superframe.

### Worst-Case Delay Bound

```
Dmax ≈ Qth × Tsf
```

With `Tsf = 100 ms` and `Qth = 3` (optimal threshold), the worst-case delay is bounded at ~300 ms — a hard guarantee absent in ACR/DCR.

---

## Algorithm — Selective CAP Preservation (Algorithm 1)

```
Input:  Queue threshold Qth, current CAP queue length Q
Init:   CAP mode ← CR

for each beacon interval (every multi-superframe):
    if Q > Qth:
        CAP mode ← NCR          # Preserve CAP in ALL superframes
        Set CAP Reduction Flag in beacon ← 0
    else:
        CAP mode ← CR           # CAP only in first superframe
        Set CAP Reduction Flag in beacon ← 1

    Broadcast beacon with updated CAP Reduction Flag
    Apply mode for next multi-superframe
```

**Why this works:** The coordinator uses the standard IEEE 802.15.4e beacon's `CAP Reduction` bit — no new control messages, no changes to the frame format. All receiving nodes automatically reconfigure their superframe structure upon receiving the next beacon.

---

## Repository Structure

```
secap-dsme/
│
├── 📂 src/
│   ├── secap/
│   │   ├── algorithm.py            # Algorithm 1: SeCAP coordinator logic
│   │   └── queue_monitor.py        # CAP queue length tracking
│   ├── dsme/
│   │   ├── superframe.py           # Multi-superframe structure (BO, MO, SO)
│   │   └── beacon.py               # Beacon generation with CAP Reduction flag
│   ├── baselines/
│   │   ├── static_cr.py            # Static CAP Reduction (paper baseline)
│   │   ├── acr.py                  # Alternating CAP Reduction [Meyer et al. 2020]
│   │   └── dcr.py                  # Dynamic CAP Reduction [Meyer et al. 2020]
│   ├── network/
│   │   ├── topology.py             # Cluster-tree topology builder
│   │   └── traffic.py              # Poisson traffic + bursty event generator
│   └── metrics/
│       └── evaluator.py            # Delay, throughput, energy, overhead metrics
│
├── 📂 configs/
│   ├── default.yaml                # Simulation parameters
│   └── sensitivity.yaml            # Threshold T sweep configuration
│
├── 📂 results/
│   ├── figures/                    # Reproduced plots (Fig. 3a–3e from paper)
│   └── logs/                       # Raw simulation output (JSON)
│
├── 📂 notebooks/
│   ├── 01_mm1_delay_model.ipynb    # Analytical delay model walkthrough
│   ├── 02_algorithm_trace.ipynb    # Step-by-step SeCAP execution trace
│   ├── 03_results_reproduction.ipynb  # Reproduce all paper figures
│   └── 04_threshold_sensitivity.ipynb # Fig. 3d: optimal Qth analysis
│
├── 📂 docs/
│   └── CAP_REDUCTION_PRIMER.md     # Background on DSME CAP/CFP structure
│
├── run_simulation.py               # Main entry point — reproduces all 5 plots
├── requirements.txt
└── README.md
```

---

## Setup

```bash
git clone https://github.com/YOUR_USERNAME/secap-dsme.git
cd secap-dsme
pip install -r requirements.txt
```

**Requirements:** Python 3.9+, NumPy, Matplotlib, PyYAML, tqdm, SciPy

---

## Running Simulations

### Reproduce all paper figures (Fig. 3a–3e)

```bash
python run_simulation.py --mode full --save results/figures/
```

### Individual experiments

```bash
# (a) Packet delay vs. offered load
python run_simulation.py --mode delay_vs_load

# (b) Throughput vs. node density
python run_simulation.py --mode throughput_vs_nodes

# (c) Energy consumption comparison
python run_simulation.py --mode energy

# (d) SeCAP delay vs. threshold T (sensitivity analysis)
python run_simulation.py --mode threshold_sweep

# (e) Control overhead comparison
python run_simulation.py --mode overhead
```

### Custom configuration

```bash
python run_simulation.py \
  --config configs/default.yaml \
  --nodes 50 \
  --load 0.7 \
  --threshold 3 \
  --so 3 --mo 5 --bo 6
```

---

## Simulation Parameters

| Parameter | Value |
|-----------|-------|
| MAC standard | IEEE 802.15.4e DSME |
| Superframe Order (SO) | 3 |
| Multi-superframe Order (MO) | 5 |
| Beacon Order (BO) | 6 |
| Superframes per multi-superframe | 2^(MO−SO) = 4 |
| CAP slots per superframe | 8 (slots 1–8) |
| CFP/GTS slots per superframe | 7 (slots 9–15) |
| Offered load range | 0.1 – 0.9 pkts/s |
| Node count range | 10 – 90 |
| SeCAP threshold (optimal) | T = 3 |
| Traffic model | Poisson + bursty events |
| Baselines compared | Static CR, ACR, DCR |

---

## Theoretical Highlights

### The U-Shaped Threshold Curve (Fig. 3d)

The SeCAP delay vs. threshold T follows a U-shape with a minimum at T = 3:

- **T too low (T=1):** CAP preserved too often → needless CFP reduction → higher data delay
- **T = 3 (optimal):** CAP preserved exactly when needed → minimum delay ~70 ms
- **T too high (T≥4):** CAP preserved too late → backlog builds → delay rises to ~95–120 ms

This analytically-guided threshold selection distinguishes SeCAP from heuristic approaches like ACR/DCR.

### Why SeCAP Outperforms DCR

DCR adjusts the *number of CAP slots* per superframe; SeCAP adjusts the *number of superframes containing a CAP*. This coarser, beacon-aligned adaptation is more efficient because:

1. It is fully encoded in the existing CAP Reduction flag — zero overhead
2. Mode changes take effect at the next beacon — one multi-superframe latency
3. The delay bound is provable from the M/M/1 analysis, not just empirical

---

## Baselines Implemented

| Scheme | Source | Description |
|--------|--------|-------------|
| Static DSME (CR) | IEEE 802.15.4-2020 | CAP only in first superframe |
| ACR | Meyer et al., AdHoc-Now 2020 | Alternates CR/NCR every multi-superframe |
| DCR | Meyer et al., AdHoc-Now 2020 | Dynamically adjusts CAP slots by traffic estimate |
| **SeCAP** | **This work** | Queue-monitored, delay-bounded CAP preservation |

---

## Citation

If you build upon this work, please cite:

```bibtex
@inproceedings{anand2025secap,
  author    = {Anand, Sonali and Gorrela, Alekhya and Rahman, Raziur
               and Choudhury, Nikumani and Hazarika, Anakhi
               and Choudhury, Dipamani and Ojha, Tamoghna},
  title     = {Delay-Bounded Adaptive {MAC} for {IEEE} 802.15.4e {DSME} Networks:
               Enhancing Resilience under Bursty and Dynamic {IoT} Traffic},
  booktitle = {[Conference Name]},
  year      = {2025},
  note      = {Supported by DST-SERB Grant SRG/2023/002016}
}
```

---

## Authors

### Lead Author & Maintainer

**Sonali Anand**
MTech, Computer Science & Information Systems
BITS Pilani, Hyderabad Campus

[![Email](https://img.shields.io/badge/Email-sonalianand2406%40gmail.com-EA4335?style=flat-square&logo=gmail)](mailto:sonalianand2406@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=flat-square&logo=linkedin)](https://linkedin.com/in/sonali-anand-aa175a189)

### Co-Authors

**Alekhya Gorrela** · BITS Pilani, Hyderabad Campus

**Raziur Rahman** · BITS Pilani, Hyderabad Campus
l
**Nikumani Choudhury** · BITS Pilani, Hyderabad Campus *(Supervisor)*

**Anakhi Hazarika** · BITS Pilani, Hyderabad Campus

**Dipamani Choudhury** · IIT (ISM) Dhanbad

**Tamoghna Ojha** · IIT (ISM) Dhanbad

---

## Related Repository

This work extends the PSO-based parameter tuning framework from our earlier paper:

> *"Improving Network Efficiency in Clustered Tree Topology through PSO Optimization in IEEE 802.15.4-DSME based IoT Networks"* — [[Repository](https://github.com/SonaliAnand24/dsme-pso)]

Both repositories share the same DSME cluster-tree simulation infrastructure and are part of a broader research effort on **adaptive MAC-layer optimisation for industrial IoT**.

---

## Acknowledgement

This work is supported by the **Science and Engineering Research Board, Department of Science and Technology, Government of India** through the Startup Research Grant under Grant **SRG/2023/002016**.

---

<div align="center">
<sub>IEEE 802.15.4e · DSME · CAP Reduction · Adaptive MAC · Bursty IoT · M/M/1 Queuing · Wireless Sensor Networks</sub>
</div>
