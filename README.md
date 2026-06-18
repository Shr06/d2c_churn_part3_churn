# d2c_churn_part3_churn
# Customer Churn Prediction Model

Predicts customers likely to churn in the next 60 days using RFM and behavioural
features from a D2C e-commerce snapshot dated 2025-09-30.

---

## Repository Contents

| File | Description |
|---|---|
| `churn_model.ipynb` | Full modeling notebook — data loading, leakage audit, preprocessing, baseline model, stronger model, evaluation, threshold selection, feature importance, error analysis, model saving |
| `model.pkl` | Final trained Logistic Regression pipeline (joblib format) |
| `metrics.json` | Key evaluation metrics from the test set at threshold 0.40 |
| `error_analysis.md` | Detailed false positive and false negative analysis with 15 real customer examples |
| `model_card.md` | Structured model card covering intended use, data, performance, limitations, ethical risks, and monitoring |
| `requirements.txt` | Python dependencies |
| `README.md` | 

---

## Setup

```bash
pip install -r requirements.txt
```

---

## Loading the Model

```python
import joblib
import pandas as pd

# Load the saved pipeline
model = joblib.load('model.pkl')

# Score new customers (must have same feature columns as training data)
# feature_cols excludes: customer_id, snapshot_date, churn_next_60d, split
new_data = pd.read_csv('new_customers.csv')
feature_cols = [col for col in new_data.columns
                if col not in ['customer_id', 'snapshot_date', 'churn_next_60d', 'split']]

probabilities = model.predict_proba(new_data[feature_cols])[:, 1]
predictions = (probabilities >= 0.40).astype(int)  # selected threshold

new_data['churn_probability'] = probabilities
new_data['churn_flag'] = predictions
```

---

## Model Summary

| Item | Value |
|---|---|
| Algorithm | Logistic Regression |
| Preprocessing | Median imputation (numeric) + Most-frequent imputation + OHE (categorical) |
| Selected threshold | 0.40 |
| Accuracy (test) | 81.5% |
| Precision (test) | 78.2% |
| Recall (test) | 87.5% |
| F1 Score (test) | 82.6% |
| ROC-AUC (test) | 0.886 |
| PR-AUC (test) | 0.879 |

---

## Dataset

Source: `rfm_modeling_snapshot.csv`
Snapshot date: 2025-09-30
Target: `churn_next_60d` (1 = churned within 60 days of snapshot, 0 = retained)

The dataset and full Google Drive folder can be accessed here:
https://drive.google.com/drive/folders/1PmLapJI1VSDgvl_AxARNKwM1MCd3WFX0

---

## Threshold Selection

The default threshold of 0.50 was lowered to **0.40** because in churn prediction,
a missed churner (false negative) costs ~₹8,000 in lost CLV, while an unnecessary
retention campaign (false positive) costs ~₹400. At threshold 0.40 the model saves
approximately ₹1,12,000 more in avoided churn loss compared to threshold 0.50,
at an additional outreach cost of only ₹6,400.

---

## Key Churn Drivers

**Increases churn risk:** `return_rate_180d`, `negative_ticket_rate_90d`,
`avg_discount_pct_180d`, `category_diversity_180d`

**Reduces churn risk:** `loyalty_tier_Platinum`, `frequency_180d`,
`marketing_consent_Yes`, `acquisition_channel_Organic`
