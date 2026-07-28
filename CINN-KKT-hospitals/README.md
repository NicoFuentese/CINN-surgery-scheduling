# CINN-KKT Hospital Surgery Scheduler

## Project Description

This project implements an automatic surgery planning system for hospitals using:

- **CINN (Constraint-Informed Neural Networks)**: Neural networks that incorporate operational constraints directly into the model
- **Lagrangian Optimization**: Dual variable method to enforce KKT constraints via ADMM
- **Post-processing with Simulated Annealing (SA)**: Refinement of solutions through multi-objective local search
- **Gumbel Softmax**: Differentiable relaxation of discrete decisions during training

### Key Features

- **Multi-Stage Optimization**: Handles 3 surgical stages (preoperative, operating room, postoperative)
- **Resource Management**: Intelligent room assignment with limited capacity (R = 4 rooms per stage)
- **Visualizations**: Detailed Gantt charts and idle-time histograms

---

## Project Structure

```
CINN-KKT-hospitals/
├── main.py                    # Main entry point
├── main_estadistica.py        # Statistical analysis and boxplots
├── requirements.txt           # Project dependencies
├── README.md
├── LICENSE                    # MIT License
│
├── src/                       # Core modules
│   ├── __init__.py
│   ├── data_loader.py         # Data loading and preprocessing
│   ├── data_cleaner.py        # Clinical filtering and tensor construction
│   ├── model.py               # CINN architecture (SchedulePINN)
│   ├── constraints.py         # KKT constraints and dual variables
│   ├── trainer.py             # ADMM training loop
│   ├── post_processing.py     # Decoding and Simulated Annealing
│   └── visualization.py       # Gantt charts and histograms
│
└── data/
    ├── raw/
    │   └── 2_dataset_procesado_actualizado.csv
    ├── processed/
    │   └── solucion_final_optimizada.csv
    └── figures/               # PDF and PNG figures (convergence, Gantt, histograms, boxplots)
```

---

## Requirements

- **Python 3.8+**
- **CUDA 11.0+** (recommended for GPU; CPU also works)

---

## Installation

### Create virtual environment

```bash
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
```

### Install dependencies

```bash
python -m pip install -r requirements.txt
```

---

## Usage

### Data

Place the file `2_dataset_procesado_actualizado.csv` in the `data/raw/` folder before running the project.

### Full Execution

```bash
python main.py
```

This command runs the complete pipeline:

1. **[1/5] Data Loading**: Reads and preprocesses the hospital CSV
2. **[2/5] Initialization**: Creates the CINN neural network
3. **[3/5] Training**: Trains with KKT constraints (8,000 steps)
4. **[4/5] Post-Processing**: Applies Simulated Annealing (3,000 iterations)
5. **[5/5] Report Generation**: Saves results and visualizations

### Configurable Parameters

In `main.py` and `src/trainer.py`:

```python
# main.py
target_date = '2023-02-01'  # Target date to analyze
num_samples = 16            # Number of patients

# trainer.py
MAX_STEPS = 8000            # Training iterations
rho_start = 100.0           # Initial ADMM penalty
tau_start = 2.0             # Initial Gumbel temperature
w_balance = 50.0            # Load balancing weight
```

---

## Generated Outputs

The program generates files in `data/processed/` and `data/figures/`:

### 1. **solucion_final_optimizada.csv**

Table with the optimized solution:

- `job_id`: Patient ID
- `stage_id`: Stage (0 = preoperative, 1 = operating room, 2 = postoperative)
- `global_machine_id`: Assigned room (1-12)
- `real_start`: Start time in minutes (setup begins)
- `surgery_start`: Start time in minutes (surgery begins)
- `real_end`: Medical end time in minutes (surgery ends)
- `machine_end`: Machine occupancy end time in minutes (includes cleanup)
- `dur_medical`: Duration of the intervention
- `dur_occupancy`: Total occupancy (intervention + setup/cleanup buffer)

### 2. **fig3_gantt.png / .pdf**

Gantt chart showing:

- Color-coded bars per patient
- Setup (yellow) and cleanup (coral) buffers
- Stage separator lines
- Patient labels (P{j}) for surgeries longer than 8 minutes

### 3. **fig4_wait_times.png / .pdf**

Histograms of inter-stage wait times with mean indicators (red dashed) and W_max threshold (black dashed).

### 4. Additional Statistical Figures

- **fig2_convergence.png / .pdf**: Primal-dual convergence curve (makespan vs. KKT violations)
- **fig5_hist_kde.png / .pdf**: Enhanced wait-time histogram with KDE
- **fig6_boxplot_util.png / .pdf**: Boxplot of resource utilization per stage
- **fig7_flow_time.png / .pdf**: Boxplot of real vs. ideal patient flow time

---

## Model Architecture

### CINN Neural Network (SchedulePINN)

```
Input: Patient ID (128-dim embedding)
    ↓
[3 Tanh layers with 128 neurons]
    ↓
Dual Output:
  ├─ Start Times: softplus(W) × 300 → [J, I]
  └─ Machine Probs: Gumbel Softmax(logits) → [J, I, R]
```

### Loss Function

$$\mathcal{L} = w_{obj} \cdot \text{makespan} + \mu^T \cdot g + \frac{\rho}{2} \|g_+\|^2 + w_{bal} \cdot \sigma(\text{loads})$$

Where:
- $g$: KKT constraint violations
- $\mu$: Lagrange multipliers (ADMM dual ascent)
- $\rho$: Penalty coefficient (increases with iterations)
- $\sigma$: Standard deviation of machine loads (load balancing)

### KKT Constraints

1. **Sequentiality**: $s_{j,i+1} \geq s_{j,i} + p_{j,i}$ for all j, i
2. **Maximum Wait Time**: $s_{j,i+1} - s_{j,i} - p_{j,i} \leq W_{max}$ with $W_{max} = 25$ min
3. **Resource Non-Overlap**: Prevents conflicts on shared machines (probabilistic over Gumbel-Softmax assignments)

Additionally, the discrete-event scheduler enforces setup time (30 min), cleanup time (20 min), setup overlap (up to 30 min), and stage ordering.

**Total KKT constraint vector**: $g \in \mathbb{R}^{4J+1}$ (65 constraints for $J = 16$ patients).

---

## Key Components

### `data_loader.py` / `data_cleaner.py`
- Null value removal
- Timestamp conversion
- Duration calculation
- Date-specific filtering
- Clinical sanity checks

### `model.py`
- SchedulePINN architecture definition
- Xavier Normal initialization
- Forward pass with Gumbel Softmax relaxation

### `constraints.py`
- KKT constraint vector construction
- Dual variables (ADMM method)
- Violation calculation

### `trainer.py`
- Alternating ADMM loop
- Dual variable update with adaptive frequency
- Temperature ($\tau$) and penalty ($\rho$) annealing
- Pareto-optimal model selection

### `post_processing.py`
- Topology extraction (machine decoding via argmax)
- Multi-objective Simulated Annealing (makespan + 0.15 × wait + 0.50 × imbalance)
- Stage-ordered scheduling with setup overlap

### `visualization.py`
- Gantt charts with setup/surgery/cleanup decomposition
- Inter-stage wait time histograms
- Convergence curves, statistical boxplots

---

## Technical References

- **Informed Neural Networks**: Family of Neural Networks informed by various sources of domain knowledge
- **Constraint-Informed Neural Networks**: Incorporate mathematical constraints into the learning objective
- **Gumbel Softmax**: Differentiable relaxation of discrete decisions (Maddison et al., 2017)
- **Job Shop Scheduling**: Base NP-hard problem of this project

---

## License

MIT License. See [LICENSE](LICENSE) for details.

---

## Contact

- nicolas.fuentes@pucv.cl
- marcelo.becerra@pucv.cl
- carlos.valle@pucv.cl

---

**Last updated**: July 2026
