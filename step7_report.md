# Step 7 report - final PoC packaging and deployable onset rule

## Scope completed
1. Refit the Stage A onset model with leave-one-session-out validation.
2. Generated per-session onset trace plots for XY1, XY2, LM, and KH.
3. Compared the current operational rule against an LM-safe fallback rule.
4. Froze a deployable onset rule and a compact feature list for the PoC.

## Stage A model summary
- Model: elastic-net logistic regression
- Target: Fresh vs Not-Fresh
- Inputs: 18 baseline-normalized 30 s window features
- Temporal score: EWMA of P(Not-Fresh), alpha = 0.2
- Validation: leave-one-session-out

## Main result
### Standard operational rule
- Confirmed onset = EWMA P(Not-Fresh) >= 0.60 for 30 s
- Hit rate on 4 sessions: 2/4 (0.50)
- Early false alarms: 2
- Late detections: 0

### Internal engineering fallback
- Before 20 min: threshold 0.60
- After 20 min: threshold 0.45
- Persistence: 30 s
- Hit rate on 4 sessions: 2/4 (0.50)
- Early false alarms: 2
- Late detections: 0

## Important caution
The fallback was tuned on the same 4 sessions used during development. It is promising, but not yet validated on new riders.

## Recommended PoC freeze
1. Watch: EWMA P(Not-Fresh) >= 0.45 for 30 s
2. Confirmed onset: EWMA P(Not-Fresh) >= 0.60 for 30 s
3. Internal fallback for difficult riders: use 0.60 before 20 min and 0.45 after 20 min
