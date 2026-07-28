# CINN-Surgery-Scheduling

Repository for the paper **"Constraint-Informed Neural Networks for Three-Station Surgical Scheduling: A Primal-Dual Augmented Lagrangian Approach"** by Marcelo Becerra-Rozas, Carlos Valle, Nicolas Fuentes Espinoza, Alberto Garces-Jimenez, Jose J. Caro-Miranda, and Jose Manuel Gomez-Pulido.

## Overview

This repository implements a Constraint-Informed Neural Network (CINN) for the Three-Station Job Shop Scheduling Problem applied to hospital surgery planning. The model operates over three sequential stages -- Pre-operative (PRE), Operating Room (QX), and Post-operative recovery (POST) -- with four parallel rooms available at each stage (12 rooms total).

The CINN is trained by minimizing an Augmented Lagrangian that penalizes three families of KKT constraints: sequentiality (operations must follow stage order), maximum inter-stage waiting time (W_max = 25 min), and resource non-overlap (two surgeries cannot occupy the same room simultaneously). The ADMM algorithm alternates between primal descent on the network parameters (AdamW) and dual ascent on the Lagrange multipliers. After training, a Gumbel-Softmax decoding step produces a discrete solution, which is refined through a multi-objective Simulated Annealing post-processing (makespan + wait times + load imbalance).

The CINN is compared against four metaheuristic baselines -- Genetic Algorithm (GA), Discrete Particle Swarm Optimization (dPSO), Secretary Bird Optimization Algorithm (SBOA), and Discrete Mantis Shrimp Optimization Algorithm (dMShOA) -- all evaluated on the same discrete-event scheduling simulator under identical conditions.

## Experimental Instances

The benchmark evaluates all methods on real surgical data from Hospital Puerto Montt (Chile). Three dates were selected from the 2023 calendar, each with two instance sizes:

| Instance | Patients | Available after clinical filtering |
|----------|----------|------------------------------------|
| 2023-02-01 | 8 jobs | 28 |
| 2023-02-01 | 16 jobs | 28 |
| 2023-06-14 | 8 jobs | 16 |
| 2023-06-14 | 16 jobs | 16 |
| 2023-10-13 | 8 jobs | 23 |
| 2023-10-13 | 16 jobs | 23 |

Each instance is executed with N = 30 independent random seeds for each of the five algorithms, totaling 900 experimental runs. Results are compared via the Relative Percentage Deviation (RPD) from the Best-Known Solution (BKS) per instance, with paired Wilcoxon signed-rank tests at p < 0.05.

## Repository Structure

```
├── CINN-KKT-hospitals/         # CINN model + SA post-processing
│   ├── src/                    # model.py, trainer.py, constraints.py, post_processing.py
│   ├── data/
│   │   ├── figures/            # PNG + PDF figures (convergence, Gantt, histograms, boxplots)
│   │   └── processed/          # Generated schedules (CSV)
│   ├── main.py                 # Standalone CINN execution
│   ├── main_estadistica.py     # Statistical visualizations
│   └── requirements.txt
│
├── MH-hospitals/               # Metaheuristic baselines
│   ├── algorithms/             # ga.py, dpso.py, sboa.py, dmshoa.py
│   ├── simulation/             # Discrete-event scheduler + emergency generator
│   ├── config/                 # JSON-based parameter configuration
│   ├── results/                # Simulation outputs and plots per mode
│   ├── main.py
│   └── requirements.txt
│
├── benchmark_runner.py         # Unified benchmark (CINN + 4 MHs, 30 seeds)
├── metrics_analyzer.py         # Q1-table generation (RPD + Wilcoxon)
├── raw_results.csv             # 900-run benchmark dataset
├── results_summary_q1.csv      # Aggregated comparison table
├── LICENSE
└── README.md
```

## CINN-KKT-hospitals

The proposed method. A feed-forward neural network with three residual blocks (128 hidden units, Tanh activation) that maps job indices to continuous start times (via Softplus) and per-machine assignment probabilities (via Gumbel-Softmax). Training uses the Augmented Lagrangian formulation with ADMM, minimizing makespan while satisfying KKT constraints. After 8,000 training steps, the best-checkpoint model is decoded into a discrete schedule and refined by 3,000 iterations of multi-objective Simulated Annealing over machine reassignments within each stage.

**Parameters**: setup time = 30 min, cleanup time = 20 min, W_max = 25 min, 4 rooms per stage, AdamW (lr = 1e-3), Gumbel temperature 2.0 to 0.05 (linear decay), rho 100 to 300,000 (exponential growth).

## MH-hospitals

Four population-based metaheuristics for the same three-station problem. All algorithms use a solution encoding that combines a job priority sequence with room assignments and are evaluated via a common discrete-event simulator that enforces setup, cleanup, personnel assignment, and resource constraints.

| Algorithm | Type | Key mechanism |
|-----------|------|---------------|
| GA | Genetic Algorithm | OX1 crossover + swap mutation + elitism |
| dPSO | Discrete Particle Swarm | Sigmoid-based discrete position update |
| SBOA | Secretary Bird Optimization | Levy flights + two-phase exploration/exploitation |
| dMShOA | Discrete Mantis Shrimp | Sigmoid velocity discretization + PTI mixing |

Default configuration: 1,000 iterations, population/swarm size = 30.

## Installation

### CINN dependencies

```bash
cd CINN-KKT-hospitals
python -m venv .venv
.venv\Scripts\activate     # Windows
pip install -r requirements.txt
```

Requirements: PyTorch >= 2.0, Pandas, NumPy, Matplotlib, Seaborn.

### MH dependencies

```bash
cd MH-hospitals
python -m venv .venv
.venv\Scripts\activate     # Windows
pip install -r requirements.txt
pip install tabulate        # required by metrics_analyzer.py
```

Requirements: NumPy, SciPy, Pandas, Matplotlib, Joblib.

## Usage

### Run CINN standalone (single seed)

```bash
cd CINN-KKT-hospitals
.venv\Scripts\python main.py
```

### Run MH baselines standalone (50 Monte Carlo runs)

```bash
cd MH-hospitals
.venv\Scripts\python main.py
```

### Run full benchmark (CINN vs 4 MHs, 30 seeds x 6 instances)

```bash
CINN-KKT-hospitals\.venv\Scripts\python benchmark_runner.py
```

### Generate Q1 comparison table

```bash
MH-hospitals\.venv\Scripts\python metrics_analyzer.py
```

## Data Availability

The datasets generated and analysed during the current study are not publicly available because they consist of anonymized clinical-operational records provided by Hospital Puerto Montt Dr. Eduardo Schutz Schroeder under institutional data-use restrictions, but are available from the corresponding author on reasonable request and subject to authorization from Hospital Puerto Montt and the Servicio de Salud del Reloncavi.

## License

MIT License. See [LICENSE](LICENSE) for details.
