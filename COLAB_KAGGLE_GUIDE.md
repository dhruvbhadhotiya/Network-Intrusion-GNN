# Colab / Kaggle Training Guide

Phases 1–3 (EDA, Preprocessing, Graph Construction) run locally.  
Phases 4–5 (Baseline Models + GNN Training) are GPU-intensive and should run on **Kaggle** or **Google Colab**.  
Phase 6 (Evaluation) runs locally again — it only loads saved model files.

---

## Overview

```
Local Machine                        Kaggle / Colab
─────────────                        ──────────────
Phase 1  EDA              ✅
Phase 2  Preprocessing    ✅  ──►  upload outputs/  ──►  Phase 4  Baselines   🚀
Phase 3  Graph Construct  ✅                              Phase 5  GNN Train   🚀
                                     download models/  ◄──
Phase 6  Evaluation       ✅  ◄──────────────────────────
```

---

## Step 1 — Files to Upload

You only need to upload the **pipeline outputs**, not the raw dataset (2.5M rows).

### Create the upload zip

Run this in PowerShell from the project root:

```powershell
$files = @(
    "outputs\preprocessed\X_train.npy",
    "outputs\preprocessed\X_val.npy",
    "outputs\preprocessed\X_test.npy",
    "outputs\preprocessed\X_train_binary_resampled.npy",
    "outputs\preprocessed\X_train_multiclass_resampled.npy",
    "outputs\preprocessed\y_binary_train.npy",
    "outputs\preprocessed\y_binary_train_resampled.npy",
    "outputs\preprocessed\y_binary_val.npy",
    "outputs\preprocessed\y_binary_test.npy",
    "outputs\preprocessed\y_multiclass_train.npy",
    "outputs\preprocessed\y_multiclass_train_resampled.npy",
    "outputs\preprocessed\y_multiclass_val.npy",
    "outputs\preprocessed\y_multiclass_test.npy",
    "outputs\preprocessed\feature_names.json",
    "outputs\preprocessed\node_features.csv",
    "outputs\preprocessed\ip_to_node.json",
    "outputs\preprocessed\node_features_meta.json",
    "outputs\graph\graph_data_train.pt",
    "outputs\graph\graph_data_val.pt",
    "outputs\graph\graph_data_test.pt",
    "outputs\graph\graph_stats.json",
    "outputs\encoders\le_attack_cat.pkl",
    "outputs\encoders\le_service.pkl",
    "outputs\encoders\le_state.pkl",
    "outputs\encoders\proto_freq_map.pkl",
    "outputs\encoders\robust_scaler.pkl"
)
Compress-Archive -Path $files -DestinationPath "unsw_nb15_outputs.zip" -Force
Write-Host "Created unsw_nb15_outputs.zip"
```

**Total size: ~1.1 GB**

---

## Step 2 — Platform Comparison

| Platform | GPU | Free Tier | Limit | Best For |
|---|---|---|---|---|
| **Kaggle** | NVIDIA T4 × 2 | ✅ Free | 30 GPU hrs/week | Phase 4 + Phase 5 |
| **Google Colab Free** | T4 | ✅ Free | Session-limited (~2–4 hrs) | Quick experiments |
| **Google Colab Pro** | T4 / A100 | $10/month | 100 compute units | Full Phase 5 GNN |

**Recommendation:** Use Kaggle — no time pressure, 30 GPU hrs/week is enough for both phases, and datasets persist across sessions.

---

## Step 3 — Kaggle Setup

### 3.1 Upload your data as a Kaggle Dataset

1. Go to [kaggle.com/datasets](https://www.kaggle.com/datasets) → **New Dataset**
2. Name it: `unsw-nb15-preprocessed`
3. Upload `unsw_nb15_outputs.zip` → Kaggle will unzip it automatically
4. Set visibility to **Private**
5. Click **Create**

### 3.2 Create a Kaggle Notebook

1. Go to [kaggle.com/code](https://www.kaggle.com/code) → **New Notebook**
2. In **Settings** (right panel):
   - Accelerator: **GPU T4 × 2**
   - Internet: **On**
3. Click **Add Data** → search `unsw-nb15-preprocessed` → Add
4. Your data will be at `/kaggle/input/unsw-nb15-preprocessed/`

### 3.3 First cell in every Kaggle notebook

```python
import os

# ── Path config: one line to change between local / Kaggle / Colab ────────────
PLATFORM = 'kaggle'   # 'local' | 'kaggle' | 'colab'

if PLATFORM == 'kaggle':
    BASE    = '/kaggle/input/unsw-nb15-preprocessed'
    OUT_DIR = '/kaggle/working/outputs'          # writable output dir
elif PLATFORM == 'colab':
    from google.colab import drive
    drive.mount('/content/drive')
    BASE    = '/content/drive/MyDrive/UNSW-NB15/outputs'
    OUT_DIR = '/content/drive/MyDrive/UNSW-NB15/outputs/models'
else:
    BASE    = r'c:\Users\Asus\OneDrive\Desktop\GNN\UNSW-NB15'
    OUT_DIR = os.path.join(BASE, 'outputs', 'models')

OUT_PRE = os.path.join(BASE, 'outputs', 'preprocessed') if PLATFORM == 'local' else os.path.join(BASE, 'preprocessed')
OUT_GRF = os.path.join(BASE, 'outputs', 'graph')        if PLATFORM == 'local' else os.path.join(BASE, 'graph')
OUT_ENC = os.path.join(BASE, 'outputs', 'encoders')     if PLATFORM == 'local' else os.path.join(BASE, 'encoders')
os.makedirs(OUT_DIR, exist_ok=True)
print(f'Platform: {PLATFORM}')
print(f'Preprocessed: {OUT_PRE}')
print(f'Graph:        {OUT_GRF}')
print(f'Output dir:   {OUT_DIR}')
```

### 3.4 Install packages on Kaggle

Kaggle has most packages pre-installed. Only add:

```python
!pip install torch_geometric -q
```

---

## Step 4 — Google Colab Setup

### 4.1 Upload to Google Drive

1. In Google Drive, create folder: `My Drive/UNSW-NB15/`
2. Inside it, create subfolders: `preprocessed/`, `graph/`, `encoders/`
3. Upload files from `unsw_nb15_outputs.zip` into the matching folders

### 4.2 First cell in every Colab notebook

```python
from google.colab import drive
drive.mount('/content/drive')

BASE    = '/content/drive/MyDrive/UNSW-NB15'
OUT_PRE = f'{BASE}/preprocessed'
OUT_GRF = f'{BASE}/graph'
OUT_ENC = f'{BASE}/encoders'
OUT_DIR = f'{BASE}/models'

import os; os.makedirs(OUT_DIR, exist_ok=True)
```

### 4.3 Install packages on Colab

```python
!pip install torch --index-url https://download.pytorch.org/whl/cu121 -q
!pip install torch_geometric -q
!pip install lightgbm xgboost imbalanced-learn -q
```

---

## Step 5 — Running the Notebooks

### Phase 4 — Baseline Models (`04_baselines.ipynb`)

Expected GPU time on Kaggle T4:

| Model | Binary Train Time | Multiclass Train Time |
|---|---|---|
| Logistic Regression | ~5 min (subsampled) | ~5 min |
| Random Forest | ~10 min | ~15 min |
| XGBoost | ~8 min | ~10 min |
| LightGBM | ~3 min | ~4 min |
| MLP (PyTorch) | ~5 min / epoch | ~5 min / epoch |

### Phase 5 — GNN Training (`05_gnn.ipynb`)

Expected GPU time on Kaggle T4:

| Model | Time per Epoch | Notes |
|---|---|---|
| Edge-GCN (binary) | ~3–5 min | Full-graph, no mini-batching |
| Edge-GAT (binary) | ~5–8 min | Multi-head attention, more memory |
| Edge-SAGE (multiclass) | ~4–6 min | GraphSAGE, better for dense graphs |

> ⚠️ With only 51 nodes and 1.6M edges, the graph is **dense** — standard `NeighborLoader`
> returns the full graph. Use full-batch training with gradient accumulation if memory limited.

---

## Step 6 — Downloading Trained Models

After training, download from Kaggle:

1. In the Kaggle notebook, the output files are in `/kaggle/working/outputs/models/`
2. Go to the notebook page → **Output** tab → Download individual files or the full output zip

After training on Colab:
- Files are saved directly to Google Drive → download from there

### What to download

**Phase 4:**
```
outputs/models/baseline_results.json   # metrics summary
outputs/models/rf_binary.pkl
outputs/models/xgb_binary.pkl
outputs/models/lgbm_binary.pkl
outputs/models/mlp_binary.pt
outputs/models/rf_multiclass.pkl
outputs/models/xgb_multiclass.pkl
outputs/models/lgbm_multiclass.pkl
outputs/models/mlp_multiclass.pt
```

**Phase 5:**
```
outputs/models/gnn_binary_best.pt
outputs/models/gnn_multiclass_best.pt
outputs/models/gnn_training_log.json
```

---

## Step 7 — After Download: Local Evaluation

Place downloaded files into `outputs/models/` in your local project, then run:

```
notebooks/06_evaluation.ipynb
```

This notebook only **loads** saved models and generates the final comparison plots and report. **No GPU required.**

---

## Quick Reference

| Task | Where | Notebook |
|---|---|---|
| EDA | Local | `01_eda.ipynb` |
| Preprocessing | Local | `02_preprocessing.ipynb` |
| Graph Construction | Local | `03_graph_construction.ipynb` |
| Baseline Models | **Kaggle / Colab** | `04_baselines.ipynb` |
| GNN Training | **Kaggle / Colab** | `05_gnn.ipynb` |
| Final Evaluation | Local | `06_evaluation.ipynb` |
