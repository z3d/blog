# Measuring consistency in agent-generated code

Status: Published
Date: 2026-05-03
Author: z3d

One of the things I noticed when I worked with good developers was how consistent their code was. You could move from one project to another, or from one client to another, and become productive almost immediately.

Part of it was the significant amount of effort that went into developer experience, but equally important was the consistency that permeated all the repositories. Patterns still evolved, but they evolved in a controlled and deliberate way.

That consistency is what I wanted to measure once agents started writing more of the code.

The check I am describing is fairly small. Pick a cohort of files that solve the same kind of problem. Pin a few examples that show the intended shape. Extract a feature vector from every file in the cohort, normalize those features, then report which files have moved away from the examples and which features caused the movement.

The report is advisory. It should not say "fail the build because this handler scored 6.2." It should say "this handler is far from the exemplars because it has an unusual dependency count, retry shape, cache behavior, or entity-load pattern." Review can then decide whether the file is drift, legitimate local complexity, or the first instance of a pattern that should become an exemplar.

Agent-generated code makes this useful because the problem can accumulate quietly. The code compiles, the unit tests pass, and the handler answers the right request. After enough of those changes, though, the codebase can start to feel less like one system and more like a folder of plausible one-offs.

A single handler might only be a little odd: one extra dependency, inline DTO construction, a cache invalidation path no sibling uses. A query might introduce a SQL JOIN in a slice where the surrounding queries are deliberately simple. An EF configuration can tuck a class inside another file even though every other configuration has its own file. None of this is automatically wrong. The point is to make it visible while review still has context.

## What I mean by consistency

Correctness, convention compliance, style, and consistency tend to get blurred together, but they want different tools.

If every command has to have a validator, that belongs in a deterministic convention test. If query handlers are not allowed to use EF Core, or command handlers are not allowed to use Dapper, encode that and let CI be boring. A formatter can settle whitespace. An analyzer can catch a narrow syntax rule. None of those tools tells you when a file has all the right ingredients but an unfamiliar shape.

In this work, consistency means idiom: proportions, dependency count, decomposition, cache behavior, private helpers, entity loads, and rare combinations of features. A file can satisfy every convention and still be the one where a reviewer should slow down and ask why it does not resemble its peers.

## Start with a cohort

A consistency check only makes sense inside a cohort, meaning a group of files that solve the same kind of problem. Measuring every class in a repository against every other class would mostly measure nonsense. In the paper, three cohorts were enough to make the idea concrete:

- 34 command handlers.
- 20 query handlers.
- 26 EF Core configurations.

Each cohort needs three to five pinned exemplars. I would not compute against the current average of the whole cohort, because the current average can already contain the drift you are trying to find. Once drift is baked into the reference point, new drift starts looking normal.

The exemplars are the files you would point a new engineer, or a new agent session, at and say: write like this. They need written justification and review, because they become part of the measuring instrument. They should represent the intended shape of the cohort, not the longest file, the most defensive file, or whatever happened to land first.

## Normalize before believing the number

The structural layer extracts a small feature vector for every member of the cohort. For handlers, useful features included IL byte size, constructor dependency count, logger presence, cache invalidation, try/catch presence, private method count, and entity load count.

The first implementation mistake was obvious in hindsight. I put raw features into the same distance calculation even though their scales were completely different. `IlByteSize` had values in the hundreds or thousands; booleans like `HasLogger` were just 0 or 1. Size won, then pretended to be a multi-feature consistency score.

On the command-handler cohort, unnormalized `IlByteSize` accounted for 98.5% of the distance contribution and was the top contributor for every handler. After z-scoring features against the cohort, the signal changed. The top contributors spread across entity loads, IL byte size, try/catch, and dependency count. The outliers became files with unusual combinations of features, not merely files with more code.

The lesson is simple, but easy to miss when the report looks official: a weak feature will dominate if its scale is allowed to dominate. Calling a feature a crude proxy does not make it harmless. Normalize it, name it honestly, and show which feature actually drove the score.

## Three signals, one did most of the work

The framework tried three scoring layers: structural distance, AST/IL shingle similarity, and embedding distance. I expected the layers to disagree in useful ways. The case study was less dramatic than that.

Structural distance carried most of the useful signal once normalization was fixed. Shingle similarity often moved with it, though it still helped when two files had similar feature vectors and different skeletons. Embedding distance was the noisiest layer. It tended to notice vocabulary before it noticed design.

Embeddings were still useful once: they overlapped with handlers that were constructing DTOs inline instead of using the matching mapper. The embedding layer probably noticed vocabulary, not intent, but the pattern it surfaced was real. After that, the right move was not to keep admiring the embedding score; the pattern graduated into a deterministic convention test.

That relationship feels right to me. Consistency scores are a staging ground. When the same advisory signal keeps pointing at a rule you can state absolutely, move it out of the advisory system and into a convention test.

## The report I kept reading

The composite distance score was useful for sorting attention, but the report I kept coming back to was per-feature divergence.

Instead of asking which file is farthest from a centroid, the divergence report asks which members differ from the exemplars on each feature. A line like "15 handlers differ from the exemplar on HasCacheInvalidator" is easier to review than a single floating-point distance. It tells you what changed, where to look, and whether the difference is isolated or spreading through the cohort.

Composite distance answers "where should I look first?" Per-feature divergence answers the question that matters once you get there: what is different?

## Case studies

### Command handlers

The command-handler cohort had 34 members and seven features. Before normalization, the largest handlers dominated the report. After normalization, the strongest outliers were two handlers that wrapped work in SERIALIZABLE transactions with retry and rollback. They were not wrong; they were carrying legitimate business complexity. The report still did its job by saying, in effect, "this is structurally unusual, please look at it."

The same cohort also surfaced the inline DTO construction pattern. That one was not just unusual; it was a rule hiding in plain sight. Once found, it became a convention test that catches DTO construction through IL inspection and skips the one result-summary case where no mapper should exist.

### Query handlers

The query-handler cohort had 20 members and three sub-shapes: by-id single-row, unpaginated list, and paginated list. Pinning one exemplar from each sub-shape mattered because there was no honest "average query handler" in that set. A single canonical exemplar would have produced a centroid that represented none of the real shapes well.

The only strong outlier was the query using a SQL JOIN. That was a good advisory result: unusual, worth noticing, and still defensible as legitimate complexity. The cohort also helped move list-query caching from an advisory concern into a hard convention, because list caches could not be reliably invalidated with the available cache abstraction.

### EF configurations

The EF-configuration cohort had 26 members. It pinned a minimal config, a canonical mid-range config, and a richer aggregate config. On the first run, governance caught a configuration class nested inside another configuration file. Every other configuration had its own file, but the rule had never been named, so the exception had been tolerated. Once surfaced, it was obvious enough to fix and enforce.

## Do not gate on the score

A consistency score should not fail a merge. Distance has a known pathology: it punishes the first good example of a new pattern and rewards the tenth copy of a bad one. If unusual means blocked, the codebase cannot evolve.

Use the score to rank attention, not to label files good or bad. An outlier has at least three possible meanings: drift to correct, legitimate local complexity, or the first instance of a pattern that may later become an exemplar. The tool cannot decide which one you are looking at; a reviewer has to.

For that reason, the consistency suite should include a meta-test that prevents literal threshold gates from creeping in. If a test starts saying "distance must be less than 5.0", the advisory layer has quietly become a brittle policy layer.

## Where prompts fit

It is tempting to give an LLM the candidate file and the exemplars, then ask whether the candidate is consistent. I think that can be useful commentary, but I would not make it the measuring layer.

Prompt-based judges are not stable enough to trend, and they may share the same blind spots as the generator. A better use is explanation after deterministic scoring: this file is structurally unusual; here are the exemplars; describe the difference for review prep.

The one open area where a prompt might earn a scoring role is semantic duplication: given the nearest existing handlers, is this new handler solving the same problem under a different name? I would still leave that unbuilt until there is a real case to tune against.

## The practical loop

The workflow I trust now is:

1. Pick one cohort large enough to measure and important enough to drift.
2. Pin three to five exemplars with written justification.
3. Extract a small feature vector that represents shape, not taste.
4. Normalize features before computing distance.
5. Read the report as advisory review guidance.
6. Promote repeatable deterministic findings into convention tests.

I am not trying to make every file look the same. I am trying to catch surprise while there is still time to decide whether it belongs.

Adapted from [z3d's consistency paper](https://github.com/z3d/z3d-consistency/blob/main/docs/papers/consistency-in-agent-generated-code.md) and the extracted [Z3D.Consistency](https://github.com/z3d/z3d-consistency) library.
