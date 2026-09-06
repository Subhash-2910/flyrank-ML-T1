# Capstone Report — Transparent Refresh Opportunity Scoring

- **Author:** Subhash Kashyap
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** [Subhash-2910/flyrank-ML-T1](https://github.com/Subhash-2910/flyrank-ML-T1)
- **Date:** September 7, 2026

## 0. Abstract

This project asks which content pages should be prioritized for human review when a content team has limited capacity. It uses the FlyRank ML Internship anonymized starter dataset, containing 30,000 pseudonymized content records across 32 clients. I compared a transparent stale-and-visible baseline with Logistic Regression and then re-ran the model with a stricter client-grouped, leakage-audited design. In the initial same-split comparison, Logistic Regression reached Precision@20 of 80.0% versus 45.0% for the baseline; under the stricter grouped validation, Precision@20 was 50.0%. The output is a ranked, human-reviewed action queue that supports content review decisions and does not claim that refreshing a page causes improved search performance.

## 1. Problem framing

This project supports a content or SEO team deciding which pages to inspect first for possible refresh work.

The unit of analysis is one pseudonymized content page. The output is a ranked review queue with a model score, an action label, and a reason code. A human reviewer uses the queue to decide which pages deserve investigation before any content change is made.

A wrong positive recommendation can waste editorial time on a page that does not need intervention. A wrong negative recommendation can leave a potentially important page unreviewed. Data and machine learning can help prioritize limited review capacity by combining several observed page, content, and market signals.

## 2. Data safety

I used the FlyRank ML Internship anonymized starter dataset: `data/raw/content_refresh_anonymized.csv`. It contains 30,000 rows, 44 columns, and 32 pseudonymized clients.

I did not use client names, domains, URLs, titles, raw search queries, credentials, or raw private exports. The fields `content_id` and `client_id` were used only for identification and client-grouped validation; they were never model features.

The outcome was whether `trend_direction` was `down`. I excluded `trend_direction` from the feature set because it is the label. I also excluded `trend_pct`, because it directly measures the magnitude of the same observed trend outcome.

The strict leakage-audited model excluded all 30-day comparison-window columns and 90-day performance totals because they can overlap with the period used to measure the observed trend. This reduced the available feature set but made the final validation claim more cautious.

## 3. Baseline

The initial baseline was a transparent stale-and-visible score:

- 60% relative content age or staleness
- 40% relative current 90-day impressions

Pages meeting the stale-and-visible conditions were assigned the reason code `STALE_VISIBLE_PAGE` and the action label `REVIEW_FOR_REFRESH`.

On the Week-5 held-out client split, the baseline achieved:

| Metric | Baseline |
|---|---:|
| Precision@20 | 45.0% |
| Average Precision | 0.476 |
| ROC-AUC | 0.473 |

This baseline is useful because a human can understand its logic. It was also evaluated on the same held-out client frame as the initial Logistic Regression comparison.

## 4. Model / analysis

I used Logistic Regression because it is transparent, fast to train, and suitable for a ranking-oriented decision-support problem.

The initial Week-5 model used 25 numeric and categorical signals, including content size, search demand, competition, historical visibility, engagement, CTR, average position, freshness, content type, and search intent. Its target was whether the observed `trend_direction` was `down`.

The Week-6 leakage audit identified that several 90-day performance totals could overlap with the time period used to define the trend label. The stricter audited model therefore retained only stable content and market features:

- `search_volume`
- `competition`
- `cpc`
- `word_count`
- `char_count`
- `content_age_days`
- `days_since_last_update`
- `competition_level`
- `content_type`
- `main_intent`

The final claim is based on this stricter feature set. The model measures associations with an observed downward trend; it does not estimate the causal effect of a future refresh.

## 5. Evaluation

The initial Week-5 evaluation used a client-grouped split:

- Training rows: 23,837
- Test rows: 6,163
- Training clients: 25
- Test clients: 7
- Client overlap: none
- Test-set downward-trend rate: 51.1%

On that same held-out split, the initial Logistic Regression outperformed the transparent baseline:

| Approach | Precision@20 | Average Precision | ROC-AUC |
|---|---:|---:|---:|
| Stale-and-visible baseline | 45.0% | 0.476 | 0.473 |
| Initial Logistic Regression | 80.0% | 0.597 | 0.596 |

However, the initial model included 90-day performance totals that the later audit treated as potential outcome-window overlap. Therefore, the 80.0% Precision@20 result is reported as an initial measurement, not the main final capability claim.

The stricter Week-6 model compared random page-level validation with client-grouped validation:

| Validation design | Precision@20 | Average Precision | ROC-AUC |
|---|---:|---:|---:|
| Random page-level split | 55.0% | 0.636 | 0.629 |
| Client-grouped split | 50.0% | 0.529 | 0.533 |

The grouped result is the more honest estimate because it tests on seven clients not seen during training. Its Precision@20 of 50.0% is close to the 51.1% downward-trend base rate in the grouped test set, while Average Precision and ROC-AUC indicate modest discrimination. This means the strict model should be treated as a limited prioritization aid, not a strong standalone classifier.

Error review found false positives: pages with higher predicted downward-trend scores that did not have an observed `down` trend. False negatives were pages with an observed downward trend but low predicted scores. These errors may reflect seasonality, search-intent changes, competition, or client-specific context not represented by the stable feature set.

## 6. Interpretation

The initial model placed notable weight on historical user/session measures, days with impressions, content age, content type, word count, and search intent. Because several historical performance measures may overlap with the outcome period, they were not used in the stricter validation model.

The key negative result is important: once potentially overlapping performance-window features were removed and validation was grouped by client, model performance became much more modest. This suggests that much of the stronger initial result may have depended on signals too close to the measured outcome window.

The project therefore does not claim to predict Google’s algorithm. It measures directional relationships in this anonymized dataset and shows that stable content and market features alone provide limited separation of pages with observed downward trends.

## 7. Recommendation

The action playbook contains 6,163 held-out pages:

| Action / reason code | Pages | Mean model score |
|---|---:|---:|
| `DECLINE_RISK_REVIEW` | 1,414 | 0.613 |
| `STALE_HIGH_DEMAND` | 512 | 0.505 |
| `STALE_THIN_CONTENT` | 203 | 0.557 |
| `LOWER_PRIORITY_MONITOR` | 4,034 | 0.428 |

A total of 2,129 pages, or 34.54% of the held-out queue, were marked `REVIEW_FOR_REFRESH`. The remaining 4,034 pages were marked `MONITOR`.

An editor should use the queue to choose what to inspect first. Before acting, the editor must check current search intent, factual accuracy, freshness, seasonality, competition, indexing or technical issues, and editorial or business value.

Nothing in this playbook should be automated. It must not automatically rewrite, publish, delete, merge, prune, change metadata, or alter URLs. A score is a prompt for human review, not permission to make a content change.

The queue should be reviewed or retrained if Precision@20 materially changes on a new evaluation period, the content/client mix changes, the outcome definition changes, or human reviewers repeatedly identify misleading recommendations.

## 8. Reproducibility

From a fresh clone:

```bash
git clone https://github.com/Subhash-2910/flyrank-ML-T1.git
cd flyrank-ML-T1
python -m pip install -r requirements.txt
```

Run the notebooks in order:

1. `work/notebooks/w04_baseline_score.ipynb`
2. `work/notebooks/w05_model.ipynb`
3. `work/notebooks/w06_validation_audit.ipynb`
4. `work/notebooks/w07_action_playbook.ipynb`

The notebooks use `random_state=42` for Logistic Regression and `GroupShuffleSplit`. The client-grouped split uses a 20% held-out client set. Dependencies are listed in `requirements.txt`.

The Week-7 notebook generates:

- `work/outputs/ranked_action_playbook.csv`
- `work/outputs/action_playbook_metrics.json`
- `work/figures/action_playbook_reason_codes.png`

The CSV is intentionally not committed because it is regenerated by the notebook and remains subject to the repository’s data-safety rules. The metrics JSON and figure are reproducibility receipts for the paper.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).
