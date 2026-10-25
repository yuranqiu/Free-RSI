# Free-RSI

Free-RSI investigates the cost and behavior of recursive self-improvement of agent safety harnesses when training-time proposal validation (TPV) is removed or replaced. Historical review is a research hypothesis for retaining useful evidence across updates. Existing experiments are exploratory: they have not established that removing TPV causes overfitting, or that adaptive historical review reliably improves safety, utility, transfer, or efficiency.

**Status, updated October 25, 2026:** experiments are paused following the October 24 review. The latest interpretation is recorded in the [stage summary](docs/stage-summary-20261024.md), with [selected complete final answers](docs/selected-outputs-20261024.md). The sections below retain the earlier baseline results and historical method description; they do not establish a validated final method.

This repository contains documentation only. Code, datasets, complete trajectories, harness artifacts, and backup archives belong to the private [Free-RSI_exp experiment repository](https://github.com/yuranqiu/Free-RSI_exp).

## 1. Current Experiments

We use a trajectory-driven evolution process to modify the System Prompt, Rule Bank, Safety Memory, and Tool Policy, examining safety, task utility, and transfer performance under different update strategies.

We have completed evolution for two baselines and frozen the selected harnesses for evaluation on Agent-SafetyBench (ASB) and AgentHarm across multiple target models. These transfer evaluations use the existing harnesses directly, without repeating evolution for each target model.

| Baseline | Update Strategy | Starting Point for Subsequent Updates |
|---|---|---|
| B1: Performance Gate | Accept a candidate only if clean UBR and attack ASR do not increase, attack UA does not decrease, and the composite score strictly improves; otherwise, restore the best version so far | Best version so far |
| B2: Free Evolution | Retain trajectory execution, scoring, and failure diagnosis, without using performance scores to restrict updates or trigger rollback | Current version |

The comparison encompasses differences in acceptance strategy, the starting point for subsequent updates, rollback, final version selection, and associated prompts.

## 2. Aggregate Results for the Two Baselines

### Agent-SafetyBench

| Baseline | Clean UBR ↓ | Attack ASR ↓ | Attack UA ↑ |
|---|---:|---:|---:|
| B1 | 6.37% | 2.14% | 83.62% |
| B2 | 8.16% | 1.79% | 84.57% |

Clean UBR measures the proportion of cases exhibiting partially or fully unsafe behavior without an attack. Attack ASR measures fully unsafe behavior under attack, while attack UA measures the proportion of safely completed tasks among those eligible for utility evaluation. ASR and UA are first micro-averaged across attack conditions within each model, then averaged with equal weight across models.

### AgentHarm

| Baseline | Harm Score ↓ | Harm Refusal ↑ | Benign NR ↑ |
|---|---:|---:|---:|
| B1 | 10.35% | 87.93% | 74.16% |
| B2 | 14.22% | 82.57% | 71.95% |

Harm Score is the mean completion score for harmful tasks, where lower values indicate greater safety. Harm Refusal measures refusal on harmful tasks. Benign NR is the mean completion score for benign tasks conditional on non-refusal; it is neither the benign non-refusal rate nor overall benign utility.

## 3. Current Findings

**Free evolution improves mean performance under ASB attacks, but the gains do not extend to all metrics.** Compared with B1, B2 reduces attack ASR by 0.35 percentage points and increases attack UA by 0.95 percentage points. Meanwhile, clean UBR rises by 1.79 percentage points, indicating worse safety without an attack.

**The gains on some ASB metrics do not carry over to AgentHarm.** B2 increases Harm Score by 3.87 percentage points, reduces Harm Refusal by 5.36 percentage points, and reduces Benign NR by 2.21 percentage points. Based on the current averages across models, B1 provides better protection against harmful tasks and better benign task completion conditional on non-refusal.

## 4. Our Method: On-Demand Historical Review and Update Correction

Our method builds on B2 free evolution by adding a conditional historical review before applying a candidate patch.

1. Determine whether to review history from the actual patch diff. Review is triggered when a patch overwrites or updates existing content, modifies the tool policy, affects multiple components, or revisits a component with previously applied changes.
2. Retrieve relevant applied changes and initial safety boundaries for the affected components, including before-and-after content, reasons for the changes, and verifiable citations. The initial implementation matches components and prioritizes recent changes, without replaying historical tasks.
3. The History Agent provides specific evidence of conflicts and recommendations, giving the original evolution model one opportunity for targeted revision. Historical constraints may also be adjusted when supported by evidence.
4. Apply the patch after checking its structure, format, and citation validity. Preserve the original proposal, review, revision, and actual changes, then proceed to the next iteration.
