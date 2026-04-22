# Prompt Surgeon

**Empirical prompt strategy comparison for agentic LLM systems.**

Most prompt engineering is untested opinion. Someone adds a rule because it sounds professional,
a confirmation step because it feels safe, a numbered process because it looks organized, and
ships it without ever measuring whether any of it helped. Prompt Surgeon runs the experiment
instead.

Four structurally distinct prompt strategies were compared across 800 controlled agent trials
with bootstrap confidence intervals. One finding was statistically unambiguous: the most
elaborately engineered prompt catastrophically hurt performance.

---

## Results

Primary model: `claude-haiku-4-5-20251001` | 20 scenarios x 10 replicates = 200 trials per strategy

| Strategy | Words | Success rate | 95% CI |
|---|---|---|---|
| Minimal | 25 | **71.0%** | [65.0%, 77.5%] |
| Instructed | 100 | 70.0% | [64.0%, 76.0%] |
| Structured | 149 | 66.0% | [59.5%, 73.0%] |
| **Overspecified** | **245** | **39.5%** | **[33.0%, 46.5%]** |

**Pairwise comparisons vs. Minimal baseline (95% bootstrap CI on delta):**

| Strategy | Delta | 95% CI | p-value | Verdict |
|---|---|---|---|---|
| Instructed | -1.0pp | [-10.0pp, +8.5pp] | 0.846 | No difference |
| Structured | -5.0pp | [-14.0pp, +3.5pp] | 0.292 | No difference |
| **Overspecified** | **-31.5pp** | **[-40.5pp, -21.5pp]** | **<0.001** | **Significantly worse** |

Minimum detectable effect: **13.4pp** at 80% power, alpha=0.05 (N=200, computed before any trials ran).

---

## What the results mean

**Prompt bloat is actively harmful.** The overspecified strategy, which added mandatory
confirmation gates before every booking, required redundant tool calls even when the user had
already provided the flight ID, imposed emotional labor requirements ("apologize profusely,
offer three alternatives"), and enforced no contractions style rules, dropped success rate by
31.5 percentage points. The 95% CI is [-40.5pp, -21.5pp], cleanly excluding zero on both sides.
This was the clearest signal in this exploration.

**And it cost more to fail.** The overspecified strategy spent $0.982 per 200 trials against
minimal's $0.661, a 49% cost increase, while completing 44% fewer tasks successfully. Verbose
prompts are not just ineffective, but anti-efficient.

**The null results are also informative.** Adding four explicit behavioral rules (Instructed,
100 words) or a full numbered decision process with grounding rules (Structured, 149 words)
produced no statistically significant change from a bare bones 25 word prompt. Both effects are
within the noise at N=200. This does not mean those additions have zero effect, it suggests any
effect they have is smaller than the 13.4pp MDE. The model correctly infers appropriate
tool use behavior from minimal context.

**Opus's diagnosis matched the data.** Claude Opus reviewed the experimental results and
synthesized a revised prompt. Its reasoning: the minimal prompt proves the model understands
the task from tool definitions alone; the overspecified prompt's mandatory steps caused the
model to prioritize protocol compliance over task completion. The Opus revised prompt (which
closely resembled Instructed) scored 70.0% [63.5%, 76.0%] which is statistically indistinguishable
from both Minimal (71.0%, delta=-1.0pp, p=0.855) and the original Instructed strategy.

---

## Study design

**Four strategies across a deliberate spectrum:**

| Strategy | Description |
|---|---|
| **Minimal** | Role declaration + tool list. No behavioral instructions. |
| **Instructed** | Adds four behavioral rules: act don't describe, report errors, ground in tool outputs, confirm actions. |
| **Structured** | Adds a numbered five step decision process and explicit grounding rules. |
| **Overspecified** | Adds mandatory confirmation gates, required redundant tool calls, emotional labor requirements, and style constraints. |

The Overspecified strategy deliberately encodes things practitioners commonly do: mandatory verification steps before irreversible actions, required lookups even when the user specifies the exact flight ID, instruction to pause and ask for user confirmation. The experiment tests whether these "safety" additions help or hurt.

**Benchmark:** 20 airline booking scenarios (8 easy, 9 medium, 3 hard) with a 6-tool deterministic environment. Success evaluated by state-based predicates, checking the actual booking state after each trajectory. Temperature 0.4 for trial-to-trial variance. All scenarios and tool implementations are included in the notebook.

**Statistics:** 2000 sample bootstrap on all rates and pairwise differences. Power analysis
run before any trials. Verdicts use 95% CIs: a result is only called significant when the CI
excludes zero.

---

## Known issue: cross-model cell

The cross model validation cell contains a variable assignment bug. Because Minimal was the
top-ranked strategy, `best_strategy` resolved to `"minimal"` which is the same string as
`baseline_name`. The cell therefore ran Minimal vs. Minimal on Sonnet, producing a
self-comparison (delta=0.0%, p=1.000) with no interpretable meaning.

The bug is fixed below. The fix selects the best non-baseline strategy for
cross-model testing:

```python
# Instead of: best_strategy = summary_df.iloc[0]["strategy"]
ranked = [r for r in summary_df["strategy"].tolist() if r != baseline_name]
best_strategy = ranked[0] if ranked else "instructed"
```

One valid piece of data did emerge from that run: Minimal on Sonnet scored 83.5% [78.5%,
88.5%] across 200 trials, 12.5pp above its Haiku score of 71.0%. The comparison between
strategies on Sonnet was not run; only the absolute Sonnet rate for Minimal is usable.

---

## Reproducibility

**Total study cost:** $6.29 (800 primary Haiku trials: $3.03 | 400 Sonnet trials: $5.06 |
Opus revision: $0.06)

**Wall time:** ~60-120 minutes on free Google Colab

**Models:**
- Primary: `claude-haiku-4-5-20251001`
- Cross-model: `claude-sonnet-4-5-20250929`
- Prompt surgeon: `claude-opus-4-5-20251101`
- Model IDs from `anthropic` SDK v0.96.0

---

## How to run

1. Open `PromptSurgeon.ipynb` in Google Colab
2. Add your Anthropic API key to Colab secrets as `ANTHROPIC_API_KEY`
3. Run all cells in order

The cross-model fix is already applied in the notebook, copy and paste into cell below fic. To adapt to your own agent, replace
`STRATEGIES` (cell 5), `SCENARIOS` (cell 4.2), and the tool implementations (cell 4.1).

---

## Outputs After Running Notebook

```
SURGERY_REPORT.md           full narrative report with all tables
trajectories.json           all 1200 trial records (scenario, strategy, outcome, tokens, latency)
strategy_summary.csv        per-strategy rates, CIs, costs
pairwise_comparisons.csv    pairwise deltas vs. minimal baseline
opus_revision.json          Opus-proposed revised prompt and reasoning
ps_strategy_comparison.png  main results bar chart with CI error bars
ps_heatmap.png              per-scenario success rate heatmap across all strategies
ps_forest_plot.png          pairwise effect forest plot
ps_cost_perf.png            cost vs. performance scatter
ps_beforeafter.png          minimal / minimal / Opus-revised before-after comparison
```

---

## Images Outputted

<img width="989" height="489" alt="image" src="https://github.com/user-attachments/assets/8baccc03-3281-4fd6-9508-0d5f0b36f019" />
<img width="1281" height="390" alt="image" src="https://github.com/user-attachments/assets/0a8f2efb-da9a-47d4-be09-1afa26f1119e" />
<img width="889" height="390" alt="image" src="https://github.com/user-attachments/assets/ac20d58e-78b8-48aa-be52-ac247bf61bfe" />
<img width="790" height="490" alt="image" src="https://github.com/user-attachments/assets/d1955649-8ad5-445b-a79b-8e0ee9e4ee45" />
<img width="889" height="489" alt="image" src="https://github.com/user-attachments/assets/d6ca9135-8db6-4cc3-ad2e-bf1fb33efde3" />


---

## Limitations

- MDE is 13.4pp at N=200. Effects smaller than this are undetectable at this sample size.
  Sentence-level ablation within a single strategy is not viable here.
- 20 scenarios on one task type (airline booking). Generalization to other domains is not
  established.
- Temperature 0.4 is higher than typical production deployment. Results may differ at
  temperature 0.
- Cross-model validation contains a bug in the original run (documented and fixed above).
  The Instructed vs. Minimal comparison on Sonnet was not executed.

---

## Stack

Python · Anthropic API · NumPy · SciPy · pandas · matplotlib · tenacity · Google Colab
