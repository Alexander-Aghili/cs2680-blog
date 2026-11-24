---
title: "Making Agent Improvements Survive Evaluation"
date: 2026-11-24
authors: ["Alexander Aghili"]
summary: "Across four assignments, improving an agent meant tracing failures, spending compute where progress remained possible, and checking whether local measurements predicted real results."
tags: ["agent-design", "testing", "observability", "serving"]
toc: true
draft: true
---

My agent solved 19 of 25 tasks locally and only 8 of 26 on the leaderboard. The task sets differed, so I could not interpret the gap as a precise regression. It still exposed a problem with how I was evaluating progress. The local result had become my reason to trust the system, even though I had little evidence that it predicted performance on unfamiliar tasks.

Across Assignments 2 through 5, I built a SWE-bench Pro harness, allocated a fixed budget across model tiers, served a model on a single GPU, and evaluated the full stack on Terminal-Bench. Each layer introduced a measurement that could look encouraging while the agent continued to fail. Passing tests, cheaper calls, and faster generation became useful only when I understood how they affected completed tasks.

<!--more-->

## Diagnose the failure before optimizing it

A score tells me how often the system succeeded. A trace can explain what prevented success. That distinction mattered when repeated attempts produced nearly identical failures. In the serving assignment, almost half of the failed gradings failed at least three quarters of their tests. This pattern suggested that many attempts were missing a basic requirement or failing before their implementation could be meaningfully exercised.

Interface mismatches were one example. A patch could contain plausible logic while exposing it in a form that callers could not use. Guidance to preserve the expected public interface was followed by an increase from 5 to 11 solves out of 26. A later regrade reached 8, which limited the size of the improvement I could confidently claim. The useful finding was the mechanism. More generation speed would have helped the agent produce the same incompatible patch sooner.

The agent's own tests also needed scrutiny. In four of five difficult local failures from the budget assignment, its test scripts passed while the hidden evaluation failed. Some scripts exercised the wrong function. In another case, the agent changed a correct fix to satisfy synthetic data it had invented. Because the implementation and the test came from the same interpretation, they could agree while both missed the requirement.

Terminal-Bench extended this problem to the environment. A command available through an interactive shell configuration might remain unavailable to the evaluator. A claimed speed improvement meant little without a comparison against the reference. These failures made me treat completion as an observable condition. The patch needed to expose the required interface, run in the expected environment, and survive checks grounded in the task.

The harness itself required the same scrutiny. In the budget assignment, it dropped the model's reasoning between tool calls. Preserving that state reduced average output from roughly 1,170 to 320 tokens per call and eliminated turns that performed no action in the measured comparison. The solve count stayed unchanged. This was an efficiency improvement with a clear cause, even though the small evaluation showed no accuracy gain.

Other harness bugs distorted the meaning of verification and resource limits. A reviewer treated an empty response as approval after spending its output allowance on hidden reasoning. Later, a context counter counted some usage twice and stopped tasks at roughly half their actual allowance. Before attributing a failure to model capability, I needed to inspect the messages the model received, the conditions that counted as success, and the accounting that could terminate its work.

## Allocate compute around useful progress

A stronger model could consume enough of the budget to reduce the number of tasks the system had a chance to solve. An expert-only strategy solved 7 of 26 tasks and exhausted its budget with seven tasks unopened. A strategy that allowed two standard-model attempts before escalating reached 20 solves at a substantially lower cost per solve. That was its best run, with lower results on repetition, so it should not be read as a guaranteed outcome. It did establish that escalation policy could matter as much as the model selected for an individual attempt.

The cheaper tier created room for more attempts. That room had value because a fresh attempt sometimes found an approach that an earlier conversation had missed. Expert use needed to justify the opportunities displaced by its cost.

Additional verification had a similar tradeoff. On one small comparison, a gate and reviewer solved fewer tasks at greater total cost than the simpler harness. A review stage can help when it detects an error and enables a useful correction. It can also consume the remaining budget, reject valid work, or return no usable judgment. Its value has to be measured through the resulting task outcomes.

Within an attempt, waste was concentrated in unusually long responses. In the serving assignment, 4.5% of calls consumed 44.5% of output tokens. Some repeated the same reasoning, while others described commands without issuing a tool call. Interrupting these responses after a token limit and prompting the model to act produced executable commands in all eight replays tested. That supported a targeted intervention for stalled calls. It did not establish that the intervention would solve the corresponding tasks.

Aggressive limits on every response had the opposite effect. A 512-token cap increased the number of calls and raised time per solve from about 14 to 21 minutes. Useful reasoning was being split across more interactions. The evidence favored limiting prolonged stalls while leaving enough room for ordinary reasoning to reach an action.

The same distinction applied to whole attempts. Failed tasks consumed most of the task time in the later assignments, making them an obvious target for savings. Yet a short deadline could end productive work before a patch existed. In one comparison, an 18-minute cap left twelve tasks without patches, compared with three in a longer run.

Fresh attempts offered another way to use that time. Some retries had simply resubmitted identical patches. Starting without the previous diff allowed a different approach, and roughly a third of the solves in the strongest budget run arrived on the third attempt or later. A useful stopping policy therefore needed evidence of stalled progress, such as repeated patches or repeated reasoning without new observations. Elapsed time alone could not distinguish a difficult solution in progress from an exhausted approach.

## Validate the measurement against the workload

Serving speed mattered only to the extent that it improved the full task. In a Terminal-Bench comparison, two models each solved four of sixteen tasks. One generated tokens more than twice as fast, yet total runtime improved only from about thirty to twenty-seven minutes. Tool execution, the number of interactions, and long failures all limited how much faster generation could help.

Even an inference benchmark could misrepresent the agent's workload. My replay benchmark fed the server's own output directly into the next prompt, producing cache reuse that the real agent did not reproduce. Cache hit rates above 80% in the replay fell to about 62% in actual runs. The benchmark was measuring a more favorable request pattern than the one I needed to optimize.

A fast engine also needed a correctness check. One configuration combined chunked prefill with multi-token prediction and produced impressive throughput alongside poor graded results. Comparison with a reference server revealed substantial token discrepancies. I had initially looked for an explanation in the agent. The serving layer was part of the behavior under evaluation, and its speed could not establish that it was functioning correctly.

Task selection introduced another source of optimism. A Terminal-Bench subset chosen for small container images produced thirteen successes out of fifteen. A harder subset produced three out of sixteen. The convenient subset reduced evaluation overhead, but its score gave little support for broad claims about capability.

These examples changed what I required from a local test. It needed to preserve the property relevant to the decision. A serving benchmark needed realistic requests and cache behavior. A correctness check needed an independent reference. A task subset needed enough difficulty and variety to expose the failures I was trying to reduce. Without that connection, a precise local result could still guide the system in the wrong direction.

## Separate improvements from favorable runs

Changing several layers at once made diagnosis unnecessarily difficult. When I changed the agent and engine together, two submissions solved almost nothing. Restoring the original agent while retaining a narrower engine change recovered thirteen solves. That recovery identified a useful configuration, although it could not isolate every cause of the earlier failures. Keeping one layer fixed would have made the experiment easier to interpret.

Repeated evaluation was equally necessary. The same submission solved eleven tasks on one run and eight on another. The strongest budget configuration scored twenty, fourteen, and eighteen across runs. A leaderboard that retains the best result rewards a favorable draw, so its headline score can overstate typical performance.

Variation also had structure. Some tasks consistently failed, while others passed reliably. Looking at which tasks changed outcome helped separate persistent limitations from unstable cases. A repeatable failure deserved investigation even when the aggregate score fluctuated.

Benchmark tuning complicated that interpretation further. Repeatedly diagnosing the same tasks could turn a general instruction into a response tailored to a familiar failure. Advice about interfaces, environment setup, or performance measurement had a plausible broader use, but improvement on the tasks that motivated it provided weak evidence of generalization. I would want to evaluate such changes on held-out tasks and repeat the comparison under the same budget before treating them as broadly useful.

The agent helping me build the system also made confident explanations from limited evidence. It sometimes blamed a configuration after a single result or treated a strategy note as if it described a measured experiment. My most useful interventions were requests for a comparison, a trace, or a return to a known baseline. Those requests exposed bugs and narrowed claims that fluent explanations had made sound settled.

The central lesson was to connect each proposed improvement to a failure mechanism and an observable result. A cheaper call could free budget for another attempt. A reasoning cap could interrupt a loop. A reference comparison could reveal an invalid speedup. When the agent wrote most of the code, my responsibility was to establish those connections and decide how much evidence each conclusion deserved.

## AI use

Claude Code wrote much of the harness and serving code and assisted with experiments and analysis. Codex contributed to part of Assignment 2 and drafted this post from my outline. I directed the experiments and made the intervention decisions described above.
