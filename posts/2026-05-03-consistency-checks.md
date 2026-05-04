# Measuring consistency in agent-generated code

Status: Published
Date: 3 May 2026
Author: z3d

One of the things I noticed when I worked with good developers was how consistent their code was. You could move from one project to another and become productive almost immediately.

Part of it was the significant amount of effort that went into developer experience, but equally important was the consistency that permeated all the repositories. Patterns still evolved, but they evolved in a controlled and deliberate way.

That consistency is what I wanted to measure once agents started writing more of the code.

I should be clear about how this came together. The consistency problem was mine: I wanted a way to notice when generated code stopped resembling the code around it. I am years away from my CS degree and the university statistics courses where I last had to think seriously about this kind of thing. The vague measurement idea was some kind of eigenvector characterisation of code shape. I let Claude Code and Codex guide the translation from that hunch into something I could actually try against a codebase.

That hunch became something narrower: an xUnit test suite around a report builder. For the structural part, each file becomes a fingerprint, which is a row of measurements such as IL byte size, constructor dependency count, whether the handler has cache invalidation, whether it uses try/catch, entity load count, and so on.

The report builder also checks compiled shape and source vocabulary. Underneath that, the loop is fairly plain: choose a cohort of files that solve the same kind of problem, pin a few examples that show the intended shape, compare the rest of the cohort with those examples, and report which files look furthest away and why.

Because it is for review, the report is advisory. It should not say "fail the build because this handler scored 6.2." It should say "this handler is far from the exemplars because it has an unusual dependency count, retry shape, cache behaviour, or entity-load pattern." Review can then decide whether the file is drift, legitimate local complexity, or the first instance of a pattern that should become an exemplar.

Convention tests come after that review. When an advisory finding turns into a rule the team can state clearly, it should leave the consistency report and become a deterministic test.

The suite also protects that boundary. It has a meta-test that scans consistency tests for literal distance thresholds such as `distance < 5.0`. Data-relative assertions for scorer tests can be allow-listed with a reason, but a hard threshold cannot drift into the report quietly.

Agent-generated code makes this useful because the problem can accumulate quietly: the code compiles, the unit tests pass, and the handler answers the right request. The cost shows up later, when someone has to change the system and cannot tell which nearby file represents the intended pattern. Does this kind of handler load entities before validation, should it invalidate a cache, should this query use Dapper or EF Core? If every file answers those questions differently, the next change starts with rediscovery.

A fair objection is that generated code may never be perfectly consistent, and I agree with that. Perfect sameness is not the goal here; the goal is that a human can come back later, without the original prompt or the same model, and still understand how the codebase wants to be changed. The tools are unlikely to vanish, but the code should still be maintainable by the people responsible for it.

A single handler might only be a little odd: one extra dependency, inline DTO construction, a cache invalidation path no sibling uses. A query might introduce a SQL JOIN in a slice where the surrounding queries are deliberately simple. An EF configuration can tuck a class inside another file even though every other configuration has its own file. None of this is automatically wrong; the point is to make it visible while review still has context.

## What I mean by consistency

Correctness, convention compliance, style, and consistency tend to get blurred together, but they want different tools.

The concrete examples here come from a .NET codebase. If every command has to have a validator, that belongs in a deterministic convention test. If query handlers are not allowed to use EF Core, or command handlers are not allowed to use Dapper, encode that and let CI be boring. A formatter and an analyser can take care of narrow surface rules, but neither tells you when a file has all the right ingredients and still has an unfamiliar shape.

In this work, consistency means idiom: proportions, dependency count, decomposition, cache behaviour, private helpers, entity loads, and rare combinations of features. A file can satisfy every convention and still be the one where a reviewer should slow down and ask why it does not resemble its peers.

## Where convention tests fit

Convention tests are for decisions the team has already made. In the .NET reference codebase, that means rules like: every command handler must have a validator, query handlers must not use EF Core, and command handlers must not use Dapper. Encode those rules directly and let CI catch them every time.

Consistency checks are for questions the team has not fully named yet. A report might show that a handler is far from the exemplars because it constructs DTOs inline instead of using the mapper. The first time that appears, it is a review question. It might be harmless local complexity, or it might be a real convention hiding in plain sight.

Once review decides "we should never do that, except for this one result-summary case", the finding should graduate into a convention test. From then on, the consistency report no longer has to rediscover it. CI can catch the rule cheaply, and the consistency check can keep looking for softer drift.

This is why I want to keep the two tools separate. A consistency score is a way to notice code that deserves a closer look, not a policy decision by itself. The convention tests are where the team records agreements it is ready to enforce. Patterns can still evolve, but they should evolve by updating exemplars and conventions deliberately, not by letting every new outlier redefine normal.

## Prior art

None of this starts from thin air. Style guides, formatters, and analysers already make small choices disappear. Architecture-test tools such as [ArchUnit](https://www.archunit.org/userguide/html/000_Index.html) and [NetArchTest](https://github.com/BenMorris/NetArchTest) let teams encode structural rules in tests. Evolutionary architecture uses [fitness functions](https://www.thoughtworks.com/en-us/insights/books/building-evolutionaryarchitectures-second-edition) as automated feedback on architectural characteristics. [Approval and snapshot testing](https://approvaltestscpp.readthedocs.io/en/latest/generated_docs/ApprovalTestingConcept.html) compare current output with a known-good artefact.

This work sits near those ideas, but it is trying to fill a different gap. The thing being measured is not whitespace or naming, so it is not a formatter. It also does not start with a rule, which separates it from architecture tests, and it is not looking for exact reproduction in the way a snapshot test does. I still want review in the loop, because the output is a reason to look closer, not a verdict.

It also differs from the common exemplar style in LLM workflows, where examples are used as instructions to the generator: here are three files, now write the next one like this. That can be useful as generation guidance, but the exemplars here have a different job: they become the measuring instrument. They anchor the reference distribution for a cohort so review can see which files are structurally far away and which features made them far away.

That distinction matters because copying one exemplar too closely would be wrong. The query-handler cohort has multiple legitimate shapes: by-id single-row, unpaginated list, and paginated list. A single "make it like this" example would flatten that variation. A pinned exemplar set can preserve the allowed variation while still making drift visible.

## Start with a cohort

A consistency check only makes sense inside a cohort, meaning a group of files that solve the same kind of problem. Measuring every class in a repository against every other class would mostly measure nonsense. In the original consistency paper, three cohorts were enough to make the idea concrete:

- 34 command handlers.
- 20 query handlers.
- 26 EF Core configurations.

In this kind of .NET application, a command handler receives a request that changes state. It loads entities, calls domain methods, saves changes, invalidates caches, and returns a DTO. A query handler receives a read request and returns read-model data. It should usually be thinner, dependency-light, and shaped around the kind of result it returns. Treating both as just "handlers" would hide the fact that they are meant to use different approaches.

Those are small cohorts, not population studies. They were enough for this work because the report was not trying to prove a universal property of code. It was trying to give review a ranked list of files that looked unusual against a set of chosen examples. I would not run the same machinery over six files and pretend the distance score meant much.

Those categories were useful because they had enough members, a clear role, and different kinds of legitimate variation. The command handlers were not one identical template. The exemplar set covered a small state transition, an entity creation path with cross-entity validation, and a heavier orchestration path that touched several reads before writing. The query handlers had a cleaner split: by-id lookup, bounded list, and paginated list. The EF configurations ranged from small table mappings to richer aggregate mappings.

That matters because measuring consistency is not the same as forcing every file into one shape. If a cohort has several intended shapes, the exemplar set needs to preserve them. Otherwise the tool will punish ordinary variation and call it drift.

The practical rule I landed on is three to five pinned exemplars per cohort. I would not compute against the current average of the whole cohort, because the current average can already contain the drift you are trying to find. Once drift is baked into the reference point, new drift starts looking normal.

The exemplars are the files you would point a new engineer, or a new agent session, at and say: write like this. They need written justification and review, because they become part of the measuring instrument. They should represent the intended shape of the cohort, not the longest file, the most defensive file, or whatever happened to land first.

## How it runs

The entry point is deliberately ordinary: run the test project. In the reference app, `dotnet test` loads the normal xUnit suite and discovers `ConsistencyReportTests` beside the other tests. There are report tests for command handlers, query handlers, and EF Core configurations. Each one creates a cohort and passes it to `CohortConsistencyReporter.Build(...)`.

The cohort is the project-specific part. It says which application assembly to inspect, how to find members of this family, how to measure one member, and which type names are pinned exemplars. The command-handler cohort reflects over the API assembly, keeps concrete `CommandHandler` types that implement the mediator handler interface, and then builds a `HandlerFingerprint` from constructor parameters and IL inspection.

That fingerprint is the row of measurements. For command handlers, it includes IL byte size, constructor dependency count, logger presence, cache invalidator dependency, try/catch usage, private method count, and entity-load calls. Query handlers use a different row because their shape is different: pagination, cache opt-in, list return shape, SQL statement count, and `JOIN` or `APPLY` usage. EF configurations use fluent API call counts, because that is where their shape lives.

`Build` then follows the same path for every cohort. It discovers the member types, extracts all fingerprints, resolves the configured exemplar names back to those fingerprints and types, then refuses to continue if the cohort is empty or an exemplar cannot be resolved. After that it computes the four pieces of the report: structural distance, IL shingle similarity, source-token similarity, and per-feature divergence.

The structural score is the measurement-row comparison. It normalises the features, builds the exemplar centroid, regularises the covariance calculation, then sorts members by distance from the exemplar shape. The shingle score ignores names and most operands and compares short runs of IL opcodes, so it is a second opinion on compiled control-flow shape. The source-token score is local and deterministic in this implementation: it hashes method names, string literals, type references, constructor references, and fields found while walking IL metadata, then compares those token vectors with cosine similarity. It is not asking a model to judge the code.

The report test writes all of that to xUnit output: structural rows with top contributors, shingle similarity rows, token-similarity rows, a per-feature divergence section, and a summary with the exemplar list and simple cohort statistics. The test asserts that the report is well formed: member counts line up, scores are finite, and sorted outputs are actually sorted. It does not assert that a handler's distance must be below some magic number.

The tests that are meant to fail are the governance tests around the report. They check that discovery still matches files on disk, fingerprints still have meaningful values, exemplar names in code still match the README, every exemplar still has written justification, and the scorer is anchored to pinned exemplars rather than the whole cohort. A separate advisory meta-test scans the consistency tests so someone cannot quietly add a literal distance threshold and turn the report into a merge gate.

In plain terms: compile the application, run the tests, reflect over the compiled types, measure each member, compare it with the pinned exemplars, print the report, and fail only when the measurement setup or a deterministic convention is broken. A high score is not a failing condition. It is a review prompt with enough detail for a person to decide what it means.

## Normalise before believing the number

Once the cohort and exemplars are chosen, the next step is to compare each candidate file with the shape the exemplars represent. That comparison starts by turning every file into a small set of measurements. For a handler, those measurements might include IL byte size, constructor dependency count, logger presence, cache invalidation, try/catch usage, private method count, and entity load count.

The distance score is just a way to combine those measurements into a review signal. It is not trying to decide whether the file is good or bad. It is trying to answer a narrower question: which files look furthest from the pinned exemplar shape?

That only works if each measurement gets a fair say.

Those features do not naturally live on the same scale: IL byte size can be in the hundreds or thousands, constructor dependency count is usually a small integer, and a boolean such as "has a logger" is only 0 or 1. They are different kinds of measurement, but a distance calculation only sees numbers.

That means the raw score can rank the wrong thing. A handler that is 600 IL bytes larger than the exemplars may look more unusual than a handler that drops cache invalidation or adds the only try/catch block in the cohort, simply because 600 is bigger than 1. The calculation has not understood that the boolean represents a meaningful structural choice; it has only been handed unscaled inputs.

So before normalisation, the feature with the biggest numeric range gets the loudest voice. The report can look like it is comparing shape while mostly comparing whichever feature happened to have the largest units.

In the command-handler cohort, this is exactly what happened: unnormalised `IlByteSize` accounted for 98.5% of the distance contribution and was the top contributor for every handler. The report looked mathematical, but it was mostly answering one question: which handler has more IL?

Normalisation fixes that by making each feature relative to the cohort before distance is calculated. A count of seven constructor dependencies matters because most handlers have fewer. A try/catch matters because almost no handlers use one. A large IL body still matters, but it no longer drowns out everything else.

After normalising the features, entity loads, IL byte size, try/catch usage, and dependency count all started showing up as contributors. The outliers were no longer just files with more code; they were files with unusual combinations of features.

That is the rule I now trust: normalise before calculating distance, and always show which features drove the score. Otherwise the number can look objective while mostly measuring the wrong thing.

## What the scoring layers showed

The three comparisons are where Claude Code and Codex most directly shaped the work. I knew what I wanted to measure, but I did not have a clean statistical design. They suggested trying a structural measure, a compiled-shape measure, and a vocabulary measure. I tried all three, then looked at which ones earned their keep.

The structural measure handled the fingerprint rows. The shingle measure compared compiled control-flow skeletons by looking at short runs of IL opcodes while ignoring most names and operands. The vocabulary measure compared source-token vectors with cosine similarity. I called this embedding distance in early notes, but in the implementation it is local and deterministic: method names, strings, type references, constructor references, and fields. No model is asked to judge the file.

I expected three mostly independent signals, but the results did not separate that cleanly. Structural distance did most of the useful ranking, shingles were a correlated second opinion, and the vocabulary layer was mostly noisy. That was still valuable, because it showed which parts of the idea were carrying weight and which parts were only attractive on paper.

The structural layer is where Mahalanobis distance came in. After normalisation, each file becomes a point in feature space, and the exemplars define a centre. Plain distance asks how far a point is from that centre. Mahalanobis also asks how the features usually move together in the exemplars. If larger handlers normally have more private helpers, that combination is less surprising than a normal-sized handler with an unusual dependency count, a try/catch, and extra entity loads.

The awkward part is covariance. Mahalanobis needs a covariance matrix to describe how features move together, and the raw exemplar matrix is often unusable here. Three exemplars and seven command-handler features means fewer examples than dimensions, so the sample covariance cannot be inverted safely. Even with cohorts of 20, 26, or 34 members, the score is not a statistical proof. The cohort is small and curated.

The implementation regularises the covariance with Ledoit-Wolf shrinkage. In plainer terms, it blends the covariance estimated from the exemplars with a scaled identity matrix. The code computes a shrinkage weight, `alpha`, and uses `Sigma_reg = (1 - alpha) * S + alpha * F`, where `S` is the sample covariance and `F` is the scaled identity target. When `alpha` is near 1, the scorer does not trust the exemplar correlations much and behaves more like standardised Euclidean distance over z-scored features. When the exemplar evidence supports correlations, `alpha` moves towards 0 and the correlations carry more of the score.

This is the concrete fallback: it keeps the distance finite and conservative when the exemplar set is small. It does not make the number a gate, and it does not remove the need for the per-feature report.

Structural distance carried the work once normalisation and shrinkage were in place. It found files with unusual combinations of features: more entity loads than the cohort expected, different dependency counts, retry shapes, private helpers, and try/catch blocks.

The shingle layer took a different approach by ignoring names and most operands, looking at short runs of IL opcodes, and asking whether the file's skeleton resembled the exemplar skeletons. I wanted this to catch the "same feature vector, different control flow" case. It did that sometimes, but it often moved with structural distance. I would keep it as a cheap second opinion, not build it before the structural and per-feature reports.

The vocabulary layer is the part I trust least. It can flag a handler because it uses domain words the exemplars do not use, even when its shape is normal. Its useful exception was inline DTO construction: it flagged handlers that built DTOs directly instead of using the matching mapper. It probably found them for vocabulary reasons, not because it understood the design problem. Once the pattern was visible, the right move was to turn it into a convention test.

The relationship between the layers is less tidy than my first version assumed. Structural distance and per-feature divergence now carry most of the report; shingles are useful when they confirm or complicate that picture, and vocabulary similarity is best treated as a hunch that needs review before it becomes anything stronger.

## Explain the score by feature

The composite distance score worked as a sort order. It helped decide which files review should open first, but it could not explain what was unusual about a file on its own. A handler can score highly because it is large, because it has a rare control-flow shape, because it loads more entities than the exemplars, or because several smaller differences line up.

The per-feature report keeps those reasons visible by using the same fingerprint as the distance score and then reporting each feature separately. Instead of only giving a single ranked list of files, it organises the output by feature name: the exemplar expectation, then the files whose values differ enough to matter.

By feature, I just mean one measurement in the fingerprint. Some features are yes/no questions. I call those boolean features: does the handler have cache invalidation, does it use try/catch, is the query cacheable? Other features are counts or sizes. I call those numeric features: IL byte size, constructor dependency count, entity load count, JOIN count.

Those two kinds of feature need different treatment. For boolean features, the report compares the file with the nearest exemplar shape, not with a global majority. That avoids false positives in cohorts with valid sub-shapes. A paginated query should be compared with the paginated-list exemplar, not with a by-id query. After that match, a difference such as `HasCacheInvalidator`, `HasTryCatch`, or `IsCacheable` is easy to read.

For numeric features, the report compares against the exemplar mean and spread. If the exemplars usually have two or three constructor dependencies and a handler has seven, the report does not need to hide that inside a 6.2 score. If every query exemplar has zero JOINs, the first handler with a JOIN should be visible even if the rest of the file looks ordinary.

The useful output is not just that one handler is far from the exemplars; it is closer to: `ConstructorDependencyCount` expected roughly 2 or 3 and found 7; `EntityLoadCount` expected 1 and found 4; `HasTryCatch` expected false and found true. That is the level where a reviewer can do something with the result.

That changes the review conversation. Instead of saying only that a handler is an outlier, the report can say that dependency count is high, entity loads are high, and try/catch is present where the exemplars do not use it. Review can then decide whether those facts are expected complexity, accidental drift, or a pattern that should become a convention.

The count of affected files matters too, because one handler missing cache invalidation is probably a local question, while fifteen handlers missing it suggests something broader has moved. The exemplar set may be stale, the convention may have shifted, or the rule may need to be encoded directly. Those are different outcomes, and the composite score tends to blur them together.

## What the cohorts surfaced

The three cohorts were useful because each stressed a different part of the idea: command handlers tested whether the feature vector could separate size from shape, query handlers tested whether one cohort could contain multiple legitimate sub-shapes, and EF configurations tested whether the machinery could find a convention nobody had bothered to name.

### Command handlers

The command-handler cohort had 34 members and seven features. Before normalisation, the largest handlers dominated the report; after normalisation, the strongest outliers were two handlers that wrapped work in SERIALIZABLE transactions with retry and rollback.

Those handlers were carrying legitimate business complexity, not making an obvious mistake. The report still did its job because it did not say "fail this"; it said "this is structurally unusual, please look at it." That is the right level of confidence for this kind of tool.

The same cohort also surfaced the inline DTO construction pattern. That one was not just unusual; it was a rule hiding in plain sight. Once found, it became a convention test that catches DTO construction through IL inspection and skips the one result-summary case where no mapper should exist.

### Query handlers

The query-handler cohort had 20 members and three sub-shapes: by-id single-row, unpaginated list, and paginated list. Pinning one exemplar from each sub-shape mattered because there was no honest "average query handler" in that set. A single canonical exemplar would have produced a centroid that represented none of the real shapes well.

The only strong outlier was the query using a SQL JOIN, which was a useful advisory result because it was unusual enough to notice and still defensible as legitimate complexity. The score could not decide that for us; it just made the question hard to miss.

The same cohort also helped move list-query caching from an advisory concern into a hard convention, because list caches could not be reliably invalidated with the available cache abstraction.

### EF configurations

The EF-configuration cohort had 26 members. It pinned a minimal config, a canonical mid-range config, and a richer aggregate config. On the first run, governance caught a configuration class nested inside another configuration file.

Every other configuration had its own file, but the rule had never been named, so the exception had been tolerated. Once surfaced, it was obvious enough to fix and enforce, which is the best kind of result for this system: it found a convention that already existed socially and made it explicit.

## Keep scores advisory

A consistency score should not fail a merge, because distance has a known pathology: it punishes the first good example of a new pattern and rewards the tenth copy of a bad one. If unusual means blocked, the codebase cannot evolve.

Use the score to rank attention rather than label files good or bad, because an outlier has at least three possible meanings: drift to correct, legitimate local complexity, or the first instance of a pattern that may later become an exemplar. The tool cannot decide which one you are looking at; a reviewer has to.

That is why the suite includes the meta-test mentioned earlier. It IL-inspects consistency tests for methods that read `CohortScore.Distance` and compare it with a literal floating-point value, unless the test is explicitly allow-listed as data-relative scorer coverage. The point is mundane but important: nobody should be able to turn an advisory report into `distance < 5.0` without having that argument in review.

## Use prompts for explanation, not scoring

It is tempting to give an LLM the candidate file and the exemplars, then ask whether the candidate is consistent. I think that can be useful commentary, but I would not make it the measuring layer.

Prompt-based judges are not stable enough to trend, and they may share the same blind spots as the generator. A better use is explanation after deterministic scoring: this file is structurally unusual; here are the exemplars; describe the difference for review prep.

If I used a prompt here, I would feed it deterministic evidence: the candidate file, the exemplars, and the top feature differences. The prompt can turn that into review notes, but the score should come from code I can rerun.

## The loop I use now

The workflow I trust now is:

1. Pick one cohort large enough to measure and important enough to drift.
2. Pin three to five exemplars with written justification.
3. Extract a small feature vector that represents shape, not taste.
4. Normalise features before computing distance.
5. Read the report as advisory review guidance.
6. Promote repeatable deterministic findings into convention tests.
7. Update exemplars when a new pattern becomes intentional.

I am not trying to make every file look the same. I am trying to catch surprise while there is still time to decide whether it belongs.

Adapted from [z3d's consistency paper](https://github.com/z3d/z3d-consistency/blob/main/docs/papers/consistency-in-agent-generated-code.md) and the extracted [Z3D.Consistency](https://github.com/z3d/z3d-consistency) library.
