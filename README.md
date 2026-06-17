# UNSW-NB15 — Network Intrusion Detection via Graph Neural Networks

A full end-to-end ML/DL pipeline for network intrusion detection on the [UNSW-NB15 dataset](https://research.unsw.edu.au/projects/unsw-nb15-dataset), culminating in a **Graph Neural Network (GNN)**-based classifier that models network flows as a directed graph.

---

## Pipeline Overview

```
Phase 1 ──► Phase 2 ──► Phase 3 ──► Phase 4 ──► Phase 5 ──► Phase 6
  EDA       Preprocess   Graph       Baseline     GNN Model   Eval &
                         Construct   Models       Training    Report
```

| Phase | Notebook | Status |
|---|---|---|
| 1 — EDA | `notebooks/01_eda.ipynb` | ✅ Complete |
| 2 — Preprocessing | `notebooks/02_preprocessing.ipynb` | ✅ Complete |
| 3 — Graph Construction | `notebooks/03_graph_construction.ipynb` | ✅ Complete |
| 4 — Baseline Models | `notebooks/04_baselines.ipynb` | 🔲 Colab |
| 5 — GNN Training | `notebooks/05_gnn.ipynb` | 🔲 Colab |
| 6 — Evaluation | `notebooks/06_evaluation.ipynb` | 🔲 Pending |

---

## Dataset

The UNSW-NB15 dataset must be downloaded separately from the [UNSW Canberra Cyber website](https://research.unsw.edu.au/projects/unsw-nb15-dataset).

Place the following files in the project root:

```
UNSW-NB15_1.csv
UNSW-NB15_2.csv
UNSW-NB15_3.csv
UNSW-NB15_4.csv
NUSW-NB15_GT.csv
NUSW-NB15_features.csv
Training and Testing Sets/UNSW_NB15_training-set.csv
Training and Testing Sets/UNSW_NB15_testing-set.csv
```

> These files are **not included** in the repository (too large for Git).

---

## Key Results (Phases 1–3)

### Dataset Statistics
- **2,540,047** raw flows across 4 files → **2,059,408** after deduplication
- **49 features** → **37** after dropping correlated and ID columns
- **50 unique IP addresses** (closed testbed) → 50 graph nodes + 1 UNK node

### Graph (Phase 3)
| Split | Edges | Attack % |
|---|---|---|
| Train | 1,647,526 | 4.84% |
| Val | 411,882 | 4.84% |
| Test | 82,332 | 55.06% |

Full results in [outputs.md](outputs.md).

---

## Project Structure

```
UNSW-NB15/
├── notebooks/                  # Jupyter notebooks (one per phase)
├── outputs/
│   ├── eda/                    # EDA plots and summary JSON
│   ├── preprocessed/           # Scaled numpy arrays (gitignored)
│   ├── encoders/               # Sklearn encoders (pkl)
│   ├── graph/                  # PyG Data objects (gitignored)
│   └── models/                 # Trained model checkpoints (gitignored)
├── src/                        # Reusable Python modules
├── plan.md                     # Full pipeline design document
├── outputs.md                  # Live results log (updated each phase)
├── requirements.txt
└── .gitignore
```

---

## Setup

```bash
# 1. Clone the repository
git clone https://github.com/YOUR_USERNAME/UNSW-NB15-GNN-NIDS.git
cd UNSW-NB15-GNN-NIDS

# 2. Create and activate a virtual environment
python -m venv venv
venv\Scripts\activate          # Windows
# source venv/bin/activate     # Linux/macOS

# 3. Install dependencies
pip install -r requirements.txt

# 4. Place dataset files in the project root (see Dataset section above)

# 5. Run notebooks in order
#    notebooks/01_eda.ipynb
#    notebooks/02_preprocessing.ipynb
#    notebooks/03_graph_construction.ipynb
```

### GPU Training (Colab / Kaggle)

Phases 4 and 5 are designed to run on Google Colab or Kaggle with GPU.  
See **[COLAB_KAGGLE_GUIDE.md](COLAB_KAGGLE_GUIDE.md)** for the full step-by-step upload, training, and download workflow.

For GPU PyTorch:
```bash
pip install torch --index-url https://download.pytorch.org/whl/cu121
pip install torch_geometric
```

---

## Graph Schema

```
G = (V, E)  — homogeneous directed graph
  Nodes V  : 51  (50 unique IPs from training set + 1 UNK node)
  Node feat: [51, 11]  — aggregated per-IP flow statistics, RobustScaled
  Edges E  : each network flow → directed edge (srcip → dstip)
  Edge feat: [E, 37]   — flow-level features, RobustScaled
  Edge label: binary (0=Normal, 1=Attack) + multiclass (0–9 attack categories)
```

---

## Attack Categories

| Label | Category | Train Count |
|---|---|---|
| 0 | Analysis | 1,729 |
| 1 | Backdoor | 1,543 |
| 2 | DoS | 4,551 |
| 3 | Exploits | 22,113 |
| 4 | Fuzzers | 17,443 |
| 5 | Generic | 20,260 |
| 6 | Normal | 1,567,812 |
| 7 | Reconnaissance | 10,739 |
| 8 | Shellcode | 1,201 |
| 9 | Worms | 135 |

---

## References

- Moustafa, N. & Slay, J. (2015). *UNSW-NB15: A Comprehensive Data set for Network Intrusion Detection Systems*. MilCIS 2015.
- PyTorch Geometric: https://pytorch-geometric.readthedocs.io
