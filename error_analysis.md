# Error Analysis — Churn Prediction Model

## Overview
Model: Logistic Regression | Threshold: 0.40 | Evaluated on: Validation set

| Metric | Value |
|---|---|
| Total False Positives | 37 |
| Total False Negatives | 23 |
| False Positive Rate | 37 / (37 + correct non-churners) |
| False Negative Rate | 23 / (23 + 147 true positives) = 13.5% |

---

## False Positives — Predicted Churn, Customer Stayed

**Business risk:** Unnecessary retention spend — discounts and outreach sent to
customers who would have stayed anyway. Estimated cost: ₹400 per customer.
Total exposure: 37 × ₹400 = **₹14,800**

| # | Customer | Recency (days) | Freq 180d | Spend 180d | Sessions 30d | Return Rate | Neg Ticket Rate | Loyalty Tier | Churn Prob | Interpretation |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | CUST00027 | 70 | 1 | ₹2,128 | 11 | 1.00 | 0.0 | None | 0.52 | High return rate (100%) triggered churn signal despite strong spend; customer returned products but kept buying |
| 2 | CUST00100 | 70 | 1 | ₹372 | 1 | 0.00 | 0.0 | None | 0.68 | High recency gap + low frequency correctly matched churn pattern, but customer made a quiet late purchase |
| 3 | CUST00144 | 86 | 2 | ₹929 | 8 | 0.00 | 1.0 | Gold | 0.61 | Long recency + 100% negative ticket rate flagged as churn; Gold tier loyalty ultimately retained them |
| 4 | CUST00165 | 103 | 3 | ₹1,826 | 2 | 0.00 | 0.0 | Gold | 0.50 | Very long recency (103 days) drove alert; high spend and Gold tier indicate a low-frequency loyal buyer |
| 5 | CUST00177 | 82 | 1 | ₹255 | 12 | 0.00 | 0.0 | None | 0.74 | High sessions (12) but low spend and frequency; browser not a buyer — model correctly flagged risk but customer stayed |
| 6 | CUST00226 | 54 | 3 | ₹2,345 | 6 | 0.33 | 1.0 | Silver | 0.42 | Mixed signals: decent frequency and high spend offset by return rate and poor support experience |
| 7 | CUST00332 | 24 | 2 | ₹2,060 | 0 | 0.50 | 1.0 | Silver | 0.43 | Zero sessions is a strong churn signal; high spend saved the relationship despite complaints and returns |
| 8 | CUST00338 | 12 | 1 | ₹887 | 6 | 1.00 | 1.0 | None | 0.74 | 100% return rate + 100% negative ticket rate + no loyalty tier — legitimate high-risk flag; customer unexpectedly stayed |

**Pattern across all 37 FPs:**
- Mean recency = **95.7 days** — the model is heavily triggered by customers who have
  not purchased recently, even when spend is high
- Mean spend = **₹1,156** — many FPs are actually decent spenders, suggesting the model
  underweights monetary value relative to recency
- 75% have return rate = 0 — return rate is not the main driver of FPs; recency is
- Most have no loyalty tier (NaN) — untiered customers are disproportionately flagged

---

## False Negatives — Predicted Stay, Customer Churned

**Business risk:** Customer lost without intervention. Estimated CLV per customer: ₹8,000.
Total exposure: 23 × ₹8,000 = **₹1,84,000**

| # | Customer | Recency (days) | Freq 180d | Spend 180d | Sessions 30d | Return Rate | Neg Ticket Rate | Loyalty Tier | Churn Prob | Interpretation |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | CUST00093 | 85 | 1 | ₹760 | 16 | 0.0 | 0.0 | Gold | 0.33 | High sessions (16) gave false retention signal; low probability (0.33) despite long recency — engagement was passive browsing |
| 2 | CUST00157 | 0 | 1 | ₹377 | 3 | 0.0 | 0.0 | None | 0.23 | Recency = 0 (purchased on snapshot day) completely masked churn intent; model saw recent activity as strong retention signal |
| 3 | CUST00188 | 29 | 2 | ₹1,880 | 11 | 0.0 | 1.0 | Gold | 0.11 | High spend + Gold tier + moderate frequency produced a very low churn score (0.11) despite 100% negative ticket rate — support dissatisfaction was ignored |
| 4 | CUST00267 | 29 | 1 | ₹442 | 8 | 0.0 | 0.0 | None | 0.11 | No strong signal in any direction; cumulative disengagement not captured — this customer drifted quietly |
| 5 | CUST00327 | 2 | 1 | ₹367 | 7 | 0.0 | 0.0 | Gold | 0.12 | Very recent purchase (2 days) and Gold tier created a strong false retention signal; customer churned immediately after |
| 6 | CUST00337 | 7 | 2 | ₹1,888 | 2 | 0.0 | 0.0 | None | 0.13 | High spend + recent purchase (7 days) masked churn; low sessions (2) was the only warning signal |
| 7 | CUST00514 | 93 | 1 | ₹1,688 | 11 | 0.0 | 0.0 | None | 0.33 | Long recency + high sessions suggests a passive browser with one historical big purchase — model misjudged as retained |

**Pattern across all 23 FNs:**
- Mean recency = **40.5 days** — much lower than FPs (95.7 days); FNs look
  recently active, which is exactly why the model misses them
- Mean sessions = **7.4** — relatively high engagement masks churn intent
- Mean spend = **₹1,442** — FNs are actually higher spenders than FPs, making
  them more costly to miss
- Return rate = 0 for almost all — these customers did not return products, so
  the model had no behavioural red flags to catch

---

## Business Risk Summary

| Error Type | Count | Unit Cost | Total Exposure |
|---|---|---|---|
| False Positives | 37 | ₹400 campaign cost | ₹14,800 |
| False Negatives | 23 | ₹8,000 CLV loss | ₹1,84,000 |

False negatives are **12.4× more expensive** than false positives.
The 0.40 threshold was selected specifically to minimise FNs at the
acceptable cost of more FPs.

---

## Recommendations to Reduce Errors

1. **Add email engagement features** (open rate, click-through) — CUST00188 and
   CUST00327 show that purchase recency alone is misleading; email disengagement
   would have flagged them earlier
2. **Weight support ticket severity** — CUST00188 had 100% negative ticket rate
   but a 0.11 churn probability; ticket severity should override monetary signals
3. **Tiered intervention strategy:**
   - Probability ≥ 0.65 → personal outreach (high-touch)
   - Probability 0.40–0.65 → automated email campaign (low-cost)
   - This reduces FP cost while maintaining recall
4. **Flag customers with recency = 0–7 days separately** — CUST00157 and CUST00327
   show that very recent purchasers can still churn; a separate rule-based flag
   for "purchased recently but low frequency + no tier" would catch these
