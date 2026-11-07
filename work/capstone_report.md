# Capstone Report — CTR / Engagement Opportunity Scoring

- **Author:** Luka Đurić
- **Lane:** CTR / Engagement Opportunity Scoring (ranked review queue, decision-support)
- **Repo:** https://github.com/LukasDjuric/FlyRank_intership
- **Date:** 2026-11-07

> All numbers below come from one month of outcomes (June 2026) on one portfolio of clients. Language throughout: observed / measured / directional / decision-support. No causal claims.

## 1. Problem framing

**Decision.** Which content pages should an editor or SEO analyst review first for a possible CTR / metadata problem, given limited review time?

**Unit of analysis.** One row = one client x content item (a pseudonymized page), with features aggregated from March to May 2026 and outcomes measured in June 2026.

**Output.** A ranked review queue. Each page gets a score, one reason code, one action label and an evidence-strength note. The queue is capped at 50 pages per cycle, with no client above 30% of it.

**Action a human takes.** Compare the title and snippet with search intent and the observed CTR gap (`review_ctr_metadata`), review a slipping page for a possible refresh (`review_content_refresh`), or first check data / technical causes (`check_technical_first`, `verify_data_first`, `verify_then_review`). A person approves any edit.

**Cost of a wrong call.** Reviewer time spent on a page that should not have been seen first (opportunity cost), and a possible bad edit if the human check is skipped. A false technical-check route also uses data or engineering time. With an assumed 12 minutes per review, a 50-page cycle is 10 reviewer hours (Section 7).

**Task type.** Ranking / scoring, not plain classification (the goal is ordering pages under limited capacity) and not clustering (pages are ordered against measured outcomes, not grouped without labels). The metric is **Precision@K** reported next to the base rate.

**Two outcomes (both observed in June, after all feature windows):**
- `y_decline`: June impressions below 80% of May impressions. Defined for every eligible page.
- `y_low_ctr`: June CTR below the expected CTR of the page's position tier. Defined only where June impressions are at least 100 (65,088 of 90,966 pages).

**Why data / ML helps at all.** The pattern depends on several tangled signals (exposure, position tier, CTR, momentum) that differ by client. A one-line rule is a fair start but is a starting point only, so the work first builds a transparent rule, then tests whether a learned ranking beats it on clients it has not seen. GA4 engagement data is only partly available (4.2% of March rows), so CTR gap is the main signal and engagement is not modeled.

## 2. Data safety

**Data used.**
- Weeks 1-2: the anonymized 30,000-row starter CSV (framing, first discoveries, leakage demo).
- Weeks 3-7: the Hugging Face release `FlyRank/internship-warehouse` (build v20260703), read with DuckDB without downloading it. Daily facts for 2026-03-01 to 2026-06-30 (43,647,556 rows), joined to `dim_content` and `dim_clients`.
- After the history and activity filters, the cached feature table has 296,221 client x content rows from 48 clients. After eligibility (at least 100 impressions in March-April, at least 50 in May, known position), **90,966 pages from 37 clients** remain.

**Contract checks (W3, March partition).** Grain probe returned 0 violations (one row per date x client x content). GA4 data was available in 413,966 of 9,841,378 rows (4.2%), so engagement is not a complete source of truth. Mean daily CTR 0.308% vs weighted CTR 0.293% (weighted = total clicks / total impressions).

**Deliberately excluded columns and why.**
- `trend_direction`, `trend_pct` (starter CSV): the starter label is computed from them, so they are never features.
- All June columns and the outcome columns: they define the label window.
- `baseline_action_score`: the baseline is something to beat, never an input.
- `client_hash_id`, `content_hash_id`: pseudonyms used for grouping and joining only.
- Raw queries, URLs, titles, keywords, client names: not present in the release and never shown.
- Product decision flags (`health_score` etc.): not in the data; if rebuilt they would be circular.

**Leakage considered.**
- Timeline: features March-May, labels June, no overlapping window.
- Injected-leak tests: adding a June-derived column pushes P@50 to 1.000 in every fold (AUC 1.000) for both outcomes (W6); a W4 leaky ranking scores 1.000 at every K. The harness detects leakage.
- Residual mild issue: the tier-expected CTR is computed from March-May of all eligible pages, including pages later in a test fold. It uses no June data but is not strictly train-only.
- A `y_low_ctr` caveat that tests cannot remove: the features include March-May CTR, so part of the skill is persistence of an existing CTR gap.

**Public safety.** Nothing client-identifying appears in `work/`, `docs/` or the paper. Row-level CSVs are gitignored; only aggregates, figures and metric JSONs are committed.

## 3. Baseline

**Rule in plain words.** A page is worth reviewing first if it is visible enough to matter, and either its visibility is slipping against its own history or its CTR is below what pages in the same position tier normally get.

```
slip       = how far May impressions fell below the page's own Mar-Apr monthly average (0 to 1)
ctr_gap    = how far CTR is below the expected CTR of its position tier (0 to 1)
visibility = percentile rank of Mar-May impressions (0 to 1)
score      = visibility x (0.5 x slip + 0.5 x ctr_gap)
```

Expected CTR per tier is total clicks / total impressions (weighted by the denominator). One reason code per page: `slipping_visibility` if `slip >= ctr_gap`, otherwise `ctr_below_tier_expected`; pages with neither signal are `no_signal`.

**Signal checks (W4).**
- CTR by position tier: weighted CTR falls from 0.474% (`top_3`) to 0.035% (`deep`), but `page_1` (0.320%) and `striking` (0.323%) are nearly equal. Verdict **MIXED**.
- June decline rate by May slip bucket: the highest-slip bucket (>= 0.9, n = 1,038) declines less (0.540) than the base rate (0.704). Verdict **OPPOSITE**. Practical meaning: a collapsed May goes to a technical check, not straight to a content rewrite.

**Baseline numbers (all 90,966 eligible pages, frozen, no training).**

| Outcome | Base rate | Ranking | P@20 | P@50 | P@100 |
|---|---:|---|---:|---:|---:|
| `y_decline` | 0.704 | baseline rule | 0.65 | 0.66 | 0.79 |
| | | visibility only | 0.65 | 0.70 | 0.65 |
| `y_low_ctr` | 0.570 | baseline rule | 0.80 | 0.78 | 0.70 |
| | | visibility only | 0.65 | 0.70 | 0.66 |

The baseline does not beat random picking at the top for `y_decline` (lift 0.92x at K=20, 0.94x at K=50). For `y_low_ctr` it beats the base rate (1.23x to 1.40x). The rule stayed frozen after model work began.

## 4. Model / analysis

**Method.** Readable first, stronger second: logistic regression, a depth-3 decision tree, then a random forest (300 trees, max depth 10, min leaf 50, `balanced_subsample`). Logistic regression and the tree use `class_weight="balanced"`. The forest only counts if it beats both the rule and the simpler models on the same folds.

**Features (15, all March-May):** `log_imp_early`, `log_imp_may`, `log_imp_feat`, `recent_ratio_c`, `slip`, `log_ctr_feat`, `ctr_gap`, `tier_expected_ctr`, `log_pos_feat`, `days_with_imp_feat`, and one-hot position tiers (`tier_deep`, `tier_page_1`, `tier_page_3_5`, `tier_striking`, `tier_top_3`). An assertion checks that no column contains June, an outcome, a score or an ID.

**Targets.** `y_decline` and `y_low_ctr` as defined in Section 1. Both are observed June outcomes, not rule-derived buckets of the feature window.

**Left out on purpose.** The baseline score, all June fields, IDs, raw text fields.

## 5. Evaluation

**Split.** 5-fold `GroupKFold` by `client_hash_id`. Every client is in a test fold exactly once and never in train and test of the same fold (asserted). The baseline rule and visibility-only are scored on the same test rows. Fold test sets hold 1, 7, 10, 8 and 11 clients; fold 1 is a single client with 22,115 pages. A time split is not possible because every page is already predicted forward in time (March-May to June).

**Model vs baseline (mean over 5 grouped folds).**

| Outcome | Base rate | Ranking | P@20 | P@50 | P@100 | AUC |
|---|---:|---|---:|---:|---:|---:|
| `y_decline` | 0.701 | baseline rule | 0.77 | 0.764 | 0.78 | 0.533 |
| | | logistic regression | 0.57 | 0.636 | 0.67 | 0.556 |
| | | decision tree (depth 3) | 0.80 | 0.828 | 0.78 | 0.540 |
| | | **random forest** | **0.86** | **0.852** | **0.83** | **0.597** |
| `y_low_ctr` | 0.566 | baseline rule | 0.70 | 0.756 | 0.71 | 0.590 |
| | | logistic regression | 0.97 | 0.940 | 0.92 | 0.726 |
| | | decision tree (depth 3) | 0.96 | 0.920 | 0.92 | 0.713 |
| | | **random forest** | **0.97** | **0.972** | **0.98** | **0.760** |

**Is the edge stable or only an average?**
- `y_low_ctr`: forest beats the rule in 5/5 folds at P@50 (mean difference +0.216, bootstrap 90% interval +0.120 to +0.312, excludes zero) and in 16/17 clients on within-client AUC (median +0.121). Stable across clients.
- `y_decline`: forest beats the rule in 3/5 folds at P@50 (mean +0.088, interval -0.008 to +0.220, includes zero); one fold (fold 4, forest P@50 1.00) carries the average. Within-client AUC is ahead in 15/19 clients (median +0.065) but absolute AUC is 0.50-0.64. **Directional only.** Logistic regression is below the rule for `y_decline`.
- The bootstrap uses only 5 folds, so intervals are rough.

**Honest split vs naive split.** Random-row CV gives forest P@50 1.000 (AUC 0.694) on `y_decline` against 0.852 (AUC 0.597) grouped by client. The frozen rule shows no gap (0.760 vs 0.764). The gap is client memorization. For `y_low_ctr` the gap is small (0.988 vs 0.972).

**Error analysis.**
- Confident wrong calls (`y_decline`): three high-volume `page_3_5` pages scored 0.91-0.95 fell by only 3-11% (May to June), below the 20% cut that defines a decline. They are near misses (a threshold effect), not unrelated picks.
- `y_low_ctr` confident wrong calls: three `deep`-tier pages with a March-May CTR gap recovered in June. Misses: two pages with no March-May CTR gap fell below tier expectation anyway, so persistence could not help.
- Calibration: predicted values sit about 0.15-0.20 below actual in every position tier (class weighting shifts scores). Scores are for ranking, not probabilities.
- Concentration: in four of five folds the `y_decline` top 50 is 84-100% one client, so P@50 there largely measures one client.

## 6. Interpretation

**What the model leans on.** Permutation importance on held-out fold 1 (AUC drop): `log_pos_feat` 0.038, `days_with_imp_feat` 0.021, `log_ctr_feat` 0.020. Forest impurity importance: `days_with_imp_feat` 0.221, `log_pos_feat` 0.112, `log_imp_early` 0.095. No feature dominates and no June or outcome column is present, so there is no sign of leakage. Plausible links: how steadily a page appears in search, how weak its average position is, and how well it converts the visibility it has.

**Ablations (grouped CV).**
- `y_low_ctr`: removing the CTR family drops AUC from 0.760 to 0.624, while P@50 barely moves (0.972 to 0.944). Exposure + position alone give P@50 0.936. The whole-list ranking depends on March-May CTR, which fits **persistence** (a page that under-captured clicks keeps doing so).
- `y_decline`: removing the momentum family does not hurt (AUC 0.604 vs 0.597), so `slip` adds little beyond exposure and position.

**Negative and surprising results.**
- The slip signal is opposite to the intuition: pages with the largest May collapse decline less in June.
- In the Week-1 starter data, search volume barely tracks impressions (correlation about 0.001 to 0.003) and word count does not separate declining from growing pages. Advice built only on "high volume" or "longer article" is not supported.
- For `y_decline` with a 70% base rate, even a good ranking has little room to show lift.

## 7. Recommendation

**Playbook (90,966 pages, one action each, ordered by severity so the first matching rule wins).**

| Action | Pages | Share | What a human does |
|---|---:|---:|---|
| `monitor_only` | 79,195 | 87.1% | Nothing now; keep monitoring |
| `review_ctr_metadata` | 7,313 | 8.0% | Check title / snippet against intent and the CTR gap |
| `review_content_refresh` | 2,849 | 3.1% | Review decline evidence after technical causes are checked |
| `check_technical_first` | 1,001 | 1.1% | May impressions collapsed 90%+: check indexing, redirects, tracking |
| `verify_data_first` | 342 | 0.4% | Position below 1 or zero clicks on 5,000+ impressions: verify measurement |
| `verify_then_review` | 266 | 0.3% | Extreme rise (top 1% of rises): verify, then review |

Pages with fewer than 5 expected clicks are never sent to review. Review items are ordered by estimated clicks at stake (missed clicks versus tier expectation), not by model score; the model score decides entry to the review pool (top 20%). Every page is scored by a forest that never saw its client.

**Graded on June, clients the model never saw.**
- `review_ctr_metadata`: `y_low_ctr` rate 0.892 vs base 0.570 (1.57x, 7,234 labelled pages). The lift holds in every position tier and expected-click bucket with at least 50 labelled pages (1.47x to 1.74x). Partly persistence.
- `review_content_refresh`: `y_decline` 0.842 vs 0.704 (1.20x). Directional only.
- `check_technical_first`: `y_decline` 0.544, below base: this route is not a decline predictor.
- `monitor_only`: 0.696 and 0.546, at or below base rates.

**Top-50 review queue (client cap 30%).** All 50 items are `review_ctr_metadata`; 9 clients (uncapped: 7 clients, largest 50%). `y_low_ctr` 0.959 (49 labelled pages) vs 0.761 (46) for the frozen W4 top 50. `y_decline` 0.680, below the 0.704 base rate: **this is a CTR/metadata queue, not a decline detector.** The top 20 are mostly high-volume `page_1` / `top_3` pages (16,123 to 280,892 May impressions); all 19 labelled pages kept a low June CTR, which is persistence in its strongest form.

**Cost / value (one month, 50-page budget, 12 minutes per review assumed).**

| Queue | Expected true low-CTR pages | Reviewer hours | Wasted reviews |
|---|---:|---:|---:|
| Playbook review queue | 47.96 | 10.0 | 2.04 |
| Frozen W4 baseline top 50 | 38.04 | 10.0 | 11.96 |
| Random pick | 28.51 | 10.0 | 21.49 |

These are decision-support estimates on one month, assuming unlabelled pages share the labelled rate. They are not a monetary model.

**Human review rules and no-go list.** The queue never authorizes a change by itself. Never auto-publish. `review_content_refresh` is always shown as directional. Review items must pass the checklist (does the reason match the live page and result page? is a sibling page absorbing demand? is there a seasonal explanation? could a snippet or AI answer take the click? is this client already over-represented?). A pre-release check blocks a queue where one client exceeds 30%.

**Monitoring and retrain triggers (proposed, not running).** Before release: row count and columns match, pre-release checks return ok, PSI drift on key features (largest client vs others showed moderate PSI for `log_pos_feat` 0.167 and `slip` 0.226). After the next month's outcomes: recompute P@50 for model and frozen rule and the new base rate. If the model's P@50 is below the baseline's for two consecutive labelled periods, fall back to the rule-only queue until retraining.

**Confidence and limits.** One month, 37 clients, no uncertainty interval on single-month rates, the top 50 comes from 9 clients, `y_low_ctr` partly measures persistence, and there is no causal design. The queue says where to look first; it does not show that editing a page improves it.

## 8. Reproducibility

**Fresh-clone commands (from the repo root):**

```bash
pip install -r requirements.txt
# Weeks 1-2 reference pipeline on the bundled starter CSV
python scripts/run_all.py
# Weeks 3-7 notebooks, in this order (Run all in each):
#   work/notebooks/w03_data_contract.ipynb
#   work/notebooks/w04_baseline_score.ipynb    # needs HF token; writes cached features + baseline_action_score.csv
#   work/notebooks/w05_model.ipynb
#   work/notebooks/w06_validation_audit.ipynb
#   work/notebooks/w07_action_playbook.ipynb   # writes figures and w07_metrics.json
#   work/notebooks/capstone.ipynb              # reads saved outputs only, no warehouse query
```

**Data access.** Request access to `FlyRank/internship-warehouse` on Hugging Face and create a READ token. Supply it with `getpass`, the `HF_TOKEN` environment variable or a local `.env` (gitignored); never hardcode it in a cell. W4 caches the heavy scan to `work/outputs/w04_warehouse_features.parquet`, so later runs do not rescan.

**Seeds.** `SEED = 42` for models, `GroupKFold(n_splits=5)` (deterministic), bootstrap seed 42, `train/test` none (grouped CV only).

**Environment of the reported run.** Python 3.10.11, pandas 2.3.3, numpy 2.2.6, scikit-learn 1.7.2, matplotlib 3.10.9. Tree-ensemble numbers can shift a few points between library versions; the stable claims are the direction and size of the effects, not the third decimal.

**What is and is not committed.** Committed: notebooks in `work/notebooks/`, figures in `work/figures/`, metrics JSON (`work/outputs/w07_metrics.json`), this report, the paper in `docs/`. Not committed (gitignored, CI blocks them): all CSV / parquet files, including the ranked queues with pseudonymous IDs.

**Lineage checks built in.** The frozen baseline reproduces the W4 numbers in later notebooks; the W7 out-of-fold scores match W5's to rounding (max difference 5.6e-16 and 7.8e-16); W7's re-check of W6 headline numbers prints "matches W6".

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support language everywhere · no causal claims without an experiment or causal design · no "predicted Google's algorithm" · no client-identifying details · numbers in this report match a fresh re-run.
> **Metrics vs. base rate:** every Precision@K above sits next to its base rate (0.70 for `y_decline`, 0.57 for `y_low_ctr`); AUC and lift over the baseline are the honest discrimination numbers.
