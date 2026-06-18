# Model Card — Customer Churn Prediction Model

---

## Model Details

| Field | Value |
|---|---|
| Model type | Logistic Regression |
| Framework | scikit-learn Pipeline (preprocessor + classifier) |
| Version | 1.0 |
| Saved file | `model.pkl` (joblib format) |
| Snapshot date | 2025-09-30 |
| Target window | 2025-10-01 to 2025-11-29 (60-day forward churn label) |
| Selected threshold | 0.40 |

---

## Intended Use

**Primary purpose:** Identify customers at risk of churning in the next 60 days so the
retention team can trigger targeted intervention campaigns before the customer leaves.

**Intended users:** Retention marketing team, CRM analysts, customer success managers,
and product teams running loyalty initiatives.

**Intended workflow:**
1. On or after each monthly snapshot date, score all active customers using `model.pkl`
2. Any customer with predicted probability ≥ 0.40 is flagged as high churn risk
3. Route flagged customers to a tiered retention campaign based on probability band:
   - Probability ≥ 0.65 → personal outreach or high-value discount
   - Probability 0.40–0.65 → automated email retention campaign

**Out-of-scope uses:** This model is not designed for predicting churn beyond 60 days,
for B2B or bulk-order accounts, or for making account termination decisions.

---

## Data Used

- **Dataset:** `rfm_modeling_snapshot.csv`
- **Snapshot date:** 2025-09-30
- **Train / Validation / Test split:** Used the pre-assigned `split` column from the dataset

| Split | Rows (approx) |
|---|---|
| Train | ~800 |
| Validation | ~200 |
| Test | ~336 |

**Features used (all confirmed leakage-safe — available on or before 2025-09-30):**

| Feature group | Columns |
|---|---|
| Recency | `recency_days` |
| Frequency | `frequency_30d`, `frequency_180d` |
| Monetary | `monetary_180d`, `avg_order_value_180d` |
| Engagement | `sessions_30d`, `pages_per_session_30d` |
| Support | `negative_ticket_rate_90d`, `tickets_90d` |
| Returns | `return_rate_180d` |
| Discount behaviour | `avg_discount_pct_180d` |
| Customer attributes | `loyalty_tier`, `acquisition_channel`, `marketing_consent`, `category_diversity_180d` |

**Columns excluded:**
- `customer_id` — identifier only, no predictive signal
- `snapshot_date` — constant value across all rows (2025-09-30)
- `churn_next_60d` — the target variable; using it as input would be direct leakage
- `split` — partition label; must not influence model training

**Preprocessing:**
- Numerical features: median imputation for missing values
- Categorical features: most-frequent imputation + one-hot encoding (unknown categories
  handled with `handle_unknown='ignore'`)

---

## Model Approach

### Baseline: Logistic Regression
A Logistic Regression with `max_iter=1000` and `random_state=42` was trained as the
baseline model. It provides globally optimised linear coefficients across all features
and serves as the interpretable reference model.

### Stronger model: Random Forest Classifier
A Random Forest with `n_estimators=300`, `max_depth=8`, `min_samples_split=10`,
`min_samples_leaf=5`, and `random_state=42` was trained to capture non-linear
interactions between features.

### Model selection
Logistic Regression outperformed Random Forest across all validation metrics and was
selected as the final model:

| Metric | Logistic Regression | Random Forest |
|---|---|---|
| Accuracy | 0.82 | 0.80 |
| Precision | 0.81 | 0.80 |
| Recall | 0.76 | 0.73 |
| F1 Score | 0.79 | 0.76 |
| ROC-AUC | 0.883 | 0.880 |

Random Forest underperformed because `max_depth=8` constrained the trees from learning
complex interactions, and the sparse one-hot encoded categorical features (loyalty tier,
acquisition channel) are better handled by Logistic Regression's global regularisation.

---

## Performance (Test Set, Threshold = 0.40)

| Metric | Value |
|---|---|
| Accuracy | 81.5% |
| Precision | 78.2% |
| Recall | 87.5% |
| F1 Score | 82.6% |
| ROC-AUC | 0.886 |
| PR-AUC | 0.879 |

**Confusion matrix (test set):**

| | Predicted: Stay | Predicted: Churn |
|---|---|---|
| **Actual: Stay** | 127 (TN) | 41 (FP) |
| **Actual: Churn** | 21 (FN) | 147 (TP) |

The model correctly identified 147 of 168 churning customers (87.5% recall) while
missing only 21 churners and generating 41 false alarms.

### Threshold justification
The default threshold of 0.50 was lowered to 0.40 because:
- In churn prediction, a missed churner (false negative) results in permanent revenue
  loss estimated at ₹8,000 CLV per customer
- An unnecessary retention campaign (false positive) costs approximately ₹400 per customer
- False negatives are therefore ~20× more expensive than false positives
- At threshold 0.40 vs 0.50, the model saves approximately ₹1,12,000 in avoided churn
  loss at an additional outreach cost of only ₹6,400

---

## Key Features Driving Predictions

### Strongest churn drivers (increase churn probability)
| Feature | Interpretation |
|---|---|
| `return_rate_180d` | Customers who frequently return products are more likely to stop purchasing |
| `negative_ticket_rate_90d` | Poor customer service experiences significantly increase churn risk |
| `avg_discount_pct_180d` | Discount-reliant customers have weaker brand loyalty |
| `category_diversity_180d` | Customers buying across many categories show less stable purchasing behaviour |

### Strongest retention signals (decrease churn probability)
| Feature | Interpretation |
|---|---|
| `loyalty_tier_Platinum` | Platinum members have significantly lower churn risk |
| `frequency_180d` | Repeat purchasing is the strongest indicator of retention |
| `marketing_consent_Yes` | Opted-in customers are more engaged and less likely to leave |
| `acquisition_channel_Organic` | Organically acquired customers show stronger long-term loyalty than paid-channel customers |

---

## Limitations

1. **Single snapshot training** — the model was trained on one snapshot date (2025-09-30).
   Customer behaviour changes seasonally and the model will degrade without periodic
   retraining on fresh data.

2. **Missing behavioural features** — email open rates, app usage, wishlist activity,
   and purchase conversion rate are not included. Their absence is the primary cause of
   false negatives where customers appear engaged but have stopped purchasing.

3. **Small training set** — approximately 800 training rows limits generalisation to
   niche segments such as B2B buyers, very high-CLV customers, or newly acquired customers
   with limited history.

4. **Binary output only** — the model predicts will/won't churn within 60 days. It does
   not predict timing, reason for churn, or which retention action is most likely to work.

5. **Threshold is assumption-dependent** — the 0.40 threshold was derived from estimated
   CLV and campaign cost assumptions. If these business parameters change, the threshold
   must be re-evaluated.

6. **No handling of cold-start customers** — customers with fewer than 2 purchases have
   insufficient history for reliable prediction.

---

## Ethical Risks

1. **Discount inequity** — customers flagged as high churn risk may systematically receive
   larger discounts than loyal customers who pay full price, creating a price-fairness
   problem for the most loyal segment.

2. **Acquisition channel bias** — the model penalises customers from paid channels by
   treating organic acquisition as a strong retention signal. Customers from paid channels
   may be systematically over-targeted without addressing the root cause (channel-product
   fit or onboarding quality).

3. **Loyalty tier blind spots** — as seen in error analysis, Gold tier customers
   (CUST00188, CUST00327) can have very low churn scores despite clear warning signals.
   High-CLV customers may be under-protected by the model.

4. **Feedback loop risk** — if retention offers are always given to predicted churners,
   the model will never observe what happens when a predicted churner receives no
   intervention. This causes distribution shift and model degradation over time.

5. **No fairness audit** — the model has not been evaluated for differential performance
   across demographic groups, geographic regions, or customer segments. Bias across these
   dimensions is unknown.

---

## Monitoring Needs

| Signal | Frequency | Alert threshold |
|---|---|---|
| Churn rate in scored population vs. baseline | Monthly | >10% relative shift |
| Model recall on newly labelled cohort | Monthly | Falls below 0.80 |
| Feature distribution drift (PSI) for top 5 features | Monthly | PSI > 0.20 for any feature |
| False negative rate by loyalty tier | Quarterly | FNR > 30% for any tier |
| False positive rate by acquisition channel | Quarterly | FPR > 40% for any channel |

**Recommended retraining cadence:** Every 3 months, or immediately when recall on a
labelled holdout drops below 0.80.

---

## When NOT to Use This Model

- **Do not use** for customers with fewer than 2 purchases — insufficient history for
  reliable scoring
- **Do not use** for B2B or bulk-order accounts — their churn patterns differ
  significantly from the D2C customers this model was trained on
- **Do not use** during or immediately after major promotional events (sales, launches)
  that distort baseline purchasing behaviour
- **Do not use** if more than 90 days have passed since the last retraining cycle
- **Do not use** as the sole basis for any account-level punitive or termination decision
- **Do not use** without human review for customers in the top 1% CLV segment, where
  the cost of a missed churner or an unnecessary intrusive offer is disproportionately high
- **Do not use** if the feature pipeline has not been validated against the current
  snapshot date — stale features will produce unreliable scores
