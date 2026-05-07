# Measuring consistency in agent-generated code

Status: Published
Date: 3 May 2026
Author: z3d

Agent-generated code has a quiet failure mode. The new handler compiles. The tests pass. The endpoint returns the right data. In isolation, nothing looks obviously wrong.

The cost shows up later, when someone has to change the system and cannot tell which nearby file represents the intended pattern. One handler loads entities before validation, another validates first. One query uses Dapper, another uses EF Core. One command invalidates a cache, another leaves the cache alone. Every file can be locally defensible while the codebase slowly becomes harder to read.

That is the problem I wanted to catch.

The tool I ended up with is deliberately modest: an advisory report that says "this file looks unlike its peers, and here are the features that made it stand out." It does not decide that the file is wrong. It does not fail the build because a handler scored 6.2. It gives a reviewer a short list of places worth opening before the context has gone cold.

The loop is simple. Pick a family of files that solve the same kind of problem. Pin a few examples that represent the intended shape. Measure each file with a small set of structural features. Compare the rest of the family with those examples. Report the files that differ most, and explain why.

In the reference .NET codebase, that meant command handlers, query handlers, and EF Core configurations. The command-handler fingerprint includes things like IL byte size, constructor dependency count, logger presence, cache invalidation, try/catch usage, private method count, and entity-load calls. Query handlers use a different fingerprint because their shape is different: pagination, cache opt-in, list return shape, SQL statement count, and `JOIN` or `APPLY` usage. EF configurations are measured through fluent API call counts, because that is where their structure lives.

None of those measurements knows whether the code is good. They only describe shape. That is enough for the job I want the report to do.

## Consistency means idiom

Correctness, convention compliance, style, and consistency are easy to blur together, but they are different problems.

If every command handler must have a validator, that should be a deterministic convention test. If query handlers must not use EF Core, encode that directly. If command handlers must not use Dapper, make CI boring and let it catch every violation. Formatters and analysers can handle narrow surface rules.

The gap is the file that passes all of those checks and still feels unfamiliar.

That is what I mean by consistency here: idiom. The proportions of the file, the number of dependencies, the way work is decomposed, whether cache invalidation appears, how many entities are loaded, whether a rare combination of features appears in a place where the surrounding files do something simpler. A file can satisfy every rule the team has written down and still be the one where review should slow down.

The report is for questions the team has not fully named yet. Once review turns one of those questions into a clear rule, it should stop being a consistency finding and become a convention test.

That promotion step matters. A vague report is a good place to notice a pattern once. It is a bad place to enforce a rule forever.

## Start with a real cohort

A consistency check only makes sense inside a cohort, meaning a group of files that solve the same kind of problem. Measuring every class in a repository against every other class mostly measures noise.

The reference implementation uses three cohorts: 34 command handlers, 20 query handlers, and 26 EF Core configurations. Those are small numbers, but they are enough for an advisory review signal. They are not enough for grand claims about code in general, and I would not run the same machinery over six files and pretend the score meant much.

The cohort boundary is doing a lot of the work. A command handler changes state. It loads entities, calls domain methods, saves changes, invalidates caches, and returns a DTO. A query handler reads data and should usually be thinner, dependency-light, and shaped around the kind of result it returns. Treating both as just "handlers" would hide the fact that they are meant to have different idioms.

Within a cohort, I pin three to five exemplars. These are the files I would point a new engineer, or a new agent session, at and say: write in this direction.

I do not want to compare against the current average of the whole cohort. The current average may already contain the drift I am trying to find. Once drift is baked into the reference point, the next bit of drift starts looking normal.

Pinned exemplars are not magic either. They need written justification and review, because they become part of the measuring instrument. They should represent intended shapes, not the longest file, the most defensive file, or whatever happened to land first.

They also need to preserve legitimate variation. The query-handler cohort has by-id lookups, unpaginated lists, and paginated lists. A single "average query handler" would represent none of those shapes honestly. Pinning one exemplar from each sub-shape gives the report room to say "this file is different" without flattening every valid pattern into one template.

## What the report does

The entry point is ordinary: run the test project. In the reference app, `dotnet test` discovers `ConsistencyReportTests` beside the rest of the xUnit suite. Each report test defines a cohort, tells the reporter how to discover its members, tells it how to measure one member, and names the pinned exemplars.

For each member, the reporter builds a fingerprint. After that it produces a few views over the same question.

The structural view is the useful one. It normalises the features, compares each member with the exemplar shape, and sorts the files that look furthest away. The implementation uses a regularised multivariate distance under the covers, but I do not think that should be the headline. The headline is simpler: put the features on comparable scales, compare like with like, and show the features that drove the result.

I also tried two secondary signals. One compares three-opcode IL shingles after dropping operands, which gives a rough view of compiled control-flow shape. The other compares local source and metadata vocabulary with cosine similarity. Neither asks a model to judge the code.

Those secondary layers were useful mostly because they taught me what not to lean on. Shingles were a cheap second opinion, but they often moved with the structural score. Vocabulary similarity was noisier; it could flag a handler because the domain words were different, even when the design was normal. Its one useful hit was inline DTO construction, and even there it probably found the pattern for vocabulary reasons rather than because it understood the design problem.

That is fine. A hunch can still be useful if review turns it into something deterministic. But I would not treat vocabulary similarity as an authority.

## Normalise before believing the number

The first version of the structural score had a basic problem: the features did not live on the same scale.

IL byte size can be in the hundreds or thousands. Constructor dependency count is usually a small integer. A boolean such as `HasTryCatch` is either 0 or 1. A distance calculation only sees numbers, so without normalisation the feature with the biggest numeric range gets the loudest voice.

That is exactly what happened. In the command-handler cohort, unnormalised `IlByteSize` accounted for 98.5% of the distance contribution and was the top contributor for every handler. The report looked mathematical, but it was mostly answering one question: which handler has more IL?

After normalisation, the contributors spread across `EntityLoadCount`, `IlByteSize`, `HasTryCatch`, and `ConstructorDependencyCount`. The outliers were no longer just larger files. They were files with unusual combinations of features.

That result changed how much I trust the score. Normalise first, then show the feature contributors. Without both, a number can look objective while measuring the wrong thing.

The regularised distance calculation is a guardrail, not a source of truth. With three exemplars and seven command-handler features, I cannot pretend the relationships between features are known with confidence. The shrinkage keeps the distance finite and conservative when the exemplar set is small. It does not make the score a gate, and it does not remove the need for the per-feature report.

## The explanation matters more than the score

The composite distance score is only a sort order. It helps decide which files review should open first. It does not explain what is unusual on its own.

The per-feature report is the part I now care about most. It answers the human question directly: which files differ, and on which feature?

For boolean features, the report compares a file with the nearest exemplar shape rather than with a global majority. That matters when a cohort has several valid sub-shapes. A paginated query should be compared with the paginated-list exemplar, not with a by-id lookup.

For numeric features, the report compares against the exemplar mean and spread. If the exemplars usually have two or three constructor dependencies and one handler has seven, the report should say that plainly. If every query exemplar has zero joins and one query introduces a `JOIN`, that should be visible even if the rest of the file looks ordinary.

This is the level where review can do something useful:

`ConstructorDependencyCount` expected roughly 2 or 3 and found 7. `EntityLoadCount` expected 1 and found 4. `HasTryCatch` expected false and found true.

That is a better conversation than "this file is an outlier." The reviewer can decide whether those differences are accidental drift, legitimate local complexity, or the first example of a new pattern that should become an exemplar.

The count of affected files matters too. One handler missing cache invalidation is probably a local question. Fifteen handlers missing it means something broader has moved. Maybe the exemplar set is stale. Maybe the convention has changed. Maybe the team finally has enough evidence to encode a rule. A single score tends to blur those outcomes together.

## What it surfaced

The command-handler cohort had 34 members and seven features. Before normalisation, the largest handlers dominated the report. After normalisation, the strongest outliers were two handlers that wrapped work in `SERIALIZABLE` transactions with retry and rollback.

Those handlers were carrying real business complexity. They were not obvious mistakes. The report still did its job because it did not say "fail this." It said "this is structurally unusual, please look at it."

The same cohort surfaced a better finding: inline DTO construction. Some handlers were building DTOs directly instead of using the matching mapper. Once the pattern was visible, it stopped being a consistency question and became a convention test. The test now catches DTO construction through IL inspection and skips the one result-summary case where no mapper should exist.

The query-handler cohort had 20 members and three legitimate shapes: by-id single-row, unpaginated list, and paginated list. Pinning one exemplar from each shape mattered because there was no honest average query handler.

The only strong outlier was a query using a SQL `JOIN`. That was a useful advisory result. It was unusual enough to notice and still defensible as local complexity. The score could not decide that for us, but it made the question hard to miss.

The query cohort also helped move list-query caching from an advisory concern into a hard convention, because list caches could not be reliably invalidated with the available cache abstraction.

The EF-configuration cohort had 26 members. On the first run, governance caught a configuration class nested inside another configuration file. Every other configuration had its own file, but the rule had never been named, so the exception had been tolerated. Once surfaced, it was obvious enough to fix and enforce.

Those are the best results this system can produce. It finds a convention that already exists socially, makes it visible, and then gets out of the way once the rule is encoded directly.

## Keep scores advisory

A consistency score should not fail a merge. Distance has a simple pathology: it punishes the first good example of a new pattern and rewards the tenth copy of a bad one. If unusual means blocked, the codebase cannot evolve.

The report should rank attention, not label files good or bad. An outlier has at least three possible meanings: drift to correct, legitimate local complexity, or the first instance of a pattern that may later become intentional. The tool cannot tell which one you are looking at. A reviewer has to.

The suite protects that boundary with a meta-test. It scans consistency tests for literal distance thresholds such as `distance < 5.0`. Data-relative scorer tests can be allow-listed with a reason, but nobody should be able to quietly turn an advisory report into a merge gate without having that argument in review.

That guardrail is mundane, but it matters. Without it, the report will eventually become a flaky policy engine wearing a review-tool costume.

## Use prompts for explanation, not scoring

It is tempting to give an LLM the candidate file and the exemplars, then ask whether the candidate is consistent. I think that can be useful commentary, but I would not make it the measuring layer.

Prompt-based judges are not stable enough to trend, and they may share the same blind spots as the generator. A better use is explanation after deterministic scoring: here is the candidate file, here are the exemplars, here are the top feature differences, now turn that into review notes.

The score should come from code I can rerun. The model can help explain the evidence, but it should not be the evidence.

## The version I trust now

If I were rebuilding this from scratch, I would start smaller than I did.

I would pick one cohort that is large enough to measure and important enough to drift. I would pin three to five exemplars with written justification. I would extract a small feature vector that represents shape rather than taste. I would normalise the features, print the per-feature differences, and keep the result advisory. When a finding becomes a rule the team can state clearly, I would promote it into a deterministic convention test. When a new pattern becomes intentional, I would update the exemplars deliberately.

Structural fingerprints and per-feature divergence are the load-bearing parts. The composite score is useful as a sort order, but only with explanations attached. Shingles are worth keeping if they are cheap. Vocabulary similarity is exploratory until it finds a repeatable pattern worth promoting.

The goal is not to make every file look the same. The goal is to catch surprise while there is still time to decide whether it belongs.

Adapted from [z3d's consistency paper](https://github.com/z3d/z3d-consistency/blob/main/docs/papers/consistency-in-agent-generated-code.md) and the extracted [`Z3D.Consistency`](https://github.com/z3d/z3d-consistency) library.

*Disclosure: I wrote this with help from Claude Code and Codex. The consistency concern and codebase examples are mine; some statistical framing and implementation choices were model-guided, then checked against the source code and the longer paper.*
