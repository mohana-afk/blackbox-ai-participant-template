# round-1 — Observe
**Team:** BB-019
**Queries used:** 63 / 150

## What we concluded

The GK-03 score is affected by multiple input features, but their effects are not equally strong.

The strongest observed region was near the upper end of account age, balance, and trust-related settings, combined with low utilisation and favorable channel/settings. The highest score observed was **0.9890 (APPROVE)**.

Key observations:

- **Account age has a strong positive effect.** Scores increased substantially as account age moved from 18 toward the 60–75 range. For example, 18 days produced 0.7211 (Q28), while 60 days produced 0.9279 (Q29), and 74–75 days reached approximately 0.9890 in the later experiments.
- **Trust score has a strong effect.** Changing trust score produced large score changes, including an observed decline from 0.5346 to 0.4314 when testing a high trust-score setting, while a low trust-score setting produced much higher scores in another controlled experiment. This indicates that trust score is not behaving according to a simple real-world assumption and needs to be interpreted strictly from observations.
- **Utilisation has a substantial effect.** Higher utilisation settings produced significantly different scores. The later experiments reached 0.9345 and then approximately 0.9890 as utilisation was adjusted toward the tested high-scoring region.
- **Channel has a relatively small effect compared with the strongest features.** In the high-scoring region, D produced 0.9890, C 0.9858, B 0.9864, and A 0.9829 (Q45–Q50).
- **Amount appears weak or inactive in some regions.** In one controlled region, amount 0 and amount 50 both produced 0.9459 (Q31–Q32), while amount 100 later reached 0.9890.
- **Beneficiaries showed little effect in tested regions.** For example, Q8 and Q9 both produced 0.7204 despite beneficiaries changing from 3 to 6.
- **Linked cards did not show a strong effect in the high-scoring region.** The score remained around 0.9890 across several tests where the other important inputs were held near the best observed configuration.
- **Recent chargebacks did not behave monotonically.** The tested values produced different scores, but increasing chargebacks did not simply cause a consistent decrease. This suggests either a nonlinear effect or interaction with another feature.
- **Months active also appears potentially nonlinear or weak in some tested regions.** We observed changes when varying it, but do not yet have enough controlled evidence to claim a simple monotonic relationship.

## How we got there

We started with a baseline configuration and changed one or a small number of inputs at a time.

The baseline was:

- Score: **0.5346**
- Decision: **APPROVE**
- Account age: 46.5
- Account balance: 50
- Amount: 50
- Beneficiaries: 3
- Channel: A
- Linked cards: 10

We then explored individual features and combinations.

Important comparisons included:

- Q1 vs Q2/Q3: large score changes under trust-related testing.
- Q28 vs Q29: account age 18 → 60 increased the score from **0.7211 → 0.9279**.
- Q30: reducing balance to 0 gave **0.9110**, showing that balance matters but does not completely determine the score.
- Q31 vs Q32: amount 0 → 50 produced the same observed score of **0.9459** in that region.
- Q25–Q27: channels B/C/D gave very similar scores around **0.945–0.946**.
- Q45–Q50: in a stronger configuration, channel differences were still relatively small, with D reaching **0.9890**.
- Q52/Q57: account ages 73 and 72 produced **0.9884** and **0.9877**, respectively.
- Q59–Q60: the strongest region remained at **0.9890** even after small changes to account age/balance.

The experiments therefore moved from broad baseline probing toward searching for a high-scoring region and then checking whether small changes disturbed that region.

## What we ruled out

We did not find evidence that any single feature completely determines the decision.

We also ruled out several simple assumptions:

- Beneficiaries are **not strongly controlling the score** in the tested regions.
- Channel is **not the dominant driver**; channel changes usually caused only small score differences compared with major feature changes.
- Amount does **not always have a meaningful local effect**; some controlled changes produced identical scores.
- Small changes near the high-scoring region do **not necessarily change the score**.
- Recent chargebacks do **not appear to have a simple monotonic relationship** with score based on the tested values.
- We cannot explain the system using ordinary real-world financial intuition alone; the challenge explicitly requires conclusions to come from observed black-box behavior.

## What we are still unsure about

Several questions remain unresolved:

1. We do not yet know the exact mathematical form of the score function.
2. We do not know the exact decision threshold separating APPROVE and DECLINE.
3. Some features may interact with each other, so isolated one-variable conclusions may not generalize to every region.
4. We have not exhaustively mapped the effects of trust score, utilisation, months active, and recent chargebacks across their full allowed ranges.
5. The observed score ceiling of **0.9890** may be a real maximum, a clipping value, or simply the best point found within our query budget.
6. We have not yet established whether the unusual behavior of some features is caused by nonlinear transformations, interactions, or hidden thresholds.

Overall, Round 1 established several strong empirical relationships and identified a high-scoring region, but the exact decision rule remains unresolved.
