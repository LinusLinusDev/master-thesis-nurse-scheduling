# Nurse Scheduling: Transitioning from Manual to Automated Rostering

This repository contains the Mixed-Integer Linear Programming (MILP) implementation described in the Master's thesis:  
> **"Nurse Scheduling: Modelling the Transition from Manual to Automated Rostering under Fairness and Employee Satisfaction Considerations"** > *Author: Linus Palm (University of Cologne)*

The primary objective of this project is to mathematically model and facilitate the structural transition from traditional, manual shift planning to automated algorithmic rostering. For a comprehensive breakdown of the underlying mathematical model, constraints, and experimental results, please refer to the full thesis text.

---

## Overview & Key Features

Using this optimization tool, healthcare facilities can generate high-quality monthly shift schedules that are:
* **Historically Consistent:** The model seamlessly integrates data from the previous month to ensure continuous constraint validation (e.g., resting periods, consecutive night shift limits) across scheduling boundaries.
* **Optimized for Fairness:** Workload distribution is balanced by minimizing the maximum positive deviation in working hours across the entire staff roster.
* **Ergonomically Sound:** Employee satisfaction is prioritized by maximizing the fulfillment of individual shift preferences, maximizing block schedules (continuity), and minimizing critical late-to-early shift transitions (short rests).
* **Highly Constrained:** Fulfills a wide array of hard legal, contractual, and institutional requirements (e.g., staffing demands, minimum rest periods, weekend regulations).

---

## Model Scope & Core Assumptions

To capture the operational reality of the ward, the optimization model is designed around the following structural parameters:
* **Three-Shift System:** The model operates on a standard three-shift rotation consisting of Early (`E`), Late (`L`), and Night (`N`) shifts.
* **Fully Qualified Personnel Only:** The scope is limited to fully qualified nursing staff; trainees and apprentices are explicitly excluded from the model.
* **Deterministic Demand:** Staffing requirements are entirely deterministic, meaning minimum and maximum staffing levels are fixed and known for every shift and day.
* **Fixed Monthly Horizon:** The planning horizon is strictly set to exactly one calendar month per optimization run.

---

## Technical Stack

The core optimization pipeline is built entirely in Python and utilizes industry-grade mathematical programming solvers:

* **Language:** Python `3.10.18`
* **Mathematical Solver:** Gurobi Optimizer `13.0.1` (utilizing native lexicographic multi-objective capabilities)
* **Data Manipulation:** pandas `2.3.3`

---

## Input Configuration Files (`.xlsx`)

The instance data is structured across nine specialized Excel configuration files. To maintain a clean separation of concerns, these files are grouped into global parameters, workforce profiles, demand requirements, and temporal constraints.

> ⚠️ **Data Protection Note:** To comply with strict data protection and privacy requirements, all original real-world hospital data has been completely omitted. The Excel files provided in this repository are populated entirely with **synthetic dummy data** for demonstration and presentation purposes.  
>   
> **Example Scenario:** The provided dummy dataset is configured to generate a complete monthly shift schedule for **June 2026** while ensuring full historical compatibility with a given manual roster from **May 2026**.

### 1. Global Framework & Institutional Rules
* **`rules.xlsx`** Contains global scheduling parameters and policy constraints that apply uniformly across the entire ward.
    * `first_monday`: The zero-based index of the first Monday within the current scheduling horizon.
    * `first_weekend`: Determines the baseline weekend rotation sequence (`0` if the first weekend corresponds to Group A, `1` for Group B).
    * `B_n`: The maximum number of night shift blocks a single nurse is allowed to work per month.
    * `L_n`: The minimum number of mandatory rest days required immediately following a night shift block.
    * `W_r`: The length of the rolling time window (in days) used for evaluating specific shift rotation patterns.
    * `V_n`: The maximum number of weekend night shifts permitted per nurse within the month.
    * `A_r`: The maximum number of allowed planned absences within the defined rotation time window.
    * `Dist_n`: The minimum required spacing (in days) between the start dates of two separate night shift blocks.
    * `alpha_vac`: The absolute number of working hours credited toward a nurse's monthly target hours for each day of paid vacation.
    * `alpha_nc`: The absolute number of working hours credited toward a nurse's monthly target hours for each non-clinical workday.
* **`shifts.xlsx`** Defines the baseline properties of the shift roster, specifically mapping each standard shift type (Early `E`, Late `L`, Night `N`) to its precise duration in hours.

### 2. Workforce Profiles & Individual Contracts
* **`working_hours.xlsx`** Configures the contractual capacity of the workforce. It establishes the baseline weekly target hours for a 100% full-time equivalent (FTE) position and defines the individual employment percentage (e.g., 50%, 75%, 100%) for each nurse on the payroll.
* **`preferences.xlsx`** Stores static, nurse-specific constraint overrides and soft scheduling preferences:
    1. *General Shift Preferences:* An ordinal score from `1` (least preferred) to `3` (most preferred) assigned by each nurse to the standard shift types (`E`, `L`, `N`).
    2. *`max_cons_s`:* The maximum number of consecutive working days allowed for the individual before a mandatory day off.
    3. *`max_cons_e` / `max_cons_l` / `max_cons_n`:* Shift-specific limits for consecutive assignments of the same type. If left blank, the global `max_cons_s` parameter serves as the fallback threshold.
    4. *`nights`:* A binary indicator (`1`) specifying if a nurse explicitly requests to be assigned the maximum permissible volume of night shifts.
    5. *`weekend`:* Assigns the employee to a fixed weekend rotation pattern (`0` for Weekend Group A, `1` for Weekend Group B, or left blank if the employee is fully flexible).

### 3. Deterministic Staffing Demand
* **`demand_min.xlsx`** A daily matrix defining the hard lower bound for staffing requirements. It specifies the minimum number of qualified nurses required to be on duty for each shift type on every individual calendar day of the planning period.
* **`demand_max.xlsx`** A daily matrix defining the upper bound for staffing capacity, specifying the maximum number of nurses that may be allocated to a given shift type per day to prevent overstaffing.

### 4. Temporal Context & Roster Overrides
* **`previous.xlsx`** Provides the historical roster data from the preceding month. This file maps the full assignment history—including active shifts (`E`, `L`, `N`), regular days off (`X`), paid vacations (`U`), and non-clinical tasks (`Z`) such as administrative duties or continuing education—to ensure full mathematical continuity across month boundaries.
* **`planned.xlsx`** Acts as a hard constraint override file for the current planning period. It pre-assigns fixed schedules before the optimization run, predominantly tracking pre-approved paid vacations (`U`) and non-clinical workdays (`Z`), while retaining the flexibility to lock in specific active shifts or mandatory days off.
* **`wishes.xlsx`** A dynamic input interface where staff members can submit date-specific soft requests for the current month, capturing individual "wish shifts" or requested "wish days off" to be processed by the multi-objective optimization layers.

---

## Output Files (`.xlsx`)

Once the optimization pipeline completes, the solver exports the results into two structured Excel spreadsheets located in the output directory.

* **`out.xlsx`** The primary deliverable of the optimization engine. It contains the final computed roster for the current planning period, fully populated with active shift assignments (`E`, `L`, `N`), regular days off (`X`), paid vacations (`U`), and non-clinical workdays (`Z`). Beyond the schedule itself, it includes two key analytical summaries:
    * *Overtime Tracking:* Displays individual overtime or undertime balances for every nurse, automatically calculated based on the actual scheduled hours versus their individual contractual monthly target hours.
    * *Staffing Level Evaluation:* Aggregates and displays the exact number of nurses deployed per shift type for each individual day, allowing immediate verification against the minimum and maximum demand thresholds.
* **`wishes_out.xlsx`** An evaluation and transparency report that audits the model's performance regarding employee satisfaction. For every employee and calendar day, it indicates whether individual preferences were successfully accommodated by the multi-objective optimization layers:
    * `Y` (*Yes*): The nurse's soft preference or wish day off was fully respected and scheduled.
    * `N` (*No*): The request could not be fulfilled due to higher-priority operational constraints or fairness goals.
    * `X` (*None*): No specific preference or wish was submitted by the nurse for this particular date.

---

## Getting Started & How to Run

Executing the optimization model requires a working Python environment, an active Gurobi solver installation, and properly formatted input configuration files.

### 1. Prerequisites & Dependencies

Ensure your environment meets the following technical requirements:
* **Python:** `3.10.x` (Tested on `3.10.18`)
* **Libraries:** `pandas` (Tested on `2.3.3`) and `jupyter` (to execute the notebook environment).
* **Solver:** Gurobi Optimizer (Tested on `13.0.1`) with a valid license (e.g., a free Gurobi Academic License).

You can install the required Python packages via pip:
```bash
pip install pandas jupyter
```

### 2. Roster Configuration

Before running the model, make sure all `.xlsx` configuration files are set up correctly according to your planning horizon (e.g., configuring requirements, preferences, and the historical schedule from the previous month).

### 3. Execution via Jupyter Notebook

The entire optimization workflow—from data ingestion and mathematical modeling via Gurobi's Python API to exporting the final spreadsheets—is contained within a single interactive notebook.

1. Open your terminal or command prompt in the repository directory.
2. Launch the Jupyter environment:
   ```bash
   jupyter notebook
   ```
3. Open **`MILP.ipynb`**.
4. Run the notebook **cell by cell** from top to bottom.

### 4. Performance & Computational Runtime

Mathematical optimization via Mixed-Integer Linear Programming is computationally intensive. The time required to find an optimal solution heavily depends on:
* The available **hardware** specifications.
* The complexity and size of the **solution space** (e.g., number of active nurses, density of constraints, and total number of scheduling conflicts).

To ensure practical usability and prevent infinite loops in highly constrained or large-scale problem instances, a solver **timeout of 5 minutes (300 seconds)** is pre-configured in the Gurobi parameters within the notebook. Gurobi will return the best feasible solution found within this time limit.