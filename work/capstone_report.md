# Capstone Report — Lane 4: Position-Adjusted CTR Opportunity Scoring

- **Author:** Shreeyesh Baral
- **Lane:** Lane 4 — CTR Opportunity Scoring
- **Repo:** [https://github.com/shreeyeshbaral/ShreeyeshAssignment1](https://github.com/shreeyeshbaral/ShreeyeshAssignment1)
- **Deployed Paper:** [https://shreeyeshbaral.github.io/ShreeyeshAssignment1/paper/](https://shreeyeshbaral.github.io/ShreeyeshAssignment1/paper/)
- **Date:** August 2026

---

## 0. Abstract

FlyRank builds content as infrastructure — researching, publishing, and optimizing thousands of content assets across client portfolios. To manage performance, FlyRank currently relies on hand-written heuristic flags (such as static CTR thresholds) to spot underperforming content for editorial review. However, fixed rules generate bloated review queues and fail to account for position context or subtle engagement signals. Using 30,000 page-level records sampled from a 79-million-row anonymized production search dataset across 32 clients, we trained a Random Forest classifier to rank published pages by their probability of under-performing their position-tier median CTR, evaluated with a client-grouped holdout split to prevent data leakage across client portfolios. The model achieved Precision@50 = 0.76 and AUC = 0.72 on held-out clients, compared with Precision@50 = 0.40 and AUC = 0.47 for FlyRank's transparent rule baseline — an observed 1.9× improvement in queue precision. Session consistency (`days_with_sessions`) and engagement rate emerged as the most informative ranking signals, showing that on-site user behavior carries predictive value for search CTR gaps. The output is a position-adjusted, volume-weighted ranked action queue with reason codes, enabling content teams to prioritize title, snippet, and metadata optimizations on high-leverage published assets.

---

## 1. Problem framing

FlyRank builds content as infrastructure: it researches, writes, publishes, and continuously monitors search performance across client portfolios. A core operational challenge in managing thousands of published assets is prioritizing content maintenance: **which published pages should content teams review first?**

Currently, FlyRank's product suite uses hand-written heuristic flags (e.g. fixed rules like "flag every page with CTR < 0.5%"). While reliable, fixed threshold rules create severe operational bottlenecks: in this dataset, a single fixed CTR rule flags 9,759 pages — representing roughly 195 review cycles for editorial teams operating at a capacity of 50 pages per cycle. Furthermore, simple rules treat position 3 and position 18 identically and ignore multi-signal engagement context.

The core technical problem is that a low CTR is not an absolute defect — a page ranking at position 18 naturally receives far fewer clicks than a page at position 3. The actionable opportunity signal is whether a page's CTR is **low relative to comparable pages in the same position tier**.

- **Unit of Analysis:** Page (identified by an anonymized content ID).
- **Output:** Position-adjusted, volume-weighted ranked queue with reason codes.
- **Human Action:** Content editors audit titles/meta descriptions (CTR gap), on-page structure (engagement gap), or flag pages for monitoring.
- **Cost of Errors:** False positive = 15–30 minutes of wasted editor time; false negative = compounding wasted search visibility across client portfolios.

---

## 2. Data safety

- **Source:** FlyRank ML Internship dataset (79-million-row warehouse on Hugging Face). Starter sample contains 30,000 page-level records across 32 anonymized clients.
- **Working Slice:** 22,006 rows after excluding pages with `avg_position = 0` (no search data) and `impressions_90d < 100` (insufficient volume for CTR estimation).
- **Excluded Fields (Leakage Prevention):**
  - `ctr`, `clicks_90d`, `clicks_last_30d`, `clicks_prev_30d`: Direct target components.
  - `trend_direction`, `trend_pct`: Label-adjacent thresholded fields.
  - `client_id`: Used strictly for client grouping in split design, never as a model feature.
- **Public Safety:** All URLs, page titles, client names, and raw search queries have been removed.

---

## 3. Baseline

The transparent, hand-written rule baseline uses the formula:

$$\text{score} = \text{visible} \times \text{actionable} \times \log(1 + \text{impressions\_90d}) \times (1 + \text{stale})$$

Where:
- `visible`: $\text{impressions\_90d} \ge 500$
- `actionable`: $3 < \text{avg\_position} \le 20$
- `stale`: $\text{days\_since\_last\_update} \ge 90$

On the 8 held-out client test fold (4,610 rows), the Rule Baseline achieved:
- **AUC:** 0.47 (below random guessing)
- **Precision@10:** 0.20
- **Precision@20:** 0.20
- **Precision@50:** 0.40

---

## 4. Model / analysis

- **Model:** Random Forest Classifier (`n_estimators=200`, `max_depth=5`, `class_weight='balanced'`, `random_state=42`).
- **Target Proxy:** `is_under_ctr` (1 if page CTR < median CTR of its position tier: top_3, page_1, striking, page_3_5, deep; 0 otherwise). Base rate = 46.8%.
- **Features (17 total):**
  - **12 Numeric:** `avg_position`, `impressions_90d`, `days_since_last_update`, `word_count`, `engagement_rate`, `scroll_rate`, `ai_traffic_pct`, `content_age_days`, `days_with_impressions`, `days_with_sessions`, `pageviews_90d`, `sessions_90d`.
  - **4 Categorical (OHE):** `content_type`, `main_intent`, `position_tier`, `freshness_tier`.
  - **1 Engineered:** `has_word_count`.

---

## 5. Evaluation

Evaluated using a 30-client **GroupShuffleSplit** (24 clients train, 8 clients test, 4,610 test rows).

### Head-to-Head Comparison Table (Grouped Test Fold)

| Model | AUC | Precision@10 | Precision@20 | Precision@50 |
|---|---|---|---|---|
| **Rule Baseline** | 0.47 | 0.20 | 0.20 | 0.40 |
| **Random Forest** | **0.72** | **0.70** | **0.80** | **0.76** |

- **Queue Precision Lift:** Random Forest achieved **Precision@50 = 0.76** vs **0.40** for the baseline — an observed **1.9× lift**.
- **Error Analysis:** Errors concentrate near the decision boundary. False positives occur where engagement signals resemble under-performers but CTR barely clears tier median. False negatives occur on pages with normal engagement where CTR gaps stem from uncaptured SERP features or branded intent.

---

## 6. Interpretation

Permutation importance on the held-out test fold identified key signal drivers:
1. `days_with_sessions` (AUC drop: **0.039**): Session consistency reflects steady user interest; sporadic sessions indicate performance risk.
2. `engagement_rate` (AUC drop: **0.022**): Low on-page engagement correlates strongly with search CTR under-performance.
3. `pageviews_90d` (AUC drop: **0.014**): Aggregate traffic context.
4. `impressions_90d` (AUC drop: **0.013**): High impression volume amplifies the value of CTR recovery.

---

## 7. Recommendation

### Ranked Editorial Action Priority

1. **High-Volume Underperformers (342 pages) — Act First:** High impressions + high model score. Audit title and meta description; highest return per editor hour.
2. **Position Decay Risk (971 pages) — Protect Visibility:** Striking distance (positions 11-20) + stale content. Refresh content, check internal linking.
3. **Low Session Consistency (3,884 pages) — Investigate Engagement:** Sporadic traffic. Audit page UX, readability, and depth.
4. **General CTR Opportunity (7,419 pages) — Bulk Review:** Sort by model score descending; batch review in cycles of 50.
5. **Monitor & Re-Score:** Track feature drift monthly (P25-P75 bounds).

---

## 8. Reproducibility

To rerun the analysis from a fresh clone:

```bash
git clone https://github.com/shreeyeshbaral/ShreeyeshAssignment1.git
cd ShreeyeshAssignment1
pip install -r requirements.txt

# Run full reference pipeline
python scripts/run_all.py

# Or run individual notebooks:
# work/notebooks/w04_baseline_score.ipynb
# work/notebooks/w05_model.ipynb
# work/notebooks/w06_validation_audit.ipynb
# work/notebooks/w07_action_playbook.ipynb
# work/notebooks/capstone.ipynb
```

- **Environment:** Python 3.12, scikit-learn 1.9.0, pandas 3.0.5, numpy 2.5.1.
- **Random Seed:** 42.
- **Metric Receipts:** `work/outputs/baseline_metrics.json`, `model_metrics.json`, `playbook_metrics.json`.

---

## 9. Acknowledgments & data credit

Built on the **FlyRank ML Internship dataset** — an anonymized, aggregated release of real production search performance data.

[https://flyrank.ai](https://flyrank.ai)

Thanks to the FlyRank team, track leads Mirza Ašćerić (ML) and Hole (data engineering), and the internship program for making this dataset available for research.
