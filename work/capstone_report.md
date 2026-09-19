## 0. Abstract

This project investigates whether a ranking system can help prioritize content pages for human review when their search-performance trend is associated with decline. The analysis uses 30,000 pseudonymized content items across 32 clients and defines a decline proxy from the dataset's `trend_direction` field. A transparent rule-based baseline was compared with Logistic Regression using a client-grouped holdout and Precision@50 as the primary metric. On the held-out clients, the baseline achieved Precision@50 of 0.400 while Logistic Regression achieved 0.380, with a decline-proxy base rate of 0.511. The resulting system is therefore positioned as decision-support for human review rather than as proof of future decline or an automatic content-refresh mechanism.

## 1. Problem framing

The decision supported by this work is which content pages an SEO or content team should review first for possible refresh.

The unit of analysis is one pseudonymized content item/page. The output is a ranked priority score and review queue.

A false positive can cause a team member to spend time reviewing a page that does not require immediate attention. A false negative can cause a potentially weakening page to be missed.

The role of ML is to combine multiple metadata signals into a ranking score that can support prioritization. The system is not intended to automatically determine which pages must be changed.

## 2. Data safety

The analysis uses the provided FlyRank internship starter dataset containing 30,000 pseudonymized content items across 32 pseudonymized clients.

The unit of analysis is one content item/page. The dataset contains aggregated search-performance measurements rather than individual user-level events.

Identifiers such as `content_id` and `client_id` were not used as model features. `client_id` was used only to create the client-grouped train/test split.

The decline proxy is defined from the dataset's `trend_direction` field. Because `trend_direction` and `trend_pct` are directly related to the outcome definition, they were excluded from the model features.

Recent 30-day and previous 30-day performance fields were also excluded because they directly define the trend window. Overlapping 90-day performance fields and derived performance indicators were excluded to reduce the risk of temporal leakage.

A deliberate leakage test using `trend_pct` alone produced an accuracy of 1.000. This was used only to demonstrate the leakage risk and was not treated as model performance.

The final feature set contains metadata and content characteristics that were kept separate from the decline-proxy definition. Missing values were handled through imputation and missingness indicators rather than blindly treating missing values as zero.

## 3. Baseline

The baseline is a transparent ranking rule designed to prioritize pages for human review.

It combines three percentile-ranked signals:

- Search-volume rank: 50%
- Days-since-last-update rank: 30%
- Content-age rank: 20%

Higher search demand increases the potential priority of a page, while longer time since update and greater content age increase its review priority.

The baseline produces a ranked queue rather than a probability of decline. It is intended to provide a simple and interpretable reference point before introducing a learned model.

For the final client-grouped test set, the baseline achieved Precision@20 of 0.500 and Precision@50 of 0.400.

The baseline was compared with Logistic Regression using the same held-out clients and the same Precision@K metric.

## 4. Model / analysis

The learned approach uses Logistic Regression to produce a score for ranking content pages.

The model uses seven numeric features:

- `search_volume`
- `competition`
- `cpc`
- `word_count`
- `char_count`
- `content_age_days`
- `days_since_last_update`

It also uses seven categorical features:

- `competition_level`
- `content_type`
- `main_intent`
- `age_tier`
- `freshness_tier`
- `word_count_tier`
- `char_count_tier`

Numeric features were median-imputed with missingness indicators and standardized. Categorical features were imputed and one-hot encoded with unknown-category handling.

The model was trained using a client-grouped 80/20 holdout. The training set contained 25 clients and 23,837 rows, while the test set contained 7 clients and 6,163 rows. There was no client overlap between the two sets.

The model was evaluated using Precision@20 and Precision@50, with Precision@50 selected as the primary metric because the practical decision involves prioritizing a small review queue.

The model produced a Precision@50 of 0.380 on the held-out client set. Its top 50 ranked items contained 31 false positives.

The strongest absolute model coefficients included `freshness_tier_181+` (-0.876), `main_intent_navigational` (-0.544), and `age_tier_181-365` (-0.334). These coefficients describe associations learned by the fitted model relative to their reference categories and should not be interpreted as causal effects.

## 5. Evaluation

The primary evaluation uses a client-grouped 80/20 holdout.

The training set contains 25 clients and 23,837 rows. The test set contains 7 clients and 6,163 rows. There is no client overlap between the two sets.

| Method | Precision@20 | Precision@50 |
|---|---:|---:|
| Decline-proxy base rate | 0.511 | 0.511 |
| W04 Rule Baseline | 0.500 | 0.400 |
| Logistic Regression | 0.300 | 0.380 |

The model therefore did not outperform the transparent baseline at Precision@50.

A validation audit also compared the model under a row-random split and a client-grouped split. Precision@50 changed from 0.780 under the row-random split to 0.380 under the client-grouped split. This demonstrates that the choice of validation design materially affects the measured result.

The final evaluation should therefore be interpreted using the client-grouped result rather than the more optimistic row-random result.

## 6. Interpretation

The Logistic Regression model's strongest absolute coefficients included:

| Feature | Coefficient |
|---|---:|
| `freshness_tier_181+` | -0.876 |
| `main_intent_navigational` | -0.544 |
| `age_tier_181-365` | -0.334 |
| `days_since_last_update` | +0.329 |
| `content_age_days` | -0.327 |

These coefficients describe associations learned by the fitted model relative to the relevant reference categories. They are not causal effects.

The error analysis found 31 false positives among the top 50 model-ranked items. This means that many high-scored items were not in the decline-proxy group on the held-out client set.

The signal audit also found weak standalone relationships for the simple signals tested. Search volume versus impressions had a correlation of 0.001, days since last update versus CTR had a correlation of -0.021, and content age versus CTR had a correlation of 0.009.

An important negative result is therefore that the learned model did not improve the ranking metric over the transparent baseline in the final client-grouped evaluation.

## 7. Recommendation

The final output should be used as a ranked human-review queue rather than an automatic content-refresh system.

Pages near the top of the queue can be reviewed first using the model score and associated reason codes as supporting context.

A reviewer should consider search intent, factual accuracy, recent developments, existing performance, content relevance, and whether changing the page could remove useful information.

The system should not automatically delete pages, rewrite factual claims, change search intent, redirect pages, publish model-generated edits, or treat a high score as proof that a page will decline.

The final client-grouped evaluation produced Precision@50 of 0.380 for Logistic Regression compared with 0.400 for the transparent baseline. This result supports using the model as an experimental decision-support tool rather than replacing the simpler baseline or making automatic content decisions.

Stronger claims about future decline or the effect of refreshing content would require additional time-forward validation and, for refresh-effect claims, an appropriate experimental or causal design.

## 8. Reproducibility

The analysis uses Python and the packages specified in the repository `requirements.txt`.

The main reproducibility requirements are:

- Random seed: `42`
- Client-grouped holdout using `GroupShuffleSplit`
- Test size: `0.20`
- Logistic Regression `max_iter=2000`
- `class_weight="balanced"`
- Same Precision@50 definition for model and baseline
- Same held-out client set for model-versus-baseline comparison

The workflow can be reproduced by cloning the repository, installing the requirements, and running the notebooks in `work/notebooks/` in sequence.

The committed notebooks contain the feature definitions, validation design, leakage checks, model configuration, evaluation results, and action-playbook logic used for the analysis.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset.

Data and internship context: https://flyrank.ai

### Run commands

```bash
git clone https://github.com/SymbolPamnani/Flyrank-ML-internship.git
cd Flyrank-ML-internship
pip install -r requirements.txt