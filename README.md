# From Data to Decision: Diagnosing & Fixing SaaS Customer Churn

**A product analytics project combining predictive modeling, NLP, and product prioritization**
*(Built on a real SaaS customer churn dataset — 2,500 records)*

---

## Problem Statement

A B2B SaaS company is losing **36.9%** of its customers. The goal of this project
was to go beyond "build a churn model" and answer three questions in sequence:

1. **Diagnose** — what actually drives churn, and can we predict it?
2. **Decide** — what should the product team build first, and why?
3. **Prove** — how do we validate that decision before committing engineering time?

**Dataset:** 2,500 customer records (2,000 train / 500 test) with account age,
login frequency, daily usage minutes, the full text of each customer's most
recent support ticket, and churn outcome.

---

## 1. Diagnose — EDA & Predictive Modeling

### Structured data patterns

| Login Frequency | Churn Rate |
|---|---|
| Daily | 12.6% |
| Weekly | 41.7% |
| Rarely | 72.8% |

| Usage Level | Churn Rate | % of Customers |
|---|---|---|
| Low (<15 min/day) | **81.5%** | 33.5% |
| Medium (15–40 min/day) | 18.2% | 25.6% |
| High (40+ min/day) | 12.0% | 40.9% |

A third of the customer base uses the product less than 15 minutes a day —
and **4 out of 5 of them churn.** This is the single biggest lever in the data.

### Adding the unstructured signal: support ticket sentiment

Using VADER sentiment analysis on each customer's most recent support ticket:

| Ticket Sentiment | Churn Rate | % of Customers |
|---|---|---|
| Negative | **83.5%** | 11.2% |
| Neutral | 34.3% | 64.0% |
| Positive | 22.4% | 24.8% |

Only ~11% of customers file a negative-sentiment ticket, but when they do,
churn is nearly certain — a smaller, high-precision warning sign that a
structured-data-only model would miss entirely.

![EDA Overview](eda_overview.png)

### Model

XGBoost classifier combining usage, account age, login frequency, and sentiment
features:

- **ROC-AUC: 0.806** (test set)
- **5-fold CV ROC-AUC: 0.848 ± 0.024** — confirms the result is stable, not an
  artifact of one lucky train/test split
- **Accuracy: 82.4%** | Precision/Recall (churn class): 0.807 / 0.637

![ROC and Precision-Recall Curves](model_curves.png)

### SHAP driver ranking

| Rank | Driver | Relative Strength (mean \|SHAP\|) |
|---|---|---|
| 1 | Daily Usage (mins) | 1.408 — dominant |
| 2 | Account age | 0.347 |
| 3 | Sentiment (compound) | 0.289 |
| 4 | Sentiment (negative) | 0.158 |
| 5 | Login frequency | 0.098 |
| 6 | Sentiment (positive) | 0.067 |

![SHAP Summary](shap_summary.png)

**Takeaway:** usage depth is, by a wide margin, the strongest churn signal.
Sentiment is a smaller but highly precise complementary signal — useful as an
early-warning flag, but usage is the lever with the most reach.

### Deliverable: risk-scored customer list

Rather than stopping at a summary accuracy metric, the model's predicted churn
probability is exported per customer — a usable artifact a retention team could
act on immediately.

📄 [`top_at_risk_customers.csv`](top_at_risk_customers.csv) — all 500 test-set
customers ranked by churn risk score (0–1), with usage, login frequency, and
their actual support ticket text attached.

---

## 2. Decide — Product Prioritization

Each SHAP driver was translated into a candidate product intervention and scored
with a RICE framework (Reach × Impact × Confidence ÷ Effort):

| Intervention | Reach | Impact | Confidence | Effort | RICE Score |
|---|---|---|---|---|---|
| **Usage-triggered nurture program** — in-app nudges + success-manager check-in for accounts averaging <15 min/day in their first 30 days | 8 | 9 | 8 | 4 | **144** |
| Automated re-engagement email flow (Weekly/Rarely login segments) | 9 | 5 | 6 | 2 | 135 |
| Early-account onboarding push (<60 days old) | 6 | 7 | 7 | 3 | 98 |
| Sentiment-triggered support escalation (auto-flag negative tickets) | 4 | 8 | 8 | 3 | 85 |

**Top priority: Usage-triggered nurture program.** Highest RICE score, and
directly targets the dominant churn driver — a third of the customer base sits
in the low-usage/high-churn segment, so this intervention reaches the most
customers with the most leverage.

*(Sentiment escalation remains a strong, cheap secondary initiative — it
catches a small, near-certain churn segment that a usage-based program alone
won't reach in time.)*

---

## 3. Prove — A/B Test Design & Business Case

### A/B Test Design

- **Hypothesis:** Proactive in-app nudges + a success-manager check-in for
  accounts averaging <15 min/day usage in their first 30 days reduces 90-day
  churn in this segment by at least 15 percentage points (from an 81.5%
  baseline).
- **Primary metric:** 90-day churn rate within the flagged segment
- **Guardrail metrics:** Support ticket volume, CSAT, feature adoption rate
- **Sample size:** calculated via `statsmodels` power analysis —
  **132 customers per arm** for 80% power at α = 0.05, detecting a 15pt
  reduction from an 81.5% baseline
- **Duration:** ~12–14 weeks (30 days to qualify + 90-day churn window)

### Projected Business Impact

Assumptions: 10,000 active customers, $50/mo ARPU, 33.5% of customers in the
low-usage segment (per this dataset's distribution).

| Scenario | Churn Reduction | Annual Retained Revenue |
|---|---|---|
| Downside | 5 pts | $100,500 |
| **Base case** | **15 pts** | **$301,500** |
| Upside | 25 pts | $502,500 |

Even the downside case comfortably justifies the ~4-person-week engineering
effort for the intervention.

### Risks

- **Causation vs. correlation:** low usage may be a symptom of poor product
  fit, not a root cause — mitigate by segmenting the test by persona/use case.
- **Success-manager capacity:** a 33.5%-of-base segment is large; confirm
  staffing before full rollout, or pilot on a random subset first.
- **Nudge fatigue:** cap in-app nudge frequency and test cadence separately.

---
