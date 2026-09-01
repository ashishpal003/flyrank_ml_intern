# Capstone Report — Decline Early-Warning

- **Author:** Ashish Pal
- **Lane:** decline early-warning for organic search traffic
- **Repo:** https://github.com/ashishpal003/flyrank_ml_intern
- **Date:** 2026-09-01

> Numbers in this report are cited from the committed receipts under `work/outputs/*.json`
> and reproduce from `work/notebooks/capstone.ipynb` (receipt-driven, runs offline). The
> deployed paper is `docs/index.html`.

## 0. Abstract

**Question.** Which content pages should a FlyRank content strategist review first for
near-future organic-search decline, per client, per month? **Data.** The FlyRank internship
warehouse — ~78.8M pseudonymised daily search rows, 104 clients, 519,606 content items,
2025-01-27 to 2026-06-30 — evaluated at decision date 2026-03-01 on 58,583 eligible pages
across 23 clients (base rate 0.045). **Method.** A frozen transparent hand-rule as the
baseline; logistic-regression and random-forest rankers on ~39 pre-decision GSC and content
features; grouped 5-fold cross-validation by client, a two-month walk-forward, one blind
month, and a planted-leak audit. **Result.** The learned rankers roughly double the
hand-rule on grouped cross-validation (average precision 0.15 vs 0.09, random 0.08), and on
the sealed June 2026 month the random forest reaches average precision 0.44 — above random
(0.34) and the frozen rule (0.39) but only level with a one-line "already-dipping" heuristic
(0.44); the label base rate is non-stationary (0.045 to 0.345 over four months). **For.** An
action playbook that orders a weekly review queue with reason codes, confidence tiers, an
8-item no-go list and a monthly drift alarm — decision support, not automation.

## 1. Problem framing

**Decision supported:** which pages a content strategist puts on this cycle's
refresh/review shortlist, before the traffic loss shows up in standard reporting. **Unit of
analysis:** one (content page x monthly decision date). **Output:** a ranked queue with a
suggested action, reason codes and a confidence tier. **Action a human takes:** open the top
pages, read each recent trend, decide the edit (refresh / expand / rewrite metadata / leave)
themselves — the queue orders attention, it does not act. **Cost of a wrong call:** a false
alarm is ~1-2 wasted editor hours; a miss is weeks of undetected traffic loss and a harder
recovery. Misses cost more, but review capacity is fixed, so the metric is precision@K
reported beside the base rate. **Why ML:** near-future decline in this portfolio depends on
the interaction of position drift, reach thinning, freshness, seasonality and traffic shape
— many signals, tangled, shifting over 17 months; too messy for a fixed threshold, worth
learning.

## 2. Data safety

**Used:** `fact_content_daily_performance` (GSC impressions, clicks, impression-weighted
position, impression-days aggregated over `[D-90d, D)`), `dim_content` (age, word/char
count, content type, competition), `dim_clients` (history start dates, for eligibility and
the grouped split). **Excluded on purpose, with reasons:**

- `trend_direction`, `trend_pct`, `is_declining_label` — the label is derived from them
  (using them means learning the rule, not the world).
- Any column whose window overlaps `[D, D+30d]` — future information.
- `fact_content_query_90d` — its single fixed 90-day window (2026-04-02 to 2026-06-30) sits
  after every development decision date; not leakage-safe for a past->future label.
- FlyRank product flags (`health_score`, `priority_score`, `action_type`) — decision-derived;
  a baseline to beat, never a feature. (They are not shipped in the data anyway.)
- GA4 columns — only ~4% of rows in the label month carry real GA4 data; the rest are
  zero-filled or NULL placeholders.
- `last_optimized_date` — populated only from ~2026-04, so unknown as of D.

**IDs:** `client_hash_id` / `content_hash_id` are pseudonyms, used for grouping, joining and
splitting only — never as features. **Confirmed:** no client names, domains, URLs, page
titles or raw search queries appear anywhere in `work/` or `docs/`.

## 3. Baseline

A frozen transparent hand-rule (ML-07): `final = 0.40 * traffic_softening + 0.30 *
position_slip + 0.30 * reach_thinning`, each component a 0-1 percentile rank, no fitted
weights. A fourth candidate — page visibility (log impressions) — was dropped before
freezing because a signal check showed bigger pages declined *less* in this data (Q4-Q1
decline lift -0.07). It is a fair comparison because it is scored on the same eligible
slice, the same forward label and the same metric as the models, and re-scored on the same
grouped-CV test folds. On the cleaned, client-grouped slice it scores precision@50 = 0.116,
precision@5% = 0.105, average precision = 0.088, per-client precision@10 = 0.212 (base rate
0.045).

Note: on the *contaminated* slice (before the label-quality fix in section 5) this same
rule scored precision@50 = **0.98** — an artifact of one client's data dropout dominating
the top of the list. The honest number is 0.12.

## 4. Model / analysis

**Method:** the decision is "which pages first?" — a ranking over a yes/no observed label —
so every model outputs a probability, ranked and scored at precision@K. Models, simplest to
strongest: `DummyClassifier(strategy="prior")`, `LogisticRegression` (StandardScaler
pipeline, `class_weight="balanced"`), `DecisionTreeClassifier(max_depth=3)`,
`RandomForestClassifier(n_estimators=200, max_depth=10, min_samples_leaf=25,
class_weight="balanced_subsample")` — configs mirror `scripts/03_train_model.py`; all
`random_state=42`.

**Features (~39, all knowable strictly before D):** `imp_90d`, `log_imp_90d`, `clk_90d`,
`ctr_90d`, `imp_last30`, `imp_prev60`, `clk_last30`, `softening_ratio` (`imp_last30 /
(imp_90d/3)`), `pos_last30`, `pos_prev60`, `pos_gap`, `days_impr_90d`, `days_impr_last30`,
`days_impr_prev30`, `reach_ratio`, `content_age_days`, `days_since_update`, `word_count`,
`char_count`, `search_volume`, `competition`, `category_count`, `backlinks`, four `has_*`
missingness flags, and one-hot `content_type` / `main_intent` / `competition_level`.
**Left out on purpose:** everything in section 2's exclusion list; and the `naive
"already-dipping"` rank (biggest recent drop) is kept only as a comparison ranker, never
a feature.

**Target:** `label_decline = 1` if the page's impressions over `[D, D+30d]` run below 75%
of its trailing-90-day pace **and** the second half of the label month is no higher than
the first (a sustained decline, not a one-day spike).

## 5. Evaluation

**Split:** grouped 5-fold cross-validation on `client_hash_id` (seed 42) — a client's pages
never span train and test, so the model must generalise to clients it never saw; this
mirrors deployment. Also: a random 5-fold split (reported beside the grouped one — the gap
is the memorisation), a two-month walk-forward (train once at D = 2026-03-01, evaluate
unchanged at 2026-04-01 and 2026-05-01), and one sealed month (June 2026, read exactly once
in `w06_validation_audit.ipynb cell-4c`, metrics in `ml09_validation_audit.json`).

**The label-quality fix.** ML-04's eligibility checked that a client had >= 90 days of
history *before* D but never that it was still reporting *in* the label window. Three
clients stopped after February — every one of their pages had zero label-window impressions
and was auto-labelled *declined*. Removing them: 60,265 -> 58,583 pages, base rate 0.072 ->
0.045. This filter uses outcome-window information (a churn-censoring choice), disclosed in
Limitations.

**Metrics, model vs baseline, same split (base rate 0.045):**

| ranker | precision@50 | precision@5% | avg precision | ROC-AUC | per-client p@10 |
|---|---|---|---|---|---|
| random | 0.064 | 0.077 | 0.081 | 0.499 | 0.133 |
| stale-visible rule | 0.088 | 0.087 | 0.082 | 0.501 | 0.153 |
| **frozen hand-rule** | 0.116 | 0.105 | 0.088 | 0.490 | 0.212 |
| dummy (prior) | 0.064 | 0.086 | 0.080 | 0.500 | 0.129 |
| **logistic regression** | 0.228 | 0.224 | 0.150 | 0.612 | 0.236 |
| decision tree (depth 3) | 0.084 | 0.098 | 0.090 | 0.537 | 0.150 |
| **random forest** | 0.212 | 0.217 | 0.152 | 0.643 | 0.269 |

- **Random vs grouped:** the random forest scores average precision 0.199 / ROC-AUC 0.808
  under a random split vs 0.149 / 0.650 grouped, and its fold-to-fold spread widens from
  +/-0.02 to +/-0.15 — the random split reports a falsely confident number.
- **Overfit check:** random-forest train average precision 0.33 vs out-of-fold 0.15 —
  some client-specific memorisation, expected with ~18 training clients per fold.
- **Walk-forward:** at the higher April/May base rates (0.22 / 0.20) the random forest holds
  up (average precision 0.34 / 0.24 vs frozen rule 0.22 / 0.25, naive 0.33 / 0.23).
- **Sealed June (base rate 0.345):** random forest average precision 0.44 vs random 0.34,
  frozen rule 0.39, naive "already-dipping" 0.44. It beats random and the rule; it ties the
  one-liner.

**Errors.** The queue head is concentrated in a few clients and in mid-volume pages the
model flags on its own (only ~5 of the top 20 out-of-fold rows actually declined, vs a base
rate of 0.045 — a 5.5x lift, but most top picks do not decline). The two characteristic
failure modes — pages whose traffic is actually *rising* while a signal fires, and small
pages riding position noise — are turned into policy holds in the playbook.

## 6. Interpretation

**What the model leans on.** `feature_importances_` and linear coefficients over-weight
`word_count` / `char_count` (near-duplicates). Permutation importance — the honest check —
puts `pos_last30` (current ranking position), `content_age_days` (page age), and
`softening_ratio` (recent traffic vs its own pace) on top; all plausibly precede a
near-future impressions decline, and the top feature's absolute correlation with the label
is only 0.11 (nowhere near a relabelled outcome).

**Surprises / negative results.**
- The intuitive heuristic — rank by biggest recent drop — scores *below* the base rate in
  the development month. Mean reversion: a page that just dropped tends to bounce.
- The label base rate is not a fixed decline frequency — it is a panel-trend statistic. It
  swings 0.045 -> 0.219 -> 0.198 -> 0.345 across four months with the panel-wide impressions
  total. This is the single most important finding for anyone deploying this.
- The frozen rule's headline 0.98 precision@50 was an artifact; on honest data it barely
  beats random.

## 7. Recommendation

The output is the **ML-10 action playbook**: a queue of all 58,583 eligible pages scored by
`final_score = 100 * (0.70 * model_out-of-fold_score + 0.30 * frozen-rule_percentile)` — a
**model score, not a probability**.

- **Reason codes** (rule + model + two policy codes) and a **suggested action** on every
  row: `refresh` (43), `review_position` (5,637), `review_metadata` (3), `monitor` (20,885),
  `monitor_only` (31,944), `watchlist` (71).
- **Three confidence tiers:** high 804 / medium 28,489 / low 29,290. `high` is deliberately
  hard: top-20% score AND real volume AND model + rule agreement AND no policy hold.
- **Two policy holds from the error analysis:** `durable_riser_hold` (traffic rising ->
  never refresh) and `low_volume_position_only` (position noise on a small page ->
  watchlist). ~15 of the top 50 are held for human judgement.
- **An 8-item no-go list:** never auto-edit; never refresh a riser; never act on position
  noise alone; never call the score a probability or a prediction; never claim a refresh
  prevents a decline; never rank across clients without per-client context; never run the
  queue while the drift alarm is tripped.
- **Monitoring:** an outcome-drift alarm (recompute the label base rate monthly vs 0.045)
  that trips in 3 of 4 months, and a PSI feature-drift harness (`content_age_days` shows
  major drift by May). A tripped trigger pauses the next weekly queue pending human
  re-review and a refit — it never rescinds actions already taken.

**How an editor uses it tomorrow:** open the weekly per-client queue, work down the
`high` and `medium` rows, skip `monitor_only` / `watchlist`, and re-run the base-rate check
before trusting the order.

**Confidence and limits, explicitly:** modest, fragile skill. The ordering beats a frozen
rule and random on honest evaluation and one blind month; it does not beat a one-line
heuristic on that month; the base rate is unstable. **The evidence supports testing this
ordering as decision support. It does not support automating actions.**

## 8. Reproducibility

**Environment:** Python 3.12, `scikit-learn` 1.9.0, `numpy` 2.5.1, `pandas` 3.0.5,
`duckdb` 1.5.5, `matplotlib` 3.11.1; `requirements.txt` in the repo; seed **42** for every
stochastic component.

**Re-run from a fresh clone:**

1. Request access to `FlyRank/internship-warehouse` on Hugging Face (instant) and put a
   READ token in `.env` at the repo root as `HF_TOKEN=hf_...` (gitignored).
2. Run `work/notebooks/w03_data_contract.ipynb` -> `w07_action_playbook.ipynb` in order
   (each writes its `work/outputs/mlXX_*.json` receipt; `w03`-`w07` query the warehouse).
3. `work/notebooks/capstone.ipynb` runs **offline** from the committed receipts + figures
   and reproduces every number in this report.

**Sealed-evaluation receipts (both committed):** the frame builder is
`work/notebooks/w06_validation_audit.ipynb` `cell-4c` (the only cell that reads
`month=2026-06`); the metrics file is `work/outputs/ml09_validation_audit.json`
(`sealed_june_2026`). "Evaluated once, blind" is checkable from the repo.

**Result -> notebook map:**

| result | regenerated by |
|---|---|
| data contract / eligibility funnel | `w03_data_contract.ipynb` -> `ml04_contract_checks.json` |
| frozen baseline metrics | `w04_baseline_score.ipynb` -> `ml07_baseline_metrics.json` |
| grouped 5-fold model comparison | `w05_model.ipynb` -> `ml08_model_comparison.json` |
| random-vs-grouped, walk-forward, sealed June | `w06_validation_audit.ipynb` -> `ml09_validation_audit.json` |
| Figure 1 (precision@K), Figure 2 (base-rate drift) | `w07_action_playbook.ipynb` exports cell -> `work/figures/*.png` |
| action queue + playbook | `w07_action_playbook.ipynb` final cell -> `ml10_playbook.json` |

## 9. Acknowledgments & data credit

Built on the **FlyRank ML Internship dataset** — https://flyrank.ai. Crediting the data
source is standard research practice and a required section; it does not replace citations.

**References:**
- Google, *Rules of Machine Learning* — https://developers.google.com/machine-learning/guides/rules-of-ml
- scikit-learn, *GroupKFold* — https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.GroupKFold.html
- scikit-learn, *RandomForestClassifier* — https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.RandomForestClassifier.html

---

> **Claims checklist:** language is observed / measured / directional / decision-support
> throughout; every precision@K carries its base rate; no causal claims; no "predicted
> Google's algorithm"; no client-identifying details; numbers match a fresh re-run of
> `work/notebooks/capstone.ipynb`.
