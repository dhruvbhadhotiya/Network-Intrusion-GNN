
## § Phase 4 — Baseline Models

> Status: ✅ Complete

### 4.1 Binary Classification Results (Val Set)

| Model | Accuracy | Precision | Recall | F1 | ROC-AUC | FPR |
|---|---|---|---|---|---|---|
| LR | 0.9745 | 0.6559 | 0.9931 | 0.7901 | 0.9853 | 0.0265 |
| RF | 0.9866 | 0.7841 | 0.9979 | 0.8782 | 0.9991 | 0.0140 |
| XGBoost | 0.9884 | 0.8112 | 0.9910 | 0.8922 | 0.9993 | 0.0117 |
| LightGBM | 0.9889 | 0.8204 | 0.9853 | 0.8953 | 0.9989 | 0.0110 |
| MLP | 0.9860 | 0.7757 | 0.9997 | 0.8736 | 0.9989 | 0.0147 |

### 4.2 Binary Classification Results (Test Set)

| Model | Accuracy | Precision | Recall | F1 | ROC-AUC | FPR |
|---|---|---|---|---|---|---|
| LR | 0.7702 | 0.7072 | 0.9942 | 0.8265 | 0.6485 | 0.5043 |
| RF | 0.8233 | 0.7572 | 0.9997 | 0.8617 | 0.9691 | 0.3928 |
| XGBoost | 0.7911 | 0.7250 | 0.9999 | 0.8405 | 0.9745 | 0.4646 |
| LightGBM | 0.7740 | 0.7093 | 0.9990 | 0.8296 | 0.9375 | 0.5017 |
| MLP | 0.7977 | 0.7314 | 0.9997 | 0.8448 | 0.9769 | 0.4498 |

### 4.3 Multiclass Classification Results (Val Set)

| Model | Accuracy | F1-Macro | F1-Weighted |
|---|---|---|---|
| LR | 0.4823 | 0.1168 | 0.6252 |
| RF | 0.9779 | 0.6131 | 0.9816 |
| XGBoost | 0.9840 | 0.6273 | 0.9845 |
| LightGBM | 0.9773 | 0.6574 | 0.9814 |
| MLP | 0.9713 | 0.5309 | 0.9766 |

### 4.4 Multiclass Classification Results (Test Set)

| Model | Accuracy | F1-Macro | F1-Weighted |
|---|---|---|---|
| LR | 0.4053 | 0.1913 | 0.4053 |
| RF | 0.6862 | 0.4605 | 0.7295 |
| XGBoost | 0.6118 | 0.3276 | 0.6450 |
| LightGBM | 0.5877 | 0.3234 | 0.6324 |
| MLP | 0.6389 | 0.3815 | 0.6938 |

### 4.5 Per-class F1 — Best Multiclass Model (lgbm_multiclass, Val Set)

| Class | F1 |
|---|---|
| Analysis | 0.1736 |
| Backdoor | 0.1535 |
| DoS | 0.4205 |
| Exploits | 0.8274 |
| Fuzzers | 0.5949 |
| Generic | 0.9467 |
| Normal | 0.9928 |
| Reconnaissance | 0.8440 |
| Shellcode | 0.9003 |
| Worms | 0.7200 |

### 4.6 Best Baseline Models

- **Binary:**     `lgbm_binary`  (val F1=0.8953, test F1=0.8296)
- **Multiclass:** `lgbm_multiclass`  (val F1-macro=0.6574, test F1-macro=0.3234)

> ⚠️  Distribution shift: Val attack ratio=4.84% | Test attack ratio=55.06%

### 4.7 Plots Generated

- [x] `outputs/models/roc_curves.png`
- [x] `outputs/models/f1_comparison.png`
- [x] `outputs/models/cm_best_binary.png`
- [x] `outputs/models/cm_best_multiclass.png`
- [x] `outputs/models/rf_importance.png`
- [x] `outputs/models/baseline_results.json`
