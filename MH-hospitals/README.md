# MH-Hospitals: Metaheuristics for Surgery Scheduling

## Description

Metaheuristic framework for the **Three-Station Job Shop Scheduling (TSJS)** problem applied to hospital surgery planning. The system models three surgical stages with dynamic personnel assignment:

1. **APR** (Pre-operative) -- Anesthesiologists
2. **OR** (Operating Room) -- Surgeons
3. **ARR** (Post-operative recovery) -- Recovery physicians

### Implemented Algorithms

| Algorithm | Type | Description |
|-----------|------|-------------|
| **GA** | Genetic Algorithm | Population-based with crossover, mutation, and elitism |
| **dPSO** | Discrete Particle Swarm Optimization | PSO adapted to discrete space via sigmoid function |
| **SBOA** | Secretary Bird Optimization Algorithm | Nature-inspired, with Levy flights |
| **dMShOA** | Discrete Mantis Shrimp Optimization Algorithm | Sigmoid-based discrete decision |

### Simulation Modes

- **Elective mode**: Scheduled surgeries only
- **Emergency mode** (TSJS): Elective surgeries plus emergency arrivals, with dynamic rescheduling

---

## Project Structure

```
MH-hospitals/
├── main.py                     # Entry point
├── requirements.txt            # Dependencies
├── check_config.py             # Configuration validation
│
├── algorithms/                 # Metaheuristics
│   ├── ga.py                   # Genetic Algorithm
│   ├── dpso.py                 # Discrete PSO
│   ├── sboa.py                 # Secretary Bird Optimization
│   └── dmshoa.py               # Discrete Mantis Shrimp Optimization
│
├── config/                     # Configuration
│   ├── config.json             # Centralized parameters
│   ├── config.py               # Configuration loader
│   ├── config.quick.json       # Quick test configuration
│   └── algorithms_loader.py    # Dynamic algorithm loading
│
├── simulation/                 # Simulation engine
│   ├── scheduler.py            # Schedule construction and fitness
│   ├── dynamic_scheduler.py    # Dynamic scheduler for emergencies
│   └── emergency_generator.py  # Emergency surgery generator
│
├── core/                       # Orchestration
│   ├── simulation_runner.py    # Parallel execution (joblib)
│   ├── report_generator.py     # Statistical reports
│   └── file_manager.py         # Output directory management
│
├── workers/                    # Per-simulation workers
│   ├── elective_worker.py
│   └── emergency_worker.py
│
├── data/                       # Data
│   ├── data_loader.py          # Real data loading
│   ├── data_generator.py       # Synthetic data generation
│   └── real_surgeries_data.csv # Real dataset (>10 MB, excluded from repository)
│
├── utils/                      # Utilities
│   ├── statistics.py           # Statistical analysis
│   ├── reporting.py            # Report generation
│   ├── plotting.py             # Visualizations
│   └── logger.py               # Logging
│
├── results/                    # Generated results
│   ├── elective/
│   │   ├── csv/                # Per-algorithm results
│   │   └── plots/              # Gantt, convergence, histograms, boxplots
│   └── emergencies/
│       ├── csv/
│       └── plots/
│
└── legacy/                     # Legacy code
    └── main_old.py
```

---

## Requirements

- Python 3.8+
- Dependencies: NumPy, SciPy, Pandas, Matplotlib, Joblib

```bash
python -m venv .venv
.venv\Scripts\activate     # Windows
python -m pip install -r requirements.txt
```

---

## Usage

### Configuration

All parameters are set in `config/config.json`:

- `experiment.num_simulations`: Number of Monte Carlo runs (default: 50)
- `experiment.num_surgeries`: Number of surgeries per simulation (default: 16)
- `resources`: Rooms per stage (default: 4 APR, 4 OR, 4 ARR)
- `algorithms.*.enabled`: Enable/disable individual algorithms
- `emergencies.enabled`: Enable emergency mode (default: false)

### Execution

```bash
# Elective mode (no emergencies)
python main.py

# Emergency mode (set emergencies.enabled to true in config.json)
python main.py
```

### Generated Outputs

Under `results/elective/` (or `results/emergencies/`):

- **CSVs**: Best schedules, strategies, statistical analysis, summary
- **Gantt Charts**: Optimal schedule visualization per algorithm
- **Convergence**: Convergence curves per simulation
- **Boxplots**: Makespan comparison across algorithms
- **Histograms**: Wait-time distributions
- **Bar plots**: Execution time per algorithm

---

## Objective Function

$$\mathcal{L} = \alpha \cdot \sum s_{j,1} + \beta \cdot \sum w_j + \gamma \cdot \max(w_j) + \delta \cdot \sigma(\text{loads})$$

Where:
- $s_{j,1}$: Start time of surgery $j$
- $w_j$: Inter-stage waiting time
- $\sigma(\text{loads})$: Room load imbalance
- $\alpha, \beta, \gamma, \delta$: Configurable weights
