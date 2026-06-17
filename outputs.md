# UNSW-NB15 NIDS — Pipeline Outputs Log

> This file records all findings, statistics, metrics, and results produced during each phase of the pipeline.
> Each phase appends its results here as it completes.

---

## § Phase 1 — EDA

> Status: ✅ Complete

### 1.1 Dataset Shape & Schema

| File | Rows | Normal | Attack | Attack % |
|---|---|---|---|---|
| UNSW-NB15_1.csv | 700,001 | 677,786 | 22,215 | 3.17% |
| UNSW-NB15_2.csv | 700,001 | 647,252 | 52,749 | 7.54% |
| UNSW-NB15_3.csv | 700,001 | 542,576 | 157,425 | 22.49% |
| UNSW-NB15_4.csv | 440,044 | 351,150 | 88,894 | 20.20% |
| **Combined** | **2,540,047** | **2,218,764** | **321,283** | **12.65%** |

- Total columns: 49 (37 numeric, 3 categorical, 2 IP, 2 timestamp, 2 target, 2 flags)
- Duplicate rows: **480,632** (18.92% of total) → must be dropped in preprocessing
- Date range: **2015-01-22** to **2015-02-18** (27 days)

### 1.2 Null / Missing Values

| Column | Null Count | Null % | Preprocessing Strategy |
|---|---|---|---|
| `ct_flw_http_mthd` | 1,348,145 | 53.10% | Fill 0 (non-HTTP flows) |
| `is_ftp_login` | 1,429,879 | 56.29% | Fill 0 (non-FTP flows) |
| All other columns | 0 | 0.00% | No action needed |

### 1.3 Class Distribution

**Binary (label)**

| Class | Count | % |
|---|---|---|
| 0 — Normal | 2,218,764 | 87.35% |
| 1 — Attack | 321,283 | 12.65% |
| **Imbalance ratio** | **6.91 : 1** | |

**Multiclass (attack_cat)**

| Category | Count | % of Total |
|---|---|---|
| Normal | 2,218,764 | 87.35% |
| Generic | 215,481 | 8.48% |
| Exploits | 44,525 | 1.75% |
| Fuzzers | 24,246 | 0.95% |
| DoS | 16,353 | 0.64% |
| Reconnaissance | 13,987 | 0.55% |
| Analysis | 2,677 | 0.11% |
| Backdoor | 1,795 | 0.07% |
| Shellcode | 1,511 | 0.06% |
| Backdoors *(duplicate label)* | 534 | 0.02% |
| Worms | 174 | 0.007% |

> ⚠️ **Note:** `Backdoor` and `Backdoors` are the same class — merge to `Backdoor` in preprocessing.

### 1.4 Feature Statistics Summary

- Zero-variance features: **0** (none to drop)
- High-correlation pairs (r > 0.95): **8 pairs**

| Feature A | Feature B | r |
|---|---|---|
| dwin | swin | 0.997 |
| Dpkts | dloss | 0.991 |
| dloss | dbytes | 0.991 |
| Dpkts | dbytes | 0.968 |
| sloss | sbytes | 0.963 |
| ct_src_dport_ltm | ct_dst_ltm | 0.960 |
| ct_srv_dst | ct_srv_src | 0.957 |
| ct_dst_src_ltm | ct_srv_dst | 0.951 |

- PCA variance explained by 2 components: **99.99%** (data is highly linearly separable)

### 1.5 Categorical Features

| Feature | Unique Values | Notes |
|---|---|---|
| proto | 135 | High cardinality — use frequency/target encoding |
| service | 13 | Low cardinality — one-hot or label encode |
| state | 16 | Low cardinality — one-hot or label encode |

### 1.6 Top-20 Features by RF Importance (binary label)

`sttl`, `ct_state_ttl`, `Dload`, `dbytes`, `Dintpkt`, `dttl`, `dmeansz`, `state`, `Dpkts`, `smeansz`, `Sload`, `sbytes`, `dur`, `ackdat`, `tcprtt`, `synack`, `dloss`, `Sintpkt`, `ct_dst_src_ltm`, `ct_dst_sport_ltm`

### 1.7 Key Observations

- Attack volume escalated across files: 3.2% → 7.5% → 22.5% → 20.2% (temporal drift)
- `Generic` dominates attacks (67% of all attack flows) — likely to dominate multiclass performance
- `Worms` and `Shellcode` are severely minority classes — will need SMOTE or oversampling
- 480k duplicate rows (18.9%) must be removed before training — significant data leakage risk
- `ct_flw_http_mthd` and `is_ftp_login` are sparse (>53% null) — fill with 0 as protocol-specific flags
- `Backdoor` / `Backdoors` label split needs merging — currently counted separately in raw data
- High correlation clusters: (swin/dwin), (Dpkts/dloss/dbytes), (sloss/sbytes) — candidate for dropping one from each pair

### 1.8 EDA Plots Generated

- [x] Class distribution bar chart → `outputs/eda/class_dist.png`
- [x] Feature correlation heatmap → `outputs/eda/correlation_heatmap.png`
- [x] PCA + t-SNE projection → `outputs/eda/pca_tsne.png`
- [x] Top-20 feature distributions → `outputs/eda/feature_histograms.png`
- [x] Box plots by attack category → `outputs/eda/boxplots_by_category.png`
- [x] Protocol/service heatmaps → `outputs/eda/attack_cat_heatmaps.png`
- [x] Temporal attack density → `outputs/eda/temporal_density.png`
- [x] Per-file distribution → `outputs/eda/per_file_distribution.png`
- [x] RF feature importance → `outputs/eda/rf_feature_importance.png`
- [x] Full summary JSON → `outputs/eda/eda_summary.json`

---

## § Phase 2 — Preprocessing

> Status: ✅ Complete

### 2.1 Missing Value Treatment

| Column | Nulls Before | Treatment | Nulls After |
|---|---|---|---|
| `ct_flw_http_mthd` | 1,348,145 (53.1%) | Fill 0 (non-HTTP flows have no method) | 0 |
| `is_ftp_login` | 1,429,879 (56.3%) | Fill 0 (non-FTP flows are not logins) | 0 |
| All other columns | 0 | No action | 0 |

### 2.2 Duplicate Rows

- Combined raw files: 2,540,047 rows
- Duplicate rows removed: **480,632** (18.92%)
- Rows after dedup: **2,059,415**

### 2.3 Categorical Encoding

| Feature | Encoding Method | Num. Categories | Notes |
|---|---|---|---|
| `proto` | **Frequency encoding** (train-only fit) | 135 | Maps protocol → its training frequency; unseen → 0 |
| `service` | Label encoding | 13 | Fit on union of all splits (fixed domain) |
| `state` | Label encoding | 16 | Fit on union of all splits (fixed domain) |
| `attack_cat` | Label encoding | 10 | Fit on full data (target encoding) |

### 2.4 Feature Selection Results

- Features before selection: **43** (49 total − 4 ID/TS − 2 targets)
- Correlated features dropped (r > 0.95): **6** (`dwin`, `dloss`, `sloss`, `ct_src_dport_ltm`, `ct_srv_src`, `ct_dst_src_ltm`)
- **Final feature count: 37**

| Dropped | Kept | r | Reason |
|---|---|---|---|
| `dwin` | `swin` | 0.997 | swin kept |
| `dloss` | `dbytes` | 0.991 | dbytes at RF rank 4 |
| `sloss` | `sbytes` | 0.963 | sbytes at RF rank 12 |
| `ct_src_dport_ltm` | `ct_dst_ltm` | 0.960 | ct_dst_ltm kept |
| `ct_srv_src` | `ct_srv_dst` | 0.957 | ct_srv_dst kept |
| `ct_dst_src_ltm` | `ct_srv_dst` | 0.951 | ct_srv_dst retained |

**Final feature set (37):** `sport, dsport, proto, state, dur, sbytes, dbytes, sttl, dttl, service, Sload, Dload, Spkts, Dpkts, swin, stcpb, dtcpb, smeansz, dmeansz, trans_depth, res_bdy_len, Sjit, Djit, Sintpkt, Dintpkt, tcprtt, synack, ackdat, is_sm_ips_ports, ct_state_ttl, ct_flw_http_mthd, is_ftp_login, ct_ftp_cmd, ct_srv_dst, ct_dst_ltm, ct_src_ltm, ct_dst_sport_ltm`

### 2.5 Class Imbalance Handling

| Strategy | Applied to | Ratio Before | Ratio After |
|---|---|---|---|
| `RandomOverSampler` (sampling_strategy=0.5) | Binary train | **19.67:1** | **2:1** (1,567,812 Normal / 783,906 Attack → 2,351,718 total) |
| `SMOTE` (k_neighbors=3, target=5000) | Multiclass train minorities | Worms=135, Shellcode=1201, Backdoor=1543, Analysis=1729, DoS=4551 | **5,000 each** (1,663,367 total) |

> `SMOTE` flag `USE_SMOTE_BINARY=True` available in notebook to switch binary strategy.

### 2.6 Train / Val / Test Split

| Split | Source | Records | Attack % | Strategy |
|---|---|---|---|---|
| Train (80%) | Raw files post-dedup | **1,647,526** | 4.84% (post-dedup) | stratified on `label` |
| Validation (20%) | Raw files post-dedup | **411,882** | 4.84% | stratified on `label` |
| Test (official) | `UNSW_NB15_testing-set.csv` | **82,332** | 55.06% (pre-balanced) | unchanged holdout |

> ⚠️ **Key finding:** After dedup, attack % in train dropped from 12.65% → **4.84%**. The 480k duplicate rows were disproportionately attack flows (Generic category). Post-dedup class distribution: Normal=1,567,812 / Attack=79,714.

### 2.7 Node-level Features Engineered (for GNN)

Computed from **training rows only** (strict anti-leakage on `*_attack_ratio` features).

| Feature | Description |
|---|---|
| `out_degree` | # flows sourced from IP |
| `in_degree` | # flows destined to IP |
| `total_degree` | out + in degree |
| `mean_sbytes` | Mean source bytes per src IP |
| `mean_dbytes` | Mean destination bytes per dst IP |
| `total_bytes` | mean_sbytes + mean_dbytes |
| `mean_dur` | Mean flow duration |
| `unique_dsport` | Unique destination ports per src IP |
| `unique_proto` | Unique protocols per IP |
| `src_attack_ratio` | Fraction of outgoing flows that are attacks (train only) |
| `dst_attack_ratio` | Fraction of incoming flows that are attacks (train only) |

- Saved → `outputs/preprocessed/node_features.csv` — **50 nodes × 11 features**
- Saved → `outputs/preprocessed/ip_to_node.json` — **50 IP entries**

### 2.8 Key Observations

- `RobustScaler` chosen over `StandardScaler`: network features are heavily skewed (|skew| > 5 for 20+ features); median/IQR scaling is far more robust
- `RandomOverSampler` used for binary (not SMOTE) to keep runtime feasible on 1.3M training rows; SMOTE available via flag
- SMOTE `k_neighbors=3` for `Worms` class which has ~140 training samples (default k=5 would fail)
- Proto frequency encoding fitted on training data only — prevents frequency distribution leakage
- IP columns retained until after split to enable per-IP node feature aggregation, then excluded from X
- Official test set column names normalised via alias map (official files use `smean`/`dmean`/`sinpkt` vs our schema's `smeansz`/`dmeansz`/`Sintpkt`)

### 2.9 Outputs Generated

- [x] `outputs/preprocessed/X_train.npy`
- [x] `outputs/preprocessed/X_val.npy`
- [x] `outputs/preprocessed/X_test.npy`
- [x] `outputs/preprocessed/X_train_binary_resampled.npy`
- [x] `outputs/preprocessed/X_train_multiclass_resampled.npy`
- [x] `outputs/preprocessed/y_binary_{train,val,test}.npy`
- [x] `outputs/preprocessed/y_binary_train_resampled.npy`
- [x] `outputs/preprocessed/y_multiclass_{train,val,test}.npy`
- [x] `outputs/preprocessed/y_multiclass_train_resampled.npy`
- [x] `outputs/preprocessed/node_features.csv`
- [x] `outputs/preprocessed/ip_to_node.json`
- [x] `outputs/preprocessed/feature_names.json`
- [x] `outputs/encoders/le_attack_cat.pkl`
- [x] `outputs/encoders/le_service.pkl`
- [x] `outputs/encoders/le_state.pkl`
- [x] `outputs/encoders/proto_freq_map.pkl`
- [x] `outputs/encoders/robust_scaler.pkl`
- [x] `outputs/preprocessed/imbalance_before_after.png`

---

## § Phase 3 — Graph Construction

> Status: ✅ Complete

### 3.1 Graph Schema

```
G = (V, E)  — homogeneous directed graph
  Nodes V  : 51 total  (50 unique training IPs  +  1 UNK node at index 50)
  Edges E  : each network flow record  →  directed edge  (srcip → dstip)
  Node feat: [51, 11]  — RobustScaled, UNK row = zeros
  Edge feat: [E, 37]  — RobustScaled flow-level features (from Phase 2)
  Edge label: y (binary) + y_multi (multiclass, 0–9)
```

### 3.2 Graph Dimensions

| Split | Nodes | Edges | Unique (src,dst) pairs | Density |
|---|---|---|---|---|
| Train | 51 | 1,647,526 | 311 | 646.09 |
| Val   | 51 | 411,882 | 286 | 161.52 |
| Test  | 51 | 82,332 | 1 | 32.29 |

### 3.3 Degree Statistics (Training Graph)

| Metric | Out-Degree | In-Degree |
|---|---|---|
| Mean | 32,304 | 32,304 |
| Max  | 154,082 | 154,219 |

- Weakly-connected components (train): **6**

> ⚠️ The graph is **extremely dense** — only 50 unique IP nodes but ~1.6M edges.
> Standard `NeighborLoader` will return full-graph subgraphs; batch size tuning required for GNN.

### 3.4 Edge Label Distribution

| Split | Normal | Attack | Attack % |
|---|---|---|---|
| Train | 1,567,812 | 79,714 | 4.84% |
| Val   | 391,953 | 19,929 | 4.84% |
| Test  | 37,000 | 45,332 | 55.06% |

### 3.5 UNK Node Coverage

- Training IPs in `ip_to_node`: **50** (UNSW-NB15 is a closed testbed — all flows use these 50 IPs)
- UNK node (index 50) created with zero features as fallback for unseen IPs
- Test set has **no IP columns** (official pre-split CSV is anonymised) → all test edges use UNK node
- Node feature scaling: `RobustScaler` fitted on the 50 training-set IPs only

### 3.6 Outputs Saved

- [x] `outputs/graph/graph_data_train.pt` — PyG `Data`, 1,647,526 edges, 296 MB
- [x] `outputs/graph/graph_data_val.pt`   — PyG `Data`, 411,882 edges, 74 MB
- [x] `outputs/graph/graph_data_test.pt`  — PyG `Data`, 82,332 edges, 15 MB
- [x] `outputs/graph/graph_stats.json`
- [x] `outputs/graph/degree_distribution.png`
- [x] `outputs/graph/edge_label_distribution.png`
- [x] `outputs/preprocessed/srcip_train.npy` / `dstip_train.npy` (cached, 1,647,526 IPs)
- [x] `outputs/preprocessed/srcip_val.npy` / `dstip_val.npy` (cached, 411,882 IPs)

---

## § Phase 4 — Baseline Models

> Status: 🔲 Not Started

### 4.1 Binary Classification Results

| Model | Accuracy | Precision | Recall | F1 | ROC-AUC | FPR | Train Time |
|---|---|---|---|---|---|---|---|
| Logistic Regression | — | — | — | — | — | — | — |
| Random Forest | — | — | — | — | — | — | — |
| XGBoost | — | — | — | — | — | — | — |
| LightGBM | — | — | — | — | — | — | — |
| MLP | — | — | — | — | — | — | — |

### 4.2 Multiclass Classification Results (F1 per category)

| Model | Normal | Fuzzers | Analysis | Backdoors | DoS | Exploits | Generic | Recon | Shellcode | Worms | Macro-F1 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Logistic Regression | — | — | — | — | — | — | — | — | — | — | — |
| Random Forest | — | — | — | — | — | — | — | — | — | — | — |
| XGBoost | — | — | — | — | — | — | — | — | — | — | — |
| MLP | — | — | — | — | — | — | — | — | — | — | — |

### 4.3 Best Baseline

- Binary: **?** with F1=—
- Multiclass: **?** with F1-macro=—

---

## § Phase 5 — GNN Model Training

> Status: 🔲 Not Started

### 5.1 Graph Construction Summary

- Framework: PyTorch Geometric
- Node features dim: —
- Edge features dim: —

### 5.2 Training Runs

#### Run 1 — NIDS-GCN

| Epoch | Train Loss | Val Loss | Val F1 |
|---|---|---|---|
| Best | — | — | — |

- Final test F1 (binary): —
- Final test F1-macro: —
- Checkpoint: `outputs/models/nids_gcn.pt`

#### Run 2 — NIDS-GAT

| Epoch | Train Loss | Val Loss | Val F1 |
|---|---|---|---|
| Best | — | — | — |

- Final test F1 (binary): —
- Final test F1-macro: —
- Checkpoint: `outputs/models/nids_gat.pt`

#### Run 3 — NIDS-SAGE

| Epoch | Train Loss | Val Loss | Val F1 |
|---|---|---|---|
| Best | — | — | — |

- Final test F1 (binary): —
- Final test F1-macro: —
- Checkpoint: `outputs/models/nids_sage.pt`

#### Run 4 — NIDS-FULL (GAT + SAGE + EdgeConv)

| Epoch | Train Loss | Val Loss | Val F1 |
|---|---|---|---|
| Best | — | — | — |

- Final test F1 (binary): —
- Final test F1-macro: —
- Checkpoint: `outputs/models/nids_full.pt`

### 5.3 Hyperparameter Search Results

| Config | Val F1 | Notes |
|---|---|---|
| — | — | — |

---

## § Phase 6 — Evaluation & Comparison

> Status: 🔲 Not Started

### 6.1 Full Comparison Table — Binary Detection

| Model | Accuracy | Precision | Recall | F1 | ROC-AUC | FPR |
|---|---|---|---|---|---|---|
| Logistic Regression | — | — | — | — | — | — |
| Random Forest | — | — | — | — | — | — |
| XGBoost | — | — | — | — | — | — |
| LightGBM | — | — | — | — | — | — |
| MLP | — | — | — | — | — | — |
| NIDS-GCN | — | — | — | — | — | — |
| NIDS-GAT | — | — | — | — | — | — |
| NIDS-SAGE | — | — | — | — | — | — |
| **NIDS-FULL** | — | — | — | — | — | — |

### 6.2 Full Comparison Table — Multiclass (F1-macro)

| Model | F1-macro | F1-weighted | Notes |
|---|---|---|---|
| Logistic Regression | — | — | — |
| Random Forest | — | — | — |
| XGBoost | — | — | — |
| MLP | — | — | — |
| NIDS-GCN | — | — | — |
| NIDS-GAT | — | — | — |
| NIDS-SAGE | — | — | — |
| **NIDS-FULL** | — | — | — |

### 6.3 Best Model Summary

- **Best Binary Model:** — | F1=— | ROC-AUC=—
- **Best Multiclass Model:** — | F1-macro=—
- **GNN improvement over best baseline:** ΔF1=—

### 6.4 Confusion Matrix Notes

- Most confused pairs: —
- Hardest category to detect: —

### 6.5 Explainability Notes (GNNExplainer)

- Key node features for attack detection: —
- Key edge features: —
- Notable subgraph patterns for DoS: —
- Notable subgraph patterns for Reconnaissance: —

### 6.6 Robustness — Temporal Generalization

| Train on | Test on | F1-binary | F1-macro |
|---|---|---|---|
| Files 1–3 | File 4 | — | — |
| Official train | Official test | — | — |

### 6.7 Plots Generated

- [ ] ROC curve comparison → `outputs/eda/roc_comparison.png`
- [ ] Confusion matrix (NIDS-FULL) → `outputs/eda/confusion_matrix_gnn.png`
- [ ] Training loss curves → `outputs/eda/training_curves.png`
- [ ] GNNExplainer subgraph → `outputs/eda/explainer_sample.png`
- [ ] Per-class F1 bar chart → `outputs/eda/perclass_f1.png`

---

## § Change Log

| Date | Phase | Change |
|---|---|---|
| 2026-06-15 | — | Plan and outputs.md created |
| 2026-06-16 | Phase 2 | Preprocessing notebook created and executed; all 19 cells ran successfully |


