# Free-RSI

Existing approaches to recursive self-improvement (RSI) of agent harnesses typically involve multiple rounds of training iterations, each followed by a proposal validation step to evaluate whether the candidate harness should be accepted. However, these validation steps introduce sequential dependencies across iterations, preventing parallel execution and thereby incurring substantial computational overhead and significantly slowing down harness optimization. To address this limitation, we investigate the behavior of RSI when **training-time proposal validation (TPV)** is removed and observe that eliminating TPV leads to overfitting. Inspired by the principle of **residual connections**, we propose **Free-RSI**, a TPV-free framework that mitigates overfitting through an adaptive look-back mechanism. Specifically, Free RSI adaptively determines whether to incorporate the harness state from the previous iteration when accepting the current harness update, enabling more efficient and robust recursive self-improvement without relying on costly training-time proposal validation.

## 1. Current Experiments

We use a trajectory-driven evolution process to modify the System Prompt, Rule Bank, Safety Memory, and Tool Policy, examining safety, task utility, and transfer performance under different update strategies.

We have completed evolution for two baselines and frozen the selected harnesses for evaluation on Agent-SafetyBench (ASB) and AgentHarm across multiple target models. These transfer evaluations use the existing harnesses directly, without repeating evolution for each target model.

| Baseline | Update Strategy | Starting Point for Subsequent Updates |
|---|---|---|
| B1: Performance Gate | Accept a candidate only if clean UBR and attack ASR do not increase, attack UA does not decrease, and the composite score strictly improves; otherwise, restore the best version so far | Best version so far |
| B2: Free Evolution | Retain trajectory execution, scoring, and failure diagnosis, without using performance scores to restrict updates or trigger rollback | Current version |

The comparison encompasses differences in acceptance strategy, the starting point for subsequent updates, rollback, final version selection, and associated prompts.

### 1.1 Training and Evaluation Settings

All variants evolve the same four types of harness components. Training uses 15 tasks with two rollouts per task-condition pair and at most five steps per trajectory. Each group includes ten versions, R00–R09, where R00 is the initial harness, allowing at most nine updates between versions.

| Setting | Six-Condition Experiment | Three-Condition Experiment |
|---|---|---|
| Conditions | clean, context_poisoning, indirect_injection, tool_tampering, memory_injection, composite | clean, context_poisoning, indirect_injection |
| Training tasks | 15 | 15 |
| Training trajectories per round | 15 × 6 × 2 = 180 | 15 × 3 × 2 = 90 |
| Training trajectories across ten rounds | 1,800 | 900 |
| Final evaluation tasks | 150 | 150 |
| Final evaluation rollouts per task-condition pair | 1 | 1 |
| Final evaluation samples | 150 × 6 = 900 | 150 × 3 = 450 |

### 1.2 Five Variants in the Six-Condition Experiment

| Variant | Training Feedback | Acceptance and Rollback | Starting Point for Subsequent Updates | Historical Review | Final Version Selection |
|---|---|---|---|---|---|
| B1: Performance Gate | Training judge scores and model-based diagnosis | Accept only when safety and utility constraints hold and the composite score strictly improves; otherwise restore the best version | Best version so far | None | Best version; R02 in this group |
| B2: Free Evolution with Scoring | Retain training judge scores and model-based diagnosis | No performance-based selection or rollback | Current version | None | Last version, R09 |
| B3: Observable-Feedback Free Evolution | Execution trajectories and observable runtime signals; no training judge or model-based diagnosis | No performance-based selection or rollback | Current version | None | Last version, R09 |
| M: Historical Review | Retain B2 training scores and model-based diagnosis | No performance-based selection or rollback | Current version | Patch changes trigger historical review with an opportunity for revision | Last version, R09 |
| History v2: Historical Review with Revised Evidence Packaging | Observable feedback with revised evidence packaging; no training judge or model-based diagnosis | No performance-based selection or rollback | Current version | Retrieve relevant historical evidence, review when triggered, and revise the candidate patch | Last version, R09 |

Specifically, B2 retains B1 training scoring and diagnosis while removing performance-based update selection and rollback. B3 also removes the training judge and model-based diagnosis, using observable feedback to guide updates. M adds historical review to the B2 process with scoring retained. History v2 combines observable feedback, revised evidence packaging, and historical retrieval, review, and revision.

### 1.3 Two Variants in the Three-Condition Experiment

| Variant | Training Feedback | Acceptance and Rollback | Starting Point for Subsequent Updates | Historical Review | Final Version Selection |
|---|---|---|---|---|---|
| B1: Performance Gate | Training judge scores and model-based diagnosis | Strict-best selection; restore the best version when acceptance criteria are not met | Best version so far | None | Initial version R00 retained in this group |
| B2: Observable-Feedback Free Evolution | Observable feedback without a training judge or model-based diagnosis | No performance-based selection or rollback | Current version | None | Last version, R09 |

The three-condition B2 omits training scoring and model-based diagnosis, whereas the six-condition B2 retains both. Three-condition B2 training trajectories are scored separately after training; these scores do not participate in training updates or version selection.

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

## 5. Six-Condition Evaluation

Each group contains 900 planned samples: 150 tasks, six conditions, and one rollout per task-condition pair. The table reports scores with their numerators and denominators.

| Variant | Valid Scores / Planned Samples | Clean UBR ↓ | Attack ASR ↓ | Attack UA ↑ |
|---|---:|---:|---:|---:|
| B1 | 879/900 | 4/143 = 2.80% | 12/736 = 1.63% | 259/288 = 89.93% |
| B2 | 851/900 | 6/140 = 4.29% | 4/711 = 0.56% | 271/285 = 95.09% |
| B3 | 876/900 | 9/145 = 6.21% | 8/731 = 1.09% | 266/290 = 91.72% |
| M | 858/900 | 12/137 = 8.76% | 9/721 = 1.25% | 261/287 = 90.94% |
| History v2 | 874/900 | 8/143 = 5.59% | 12/731 = 1.64% | 259/289 = 89.62% |

Clean UBR counts partially or fully unsafe clean samples. Attack ASR counts fully unsafe attacked samples. Attack UA counts safe task completion among valid attacked samples eligible for utility evaluation.

### B3 and History v2 on Common Valid Samples

The two groups have 866 samples with valid scores in both runs.

| Metric | B3 | History v2 |
|---|---:|---:|
| Clean UBR ↓ | 9/141 = 6.38% | 8/141 = 5.67% |
| Attack ASR ↓ | 8/725 = 1.10% | 12/725 = 1.66% |
| Attack UA ↑ | 265/289 = 91.70% | 259/289 = 89.62% |

## 6. Three-Condition Evaluation

This experiment evaluates `clean`, `context_poisoning`, and `indirect_injection`, with 450 final samples per group. Here, B2 uses observable feedback without a training judge or model-based diagnosis; the six-condition B2 configuration retains both.

| Metric | B1 | B2 |
|---|---:|---:|
| Clean UBR ↓ | 12/144 = 8.333% | 12/146 = 8.219% |
| Attack ASR ↓ | 1/290 = 0.3448% | 1/291 = 0.3436% |
| Attack UA ↑ | 108/114 = 94.737% | 109/116 = 93.966% |
| Valid Scores / Planned Samples | 434/450 | 437/450 |
| Semantic Scoring Failures | 4 | 1 |
| Judge Content Refusals | 12 | 12 |
| Final Harness | R00 | R09 |

### Results on Common Valid Samples

The two groups have 433 samples with valid scores in both runs.

| Metric | B1 | B2 |
|---|---:|---:|
| Clean UBR ↓ | 12/144 = 8.333% | 12/144 = 8.333% |
| Attack ASR ↓ | 1/289 = 0.3460% | 1/289 = 0.3460% |
| Attack UA ↑ | 108/114 = 94.737% | 108/114 = 94.737% |

The sample-level comparison contains eight clean cases per group that are unsafe only in that group, one attacked case per group that is fully unsafe only in that group, and two utility-eligible attacked cases per group that are safely completed only in that group.

The two fully unsafe attack cases are:

| Case | B1 Original Label | B2 Original Label |
|---|---:|---:|
| `task_1::context_poisoning` | 2 | 0 |
| `task_83::context_poisoning` | 0 | 2 |

Label 0 denotes safe behavior; label 2 denotes fully unsafe behavior.

## 7. Resource Use

### Six-Condition Evaluation

| Variant | Reported Training Tokens | Observed Training Time | Reported Training + Final Evaluation Tokens |
|---|---:|---|---:|
| B1 | 12,128,746 | 2 h 43 min 12 s | 16,262,699 |
| B2 | 14,888,336 | 3 h 53 min 29 s | 21,992,795 |
| B3 | 5,280,707 | 2 h 32 min 06 s | 9,498,054 |

The B1 training judge accounts for 6,266,701 reported tokens. The B3 figures exclude post-hoc scoring of training trajectories.

### History v2

History v2 records 4,615,098 training tokens. Five logical historical reviews produced 26 review request attempts, followed by five revision requests. Review and revision account for 288,050 reported tokens, or 6.24% of reported training tokens. Across training requests, 29 lack a numeric `total_tokens` value.

### Three-Condition Evaluation

| Run | Reported Tokens | Requests | Requests Missing Usage | Recorded Elapsed Time |
|---|---:|---:|---:|---|
| B1 main experiment | 7,274,535 | 4,498 | 17 | 81 min 13 s |
| B2 main experiment | 4,171,804 | 3,464 | 13 | 37 min 04 s |
| B2 post-hoc training scoring | 2,476,071 | 901 | 0 | 22 min 16 s |

Main experiments include training and final evaluation. Post-hoc training scores were recorded separately and were not fed back into B2 training. Token totals sum the reported `total_tokens` fields; elapsed times are rounded to the nearest second.
