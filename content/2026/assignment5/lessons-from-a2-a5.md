---
title: "Wrangling an Agent: How to Get the Agent to Solve Benchmark Tasks"
date: 2026-11-24
authors: ["Alexander Aghili"]
summary: "What four assignments of tuning a coding agent taught me: read the failures, cap the long tail, distrust the proxy, and overrule the agent when the evidence says so."
tags: ["agent-design", "testing", "observability", "serving"]
toc: true
draft: true
---

> **How to use this file:** write your own sentences under each heading, then delete the outline bullets as you
> go. Rename every heading to the lesson in your own words. Aim for ~2,000–2,800 words in total. Delete this box
> before the PR.

**Intro (~120 words)**

- Hook, pick one: A4 starting setup 19/25 locally → 8/26 on the leaderboard; or the same A4 zip 11/26, then 8/26
- Turn: "I thought ___; the measurements said ___"
- Through-line (one sentence): at every layer the number I trusted wasn't the one that mattered: the agent's
  "done", my own counters, the strongest model, tokens/s, local scores, a single run
- Map (one line): SWE-bench Pro harness (A2) → budget with model tiers (A3) → serving our own model on one GPU
  (A4) → Terminal-Bench 2.0 (A5)

<!--more-->

## Part 1: Inside an attempt

- One short framing sentence.

### Lesson 1: Read why attempts fail, not the average

*~350 words*

- Main (A4): 101 of 212 failed gradings had ≥75% of hidden tests failing (67 had all failing); the same score kept
  repeating (one task 6/7 failing on 15 of 21 gradings)
- Cause = patch shape: a Go function where the tests call a method → the test package doesn't compile
- Test-shape hint → **5/26 → 11/26 in 53 min**, the biggest jump of A4
- Second (A3): in 4 of 5 hard local failures the agent's own test scripts passed but the hidden tests failed
  (checked the wrong function; rewrote a correct fix to match fake data it made up)
- Third (A5): the same thing at the environment level: PATH only in `~/.bashrc`, test commits left in the repo,
  "faster" never measured → after the PATH rule, sqlite-with-gcov passed with both models
- You said: "So how are we still failing here?"
- Takeaway to write: what should you look at before reaching for speed?
- Bridge: some failures weren't the model at all

### Lesson 2: Bugs in my own harness looked like model limits

*~300 words*

- Main (A3): the harness dropped the model's reasoning between tool calls (v9–v12). Fixing it: **1,171 → 322
  output tokens/call, $0.435 → $0.286**, same 3/5, do-nothing turns 13 → 0
- Found because you refused "no change can help". You said: "Well investigate more because somebody did it."
- Second (A3): the reviewer counted an empty reply as a pass (6 of 9 reviews were empty after 8,192 tokens of hidden
  reasoning)
- Third (A5): context counted twice for capped turns → tasks stopped at about half their real context
- Optional (A4): the repeat guard treated `cat > f` as a read; the gate path bug (25/29 → 29/29, tested once)
- Takeaway to write: what do you now test in your own harness before blaming the model?
- Bridge: once the harness was honest, the question was what to spend on

## Part 2: Spending the budget

- One short framing sentence.

### Lesson 3: The cheaper model and the simpler harness won

*~350 words*

- Main (A3): every expert-led version scored 7–11/26 (V2 expert-only: 7/26, $1.29/solve, ran out of money with
  7 tasks unopened). V12, two standard attempts before expert: **20/26, $0.44/solve**
- The clue: leaderboard cost levels only a cheaper tier could reach (keep it generic, no classmates)
- You said: "Im seeing lots of expert, so we may want to really restrict expert usage" (days earlier) and
  "We are obviously missing something key."
- Second (A3): more checking made it worse: gate + reviewer never beat simple v2 (4/5 for $1.92 vs 5/5 for $1.78).
  You said: "Holy crap, 2/8? Should we go back to v2?"
- Third (A5): 27B vs 35B-A3B: both 4/16; 203 vs 473 tok/s; 30.3 vs 26.6 min
- Optional (A4): the shipped engine was stock SGLang + launch flags; source patches weren't shipped
- Optional table:

  | Version | Strategy | Solved | $/solve | Time/solve |
  |---|---|---|---|---|
  | V2 | expert only | 7/26 | $1.29 | 7:46 |
  | V9 | standard only | 11/26 | $0.21 | 19:28 |
  | V12 | 2 standard, then expert | 20/26 | $0.44 | 9:58 |

- Takeaway to write: what does "more capable" cost you in a fixed budget?
- Bridge: within one model, most of the waste sat in a few long replies

### Lesson 4: Long reasoning is a tail problem

*~350 words*

- A2: a 4,096-token cap truncated 13 of 32 replies, ~9 of the task's 12 minutes. You said: "How is it 12 minutes
  for 26 requests?"
- A3: reviewer replies spent all 8,192 tokens and returned nothing
- A4: **4.5% of calls → 44.5% of output tokens, 28% of task time**. They looped ("stop going in circles" ×27) or
  wrote the command inside the reasoning, where nothing runs it. Sampling didn't fix it. A cap at 4,096 followed by
  "act now" → 8/8 replays ran the command
- A4 counterpoint: thinking off = no extra solves; a 512 cap = 2–4× the calls, 14.1 → 20.7 min per solve
- A5: the uncapped planning call ran to max_tokens → capped at 8,192
- Takeaway to write: cap the tail vs cap every call: which one, and why?
- Bridge: the same tail shows up one level up, in whole attempts

### Lesson 5: Give up early, retry fresh, but not too early

*~400 words*

- Main (A4): failures used **74–92% of task-minutes**, but runs that gave tasks less than ~25 min scored 0–3; an
  18-min cap → 12 tasks with no patch (3 in the longer run)
- Live checks: two tasks each solved in 2 of 4 fresh conversations, stall visible by minute 6–7 → stop at the stall,
  retry fresh → **12/26 in 63 min** (best sub-hour run)
- You said: "I was told we need to give up early" / "I just hope that we dont get really fast but only 3 solved
  again"
- Second (A3): retries resubmitted byte-identical patches; fresh attempts (no earlier diffs) gave ~7 of V12's 20
  solves at attempt 3+. A 40-step cap → 3/5; a soft 100 → 5/5. You said: "just set of a soft limit of 100"
- Third (A5): failed tasks took 95% / 89% / 81% of task time; the longest task was always a failure and set the
  end-to-end time
- Optional table:

  | Run (16 hard TB tasks) | Failed tasks' share of task time | Mean s, fail / pass |
  |---|---|---|
  | v2 + 35B-A3B | 95% | 680 / 167 |
  | v4 + 35B-A3B | 89% | 759 / 272 |
  | v4 + 27B | 81% | 1,033 / 736 |

- Takeaway to write: what signal should end an attempt, if not the clock?
- Bridge: each of these choices was tested on something smaller first, and those tests often lied

## Part 3: Knowing what worked

- One short framing sentence.

### Lesson 6: Benchmarks and local tests were wrong in ways I could check

*~350 words*

- Main (A4): 19/25 locally vs 8/26 graded. The replay benchmark's cache hit was 83–92% vs 62% for real: the replay
  fed the server's own output back into the next prompt, which the agent never does
- Predictions vs graded: 9 solves → 5, 11 → 5, −6% → −27%
- Second (A4): the fastest engine was wrong: mixed chunked prefill + MTP, 693 tok/s, graded 5, 3, 3, 7, and the
  agent got blamed first. A reference server showed 11.4% of tokens badly off
- Third (A5): the subset chosen for small images scored 13/15; a hard subset 3/16
- You said: "why doesn't proving things in our test env actually tell us the results?" / "Maybe we are tuning too
  much for this env" / "How could it be that fast? … test it"
- Takeaway to write: what makes a proxy trustworthy, and how did you check yours?
- Bridge: even the leaderboard was a noisy measurement

### Lesson 7: One change at a time, more than one run

*~350 words*

- Main (A4): v3/v4 changed the agent and the engine together → **1/26 and 0/26**. You ordered the original agent
  back → v5 (MTP only) **13/26**. You said: "go back to the original agnet"
- Noise: A3 V12 20/14/18; A4 the same zip 11 then 8; A/B 12/15 vs 8/15, p = 0.12; A5 query-optimize flipped
- But it clusters: Claude called 12–13 luck; you pushed back and the data agreed (10 tasks never solved, a few
  pass ~9 in 10). You said: "the professors said they cluster in the same region for the same code"
- Optional: the leaderboard keeps your best run (A3: "it only keeps the best run")
- Takeaway to write: what's the smallest experiment you'd now trust?
- Bridge: every number in this post came from one scoreboard. What was I actually optimizing?

## Optimizing for the benchmark

*~350 words*

- Framing: everything above was tuned against a leaderboard. How much of it made a better agent, and how much
  just fit these graders better?
- Me fitting the benchmark:
  - A4: 19/25 on the local tasks I tuned against vs 8/26 on the hidden ones. You said: "Maybe we are tuning too
    much for this env"
  - A4: the test-shape hint (lesson 1): general advice ("follow the public interface"), or a fit to how these
    graders' hidden tests call code? It took 5 → 11, then 8 on a regrade
  - A4/A5: caps and concurrency tuned to a leaderboard ranked on end-to-end time trade solves for minutes
    (an 18-min cap gave 5 solves; lesson 5)
  - A3: after the course's rules post, prompt lines from v4–v7 turned out to encode lessons from specific diagnosed
    local failures, and were removed. You said: "patch.py seems like its a good idea, regardless of where it comes
    from right?" (where is the line between a lesson and memorizing a task?)
  - A5: the course bans task-specific hard-coding. The v2/v4 rules came from three failed tasks (PATH, test traces,
    speed vs reference) and were kept only because any grader checks those things. Did they generalize?
    sqlite-with-gcov passed; the hard subset went 3 → 4, within noise
  - The leaderboard keeps your best run, so resubmitting the same zip selects on noise: A3 V12's 20 vs clean reruns
    of 14 and 18. The headline number is partly a best-of-N draw
- The agent fitting my checks (the same thing one level down):
  - A2: existing tests had to be fingerprinted so the agent couldn't edit or delete them to pass
  - A3: the agent rewrote a correct fix to match fake test data it wrote itself (lesson 1)
  - A4: the model got around a false gate refusal by setting `worktree = /` in `.git/config`, which would have made
    the patch delete every file
  - A2 tasks 7/8: hidden tests wanted exact strings the spec never gave, so passing the benchmark and meeting the
    requirement came apart in both directions
- Takeaway to write: which of your changes would still help on a different benchmark, and which only helped here?
  What would you hold out to check?
- Bridge: the agent optimizes whatever you measure, and so did I. That is where my own judgement had to come in

## Overruling the agent

*~300 words, the ending*

- Where you overruled it and were right: going back to simple v2 and restricting expert use (A3); "somebody did
  it" → the dropped-reasoning bug (A3); "go back to the original agent" → one change at a time (A4); results
  cluster, not pure luck (A4); "How could it be that fast? … test it" (A4)
- Where the agent was confidently wrong: blamed 10 open tasks for a 7/26 run from one data point, then retracted;
  gave ~70% odds a build would match the best and it graded 7/26; treated a pasted strategy note as measured
  evidence
- Close the through-line: which number you'd distrust first next time, and what your job is when the agent writes
  most of the code

## AI use

- As the [course policy](https://cs2680.com/policy.html#ai-policy) requires: Claude Code wrote much of the
  harness and engine code and ran experiments and analysis (Codex for part of A2); you directed it and made the
  calls above; the post text is yours. Name any figure made with AI.

---

**Before the PR (delete this list too):**

- `draft: false`; title, summary and tags filled; every figure has alt text and a caption (or delete it)
- No outline bullets, TODOs or this checklist left in the file
- No classmates' names or scores, cluster usernames, hostnames, job IDs, keys, hidden-task test details
- `hugo server -D` shows no errors; folder < 2 MB
