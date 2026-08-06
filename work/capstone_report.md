# Capstone Report — Lane 4 CTR Opportunity Scoring

- **Author:** Rim Lazrek
- **Lane:** Lane 4 — CTR / Engagement Opportunity Scoring
- **Repo:** flyrank-internship (work notebooks + outputs)
- **Date:** 2026-08-01

> Sections 1–8 mirror the Pass / Needs-Work rubric axes. Sections 0 and 9 are required
> paper sections for the deployed research page.

## 0. Abstract

Editors must decide which pages to review first when click-through rate looks weak relative to similar Google ranks. Using March 2026 page-level search performance from the FlyRank ML Internship dataset (~101k pages; ~62k with at least 500 impressions), we compare a transparent `ctr_gap` ranking rule to logistic regression scored on a client holdout. On the same eligible holdout pool (base rate 46.4%), the rule reaches Precision@50 = 1.00 by construction, while the model reaches 0.84 and surfaces high-impression, non-zero-CTR underperformers the rule buries. The output is a decision-support review queue with action archetypes — not a claim that editing a page will raise clicks, and not a prediction of Google’s ranking algorithm.

## 1. Problem framing

**Decision.** Which pages should a reviewer check first because CTR looks too low for that page’s position tier, even though impressions are high enough to trust?

**Unit of analysis.** One page (`content_hash_id`) per client (`client_hash_id`).

**Output.** A ranked review list scored by `P(is_ctr_underperformer)`, plus action archetypes (`review_title_meta`, `check_tracking_first`, `ranking_not_ctr`, `monitor_only`, `no_action`).

**Human action.** Open the page, inspect snippet/content/tracking, then rewrite title/meta, improve match, check tracking, or monitor.

**Cost of a wrong call.** False alarms waste reviewer time; misses bury fixable high-impression pages.

**Why data/ML helps.** Expected CTR differs sharply by position tier, and thin-volume CTR is noisy. A scored list that respects tier and volume beats a raw “low CTR” flag.

## 2. Data safety

**Used.** FlyRank ML Internship warehouse; `fact_content_daily_performance` aggregated for `month=2026-03` (~101,441 page-rows; 61,924 with `imp_mar >= 500`). Features: `imp_mar`, `ctr_mar`, `pos_avg_mar`, `position_tier`, `engagement_rate_mar` (when GA4 available). Label: `is_ctr_underperformer`.

**Excluded.** Hash IDs as features; `trend_direction` / `trend_pct`; product flags; raw queries; GA4 zeros when tracking unavailable; `ctr_gap` / tier median as model features; overlapping 90-day query facts for this design; `pos_avg_mar = 0`.

**Leakage risks considered.** Label-derived `ctr_gap` and `trend_*` fields; pseudonymous IDs used for grouping/splitting only. Leakage trap: honest five-feature AUC ≈ 0.92 vs leaky AUC = 1.0 when `ctr_gap` is included as a feature.

**Public safety.** No client-identifying names, private URLs, or private queries in `work/`.

## 3. Baseline

**Rule.** Rank `imp_mar >= 500` pages by `ctr_gap` = tier-median CTR − page CTR; tie-break `imp_mar`.

**Why fair.** Transparent, uses the same eligible pool and Precision@K metric as the model, and is the obvious hand rule a team might ship without ML.

**Numbers (eligible client-holdout test, n = 17,110, base rate 0.464).** Precision@10/20/50 = **1.00 / 1.00 / 1.00** by construction. On the full eligible March pool (n = 61,924, base rate 0.362) the same perfect Precision@K holds.

**Limitation of the baseline as a queue.** Perfect Precision@K fills the top with zero-click `top_3` ties — high precision, weak stake ordering for human review.

## 4. Model / analysis

**Method.** Logistic regression (`max_iter=1000`, `random_state=42`) ranking by predicted probability of the underperformer label. Chosen because it is readable and enough to test whether multi-feature scoring improves review order vs the hand rule.

**Features.** `imp_mar`, `ctr_mar`, `pos_avg_mar`, `engagement_rate_mar`, `position_tier`. Left out on purpose: `ctr_gap`, tier median, IDs, trends, product flags.

**Target (proxy).** `is_ctr_underperformer`: CTR below same-tier median and `imp_mar >= 500` — a “worth a look” proxy, not proof the page is broken.

## 5. Evaluation

**Split.** Client holdout (~75% / ~25% of clients; 33 train / 11 test; seed 42). Pages from the same client stay together so site patterns do not leak across the split. Same-month snapshot (not a time forecast).

**Metrics on the same eligible test pool.**

| Method | n test | Base rate | P@10 | P@20 | P@50 |
|---|---:|---:|---:|---:|---:|
| Baseline `ctr_gap` | 17,110 | 0.464 | 1.00 | 1.00 | 1.00 |
| Logistic regression | 17,110 | 0.464 | 0.70 | 0.80 | 0.84 |

**Error / usefulness note.** Baseline wins Precision@K by construction. Model top-10 vs baseline top-10 showed 0/10 overlap in Week 5: the model lifts high-impression, non-zero-CTR underperformers the rule ranks thousands of places lower, and can still include false alarms (high score, label = 0).

**Sensitivity.** A random page split looked weaker (P@50 = 0.80); published skill claims use the client-holdout numbers.

## 6. Interpretation

Main drivers observed in the logistic coefficients were CTR and position tier, with impressions contributing a volume signal. The practical finding is not “ML beats the rule on Precision@K,” but “the rule’s perfect Precision@K is the wrong success story for reviewers.” Negative / careful result: if the only metric is Precision@K on this proxy label, the hand rule looks unbeatable — and that is exactly why stake ordering and archetype actions matter.

## 7. Recommendation

1. Rebuild the monthly queue with the logistic score; sort actionable archetypes first, then score.
2. Prioritize `high_volume_ctr_gap` → review title/meta; `zero_click_visible` → check tracking first.
3. Do **not** treat `deep_tier` as a CTR fix; leave `thin_volume` on monitor-only.
4. Human gates: imp ≥ 500, own-tier median comparison, zero-click SERP check, tracking check.
5. Confidence: moderate for **ranking review attention** on this March slice; low for promising click lifts.

**No-go.** Auto-publish rewrites; prune/merge without a human; act on thin volume; promise click lifts; use `ctr_gap`/`trend_*` as features; ship the raw queue as a client audit.

**Ops.** Retrain/rebuild if top-50 overlap collapses, base rate drifts, client set changes, or GA4 coverage drops. Half-month top-50 overlap was 0.28 — rankings perish quickly.

## 8. Reproducibility

**Commands / path.**
1. Clone the repo; install the project environment.
2. `HF_TOKEN` in `.env` only if regenerating warehouse frames.
3. Run in order: `w03_data_contract.ipynb` → `w04_baseline_score.ipynb` → `w05_model.ipynb` → `w06_validation_audit.ipynb` → `w07_action_playbook.ipynb` → `capstone.ipynb`.
4. Capstone alone reloads committed `work/outputs/*.json` and `work/figures/*.png` (no warehouse required).

**Seeds.** `random_state=42` (model + client shuffle). Receipts: `work/outputs/model_vs_baseline_metrics.json`, `playbook_metrics.json`, `validation_*.json`.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).

---

> **Claims checklist:** observed / measured / directional / decision-support language throughout · Precision@K reported next to base rate · no causal claims · no “predicted Google’s algorithm” · no client-identifying details · numbers match a fresh re-run of the receipts above.
