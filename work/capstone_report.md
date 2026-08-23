# Which Content Should Editors Refresh First?

**Author:** thany-8  
**Lane:** Content refresh prioritization  
**Repository:** [github.com/thany-8/content-refresh-prioritizer](https://github.com/thany-8/content-refresh-prioritizer)  
**Analysis date:** 2026-08-22

## Abstract

This study asks which existing pages an editor should review first when refresh capacity is limited. It uses the bundled FlyRank anonymized starter release: 30,000 content items from 32 pseudonymous clients, with trailing-90-day search and engagement measurements. I compare a transparent rule score with Logistic Regression, a constrained Decision Tree, and a Random Forest, keeping label-derived fields and identifiers out of the features and holding out entire clients for evaluation. On the reference client holdout, Random Forest raised precision among the first 50 recommendations from 0.24 for the rule baseline to 0.74, while the held-out average precision rose from 0.468 to 0.618. The result supports a human-reviewed refresh queue; it does not show that refreshing a recommended page will cause traffic to improve.

## Introduction / Problem Statement

An editor may manage thousands of pages but have time to refresh only a small batch each week. The useful decision is therefore not “is this page good?” but “which page should be inspected first?”

The unit of analysis is one pseudonymized content item. The output is a ranked score with a suggested action and reason codes. A human editor uses the queue to choose a review batch, checks each page in context, and decides whether to refresh, expand, investigate click-through rate, investigate engagement, or monitor. A false positive costs review time; a false negative can leave a declining, still-visible page unattended.

## Data

The analysis uses `data/raw/content_refresh_anonymized.csv`, the bundled FlyRank starter release. It contains 30,000 rows and 44 columns at one row per pseudonymized content item across 32 pseudonymous clients. Its metrics summarize a trailing 90-day snapshot; the starter release does not expose exact calendar start and end dates, so this paper does not invent them or make time-forward claims.

The release includes search visibility, clicks, sessions, content age, freshness, position, click-through rate, engagement, and content-shape fields. Rate columns are stored as percentages multiplied by 100, so `ctr = 0.76` means 0.76%. An `avg_position` value of zero means no position data, not rank zero.

I excluded `trend_direction` and `trend_pct` because they define the target. I also excluded recent-versus-previous 30-day fields that overlap the label window, provider/model metadata, pseudonymous IDs, and text identifiers. `client_id` is used only to separate clients during validation; it is never a model feature. The paper contains no client names, URLs, domains, titles, keywords, or raw queries.

## Methodology

### Target and assumptions

The binary proxy target is `is_declining_label = (trend_direction == "down")`. It identifies observed decline inside the release snapshot. It is not a future outcome and is not evidence that a refresh would reverse the decline.

Numeric features include demand, content length, log-transformed traffic volume, days with activity, age and freshness, click-through rate, average position, engagement, scroll rate, and AI-traffic share. Categorical features include competition level, content type, intent, age and freshness tiers, word-count tier, impression tier, and position tier. Missingness is handled inside preprocessing rather than treating every missing value as a measured zero.

### Baseline

The baseline is a readable rule score. It prioritizes pages that are stale but visible, visible with low click-through rate, thin but visible, or old while still ranking on page one. Visibility breaks ties. The baseline uses the same held-out rows and ranking metrics as the learned models.

### Models and validation

The reference pipeline compares Logistic Regression, a constrained Decision Tree, and a constrained Random Forest with random seed 42. The main evaluation holds out complete clients, producing 27,675 training rows and 2,325 test rows. This tests transfer to unseen pseudonymous clients and prevents pages from the same client appearing on both sides.

Model selection uses precision@50 first, then average precision and ROC AUC as tie-breakers. Precision@50 answers the operational question: among the first 50 pages sent to an editor, what share carries the observed declining label?

### Leakage checks

The pipeline asserts that label sources, identifiers, and excluded overlapping windows never enter the feature matrix. Models are fit only on the training partition. Baseline and model scores are evaluated on the identical held-out client partition.

## Results

| Method | ROC AUC | Average precision | Precision@20 | Precision@50 | Precision@100 |
|---|---:|---:|---:|---:|---:|
| Rule baseline | 0.627 | 0.468 | 0.15 | 0.24 | 0.36 |
| Logistic Regression | 0.700 | 0.522 | 0.35 | 0.40 | 0.44 |
| Decision Tree | 0.742 | 0.575 | 0.50 | 0.54 | 0.53 |
| Random Forest | **0.750** | **0.618** | **0.65** | **0.74** | **0.72** |

The observed positive rate across the release is 0.542. On the reference holdout, Random Forest places 37 declining-label pages in the first 50 recommendations, compared with 12 for the baseline. This is a measured ranking improvement on that client split, not a causal business outcome.

![Top model features](../outputs/charts/top_feature_importance.svg)

*Takeaway: days with search impressions, search volume, average position, and content age carry much of the fitted model signal; importance indicates reliance, not causality.*

![Recommended action mix](../outputs/charts/action_mix.svg)

*Takeaway: the queue separates immediate refresh reviews from monitoring and targeted CTR or engagement checks instead of prescribing one action for every page.*

### Split-sensitivity check

A separate Week-5 grouped holdout used 25 training clients and 7 unseen test clients. On that split, Logistic Regression was selected at precision@50 = 0.72 versus 0.38 for the same-split rule baseline; Random Forest reached 0.54. The learned ranking beat the rule again, but the winning model changed. Therefore the defensible conclusion is that learned signals improve prioritization on the tested client holdouts, while the exact winning algorithm is split-sensitive.

## Limitations & Honest Framing

- The target is an in-window decline proxy, not a future label.
- The starter release is a cross-sectional trailing-90-day snapshot with no exposed calendar bounds.
- A client holdout tests transfer across clients but does not test performance through time.
- Only 32 clients are available, and model selection changes across grouped holdouts.
- Feature importance describes what a fitted model uses; it does not identify causes of decline.
- Precision@50 describes the top review batch and does not guarantee value for every lower-ranked page.
- The queue has not been tested in a randomized refresh experiment, so it cannot estimate incremental traffic from taking action.
- Recommendations require editorial review and must not trigger automatic publishing, deletion, or client communication.

## Ranked Recommendations

The repeatable pipeline scores all 30,000 items after validation and combines model probability with the transparent baseline score. It emits a ranked queue with a suggested action, confidence band, and reason codes. The first review batch should be handled in this order:

1. **Review high-confidence visible decline candidates.** Start with high model probability, meaningful impressions, and multiple agreeing reason codes.
2. **Refresh and review CTR.** Inspect title, snippet alignment, search intent, and ranking context for visible pages with weak click-through rate.
3. **Refresh and review engagement.** Inspect whether the page satisfies intent when sessions are meaningful but engagement or scroll signals are weak.
4. **Expand and refresh thin visible pages.** Add missing coverage only after a human confirms that length or topic coverage is genuinely insufficient.
5. **Monitor low-confidence items.** Do not spend scarce editorial time solely because a page has a nonzero model score.

The generated queue contains 3,605 high-confidence, 11,395 medium-confidence, and 15,000 low-confidence items. Suggested actions are 13,093 monitor, 8,178 refresh, 6,657 refresh-and-review-CTR, 1,990 refresh-and-review-engagement, and 82 expand-and-refresh. These counts describe a decision-support artifact, not mandatory work orders.

Before acting, an editor should verify page purpose, current search results, seasonality, business priority, conversion role, previous edits, and whether tracking is complete. Never automate deletion, publication, outreach, or client-facing claims from this score.

## Reproducibility

From a fresh clone:

```bash
git clone https://github.com/thany-8/content-refresh-prioritizer.git
cd content-refresh-prioritizer
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python scripts/run_all.py
```

The command rebuilds cleaned features, the baseline, three models, the ranked queue, charts, JSON metrics, Markdown report, and PDF report. The main pipeline uses random seed 42. Core receipts are `outputs/model_results.json`, `outputs/summary.json`, and `outputs/model_report.md`.

Supporting notebooks:

- [Research question](https://github.com/thany-8/content-refresh-prioritizer/blob/main/work/notebooks/w01_research_question.ipynb)
- [ML task framing](https://github.com/thany-8/content-refresh-prioritizer/blob/main/work/notebooks/w02_ml_task_framing.ipynb)
- [Data contract](https://github.com/thany-8/content-refresh-prioritizer/blob/main/work/notebooks/w03_data_contract.ipynb)
- [Baseline score](https://github.com/thany-8/content-refresh-prioritizer/blob/main/work/notebooks/w04_baseline_score.ipynb)
- [Grouped model comparison](https://github.com/thany-8/content-refresh-prioritizer/blob/main/work/notebooks/w05_model.ipynb)

## Acknowledgments & Data Credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai/). The analysis uses only the bundled anonymized starter release and follows the repository's public data-use rules.
