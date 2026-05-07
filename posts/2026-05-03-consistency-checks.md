# Measuring consistency in agent-generated code

Status: Published
Date: 3 May 2026
Author: z3d

The thing that bothers me about agent-written code is not usually the dramatic failure. The dramatic failures are noisy. They break tests, fail to compile, or do something obviously strange in review.

The quieter problem is code that is locally fine and collectively tiring. A new handler compiles, passes its unit tests, and answers the request it was asked to answer. A month later, someone has to change the same area and discovers that nearby files no longer agree on the shape of the work. One command loads entities before validation while another validates first. One query uses Dapper and another reaches for EF Core. One handler invalidates a cache and its sibling does not. None of those choices has to be wrong in isolation, but the next change now starts with rediscovery instead of recognition.

That was the kind of drift I wanted to make visible. I was not trying to build a judge for code quality, and I definitely did not want a number that could fail a merge because a file was a bit unusual. I wanted something closer to a review prompt: this file does not look much like the files around it, and these are the reasons.

The version I built is an xUnit report over repeated shapes in a .NET codebase. For each family of files, it pins a few examples that represent the intended shape, measures the rest with a small structural fingerprint, and prints the members that sit furthest from those examples. The fingerprint is deliberately plain: IL byte size, constructor dependency count, logger presence, cache invalidation, try/catch usage, private helper count, entity-load calls, SQL joins, fluent API call counts, and similar features depending on the cohort.

Those measurements do not know whether code is good. They are just enough to notice when a file has started answering familiar design questions in an unfamiliar way.

## Consistency means idiom

Correctness, style, convention compliance, and consistency all get bundled together in conversation, but they need different tools.

If every command handler must have a validator, that belongs in a deterministic convention test. If query handlers must not use EF Core, or command handlers must not use Dapper, encode those rules directly and let CI catch them. Formatting and narrow style choices can stay with formatters and analysers. Those tools are good at decisions the team has already made clearly enough to write down.

The gap is the file that obeys the written rules and still feels unfamiliar. In this work, consistency means idiom: the proportions of the file, the amount of orchestration, how dependencies are arranged, whether cache behaviour appears, how many entities get loaded, whether there is a rare combination of features in a place where the surrounding files stay simple. A reviewer might notice that kind of thing by instinct, but instinct gets unreliable when there are twenty generated files in a row and the code all looks plausible at a glance.

The consistency report sits before convention tests in that lifecycle. It is a place to notice a pattern before the team has decided what to call it. If review decides the pattern is a real rule, the finding should leave the report and become a deterministic test. Keeping that boundary clear is the main thing that stops the tool becoming an overconfident policy engine.

## Cohorts before scores

The measurement only makes sense inside a cohort: a group of files that solve the same kind of problem. Comparing every class in a repository with every other class would mostly produce noise, because a query handler, a command handler, and an EF configuration are not trying to have the same shape.

The reference implementation uses three cohorts: 34 command handlers, 20 query handlers, and 26 EF Core configurations. Those numbers are small, but they are large enough for an advisory review signal. I would not run the same machinery over six files and pretend the resulting score meant much.

The cohort boundary carries a lot of meaning. A command handler changes state, so it often loads entities, calls domain methods, saves changes, invalidates caches, and returns a DTO. A query handler reads data and should usually be thinner, lighter on dependencies, and shaped around the result it returns. EF configurations have their structure in fluent mapping calls. Treating all of those as generic "classes" would blur the thing I am trying to measure.

Within each cohort, I pin three to five exemplars. These are the files I would point a new engineer, or a new agent session, at and say: start here; this is the direction the codebase wants you to move in.

I prefer pinned exemplars to the current average of the cohort. The current average can already contain drift, and once drift becomes part of the reference point, new drift starts to look normal. Exemplars also force a little governance. They need written justification, because they are no longer just examples in the repo; they are part of the measuring instrument.

The exemplar set has to preserve real variation. The query-handler cohort, for example, has by-id lookups, unpaginated lists, and paginated lists. There is no honest single average query handler across those shapes. Pinning one example from each shape lets the report notice odd files without flattening ordinary variation into a template.

## What the report measures

In the reference app, the report runs as part of the test project. `dotnet test` discovers `ConsistencyReportTests` beside the normal xUnit tests. Each report test defines a cohort, tells the reporter how to discover members, tells it how to measure one member, and names the pinned exemplars.

For command handlers, the fingerprint includes IL byte size, constructor dependency count, logger presence, cache invalidator dependency, try/catch usage, private method count, and entity-load calls. Query handlers get a different row because they have different concerns: pagination, cache opt-in, list return shape, SQL statement count, and `JOIN` or `APPLY` usage. EF configurations use fluent API call counts, because that is where their shape lives.

The structural view is the part I trust most. It normalises the features, compares each member with the exemplar shape, and sorts the files that look furthest away. The implementation uses a regularised multivariate distance internally, but the useful idea is simpler than the math sounds: put the features on comparable scales, compare like with like, and show the reviewer which features drove the result.

I also tried two secondary views. One compares three-opcode IL shingles after dropping operands, giving a rough signal for compiled control-flow shape. The other compares local source and metadata vocabulary with cosine similarity. Neither of these asks a model to judge the code. They are deterministic signals, and in practice they mostly taught me how much weight not to put on them.

Shingles were a cheap second opinion, but they often moved with the structural score. Vocabulary similarity was noisier. It could flag a handler because the domain words were different, even when the design was normal. Its useful exception was inline DTO construction: a few handlers built DTOs directly instead of using the matching mapper, and the vocabulary view helped put those files in front of review. Once the pattern was visible, though, the right answer was not to keep trusting vocabulary similarity. The right answer was to turn DTO construction into a convention test.

## Normalisation changed the report

The first version of the structural score had an embarrassingly simple flaw. Its features did not live on the same scale.

IL byte size can be in the hundreds or thousands, while constructor dependency count is usually a small integer and a boolean such as `HasTryCatch` is only 0 or 1. A distance calculation does not understand intent; it just sees numbers. If the inputs are not normalised, the largest numeric range gets the loudest voice.

That is exactly what happened in the command-handler cohort. Before normalisation, `IlByteSize` accounted for 98.5% of the distance contribution and was the top contributor for every handler. The report looked like it was measuring structure, but it was mostly sorting by "which handler has more IL?"

After normalisation, the contributors spread across `EntityLoadCount`, `IlByteSize`, `HasTryCatch`, and `ConstructorDependencyCount`. The outliers were no longer merely larger files. They were files with unusual combinations of features.

That result made the report much more useful, but it did not make it a statistical oracle. With three exemplars and seven command-handler features, I cannot claim the relationships between features are known with confidence. The regularisation keeps the distance finite and conservative when the exemplar set is small. It does not make the number suitable for gating, and it does not remove the need to show the feature-level explanation.

## The explanation matters more than the rank

The composite score is mainly a sort order. It helps decide which files a reviewer should open first, but it is not the interesting part of the output.

The per-feature report is more useful because it says what changed in terms a human can review. For boolean features, it compares a file with the nearest exemplar shape rather than with a global majority. That matters when a cohort has several valid sub-shapes; a paginated query should be compared with the paginated-list exemplar, not with a by-id lookup. For numeric features, it compares against the exemplar mean and spread.

The useful output reads more like this: `ConstructorDependencyCount` expected roughly 2 or 3 and found 7; `EntityLoadCount` expected 1 and found 4; `HasTryCatch` expected false and found true. At that level, review has something concrete to discuss. The file might be accidental drift, legitimate local complexity, or the first example of a new pattern that should become an exemplar.

The number of affected files changes the interpretation too. One handler missing cache invalidation is probably a local question. Fifteen handlers missing it suggests that something broader has moved: the exemplar set might be stale, the convention might have changed, or the team might finally have enough evidence to encode a rule. A single composite score tends to blur those cases together.

## What showed up

The first command-handler run was useful partly because it was humbling. Before normalisation, the report was dominated by size. After normalisation, the strongest outliers were two handlers that wrapped work in `SERIALIZABLE` transactions with retry and rollback. Those handlers were not mistakes; they were carrying real business complexity. The report still did what I wanted because it did not block them. It simply made their unusual shape visible.

The better find in the same cohort was inline DTO construction. Some handlers were constructing DTOs directly instead of using the matching mapper. Once that showed up in review, it stopped being a vague consistency concern and became a convention test. The test now catches DTO construction through IL inspection and skips the one result-summary case where no mapper should exist.

The query-handler cohort had 20 members and three legitimate shapes: by-id single-row, unpaginated list, and paginated list. Pinning one exemplar from each shape mattered because there was no useful average across them. The only strong outlier was a query using a SQL `JOIN`, which was exactly the kind of thing I wanted the report to surface. It was unusual enough to deserve a look and still defensible as local complexity.

The same cohort also helped move list-query caching from advisory concern to hard convention, because list caches could not be reliably invalidated with the available cache abstraction.

The EF-configuration cohort surfaced a quieter convention. One configuration class was nested inside another configuration file, while every other configuration had its own file. The rule had never been named, so the exception had sat there unnoticed. Once the report made it visible, it was easy to fix and enforce.

Those are the best outcomes for this kind of tool. It notices a convention that already exists socially, helps review decide whether the difference matters, and then gets out of the way if the team promotes the finding into a deterministic test.

## Keep it advisory

A consistency score should not fail a merge. Distance has a nasty little pathology: it punishes the first good example of a new pattern and rewards the tenth copy of a bad one. If unusual means blocked, the codebase cannot evolve.

The report should rank attention rather than label files as good or bad. An outlier might be drift to correct, legitimate local complexity, or the first instance of a pattern that will later become intentional. The tool cannot make that call.

The suite has a meta-test to protect that boundary. It scans consistency tests for literal distance thresholds such as `distance < 5.0`. Data-relative scorer tests can be allow-listed with a reason, but nobody should be able to quietly turn an advisory report into a merge gate without having that argument in review.

That guardrail is mundane, but it matters. Without it, the report would eventually become a flaky policy engine in review-tool clothing.

## Where prompts fit

I would not ask an LLM to be the scoring layer. Prompt-based judges are not stable enough to trend, and they may share the same blind spots as the generator. If I use a prompt here, I want it downstream of deterministic evidence: here is the candidate file, here are the exemplars, here are the top feature differences, turn that into review notes.

The score should come from code I can rerun. The model can help explain the evidence, but it should not be the evidence.

## The version I would keep

If I were rebuilding this from scratch, I would start smaller than I did. I would pick one cohort that is large enough to measure and important enough to drift, pin three to five exemplars with written justification, extract a small feature vector that represents shape rather than taste, normalise the features, and print the per-feature differences. The result would stay advisory until review turned a repeated finding into a rule.

Structural fingerprints and per-feature divergence are the parts that carry the tool. The composite score is useful only as a way to order review attention. Shingles are worth keeping if they are cheap. Vocabulary similarity is exploratory until it finds a repeatable pattern worth promoting.

I am not trying to make every file look the same. I am trying to keep the codebase recognisable enough that the next person can change it without reconstructing the local idiom from scratch.

Adapted from [z3d's consistency paper](https://github.com/z3d/z3d-consistency/blob/main/docs/papers/consistency-in-agent-generated-code.md) and the extracted [`Z3D.Consistency`](https://github.com/z3d/z3d-consistency) library.

*Disclosure: I used Claude Code and Codex while building and writing this. The consistency concern and codebase examples are mine; some statistical framing and implementation choices were model-guided, then checked against the source code and the longer paper.*
