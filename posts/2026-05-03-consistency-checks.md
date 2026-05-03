# Measuring consistency in agent-generated code

Status: Published
Date: 2026-05-03
Author: z3d

Convention tests enforce rules. Consistency measurement finds drift: files that are individually plausible, but structurally unlike the cohort they belong to.

The interesting failure mode for agent-generated code is not always a bug. Bugs get caught by tests, types, reviewers, or production. The quieter failure mode is drift: a steady accumulation of files that pass every local check while making the codebase less coherent as a whole.

A handler can be correct and still not look like the other handlers. A query can use the right dependency and still have a shape that no other query in the system has. A configuration file can live in the right layer and still break the convention everyone has been following without naming it. Nothing fails. The build is green. The repository just becomes harder to reason about.

That is the gap consistency measurement is meant to cover. It is not formatting, style, or architecture conformance. It is a relative question: does this artifact resemble the trusted examples that solve the same kind of problem?

## What consistency is not

Correctness is not consistency. Convention compliance is not consistency. Style is not consistency. Architectural conformance is not consistency.

Those distinctions matter because they point to different tools. Correctness belongs to tests and types. Convention compliance belongs to deterministic checks: every command has a validator, query handlers do not use EF Core, command handlers do not use Dapper, SQL queries avoid `SELECT *`. Style belongs to formatters and analyzers.

Consistency sits above those layers. It is about idiom: shape, proportion, decomposition, dependency count, rare feature combinations, and the way a file composes a solution compared with its peers. A file can satisfy every convention test and still be the file that should make a reviewer slow down.

## The measurable frame

The useful unit is a cohort: a group of files that solve the same kind of problem. In the reference implementation, three cohorts were enough to test the idea against real code:

- 34 command handlers.
- 20 query handlers.
- 26 EF Core configurations.

Each cohort gets three to five pinned exemplars. That word is load-bearing. The score should not be computed against the current average of the whole cohort, because the current average may already contain the drift you are trying to detect. If drift gets averaged into the reference point, the measure starts rewarding new drift for resembling old drift.

Exemplars are the files maintainers would point a new engineer or a new agent session at and say: write like this. They need written justification, version control, and review. They should represent the intended shape of the cohort, not the longest file, the most defensive file, or whatever happened to be written first.

## Distance is useful only after normalization

The structural layer extracts a feature vector for every member of the cohort. For handlers, useful features included IL byte size, constructor dependency count, logger presence, cache invalidation, try/catch presence, private method count, and entity load count.

The first implementation mistake was predictable in hindsight: raw features were put into the same distance calculation even though their scales were wildly different. `IlByteSize` had values in the hundreds or thousands; booleans like `HasLogger` were just 0 or 1. The result was not a multi-feature consistency score. It was a size sort wearing a statistics hat.

On the command-handler cohort, unnormalized `IlByteSize` accounted for 98.5% of the distance contribution and was the top contributor for every handler. After z-scoring features against the cohort, the signal changed. The top contributors spread across entity loads, IL byte size, try/catch, and dependency count. The outliers became files with unusual combinations of features, not merely files with more code.

That was the first hard lesson from the paper: a weak feature will dominate if its scale is allowed to dominate. Calling a feature a crude proxy does not make it harmless. Normalize it, name it honestly, and make the report show which feature actually drove the score.

## Three layers, one load-bearing signal

The framework tried three scoring layers: structural distance, AST/IL shingle similarity, and embedding distance. The original hope was that the layers would disagree in useful ways. The case study made the story narrower.

Structural distance did most of the work once normalization was fixed. Shingle similarity was correlated with it, but still occasionally useful when two files had similar feature vectors and different skeletons. Embedding distance was the noisiest: it often flagged vocabulary differences rather than real drift.

That does not make embeddings useless. In one useful accident, embedding outliers overlapped with handlers that were constructing DTOs inline instead of using the matching mapper. The embedding layer noticed vocabulary difference, not design intent, but the flagged pattern was real. Once the pattern was understood, it graduated into a deterministic convention test.

That is the right relationship. Consistency scores are a staging ground. When a signal repeatedly points at a rule you can state absolutely, promote it out of the advisory system and into a convention test.

## The best report was the simplest one

The most useful output was not the composite distance score. It was the per-feature divergence report.

Instead of asking which file is farthest from a centroid, the divergence report asks a simpler question: which members differ from the exemplars on each feature? A line like "15 handlers differ from the exemplar on HasCacheInvalidator" is more reviewable than a single floating-point distance. It tells the reviewer what changed, where to look, and whether the difference is isolated or cohort-wide.

Composite distance is good for ranking attention. Per-feature divergence is good for explaining the attention. The second one is closer to the reviewer's actual question.

## Case studies

### Command handlers

The command-handler cohort had 34 members and seven features. Before normalization, the largest handlers were the outliers. After normalization, the strongest outliers were the two handlers wrapping work in SERIALIZABLE transactions with retry and rollback. Those handlers were not wrong. They were legitimate business complexity. But the report correctly said: this is structurally unusual.

The same cohort also surfaced the inline DTO construction pattern. That one was not just unusual; it was a rule hiding in plain sight. Once found, it became a convention test that catches DTO construction through IL inspection and skips the one result-summary case where no mapper should exist.

### Query handlers

The query-handler cohort had 20 members and three sub-shapes: by-id single-row, unpaginated list, and paginated list. Pinning one exemplar from each sub-shape mattered. A single canonical query handler would have produced a centroid that represented none of the real shapes well.

The only strong outlier was the one query using a SQL JOIN. That was a good advisory result: unusual, worth noticing, defensible as legitimate complexity. The cohort also helped move list-query caching from an advisory concern into a hard convention, because list caches could not be reliably invalidated with the available cache abstraction.

### EF configurations

The EF-configuration cohort had 26 members. It pinned a minimal config, a canonical mid-range config, and a richer aggregate config. On the first run, governance caught a configuration class nested inside another configuration file. Every other configuration had its own file. The drift had been tolerated because nobody had named the rule. Once surfaced, it became obvious enough to fix and enforce.

## Do not gate on the score

A consistency score should not fail a merge because distance has a known pathology: it punishes the first good example of a new pattern and rewards the tenth copy of a bad one. The first file that introduces a better shape will look unusual. If unusual means blocked, the codebase cannot evolve.

The score should rank attention, not label files good or bad. An outlier has at least three possible meanings: it is drift to correct, it is legitimate local complexity, or it is the first instance of a pattern that may later become an exemplar. The tool cannot decide which one it is. Reviewers can.

That is why the consistency suite should include a meta-test that prevents literal threshold gates from creeping in. If a test starts saying "distance must be less than 5.0", the advisory layer has quietly become a brittle policy layer.

## Where prompts fit

It is tempting to give an LLM the candidate file and the exemplars and ask whether the candidate is consistent. That can be useful commentary, but it should not be the measurable layer.

Prompt-based judges are not stable enough to trend, and they may share the same blind spots as the generator. A better use is explanation after deterministic scoring: this file is structurally unusual; here are the exemplars; describe the difference for review prep.

The one open area where a prompt might earn a scoring role is semantic duplication: given the nearest existing handlers, is this new handler solving the same problem under a different name? That question is still better left unbuilt until there is a real case to tune against.

## The practical loop

1. Pick one cohort that is large enough to measure and important enough to drift.
2. Pin three to five exemplars with written justification.
3. Extract a small feature vector that represents shape, not taste.
4. Normalize features before computing distance.
5. Read the report as advisory review guidance.
6. Promote repeatable deterministic findings into convention tests.

The point is not to make a codebase uniform. The point is to make surprise visible while there is still time to decide whether it belongs.

Adapted from z3d's consistency paper and the extracted `Z3D.Consistency` library.
