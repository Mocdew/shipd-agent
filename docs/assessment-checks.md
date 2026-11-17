# Assessment checks

[← Back to README](../README.md)

This guide describes the rules implemented in this checkout, based on reported assessment feedback and the bundled rubric. It is not independently verified platform policy. Confirm changes against the assessment instructions you receive.

## Prompt quality

The local word limit is **1,000 words**. The checker measures explicit prompt files (`problem.md`, `PROBLEM.md`, `description.md`, or `prompt.md`), not the target repository’s README. If it cannot find a prompt, it cannot verify its length.

“Too Many Requirements” has two inputs:

- The platform’s visible Holistic Check flag, when available.
- A local estimate based on list items, modal verbs, and imperative sentences, ignoring code fences and headings.

The estimate defaults to flagging more than **10** requirements. This number is a project heuristic, not an official published threshold. The current implementation makes a MAJOR finding and fails this part of Phase 1 when it exceeds the threshold. Review the evidence: lists of background facts can overcount, and complex prose can undercount.

Configure `REVIEW_MAX_PROMPT_WORDS` and `REVIEW_MAX_PROMPT_REQUIREMENTS` in `.env`. A focused prompt can still need work on clarity, testability, or repository fit even when these checks pass.

## Agent run matrix

Each of Nova, Vega, and Orion needs at least four finished unhinted runs. More than four is allowed. Hinted runs never count.

On Olympus, the models have separate bars:

- **Orion** solves the task in at least one run and at most 25% of runs.
- **Vega** solves it in at most 25% of runs. Zero is fine.
- **Nova** never solves it.

Orion and Vega may both solve it. Mars keeps the older bar: exactly one model solves the task, above 0% and at most 25%, and the other two stay at 0%.

Olympus long horizon is the average sequence length of **Orion's solving runs**, which must be at least 256,000 tokens. The page must label the figure as sequence length; a generic token count does not count. The checker uses an explicit solving-run average ("Orion solving runs average sequence length", "Long Orion solves: 300K sequence length"), or else averages the lengths on unhinted Orion runs marked PASS. An Orion average over all runs is not the bar. Olympus has no solution-LOC floor and no file or message median. Mars still requires a median of at least 100 LOC when that figure is shown.

| Example | Vega | Nova | Orion | Verdict |
| --- | --- | --- | --- | --- |
| Orion at 25%, solving runs average 300K | 0/4 | 0/4 | 1/4 | Pass |
| Orion and Vega both solve | 1/4 | 0/4 | 1/4 | Pass |
| 2 of 8 is still 25% | 0/4 | 0/4 | 2/8 | Pass |
| Exactly 256K | 0/4 | 0/4 | 1/4 | Pass |
| Orion solving runs average 200K | 0/4 | 0/4 | 1/4 | Fail |
| Only Vega solves | 1/4 | 0/4 | 0/4 | Fail |
| Nova solves | 0/4 | 1/4 | 1/4 | Fail |
| Orion at 50% | 0/4 | 0/4 | 2/4 | Fail |
| Vega at 50% | 2/4 | 0/4 | 1/4 | Fail |
| Fewer than 4 unhinted runs | 0/4 | 0/3 | 1/4 | Fail |

Fractions mean **passing runs / unhinted runs**. The reviewer reads existing results. It does not schedule these evaluations or click run/re-run controls.

## Auto Review prediction

Shipd's Auto Review is the last and hardest stage, and its criteria are not published. The agent predicts it in a final stage that runs after the scored review and before the decision guards. One structured model call reads the draft decision and scores, all earlier findings, the prompt, `test.patch`, `solution.patch`, the agent runs, and the expected agent line-up. It looks for parts that disagree.

- A concrete, cited contradiction is **MAJOR**. It makes the `auto_review` stage FAIL, which turns an approve into request_changes.
- Speculative notes are **QUESTION** and do not fail the stage.
- A test patch with an empty or missing solution is a local MAJOR, even when the model is unavailable.
- When the model errors or `REVIEW_AUTO_REVIEW=0`, the stage is skipped. A skip is not a coverage gap.

Every finding is labeled "Auto Review prediction" and is not Shipd's verdict. `REVIEW_AUTO_REVIEW_MODEL` picks the model and defaults to `REVIEW_MODEL`.

The agent line-up (Vega, Orion, Nova) is a guess, because Shipd does not publish it. Override it with `REVIEW_VEGA_MODEL`, `REVIEW_ORION_MODEL`, and `REVIEW_NOVA_MODEL`, or with the `REVIEW_AGENT_MODELS` JSON. These names are recorded with the review for staleness and never decide a pass or a fail.

## Evidence limits

An unavailable or unreadable Agent Runs panel returns **SKIP**. An aggregate-only line such as `1/12 pass` also returns **SKIP**. It cannot show per-agent unhinted counts or Orion's solving-run average, so it is not a pass. When the solve bar is met but Orion's solving-run sequence length is not labeled, Olympus returns **SKIP** for that missing bar. Cheat rate, fair-task flags, and environment blockers are each skipped when the page does not show them; the summary says so instead of calling them passed.

## Updating the rules

Keep these files aligned when rules change:

- [Rubric](../shipd-rubric.md) and [shared defaults](../shipd-agent/review/rubric_defaults.py)
- [Prompt checks](../shipd-agent/review/prompt_checks.py) and [agent run checks](../shipd-agent/review/agent_runs_checks.py)
- [Configuration](../.env.example) and this guide
- Regression tests for boundary cases and propagation into the final review

```bash
python -m pytest shipd-agent/tests/test_prompt_checks.py \
  shipd-agent/tests/test_agent_runs_checks.py \
  shipd-agent/tests/test_assessment_criteria_flow.py
```
