
# Pregel‑style Fraud Detection — TrustRank on Payment Graphs (Python, Notebook)

This project implements a **Pregel‑style** message‑passing engine (pure Python, threads) to compute a **TrustRank**‑like score on a directed, weighted **payment graph**.  

---

## Repository layout

```
/Pregel based Fraud detection
├── Pregel-based-Fraud-detection.ipynb        # Working notebook
└── Pregel-based-Fraud-detection.pdf                 # Assignment brief (reference)
```
Paths and file names above come from your submitted materials; open the notebook to run the full pipeline. 

---

## Data contracts (required columns)

The notebook reads two Excel files (use **exact** column names):

**1) Payments graph — `Payments.xlsx`**
- `Sender` — string account ID  
- `Receiver` — string account ID  
- `Amount` — numeric; non‑numeric rows are skipped during graph build

**2) Seed set — `bad_sender.xlsx`**
- `Bad Sender` — string account ID (known fraud “seeds”)  

> In the notebook, the paths are hard‑coded for Colab (`/content/Payments.xlsx`, `/content/bad_sender.xlsx`). Update them to e.g. `data/Payments.xlsx`, `data/bad_sender.xlsx` for local runs. 

---

## Graph model & initialization 

- **Vertices** = accounts.  
- **Directed edges** = payments `Sender → Receiver` with **weight = Amount** (normalized per sender when sending messages).  
- Each vertex holds: `out_edges`, `value` (TrustScore), `bias` (seed prior), message buffers, and an `active` flag.  
- **Initialization:** if a vertex `p` is in the fraudulent seed set, `value = bias = 1/|seeds|`; otherwise `value = bias = 0`. 

---

## Algorithm 

Per vertex, per superstep:

```
t_new(p) = (1 - α) * d(p) + α * Σ_incoming
```
- **Damping** `α = 0.85` (configurable in the notebook).  
- **Deactivation:** vertex becomes inactive when `|t_new − t_old| < ε`, with `ε = 0.001`. 

**Message generation:** from sender `q` to each out‑neighbor `p`

```
msg(q→p) = t(q) * (w_qp / Σ_out_weights(q))
```
where `w_qp = Amount(q→p)` and the denominator is the **sum of outgoing weights** for `q`. 

---

## Pregel engine (threaded simulation)

- **Workers:** `num_workers = 4` threads.  
- **Partitioning:** `hash(vertex.id) % num_workers`.  
- **Superstep loop:** parallel **compute**, then **message exchange**, then **reactivation** of receivers; stop when converged or `superstep ≥ 100`.  


---

## Output & fraud categorization

After convergence, the notebook assembles:

```python
output_data = [{'AccountID': v.id, 'TrustScore': v.value} for v in graph.values()]
df_output = pd.DataFrame(output_data)
```

- **Redistribution step:** any residual trust mass is redistributed uniformly across seed vertices to keep mass consistent.  
- **Thresholding:** define **outlier_threshold = 90th percentile** of `TrustScore`.  
- **Classes:**  
  - *Known Fraud Sender* — in the seed set  
  - *Discovered Fraud Sender* — top decile not in seed set  
  - *Genuine Sender* — below the 90th percentile  

Add `df_output.to_csv("trust_scores_trustrank.csv", index=False)` to export results. 

---

## Environment & setup

```bash
python -m venv .venv
source .venv/bin/activate    # Windows: .venv\Scripts\activate
pip install --upgrade pip
pip install pandas numpy matplotlib openpyxl
```

**Data folder (recommended):**
```
project/
├── data/
│   ├── Payments.xlsx
│   └── bad_sender.xlsx
└── Pregel-based-Fraud-detection.ipynb
```
Then, open the notebook and set:
```python
payments_file = 'data/Payments.xlsx'
fraudulent_file = 'data/bad_sender.xlsx'
damping = 0.85
threshold = 0.001
max_supersteps = 100
num_workers = 4
```

---

## Parameters 

| Name | Default |
|---|---|
| Damping `α` | 0.85 |
| Convergence `ε` | 0.001 |
| Max supersteps | 100 |
| Workers (threads) | 4 |
| Outlier cutoff | 90th percentile |

 

---


