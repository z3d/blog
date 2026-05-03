# Measuring consistency in agent-generated code

Status: Published
Date: 3 May 2026
Author: z3d

One of the things I noticed when I worked with good developers was how consistent their code was. You could move from one project to another and become productive almost immediately.

Part of it was the significant amount of effort that went into developer experience, but equally important was the consistency that permeated all the repositories. Patterns still evolved, but they evolved in a controlled and deliberate way.

That consistency is what I wanted to measure once agents started writing more of the code.

The check I am describing is fairly small. Pick a cohort of files that solve the same kind of problem. Pin a few examples that show the intended shape. Extract a feature vector from every file in the cohort, normalise those features, then report which files have moved away from the examples and which features caused the movement.

The report is advisory. It should not say "fail the build because this handler scored 6.2." It should say "this handler is far from the exemplars because it has an unusual dependency count, retry shape, cache behaviour, or entity-load pattern." Review can then decide whether the file is drift, legitimate local complexity, or the first instance of a pattern that should become an exemplar.

Convention tests come after that review. When an advisory finding turns into a rule the team can state clearly, it should leave the consistency report and become a deterministic test.

Agent-generated code makes this useful because the problem can accumulate quietly. The code compiles, the unit tests pass, and the handler answers the right request. After enough of those changes, though, the codebase can start to feel less like one system and more like a folder of plausible one-offs.

A single handler might only be a little odd: one extra dependency, inline DTO construction, a cache invalidation path no sibling uses. A query might introduce a SQL JOIN in a slice where the surrounding queries are deliberately simple. An EF configuration can tuck a class inside another file even though every other configuration has its own file. None of this is automatically wrong. The point is to make it visible while review still has context.

## What I mean by consistency

Correctness, convention compliance, style, and consistency tend to get blurred together, but they want different tools.

The concrete examples here come from a .NET codebase. If every command has to have a validator, that belongs in a deterministic convention test. If query handlers are not allowed to use EF Core, or command handlers are not allowed to use Dapper, encode that and let CI be boring. A formatter can settle whitespace. An analyser can catch a narrow syntax rule. None of those tools tells you when a file has all the right ingredients but an unfamiliar shape.

In this work, consistency means idiom: proportions, dependency count, decomposition, cache behaviour, private helpers, entity loads, and rare combinations of features. A file can satisfy every convention and still be the one where a reviewer should slow down and ask why it does not resemble its peers.

## Where convention tests fit

Convention tests are for decisions the team has already made. In the .NET reference codebase, that means rules like: every command handler must have a validator, query handlers must not use EF Core, and command handlers must not use Dapper. Encode those rules directly and let CI catch them every time.

Consistency checks are for questions the team has not fully named yet. A report might show that a handler is far from the exemplars because it constructs DTOs inline instead of using the mapper. The first time that appears, it is a review question. It might be harmless local complexity, or it might be a real convention hiding in plain sight.

Once review decides "we should never do that, except for this one result-summary case", the finding should graduate into a convention test. From then on, the consistency report no longer has to rediscover it. CI can catch the rule cheaply, and the consistency check can keep looking for softer drift.

That split matters because consistency scores are not policy. Scores point attention at unusual code. Convention tests protect settled agreements. Patterns can still evolve, but they evolve by updating exemplars and conventions deliberately, not by letting every new outlier redefine normal.

## Prior art

None of this starts from empty air. Style guides, formatters, and analysers already make small choices disappear. Architecture-test tools such as [ArchUnit](https://www.archunit.org/userguide/html/000_Index.html) and [NetArchTest](https://github.com/BenMorris/NetArchTest) let teams encode structural rules in tests. Evolutionary architecture uses [fitness functions](https://www.thoughtworks.com/en-us/insights/books/building-evolutionaryarchitectures-second-edition) as automated feedback on architectural characteristics. [Approval and snapshot testing](https://approvaltestscpp.readthedocs.io/en/latest/generated_docs/ApprovalTestingConcept.html) compare current output with a known-good artefact.

This work sits near those ideas, but it is trying to fill a different gap. It is not a formatter, because the thing being measured is not whitespace or naming. It is not an architecture test, because it does not start with a rule. It is not a snapshot test, because the goal is not exact reproduction. It is not a replacement for review, because the output is a reason to look closer, not a verdict.

It also differs from the common exemplar style in LLM workflows, where examples are used as instructions to the generator: here are three files, now write the next one like this. That can be useful, but it is generation guidance. The exemplars in this framework are a measuring instrument. They anchor the reference distribution for a cohort so review can see which files are structurally far away and which features made them far away.

That distinction matters because copying one exemplar too closely would be wrong. The query-handler cohort has multiple legitimate shapes: by-id single-row, unpaginated list, and paginated list. A single "make it like this" example would flatten that variation. A pinned exemplar set can preserve the allowed variation while still making drift visible.

## Start with a cohort

A consistency check only makes sense inside a cohort, meaning a group of files that solve the same kind of problem. Measuring every class in a repository against every other class would mostly measure nonsense. In the original consistency paper, three cohorts were enough to make the idea concrete:

- 34 command handlers.
- 20 query handlers.
- 26 EF Core configurations.

In this kind of .NET application, a command handler receives a request that changes state. It loads entities, calls domain methods, saves changes, invalidates caches, and returns a DTO. A query handler receives a read request and returns read-model data. It should usually be thinner, dependency-light, and shaped around the kind of result it returns. Treating both as just "handlers" would hide the fact that they are meant to use different approaches.

Those categories were useful because they had enough members, a clear role, and different kinds of legitimate variation. The command handlers were not one identical template. The exemplar set covered a small state transition, an entity creation path with cross-entity validation, and a heavier orchestration path that touched several reads before writing. The query handlers had a cleaner split: by-id lookup, bounded list, and paginated list. The EF configurations ranged from small table mappings to richer aggregate mappings.

That matters because measuring consistency is not the same as forcing every file into one shape. If a cohort has several intended shapes, the exemplar set needs to preserve them. Otherwise the tool will punish ordinary variation and call it drift.

Each cohort needs three to five pinned exemplars. I would not compute against the current average of the whole cohort, because the current average can already contain the drift you are trying to find. Once drift is baked into the reference point, new drift starts looking normal.

The exemplars are the files you would point a new engineer, or a new agent session, at and say: write like this. They need written justification and review, because they become part of the measuring instrument. They should represent the intended shape of the cohort, not the longest file, the most defensive file, or whatever happened to land first.

## Normalise before believing the number

Once the cohort and exemplars are chosen, the next step is to compare each candidate file with the shape the exemplars represent. That comparison starts by turning every file into a small set of measurements. For a handler, those measurements might include IL byte size, constructor dependency count, logger presence, cache invalidation, try/catch usage, private method count, and entity load count.

The distance score is just a way to combine those measurements into a review signal. It is not trying to decide whether the file is good or bad. It is trying to answer a narrower question: which files look furthest from the pinned exemplar shape?

That only works if each measurement gets a fair say.

Those features do not naturally live on the same scale. IL byte size can be in the hundreds or thousands. A boolean such as "has a logger" is only 0 or 1. If those raw values go straight into one distance calculation, the feature with the biggest numbers gets the loudest voice.

That is exactly what happened. In the command-handler cohort, unnormalised `IlByteSize` accounted for 98.5% of the distance contribution and was the top contributor for every handler. The report looked mathematical, but it was mostly answering one question: which handler has more IL?

Normalisation fixes that by making each feature relative to the cohort before distance is calculated. A count of seven constructor dependencies matters because most handlers have fewer. A try/catch matters because almost no handlers use one. A large IL body still matters, but it no longer drowns out everything else.

After normalising the features, the report changed. Entity loads, IL byte size, try/catch usage, and dependency count all started showing up as contributors. The outliers were no longer just files with more code. They were files with unusual combinations of features.

That is the rule I would keep: normalise before calculating distance, and always show which features drove the score. Otherwise the number can look objective while mostly measuring the wrong thing.

## What the scoring layers showed

The implementation tried three ways to compare a file with the exemplars: structural distance, AST/IL shingle similarity, and embedding distance. I expected three mostly independent signals. What I got was more useful than that, but less elegant.

The structural layer is where Mahalanobis distance came in. Each file became a point in feature space. Plain distance asks how far that point is from the exemplar centre. Mahalanobis asks a better question: given how the exemplar features usually move together, how surprising is this combination of values? A bigger handler with more private helpers may be normal for the cohort. A handler with an ordinary size but an unusual dependency count, try/catch shape, and entity-load pattern may be much more interesting.

That is why normalisation mattered so much. Mahalanobis can account for feature scale and correlation, but only after the inputs are put on comparable footing. With a small exemplar set, the covariance estimate also needs regularisation so the score does not become brittle. The practical version is simple enough: when there is not enough evidence to trust feature correlations, the score falls back towards standardised distance; when the cohort has enough signal, the correlations start to matter.

Structural distance carried the work once normalisation was fixed. It found files with unusual combinations of features: more entity loads than the cohort expected, different dependency counts, retry shapes, private helpers, try/catch blocks.

The shingle layer took a different approach. It ignored names and most operands, looked at short runs of IL opcodes, and asked whether the file's skeleton resembled the exemplar skeletons. I wanted this to catch the "same feature vector, different control flow" case. It did that sometimes, but it often moved with structural distance. That made it useful as a second opinion, not a separate source of truth.

The embedding layer was the third approach. It turned each file's source tokens into a vector and compared those vectors with cosine similarity, hoping to find files that looked structurally normal but talked about a different problem. In practice it often noticed vocabulary before it noticed design. A handler with domain-specific names can look far from the exemplars even when its shape is normal. That makes embeddings noisy as a score.

The useful exception was inline DTO construction. The embedding layer flagged handlers that built DTOs directly instead of using the matching mapper. It probably found them for vocabulary reasons, but the pattern was real. Once we understood it, the right move was to stop treating it as an embedding result and turn it into a convention test.

That is the relationship I want between the layers. Scores and distance reports are a way to discover candidates for review. Convention tests are where repeatable rules end up.

## Per-feature divergence explained the score

The composite distance score was useful for sorting attention, but it did not explain enough on its own. If a report says a handler is a 6.2, the next question is obvious: why?

Per-feature divergence answered that better because it removes the aggregate score from the middle of the conversation. Instead of asking which file is farthest from a centroid, it asks a smaller question for each feature: does this file differ from the exemplar shape on this one thing?

For boolean features, the comparison is against the exemplar mode. If the pinned command-handler exemplars all have a cache invalidator and 15 handlers do not, the report can say that directly. For numeric features, the comparison is against the exemplar range: if the exemplars have one or two constructor dependencies and a handler has seven, the report does not need to hide that behind a single score.

That changes the review conversation. "This handler is an outlier" is only a prompt to investigate. "This handler is unusual because it has seven constructor dependencies, loads four entities, and has the only try/catch in the cohort" is already review material. The reviewer can ask whether those differences reflect real business complexity, drift, or a pattern that deserves to become part of the cohort.

It also shows whether a difference is local or spreading. One handler without `HasCacheInvalidator` is probably a local question. Fifteen handlers differing on that feature means the exemplar set may be stale, the convention may have shifted, or a rule needs to be written. Those are different outcomes, and a composite score tends to blur them together.

Composite distance answers "where should I look first?" Per-feature divergence answers the question that matters once you get there: what is different, and is it isolated or becoming normal?

## What the cohorts surfaced

The three cohorts were useful because they failed in different ways. Command handlers tested whether the feature vector could separate size from shape. Query handlers tested whether one cohort could contain multiple legitimate sub-shapes. EF configurations tested whether the machinery could find a convention nobody had bothered to name.

### Command handlers

The command-handler cohort had 34 members and seven features. Before normalisation, the largest handlers dominated the report. After normalisation, the strongest outliers were two handlers that wrapped work in SERIALIZABLE transactions with retry and rollback.

Those handlers were not wrong. They were carrying legitimate business complexity. The report still did its job because it did not say "fail this." It said "this is structurally unusual, please look at it." That is the right level of confidence for this kind of tool.

The same cohort also surfaced the inline DTO construction pattern. That one was not just unusual; it was a rule hiding in plain sight. Once found, it became a convention test that catches DTO construction through IL inspection and skips the one result-summary case where no mapper should exist.

### Query handlers

The query-handler cohort had 20 members and three sub-shapes: by-id single-row, unpaginated list, and paginated list. Pinning one exemplar from each sub-shape mattered because there was no honest "average query handler" in that set. A single canonical exemplar would have produced a centroid that represented none of the real shapes well.

The only strong outlier was the query using a SQL JOIN. That was a good advisory result: unusual, worth noticing, and still defensible as legitimate complexity. Nothing about the score could decide that for us. It just made the question hard to miss.

The same cohort also helped move list-query caching from an advisory concern into a hard convention, because list caches could not be reliably invalidated with the available cache abstraction.

### EF configurations

The EF-configuration cohort had 26 members. It pinned a minimal config, a canonical mid-range config, and a richer aggregate config. On the first run, governance caught a configuration class nested inside another configuration file.

Every other configuration had its own file, but the rule had never been named, so the exception had been tolerated. Once surfaced, it was obvious enough to fix and enforce. That is the best kind of result for this system: it found a convention that already existed socially and made it explicit.

## Keep scores advisory

A consistency score should not fail a merge. Distance has a known pathology: it punishes the first good example of a new pattern and rewards the tenth copy of a bad one. If unusual means blocked, the codebase cannot evolve.

Use the score to rank attention, not to label files good or bad. An outlier has at least three possible meanings: drift to correct, legitimate local complexity, or the first instance of a pattern that may later become an exemplar. The tool cannot decide which one you are looking at; a reviewer has to.

For that reason, the consistency suite should include a meta-test that prevents literal threshold gates from creeping in. If a test starts saying "distance must be less than 5.0", the advisory layer has quietly become a brittle policy layer.

## Use prompts for explanation, not scoring

It is tempting to give an LLM the candidate file and the exemplars, then ask whether the candidate is consistent. I think that can be useful commentary, but I would not make it the measuring layer.

Prompt-based judges are not stable enough to trend, and they may share the same blind spots as the generator. A better use is explanation after deterministic scoring: this file is structurally unusual; here are the exemplars; describe the difference for review prep.

The one open area where a prompt might earn a scoring role is semantic duplication: given the nearest existing handlers, is this new handler solving the same problem under a different name? I would still leave that unbuilt until there is a real case to tune against.

## The loop I would run

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
