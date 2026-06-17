# UNSW-NB15 — Network Intrusion Detection via Graph Neural Network
## Full ML/DL Pipeline Plan

---

## Objective

Build a **Graph Neural Network (GNN)-based Network Intrusion Detection System (NIDS)** on the UNSW-NB15 dataset.  
Goal: Accurately classify network flows as **normal** or one of **9 attack categories** (Fuzzers, Analysis, Backdoors, DoS, Exploits, Generic, Reconnaissance, Shellcode, Worms).

---

## Dataset Summary

| File | Records | Role |
|---|---|---|
| `UNSW-NB15_1-4.csv` | ~2.54M | Raw traffic (no header) |
| `NUSW-NB15_GT.csv` | 188,913 | Ground truth labels |
| `NUSW-NB15_features.csv` | 49 features | Feature schema |
| `UNSW_NB15_training-set.csv` | 175,341 | Pre-split training set |
| `UNSW_NB15_testing-set.csv` | 82,332 | Pre-split testing set |

---

## Pipeline Phases

```
Phase 1 ──► Phase 2 ──► Phase 3 ──► Phase 4 ──► Phase 5 ──► Phase 6 ──► Phase 7
  EDA       Preprocess   Graph       Baseline     GNN Model   Eval &      Deploy
                         Build       Models                   Compare     Notebook
```

---

## Phase 1 — Exploratory Data Analysis (EDA)

**Goal:** Understand the data distribution, feature quality, class imbalance, and traffic patterns.

### 1.1 Data Loading & Schema Validation
- Load `UNSW_NB15_training-set.csv` and `UNSW_NB15_testing-set.csv` (pre-split, headered).
- Cross-reference features against `NUSW-NB15_features.csv` dictionary.
- Report dtypes, null counts, duplicate rows.

### 1.2 Target Distribution Analysis
- Class counts for `label` (binary) and `attack_cat` (multiclass).
- Visualize: bar chart, pie chart.
- Quantify **class imbalance ratio**.

### 1.3 Feature Statistics
- Descriptive stats (mean, std, min/max, quartiles) for all 45 numeric/categorical columns.
- Identify zero-variance or near-zero-variance features.
- Identify highly correlated feature pairs (Pearson r > 0.95).

### 1.4 Categorical Feature Analysis
- `proto`, `service`, `state` — unique value counts, per-class distributions.
- Cardinality check.

### 1.5 Numerical Feature Distributions
- Histograms / KDE plots for top 20 features.
- Box plots grouped by `attack_cat`.
- Skewness & kurtosis report.

### 1.6 Temporal Patterns (raw files)
- `Stime` / `Ltime` distribution across UNSW-NB15_1–4.
- Attack density over time (time-series plot).

### 1.7 Attack Category Deep-Dive
- Per-category: top protocols, services, source/destination IP ranges.
- Pairwise feature separability (PCA 2D scatter, t-SNE).

**Outputs:** EDA plots saved to `outputs/eda/`, summary stats written to `outputs.md § Phase 1`.

---

## Phase 2 — Preprocessing & Feature Engineering

**Goal:** Produce a clean, model-ready feature matrix from the raw/split data.

### 2.1 Missing Value Treatment
- Identify columns with nulls (especially `service`, `attack_cat`).
- Strategy: mode imputation for categoricals, median for numerics.
- Flag rows with >50% missing as candidates for removal.

### 2.2 Categorical Encoding
- `proto`, `service`, `state`: **Label encoding** (low cardinality) or **frequency encoding**.
- `attack_cat`: **Label encoding** for multiclass target.
- Store encoder mappings to `outputs/encoders/`.

### 2.3 Feature Scaling
- Apply **RobustScaler** (resistant to outliers) on all numeric features.
- Fit on train, transform on test (no leakage).

### 2.4 Feature Selection
- Remove features with zero variance.
- Remove one of each highly-correlated pair (r > 0.95).
- Use **Random Forest feature importance** to rank and select top-K features (K ≈ 30–35).
- Optional: mutual information score as a second opinion.

### 2.5 Class Imbalance Handling
- Binary task: apply **SMOTE** on training set minority classes.
- Multiclass task: use **class-weighted loss** in models (avoids synthetic sample artifacts).
- Document before/after class counts.

### 2.6 Train/Val/Test Split Strategy
- Use the official pre-split files as train/test.
- Further split train → 80% train / 20% validation (stratified).

### 2.7 Feature Engineering (graph-specific)
- Derive **per-IP aggregated features** (node-level):
  - `src_degree`: number of flows sourced from an IP
  - `dst_degree`: number of flows destined to an IP
  - `src_mean_bytes`, `dst_mean_bytes`
  - `src_attack_ratio`: fraction of flows from IP that are attacks (train only)
- These become **node feature vectors** for the GNN.

**Outputs:** Preprocessed train/val/test tensors saved to `outputs/preprocessed/`, stats to `outputs.md § Phase 2`.

---

## Phase 3 — Graph Construction

**Goal:** Model network traffic as a heterogeneous temporal graph suitable for GNN input.

### 3.1 Graph Schema Design

```
Graph G = (V, E)

Nodes V:  Each unique IP address  →  node features = aggregated flow statistics
Edges E:  Each network flow record →  edge features = 49 flow-level features
          (directed: src_ip → dst_ip)

Node types:  source_ip, destination_ip  (can be same IP in bipartite view)
Edge types:  flow (with timestamp, label)
```

### 3.2 Node Feature Matrix
- For each unique IP, compute:
  - Total flows (in + out), total bytes (in + out)
  - Mean duration, mean packet size
  - Unique services used, unique protocols
  - Attack label ratio (training only, to avoid leakage at test time: use 0)
- Shape: `[num_nodes, node_feat_dim]`

### 3.3 Edge Feature Matrix
- Each row in the traffic table becomes a directed edge `(src_ip → dst_ip)`.
- Edge features: preprocessed 30–35 flow-level features.
- Edge label: `label` (binary) and `attack_cat` (multiclass).
- Shape: `[num_edges, edge_feat_dim]`

### 3.4 Adjacency Construction
- Build **COO (Coordinate) sparse format** `edge_index` tensor: shape `[2, num_edges]`.
- Keep directed edges (asymmetric traffic patterns matter for attacks).

### 3.5 Temporal Sub-graphs (optional advanced)
- Divide traffic into fixed time windows (e.g., 60-second bins).
- Build one graph snapshot per window → **Temporal GNN** input.
- Useful for detecting time-based attack patterns (DoS bursts, scan sweeps).

### 3.6 Graph Statistics
- Number of nodes, edges.
- Degree distribution (in-degree / out-degree).
- Graph density, diameter (sampled).
- Connected components.

### 3.7 Serialization
- Save graph as `outputs/graph/graph_data.pt` (PyTorch Geometric `Data` object).
- Save node/edge mappings to `outputs/graph/ip_to_node.json`.

**Outputs:** Graph object, statistics to `outputs.md § Phase 3`.

---

## Phase 4 — Baseline Models

**Goal:** Establish performance benchmarks before the GNN.

### 4.1 Logistic Regression
- Binary and multiclass (OvR).
- Metrics: Accuracy, F1-macro, ROC-AUC.

### 4.2 Random Forest
- 100 trees, class_weight='balanced'.
- Feature importance extraction.

### 4.3 XGBoost / LightGBM
- Gradient boosted trees — typically strongest tabular baseline.
- Scale-pos-weight for imbalance.

### 4.4 Multi-Layer Perceptron (MLP)
- 3-layer fully connected: [input → 256 → 128 → num_classes]
- Dropout 0.3, BatchNorm, ReLU.
- Adam optimizer, cross-entropy loss with class weights.

### 4.5 Evaluation Protocol (all baselines)
- Binary: Accuracy, Precision, Recall, F1, ROC-AUC, Confusion Matrix.
- Multiclass: F1-macro, F1-weighted, per-class F1, Confusion Matrix.
- Report on **test set** only (no peeking).

**Outputs:** Baseline results table in `outputs.md § Phase 4`.

---

## Phase 5 — GNN Model Design & Training

**Goal:** Build and train a GNN that leverages the graph structure of IP communication for intrusion detection.

### 5.1 Task Framing
- **Primary task:** Edge classification (each flow/edge → normal or attack category).
- **Secondary task:** Node classification (each IP → malicious or benign) as auxiliary loss.

### 5.2 Model Architecture — `NIDS-GNN`

```
Input:
  node_feat  [N, d_node]
  edge_feat  [E, d_edge]
  edge_index [2, E]

Layer 1 — Edge-conditioned Message Passing (GATv2Conv):
  • Multi-head attention (8 heads) on edges
  • Incorporates edge features into attention weights
  • Output: updated node embeddings [N, 128]

Layer 2 — GraphSAGE (SAGEConv):
  • Aggregates neighborhood (mean pooling)
  • Captures multi-hop context
  • Output: [N, 128]

Layer 3 — Edge Feature Update:
  • Concatenate: [src_node_emb || edge_feat || dst_node_emb]
  • Linear projection → [E, 256]
  • Output: updated edge embeddings [E, 256]

Classifier Head:
  • Dropout(0.4)
  • Linear(256 → 128) + ReLU + BatchNorm
  • Linear(128 → num_classes)
  • Softmax (multiclass) / Sigmoid (binary)
```

### 5.3 Architecture Variants to Experiment

| Variant | Conv Layer | Notes |
|---|---|---|
| `NIDS-GCN` | GCNConv × 3 | Simple baseline GNN |
| `NIDS-GAT` | GATv2Conv × 3 | Attention-based |
| `NIDS-SAGE` | SAGEConv × 3 | Inductive, scalable |
| `NIDS-FULL` | GAT + SAGE + EdgeConv | Full proposed model |

### 5.4 Training Configuration

| Hyperparameter | Value |
|---|---|
| Optimizer | AdamW |
| LR | 1e-3 with CosineAnnealingLR |
| Weight decay | 1e-4 |
| Batch size | Mini-batch via NeighborLoader (2-hop) |
| Epochs | 100 (early stopping patience=10) |
| Loss | Cross-entropy with class weights |
| Device | CUDA if available else CPU |

### 5.5 Mini-batch Training (for scale)
- Use PyTorch Geometric `NeighborLoader` for scalable training on the full ~2.54M flow graph.
- Sample 2-hop neighborhood per edge batch.
- Enables training on graphs too large for full-batch GPU memory.

### 5.6 Regularization
- Dropout on node embeddings (p=0.4).
- L2 weight decay.
- DropEdge (randomly remove edges during training, p=0.1) for robustness.

**Outputs:** Training curves, model checkpoint to `outputs/models/`, logs to `outputs.md § Phase 5`.

---

## Phase 6 — Evaluation & Comparison

**Goal:** Rigorously evaluate the GNN and compare against baselines.

### 6.1 Metrics

| Metric | Binary | Multiclass |
|---|---|---|
| Accuracy | ✓ | ✓ |
| Precision / Recall / F1 | ✓ | per-class + macro/weighted |
| ROC-AUC | ✓ | OvR macro |
| Matthews Correlation Coefficient (MCC) | ✓ | — |
| Confusion Matrix | ✓ | ✓ |
| False Positive Rate (FPR) | ✓ | — |

### 6.2 Comparison Table

All models evaluated on the same test set:

| Model | Acc | F1-macro | ROC-AUC | FPR |
|---|---|---|---|---|
| Logistic Regression | — | — | — | — |
| Random Forest | — | — | — | — |
| XGBoost | — | — | — | — |
| MLP | — | — | — | — |
| NIDS-GCN | — | — | — | — |
| NIDS-GAT | — | — | — | — |
| NIDS-SAGE | — | — | — | — |
| **NIDS-FULL** | — | — | — | — |

*(To be filled in `outputs.md` as phases complete)*

### 6.3 Explainability
- **GNNExplainer** on misclassified attack edges.
- Feature importance heatmap for attention weights (GAT heads).
- Highlight suspicious IP sub-graphs for each attack category.

### 6.4 Robustness Tests
- Performance on each attack category individually.
- Performance vs. graph sparsity (ablation on edge removal ratio).
- Generalization: train on files 1–3, test on file 4 (temporal generalization).

**Outputs:** All metrics, plots, confusion matrices to `outputs.md § Phase 6`.

---

## Phase 7 — Final Notebook & Summary

**Goal:** Clean, reproducible Jupyter notebook with all phases end-to-end.

### 7.1 Notebooks
- `01_eda.ipynb` — Phase 1
- `02_preprocessing.ipynb` — Phase 2
- `03_graph_construction.ipynb` — Phase 3
- `04_baselines.ipynb` — Phase 4
- `05_gnn_model.ipynb` — Phase 5
- `06_evaluation.ipynb` — Phase 6

### 7.2 Directory Structure
```
UNSW-NB15/
├── plan.md                    ← this file
├── outputs.md                 ← phase outputs log
├── notebooks/
│   ├── 01_eda.ipynb
│   ├── 02_preprocessing.ipynb
│   ├── 03_graph_construction.ipynb
│   ├── 04_baselines.ipynb
│   ├── 05_gnn_model.ipynb
│   └── 06_evaluation.ipynb
├── outputs/
│   ├── eda/                   ← plots
│   ├── preprocessed/          ← tensors, scalers
│   ├── encoders/              ← label encoders
│   ├── graph/                 ← graph_data.pt
│   └── models/                ← checkpoints
└── src/
    ├── data_loader.py
    ├── preprocessing.py
    ├── graph_builder.py
    ├── models.py
    ├── train.py
    └── evaluate.py
```

---

## Technology Stack

| Component | Library |
|---|---|
| Data manipulation | `pandas`, `numpy` |
| Visualization | `matplotlib`, `seaborn`, `plotly` |
| ML baselines | `scikit-learn`, `xgboost`, `lightgbm` |
| Imbalance handling | `imbalanced-learn` (SMOTE) |
| Deep learning | `PyTorch` |
| GNN framework | `PyTorch Geometric (PyG)` |
| Graph utils | `networkx` |
| Experiment tracking | `tqdm` for progress bars |
| Notebook | `Jupyter` |

---

## Success Criteria

| Metric | Target |
|---|---|
| Binary F1 on test | ≥ 0.95 |
| Multiclass F1-macro | ≥ 0.85 |
| False Positive Rate | ≤ 0.05 |
| GNN > best baseline | ΔF1 ≥ +0.02 |

---

## Phase Execution Order

```
[Phase 1: EDA]          → understand data, identify issues
       ↓
[Phase 2: Preprocessing] → clean + feature-engineer + scale
       ↓
[Phase 3: Graph Build]   → construct PyG Data object
       ↓
[Phase 4: Baselines]     → RF, XGB, MLP benchmarks
       ↓
[Phase 5: GNN Training]  → NIDS-GCN → NIDS-GAT → NIDS-FULL
       ↓
[Phase 6: Evaluation]    → compare, explain, ablate
       ↓
[Phase 7: Notebooks]     → clean reproducible pipeline
```
