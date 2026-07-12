# Benchmarking whether agents can maintain a codebase

Status: Draft
Date: 12 July 2026
Author: z3d

Most coding-agent benchmarks measure whether a model can produce one passing patch. That is a
reasonable question, but it is not the job I actually give agents. The job I give them is
maintenance: add a field, add an endpoint, chase a bug report, extend a resource — in a codebase
that already has opinions, and without leaving it worse than they found it. Worse rarely means
broken. It means conventions quietly violated, documents that no longer match the code, a diff
three times larger than the change deserved, or new files that obey every written rule and still
sit oddly next to their neighbours.

In [an earlier post](../consistency-checks) I described a small tool for making that last kind
of degradation visible: structural fingerprints over families of similar files, scored against
pinned exemplars, reported as a reviewer nudge. This post is about what happened when I built a
benchmark around the same idea and ran four models through it.

## The setup

The testbed is a .NET 10 clean-architecture template I maintain for agent-driven work. Its
defining property is that the rules are mechanical: convention tests enforce structure (every
command has a validator, queries use Dapper and never EF Core, constraints follow a naming
scheme), snapshot tests pin event wire contracts, and a docs-sync test fails the build if the
agent documentation drifts from the code. That property makes benchmarking cheap, because the
testbed's own suite is the marking rubric.

The benchmark, drift-bench, is a runner around it. Each run clones the testbed into a sandbox,
commits the task description as a baseline, launches a headless agent, and then grades:

1. `dotnet build` — a gate. Failure scores zero.
2. The testbed's full suite — also a gate. An agent that completes the task but breaks one
   convention test scores zero, as does one that never commits. In maintenance, unverified or
   uncommitted work has no value.
3. Hidden acceptance tests, written per task and copied in only after the agent finishes. They
   speak plain JSON against the public API, so they compile before the task is done and pass
   only when the behaviour exists.
4. Diff economy — the size of the agent's diff against a calibrated reference for an idiomatic
   complete solution. Acceptance carries 70% of the composite, economy 30%.

There are five tasks: a warm-up field addition, a query endpoint, a new domain event through the
outbox, a planted defect presented as a support ticket (with git history squashed so the bug
must be diagnosed rather than recovered from a diff), and a full new resource — entity,
validators, migration, three endpoints, documentation. Each task was verified against a null
agent first: the acceptance tests must fail before the work is done, for behavioural reasons.

## Results

Four models, five tasks, one repetition each, same testbed commit, same claude CLI harness.
Composite scores out of 100; costs are the CLI's own token accounting.

| Task | Fable* | Opus 4.8 | Sonnet 5 | Haiku 4.5 |
|------|--------|----------|----------|-----------|
| 01 field add | 87 | 90 | 92 | 100 |
| 02 query endpoint | 96 | 100 | 93 | 100 |
| 03 domain event | 97 | 100 | 94 | 100 |
| 04 planted bug | 99 | 99 | 98 | 96 |
| 05 full resource | 94 | 96 | 90 | 0 |
| Average | 94.6 | 97.0 | 93.4 | 79.2 |
| Cost, full suite | ~$50 | $22.41 | $20.94 | $3.95 |

\* Fable ran first, before the economy dimension was calibrated; its column is computed
retrospectively and its task-05 score is a re-grade after harness failures (more on those
below). Treat it as indicative.

## The Haiku result is the interesting one

Haiku 4.5 scored 100, 100, 100, 96 on the first four tasks, at between forty cents and $1.40
each. It root-caused the planted inventory bug from the symptom description alone — for less
than the price of the coffee I drank while it did — wrote the reproduction test first, and
fixed the cause rather than the symptom.

Then it scored zero on the resource task, and the way it scored zero is the whole point of the
exercise. Its code worked: the build passed and six of seven hidden acceptance tests passed.
What failed were two consistency tests from the machinery described in the earlier post. Its
new query handler was missing a structural feature that every exemplar query handler in the
codebase shares, and its command handler measured more than a standard deviation outside the
cohort. Haiku produced code that works but does not belong — which is precisely the quiet
degradation the benchmark was built to detect, produced under laboratory conditions by the
cheapest model in the lineup, on the first task large enough to give structure room to drift.

I want to be careful with a single observation: this is one repetition, and it may be a coin
flip rather than a ceiling. Repetitions on this cell are the obvious next experiment. But the
shape of the failure is exactly what the maintenance framing predicts and patch benchmarks
cannot see: the gap between top-tier and budget models did not appear as broken code, it
appeared as unfamiliar code.

## Frontier models separated on price, not quality

Opus 4.8, Sonnet 5, and Fable passed every gate on every task. All three kept the conventions,
kept the docs in sync, wrote validators without being reminded, and produced structurally
conforming code that the consistency cohorts accepted. The differences were second-order:
Opus was steadiest (nothing below 90), Sonnet was consistently a few points behind on diff
economy, Fable's diffs on the easy tasks carried the most test padding.

On a codebase with mechanical enforcement, the practical discriminator among frontier models
was cost per maintained change — roughly $21–22 for the suite with Opus or Sonnet against
about $50 with Fable. The enforcement does real work here: the convention tests are load-bearing
guard rails that make mid-tier models safe to use for most maintenance, with the structural
judgment gap only opening on compositional work.

## What actually failed: operations

Across the whole campaign the agents almost never failed. The campaign itself failed
repeatedly, and every failure taught the runner something:

- Two runs died to a transient `Connection closed mid-response` API error, one of them 150
  turns and $25 into the hardest task. The runner now resumes the interrupted session.
- One agent finished its work, then ended its print-mode turn saying it would commit "when the
  background test run completes." In a one-shot harness there is no later; the process exits
  with the turn. Its work re-graded at 94. It was recorded as zero, and the gate is right —
  the work was never verified or committed — but the cause was a wrong assumption about the
  harness, not ability.
- A nearly full disk made every container-backed integration test on the host fail with socket
  errors, indistinguishable from agent failure. The runner now refuses to start on a starved
  host, and a sweep waits for disk to recover rather than failing cells.
- An expired CLI login briefly scored three cells as if the model had attempted and failed
  them. Agent-startup failures now abort grading instead of producing plausible-looking junk.
- The machine slept mid-sweep; sweeps now hold a wake assertion. A zombied VM reported itself
  as running while refusing connections; the pre-flight caught it.

If there is one transferable lesson, it is that benchmark variance at this difficulty is
dominated by operational failures, not model capability. Anyone comparing agents without
repetitions, host pre-flight checks, and a hard rule that unverified work scores zero is
mostly measuring their harness.

## Caveats

One repetition per cell. One testbed, with one set of conventions, in one language. All four
models ran through the same CLI harness, so harness-specific behaviour is confounded with
model behaviour. The reference sizes behind the economy score are reviewer judgment, not
ground truth. And the diff-economy dimension has a known blind spot this run exposed: it
rewards small diffs, and cannot distinguish lean from under-tested — one model scored a
perfect 100 on the warm-up partly by writing no tests at all, which the task text did not
explicitly require. Grading rubrics get gamed by exactly the agents you most want to catch.

## What's next

Repetitions on the interesting cells first, Haiku on the resource task especially. Then the
piece I care most about: turning the consistency measurement from an advisory pass/fail into a
numeric score inside the composite, and multi-turn sequences where each task starts from the
previous task's graded output — maintenance over time rather than per-change quality. The
benchmark repository stays private for now; the hidden acceptance tests are an answer key, and
tasks that leak into training data stop measuring anything.
