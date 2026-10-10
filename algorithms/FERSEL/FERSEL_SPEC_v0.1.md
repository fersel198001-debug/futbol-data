# FERSEL — Football Strength & Match Prediction Algorithm
Version: 0.1 (design baseline; not yet validated)

## Purpose
FERSEL estimates team strength and match probabilities by combining results with the quality of opponents, team-level match statistics, venue splits, recent form, and the gap between expected and observed performance. FERSEL is a candidate research model, not a proven predictor until it passes out-of-sample tests.

## Non-negotiable rules
- Use only information available before the target match kickoff.
- Historical opponent strength and expected performance must be calculated as they were knowable before each historical match; never use later matches to rewrite earlier expectations.
- Use FINISHED matches only for history; exclude the target match itself from all features.
- Scope comparisons by competition and season unless a separately tested cross-competition normalization is introduced.
- No player-level statistics in collection, features, training, or prediction.
- Preserve missing values as NULL; never turn missing stats into zero. Record coverage and use a documented fallback when optional fields are missing.
- Do not replace ALGO V1 or promote FERSEL to production until independent validation supports it.
- No unnecessary schema changes. Supabase remains the source of truth; source/spec is versioned in GitHub.

## Team strength output
Produce an interpretable overall score on a 0–100 scale, plus separate attack and defense ratings. Keep venue and form profiles separate:
1. Overall
2. Home-only
3. Away-only
4. Most recent 5 matches
5. Most recent 1 match
6. Most recent 3 matches

The first score becomes available after five completed matches as an initial estimate, not as a claim of certainty. Ratings update after each subsequent finished match, with shrinkage toward the broader team/competition baseline when the sample is small.

## Inputs (team-match level)
Primary match outcome:
- goals for, goals against, points earned

Priority performance measures:
- total shots
- shots on target
- big chances
- touches in opposition penalty area (only if sourced and present)

Supporting measures when available:
- team xG
- opponent xG as the team's xGA proxy
- xT (only if sourced and present)
- other verified team-level match metrics

The current public.match_stats schema contains shots, shots_on_target, big_chances and xg. It does not currently expose a dedicated opposition-penalty-area-touches or xT column. These must remain unavailable/NULL until a verified source and storage path are established. xGA can be derived per match from the opponent's team xG, where both values are available.

## Opponent-adjusted performance
For every historical match, calculate pre-match expectations from each team's information available before that historical kickoff. Compare:
- expected vs actual goals for and against
- expected vs actual points/result
- expected vs actual shots, shots on target and big chances where coverage is adequate
- xG/xGA gaps where available

Use these residuals as performance signals, not as guaranteed evidence of future improvement. Stabilize one-match outliers and test whether the residual signals add predictive value.

## Form and venue comparisons
Maintain overall, home, away, last-5, last-3 and last-1 views. Compare:
- attack production versus opponent defensive allowance
- defensive allowance versus opponent attack production
- results and points against opponent strength
- current profile against the team's earlier profile
- observed performance against pre-match expected performance

Recent windows must be calculated in chronological order. Small samples should carry wider uncertainty and should not dominate the full rating.

## Match prediction
For a target fixture, combine:
- home team's overall and home ratings
- away team's overall and away ratings
- attack-vs-opponent-defense matchups
- opponent-adjusted performance residuals
- recent direction of change
- data-coverage and sample-size confidence

Return home/draw/away probabilities that sum to 100%, expected goal ranges or score probabilities, the main supporting signals, data coverage, and explicit caveats. Do not output a probability when the required minimum evidence is absent; return insufficient-data status instead.

## Validation protocol
1. Audit data coverage and integrity by competition/season and metric.
2. Establish simple baselines (league home/draw/away rates and a goals-only model).
3. Run chronological walk-forward tests; each prediction must use only earlier matches.
4. Compare FERSEL against the baselines using log loss and Brier score for 1X2 probabilities, calibration, and goal error metrics. Also report sample size and missing-data coverage.
5. Freeze model choices before out-of-sample evaluation. Do not tune on the final test window.
6. Report all tests, including negative results. A hit rate alone is insufficient.
7. Only promote a signal if it improves out-of-sample performance consistently.

## Current implementation status
- Design requirements recorded: yes.
- Supabase project and schema inspected: yes.
- Preliminary row counts: 1,946 matches; 3,892 team-match-result rows; 3,892 match-stat rows.
- Preliminary metric coverage in match_stats: shots 3,870/3,892; shots on target 3,868/3,892; big chances 2,966/3,892; xG 3,052/3,892.
- Opponent penalty-area touches and xT: not represented as dedicated columns in the inspected schema.
- Algorithm code deployed: no.
- Backtest / independent validation completed: no.
