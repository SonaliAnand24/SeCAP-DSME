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

**Sonali Anand**, Alekhya Gorrela, Raziur Rahman, Nikumani Choudhury, Anakhi Hazarika, Dipamani Choudhury, Tamoghna Ojha

BITS Pilani, Hyderabad Campus · IIT (ISM) Dhanbad

*Supported by DST-SERB Startup Research Grant SRG/2023/002016*

</div>

---

## Abstract

IEEE 802.15.4e DSME's static CAP Reduction (CR) mechanism improves channel efficiency by minimising Contention Access Periods, but fails under bursty or dynamic IoT traffic — causing queue buildup, high latency, and degraded responsiveness.

We propose **SeCAP (Selective CAP Preservation)**, a delay-aware adaptive MAC algorithm that monitors CAP queue occupancy at each coordinator and dynamically toggles between CR and non-CR (NCR) modes. When the queue length exceeds a threshold `Qth`, SeCAP restores CAPs across all superframes; when traffic is light, it re-enables CR to maximise CFP throughput. We derive a delay bound analytically using M/M/1 queueing theory and validate against Static DSME, ACR, and DCR baselines.

---

## Key Results

| Scheme | Throughput (%) | Avg Delay @ High Load (ms) |
|--------|---------------|--------------------------|
| Static DSME (CR) | 95% | 200 ms |
| ACR | 90% | 150 ms |
| DCR | 93% | 120 ms |
| **SeCAP (ours)** | **92%** | **80 ms** |

> 50 nodes, high load. SeCAP reduces delay by **50%** vs. static CR with only 1–2% throughput penalty vs. DCR.

---

## Problem Statement

Static DSME CAP Reduction allocates a CAP only in the first superframe of each multi-superframe. This maximises CFP (data) slots but creates a bottleneck for contention-based traffic during bursts:

```
Static CR (BO=6, MO=5, SO=3):
┌─────────────────────────────────────────────────────────────────┐
│ SF1: [BS][CAP][CFP] │ SF2: [BS][CFP] │ SF3: [BS][CFP] │ SF4: [BS][CFP] │
└─────────────────────────────────────────────────────────────────┘
      ↑ Only 1 CAP per multi-superframe
      → Queue builds up under bursty load → high delay

SeCAP NCR mode (triggered when Q > Qth):
┌──────────────────────────────────────────────────────────────────────┐
│ SF1: [BS][CAP][CFP] │ SF2: [BS] [CAP] [CFP] │ SF3: [BS][CAP][CFP] │ ... │
└──────────────────────────────────────────────────────────────────────┘
      ↑ CAP in every superframe → backlog cleared rapidly
```

---

## Analytical Model

### M/M/1 Queue (Equations 1–4 from paper)

Each coordinator's CAP queue is modelled as an M/M/1 queue (arrival rate λ, service rate µ):

```
D_avg  = 1 / (µ - λ)                   [Eq. 1 — average total delay]
L_q    = ρ² / (1 - ρ),   ρ = λ/µ      [Eq. 2 — average queue length]
E[W_q] = λ / (µ(µ - λ))               [Eq. 3 — average waiting time]
D_ub   = 1 / (µ - λ)                   [Eq. 4 — worst-case delay upper bound]
```

### Delay Threshold Derivation

Given maximum acceptable delay `D_thr`, the critical arrival rate threshold is:

```
λ_thr = µ - 1/D_thr
```

When λ > λ_thr (equivalently Q > Qth), SeCAP switches to NCR mode, effectively increasing µ by providing a CAP in every superframe. This drives ρ back below 1 and restores bounded delay.

### Worst-Case Delay Bound

```
D_max ≈ Qth × T_sf
```

Example: `Qth = 10`, `T_sf = 100 ms` → `D_max ≈ 1 s`. The threshold `Qth` is therefore a direct design knob for latency SLAs.

---

## SeCAP Algorithm

**Algorithm 1 — Selective CAP Preservation (runs at each Coordinator)**

```python
# Input:  Qth — queue threshold derived from M/M/1 model
# Init:   CAP mode = CR

for each beacon interval (every multi-superframe):
    Q = measure_cap_queue_length()

    if Q > Qth:
        mode = NCR                  # CAP in every superframe
        beacon.cap_reduction = 0    # standard beacon flag
    else:
        mode = CR                   # CAP only in first superframe
        beacon.cap_reduction = 1

    broadcast_beacon(mode)
    apply_mode_next_multisuperframe(mode)
```

Key properties:
- **Standard-compliant** — reuses the existing CAP Reduction flag in the beacon's Superframe Specification; no new messages or fields
- **Lightweight** — O(1) per beacon interval, no global coordination needed
- **Delay-bounded** — `Qth` is analytically derived, not heuristically tuned
- **Only scheme with a formal delay guarantee** among all compared approaches

---

## Repository Structure

```
secap-dsme/
│
├── 📂 src/
│   ├── secap/
│   │   ├── coordinator.py          # SeCAP logic: queue monitor + CR/NCR toggle
│   │   ├── queue_model.py          # M/M/1 model (D_avg, L_q, E[Wq], λ_thr)
│   │   └── threshold.py            # Qth derivation from D_thr and µ
│   ├── dsme/
│   │   ├── superframe.py           # Multi-superframe timing model
│   │   └── mac_layer.py            # CAP/CFP slot accounting
│   ├── baselines/
│   │   ├── static_dsme.py          # Static CAP Reduction baseline
│   │   ├── acr.py                  # Alternating CAP Reduction
│   │   └── dcr.py                  # Dynamic CAP Reduction
│   ├── network/
│   │   └── topology.py             # Cluster-tree topology + Poisson traffic
│   └── metrics/
│       └── evaluator.py            # Delay, throughput, energy, overhead
│
├── 📂 configs/
│   ├── default.yaml                # Default simulation parameters
│   └── sensitivity.yaml            # Threshold sensitivity sweep
│
├── 📂 results/
│   ├── figures/                    # Reproduced Fig. 3a–3e
│   ├── logs/                       # Raw JSON output
│   └── tables/                     # CSV of Tables I–III
│
├── 📂 notebooks/
│   ├── 01_mm1_delay_analysis.ipynb
│   ├── 02_secap_vs_baselines.ipynb
│   ├── 03_threshold_sensitivity.ipynb
│   └── 04_energy_overhead.ipynb
│
├── run_simulation.py               # Main entry point
├── requirements.txt
└── README.md
```

---

## Setup

```bash
git clone https://github.com/SonaliAnand24/secap-dsme.git
cd secap-dsme
pip install -r requirements.txt
```

---

## Running Simulations

```bash
# Reproduce all paper figures at once
python run_simulation.py --mode full --save results/figures/

# Fig. 3a — delay vs. offered load
python run_simulation.py --mode delay_vs_load

# Fig. 3b — throughput vs. node density
python run_simulation.py --mode throughput_vs_nodes

# Fig. 3c — energy per node
python run_simulation.py --mode energy

# Fig. 3d — SeCAP delay vs. threshold T
python run_simulation.py --mode threshold_sensitivity

# Fig. 3e — control overhead
python run_simulation.py --mode overhead
```

---

## Simulation Parameters

| Parameter | Value |
|-----------|-------|
| BO / MO / SO | 6 / 5 / 3 |
| Superframes per multi-SF | 4  (= 2^(MO−SO)) |
| CAP slots per superframe | 8 |
| CFP (GTS) slots | 7 |
| Traffic model | Poisson, λ = 0.1–0.9 pkts/s |
| Node range | 10–90 nodes |
| Optimal threshold T | 3 |
| Superframe duration T_sf | 100 ms |

---

## Baseline Comparison

| Scheme | CAP Policy | Delay Guarantee | Standard-Compliant |
|--------|-----------|----------------|-------------------|
| Static DSME (CR) | Fixed — 1 CAP per multi-SF | ✗ | ✓ |
| ACR | Alternates CR / NCR each multi-SF | ✗ | ✓ |
| DCR | Heuristic CAP slot count adjustment | ✗ | ✓ |
| **SeCAP (ours)** | Queue-triggered CR ↔ NCR | **✓ (M/M/1 bound)** | ✓ |

SeCAP is the **only** scheme with a formal, analytically derived delay bound.

---

## Related Work in This Series

| Repository | Contribution |
|-----------|-------------|
| **[secap-dsme](https://github.com/SonaliAnand24/secap-dsme)** ← *this repo* | Delay-bounded adaptive CAP management via SeCAP |
| **[dsme-pso](https://github.com/SonaliAnand24/dsme-pso)** | PSO-based multi-superframe parameter optimisation |

---

## Citation

If you build upon this work, please cite:

```bibtex
@inproceedings{anand2025secap,
  author    = {Anand, Sonali and Gorrela, Alekhya and Rahman, Raziur
               and Choudhury, Nikumani and Hazarika, Anakhi
               and Choudhury, Dipamani and Ojha, Tamoghna},
  title     = {Delay-Bounded Adaptive {MAC} for {IEEE} 802.15.4e {DSME}
               Networks: Enhancing Resilience under Bursty and Dynamic
               {IoT} Traffic},
  booktitle = {[ANTS2025]},
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
[![GitHub](https://img.shields.io/badge/GitHub-SonaliAnand24-181717?style=flat-square&logo=github)](https://github.com/SonaliAnand24)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=flat-square&logo=linkedin)](https://linkedin.com/in/sonali-anand-aa175a189/)

### Co-Authors

**Alekhya Gorrela** · BITS Pilani, Hyderabad Campus

**Raziur Rahman** · BITS Pilani, Hyderabad Campus

**Nikumani Choudhury** · BITS Pilani, Hyderabad Campus *(Supervisor)*

**Anakhi Hazarika** · BITS Pilani, Hyderabad Campus

**Dipamani Choudhury** · IIT (ISM) Dhanbad

**Tamoghna Ojha** · IIT (ISM) Dhanbad

---

## Acknowledgement

This work is supported by the **Science and Engineering Research Board, Department of Science and Technology, Government of India** through the Startup Research Grant under Grant **SRG/2023/002016**.

---

<div align="center">
<sub>IEEE 802.15.4e · DSME · Adaptive MAC · CAP Reduction · Bursty IoT Traffic · M/M/1 Queueing · Resilient Protocols</sub>
</div>
