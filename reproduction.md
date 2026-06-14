# Reproducing the TSAD Orchestra Experiments

This document provides a guide on how to reproduce the experiments for the TSAD Orchestra framework. It explains the structure of the benchmarking suite, the roles of various scripts, and provides step-by-step instructions.

## Overview

The benchmarking suite is designed to compare the performance and execution time of the agent-based anomaly detection solution ("agentic") against standard Time Series Anomaly Detection (TSAD) baselines (e.g., LOF, HBOS, iForest, PCA, Poly) and an ensemble approach.

The framework:
1. Randomly selects a subset of datasets (tables) from the database.
2. Runs the anomaly detection algorithms.
3. Calculates a comprehensive set of metrics (AUC-ROC, Precision, Recall, F-score, VUS-ROC, etc.) or execution times.
4. Stores the results directly into a PostgreSQL database in the `experiments` and `execution_time` tables.

## The `benchmark.py` Script

The entry point for running the experiments is the `benchmark.py` script located at the root of the repository. It provides a simple command-line interface to orchestrate the benchmarking process.

### Usage

```bash
python benchmark.py [OPTIONS]
```

### Options

**Performance Metrics Evaluation:**
- `--baselines`: Runs the standard baseline algorithms (LOF, HBOS, iForest, PCA, Poly).
- `--ensemble`: Runs the ensemble baseline.
- `--agentic`: Runs the main agent-based solution.
- `--agentic-no-validator`: Runs the agent-based solution without the validator (an ablation study).

**Execution Time Measurement:**
- `--time-baselines`: Measures execution time for the baseline algorithms.
- `--time-ensemble`: Measures execution time for the ensemble baseline.
- `--time-agentic`: Measures execution time for the agent-based solution.
- `--time-agentic-no-validator`: Measures execution time for the agent-based solution without the validator.

**Other Measurements:**
- `--token-usage`: Measures and records the agent's LLM token usage (must be used in conjunction with `--agentic` or `--agentic-no-validator`).
- `--all`: Runs all baselines, ensemble, agentic methods, and measures all execution times.

## Structure of the `src/benchmark/` Directory

The actual logic for each specific benchmark is stored in the `src/benchmark/` directory. `benchmark.py` calls the `main()` functions from these respective scripts.

- `configurations.py`: Contains global constants for the benchmarking suite, specifically the `RANDOM_SEED` (default: 42) to ensure reproducibility, and the `SUBSET_SIZE` (default: 45) to define how many datasets to sample.
- `run_baselines.py`: Evaluates standard baselines. It connects to the database, fetches the time series, runs the detectors, calculates the metrics, and stores the results in the `experiments` table.
- `ensemble.py`: Evaluates an ensemble baseline and stores the metrics.
- `agentic.py`: Evaluates the agent-based anomaly detection solution and stores the metrics.
- `agentic_no_validator.py`: Evaluates the agent-based solution without the validator and stores the metrics.
- `run_time_baselines.py`, `run_time_ensemble.py`, `run_time_agentic.py`, `run_time_agentic_no_validator.py`: These scripts are analogous to the ones above, but instead of calculating accuracy metrics, they measure the execution time and log the results to the `execution_time` table in the database.

## Output and Data Storage

The benchmarking suite automatically creates necessary tables in your connected database (defined by your `.env` or `secrets.toml`) and stores the results there. This allows for persistent, easily queryable results.

1. **`experiments` Table**: Stores performance metrics.
   - Columns: `dataset_name`, `method`, `auc_roc`, `auc_pr`, `precision`, `recall`, `f`, `precision_at_k`, `rprecision`, `rrecall`, `rf`, `r_auc_roc`, `r_auc_pr`, `vus_roc`, `vus_pr`, `affiliation_precision`, `affiliation_recall`, `created_at`.
2. **`execution_time` Table**: Stores runtime measurements.
   - Columns: `dataset`, `method`, `time`, `created_at`.

*Note: The scripts implement a caching mechanism. If an experiment for a given dataset and method already exists in the database, the script will skip computing it again. To truly re-run an experiment, you need to manually drop the rows or the tables from your database.*

## Step-by-Step Reproduction Instructions

1. **Environment Setup**
   Ensure your environment is set up according to the project's main `README.md`. You need your database running, and all dependencies installed via `uv`.

2. **Configure the Benchmark (Optional)**
   If you want to run on a different number of datasets or use a different seed, edit `src/benchmark/configurations.py`:
   ```python
   RANDOM_SEED = 42
   SUBSET_SIZE = 45
   ```

3. **Run the Full Suite**
   To reproduce all the performance metrics and execution time measurements in one go, simply run:
   ```bash
   python benchmark.py --all
   ```

4. **Run Specific Experiments**
   You can choose to run only parts of the suite. For example, to run just the baselines and the main agentic solution:
   ```bash
   python benchmark.py --baselines --agentic
   ```

   To measure execution times for the same:
   ```bash
   python benchmark.py --time-baselines --time-agentic
   ```

5. **Measure Token Usage**
   To measure how many tokens the agent is consuming:
   ```bash
   python benchmark.py --agentic --token-usage
   ```

6. **Viewing the Results**
   Connect to your database and query the tables to analyze the benchmarking output:
   ```sql
   SELECT method, AVG(auc_roc) as avg_auc, AVG(vus_roc) as avg_vus 
   FROM experiments 
   GROUP BY method;
   ```
