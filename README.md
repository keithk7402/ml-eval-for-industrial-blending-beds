# Blending Bed ML & Simulation Pipeline
_A compact, reproducible pipeline to (1) generate data with a physics simulator, (2) train/optimise surrogate models and search predicted Pareto fronts, and (3) **re‑evaluate** those predicted fronts with the simulator._

Pipeline: **`dataset → model → front_reeval`**  
Notebooks (in this repo):
- `dataset.ipynb` — generate training data (`y1`, `y2`, `x1..x70`) using the BMH simulator.
- `Model_NSGA-II_pop_eval.ipynb` — optimise on **surrogate** objectives (ML models) to produce predicted fronts and a combined `front_ml.csv`.
- `02_front_reeval_sensitivity.ipynb` — re‑simulate the predicted fronts to obtain **ground‑truth** `y1_sim`, `y2_sim`, plus figures and summary tables.

> **Tip:** Use **relative paths** (defaults below) instead of hard‑coded absolute paths. The notebooks already expose a few config variables you can edit in the first cell.

---

## 1) What the objectives/variables mean
- **Objectives**
  - `y1`: volume‑weighted stdev of reclaimed quality (minimise).
  - `y2`: volume/height variability of reclaimed bed (minimise).
- **Decision variables**
  - `x1..x50`: the **quality curve** (q) over time/stacking (fixed during the model search for each material curve).
  - `x51..x70`: the **deposition X‑positions** (20 control points) to be optimised.

The dataset rows therefore look like:
```
y1, y2, x1, x2, ..., x50, x51, ..., x70
```

---

## 2) Project layout (recommended)
You can keep any structure, but these defaults match the notebooks:
```
.
├── dataset.ipynb
├── Model_NSGA-II_pop_eval.ipynb
├── 02_front_reeval_sensitivity.ipynb
├── Results/
│   ├── exp/                      # predicted fronts (surrogate/ML) written here
│   └── sim re-eval/
│       ├── fronts/               # re-evaluated copies of predicted fronts
│       ├── figs/                 # figures produced during re-eval
│       └── *.csv                 # summary tables/metrics
└── libraries/
    └── bmh/                      # local BMH simulator package (see below)
```

---

## 3) Environment setup

### Python
- Python **3.10+** recommended (tested on 3.11/3.12).
- Create and activate a virtual env (conda or venv). Example (venv):
  ```bash
  python -m venv .venv
  source .venv/bin/activate  # Windows: .venv\Scripts\activate
  pip install --upgrade pip
  ```

### Install Python packages
Install requirements (CPU versions are fine):
```bash
pip install -r requirements.txt
```
If `pymoo` is missing, `02_front_reeval_sensitivity.ipynb` will auto‑install it in‑notebook.

### Install the BMH simulator (local package)
The notebooks import `bmh.*` modules. If you have the simulator source in `libraries/bmh`, install it in editable mode:
```bash
pip install -e libraries/bmh
```
> If your BMH library lives elsewhere, either install it similarly or add its path to `PYTHONPATH`.

---

## 4) Running the pipeline

### Step A — Generate the dataset (`dataset.ipynb`)
- Purpose: simulate random `(q, x)` pairs and compute ground‑truth `(y1, y2)`.
- Output (default): `matrix_f1f2_variant_smooth.csv` in the repo root.
- Key knobs inside the notebook:
  - `N_SAMPLES`: number of synthetic samples (increase to scale dataset).
  - Bed geometry & flow constants (at the top of the generator cell).

**Run:** open the notebook and run all cells, or from CLI:
```bash
jupyter nbconvert --to notebook --execute dataset.ipynb --output dataset_executed.ipynb
```

### Step B — Optimize with surrogate models (`Model_NSGA-II_pop_eval.ipynb`)
- Purpose: train/evaluate surrogate models (MLP / XGBoost / LSTM) and **search** predicted Pareto fronts using NSGA‑II on the **predicted** objectives.
- Inputs:
  - A training CSV like `matrix_f1f2_200000.csv`.
- Outputs:
  - Per‑run predicted front CSVs in `Results/exp/` with columns:
    - `y1_pred, y2_pred, x1..x70, model, pop, evals, seed, curve_id`
  - A combined `front_ml.csv` (predicted fronts stitched together), used by Step C.
- Important config (edit in the “CONFIG” cell):
  - `RESULTS_DIR` (default `Results/exp`)
  - `CURVE_CSV` (path to quality curves, often `front_ml.csv` or another curves file)
  - `POPS`, `EVALS`, `SEEDS`, `N_CURVES`

**Run:** execute all cells. The notebook prints progress as it saves CSVs under `Results/exp/`.

### Step C — Re‑evaluate predicted fronts with the simulator (`02_front_reeval_sensitivity.ipynb`)
- Purpose: take the predicted fronts from Step B and compute **true** `(y1_sim, y2_sim)` by re‑simulating each solution with BMH.
- Config (top of the notebook):
  - `ML_FRONT_PATH` → the combined `front_ml.csv` from Step B.
  - `NSGA_OUTPUTS_DIR` → directory with predicted fronts (default `Results/exp`).
  - Output folders are created under `Results/sim re-eval/`:
    - `fronts/` — copies of predicted front files with added `y1_sim`, `y2_sim`.
    - `figs/` — comparison plots.
    - summary CSVs in `Results/sim re-eval/` (metrics, calibration, etc.).
- Key function: `resimulate_solution(q_vec, x_vec, ...)` — builds `Material` and `Deposition`, runs `BslBlendingSimulator`, and evaluates both objectives via `ReclaimedMaterialEvaluator`.

**Run:** execute all cells. You’ll see a summary like:
```
[done] wrote N files to Results/sim re-eval/fronts (updated U, skipped S)
```

---

## 5) Data & file conventions

### Training dataset (from Step A)
- CSV columns: `y1, y2, x1..x50 (q curve), x51..x70 (x deposition)`.

### Predicted fronts (from Step B)
- CSV columns:
  - `y1_pred, y2_pred`
  - `x1..x50` (fixed q curve used for this run)
  - `x51..x70` (optimized deposition control points)
  - `model` (`mlp` / `xgb` / `lstm`), `pop`, `evals`, `seed`, `curve_id`

### Re‑evaluated fronts (from Step C)
- Same as predicted fronts **plus** `y1_sim, y2_sim` columns.

---

## 6) Reproducibility & configuration
- Randomness is controlled via `SEEDS` in Step B. Use a fixed list (e.g. `[0,1,2]`).
- Use **relative paths**. If migrating from absolute paths, set:
  ```python
  BASE_DIR = Path(".").resolve()
  RESULTS_DIR = BASE_DIR / "Results" / "exp"
  ML_FRONT_PATH = BASE_DIR / "front_ml.csv"
  ```
- (see `requirements.txt`).

---

## 7) Troubleshooting

**Q: `ModuleNotFoundError: No module named 'bmh'`**  
A: Install the simulator package locally: `pip install -e libraries/bmh` (or point `PYTHONPATH` to your BMH source).

**Q: TensorFlow or XGBoost installation issues**  
A: Try CPU‑only first. For Apple Silicon, prefer `tensorflow-macos` and `tensorflow-metal` 

**Q: The notebooks reference `/Users/...` paths**  
A: Edit the config cell at the top of each notebook and switch to relative paths (see Section 6).

**Q: Re‑eval is slow**  
A: Reduce the number of fronts/files, or downsample per‑front solutions before re‑simulating. You can also parallelize `resimulate_solution` with `concurrent.futures` if needed.

---

## 9) License & citation
- Internal research code. If you use the BMH simulator or re‑use parts of this pipeline, please credit the original simulator authors as appropriate (jcbachmann/blending-simulation).

