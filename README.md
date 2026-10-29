# Smart Case Router: ML-assisted triage and tenure-aware routing (prototype)

A small prototype that predicts the severity of an ad-appeal case from its text,
then routes it to the right reviewer: sensitive cases to experienced agents,
simple cases to newer agents, and uncertain cases to a human for triage.

> **All data is synthetic.** Every case description was generated for this
> project. No real cases, policies, or internal data from any employer are used.
> Results show how the *approach* behaves, not how it would perform in production.

## Product brief

**Problem.** When cases are assigned manually in arrival order (first-in,
first-out), complex or sensitive cases can reach reviewers who are still
learning the policy, while experienced reviewers spend time on simple cases.
Shift leads also spend time sorting cases by hand.

**Users.**
- *Reviewers* get cases that match their experience.
- *Shift leads* stop sorting cases manually and review only the uncertain ones.
- *Policy and quality teams* get fewer sensitive cases handled by the wrong reviewer.

**Solution.** Three steps:
1. A text classifier predicts severity (Low / Medium / High) with a confidence score.
2. A rule engine applies a tenure policy: pushbacks and High/Medium go to
   tenured agents; Low goes to newer agents first, overflowing to tenured agents
   when newer agents are full.
3. If model confidence is below a threshold, the case goes to human triage.

**Success metrics.**
- *Primary:* share of sensitive cases (true High severity or pushback) that reach tenured agents.
- *Guardrail:* share of cases sent to human triage (automation rate) and classification accuracy on auto-routed cases.

## Results (synthetic data, 600 held-out cases)

**1. Classifier vs. baselines**

| Model | Accuracy | Macro-F1 | High-severity recall |
|---|---|---|---|
| Majority class | 0.390 | 0.187 | 0.000 |
| Keyword rules | 0.870 | 0.870 | 0.979 |
| TF-IDF + Logistic Regression | 0.870 | 0.869 | 0.900 |

Honest reading: on this synthetic data the keyword rules match the model.
That is expected, because I wrote the rules against the same phrases used to
generate the data. This test favors the rules.

**2. Unseen wording (24 hand-written cases, directional only)**

| Model | Accuracy |
|---|---|
| Keyword rules | 0.333 |
| TF-IDF + Logistic Regression | 0.958 |

Honest reading: this test favors the model. The rules fail on any wording they
were not written for. The 24 cases are few and share vocabulary with the training
templates, so 0.958 is an upper bound. Together, the two tests show the trade-off
but do not prove the model would win on real cases.

**3. Confidence threshold (human-in-the-loop trade-off)**

| Threshold | Share auto-routed | Accuracy on auto-routed |
|---|---|---|
| 0.0 | 100% | 0.870 |
| 0.6 | 88% | 0.922 |
| 0.7 | 86% | 0.936 |
| 0.8 | 84% | 0.938 |
| 0.9 | 40% | 0.925 |

See `threshold_tradeoff.png`. A threshold of 0.7 keeps about 86% of cases
automated at about 94% accuracy. Raising it to 0.9 sends most cases to humans
without improving accuracy further, so more caution is not automatically better.

**4. Routing simulation (FIFO vs. ML-assisted routing)**

18 agents (10 tenured, 8 newer), 40 cases per agent, 242 sensitive cases.

| Metric | FIFO | ML-assisted (threshold 0.7) |
|---|---|---|
| Sensitive cases sent to newer agents | 42.6% | 0.0% |
| Sensitive cases sent to tenured agents | 57.4% | 82.2% |
| Sensitive cases in human triage or waiting | 0.0% | 17.8% |
| All cases assigned automatically | 100% | 85.7% |
| All cases sent to human triage | 0% | 14.3% |

The cost of the safer routing is that about 14% of cases need a human to look at
them first. That is a product decision: tune the threshold to match how much
review capacity the team has.

## Limitations

- Synthetic data: results are not evidence of real-world performance.
- Text is short and templated. Real cases are longer, noisier, and multilingual.
- TF-IDF with logistic regression is a simple baseline. It reads words, not meaning.
- The simulation ignores arrival times, shift schedules, skills beyond tenure, and case age.
- 8% label noise was added to mimic reviewer disagreement, but its pattern is simplistic.

## Responsible-use notes

- Severity predictions are a routing aid, not an enforcement decision.
- Uncertain cases go to humans by design.
- Real deployment would need to check for uneven error rates across languages or
  regions, and keep a feedback loop so reviewers can correct wrong routing.

## Roadmap

1. Replace TF-IDF with a multilingual embedding model and test on real labeled data.
2. Add case age and queue length, so older cases are prioritized.
3. Run a small pilot with a control group, measuring sensitive-case placement,
   handling time, and quality score.
4. Let reviewers flag mis-routed cases and retrain on that feedback.

## How to run

```bash
pip install -r requirements.txt
python data_gen.py          # creates cases.csv (synthetic)
python train.py             # trains model, prints metrics, saves chart
python paraphrase_eval.py   # unseen-wording test
python simulate.py          # FIFO vs ML-assisted routing
python demo.py "Ad tells voters the polling place is closed when it is open."
```

## Files

| File | Purpose |
|---|---|
| `data_gen.py` | Generates synthetic cases with class overlap and label noise |
| `train.py` | Trains TF-IDF + logistic regression, evaluates, compares baselines |
| `router.py` | Tenure-aware routing rules with a confidence threshold |
| `simulate.py` | Compares FIFO with ML-assisted routing |
| `train_rules.py` | The keyword-rule baseline, shared by `paraphrase_eval.py` |
| `paraphrase_eval.py` | Tests on wordings the model has not seen |
| `demo.py` | Routes a single case from the command line |
