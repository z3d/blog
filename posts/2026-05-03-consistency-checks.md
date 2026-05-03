# Consistency checks

Status: Draft
Date: 2026-05-03

Convention tests are for rules. Consistency checks are for patterns worth looking at.

Some architecture rules are simple enough to fail a build. If every command needs a validator, write that as a convention test and let CI enforce it.

Other things are softer. A handler might have more dependencies than usual. A query might have a shape that does not look like the rest of the queries. That does not always mean it is wrong, but it is worth noticing.

I have been treating those as consistency checks: compare repeated code patterns, report outliers, and leave the decision to review.

## The split

Convention tests should be boring. They encode rules that are always true.

- Commands and queries have validators.
- Query handlers do not use EF Core.
- Command handlers do not use Dapper.
- SQL queries avoid `SELECT *`.

Consistency checks are different. They answer a weaker question: does this file look unusually far from the examples we trust?

## Why keep them advisory?

Because unusual code can be the right code. Turning every outlier into a failure makes the test suite noisy and brittle.

The useful loop is smaller: report the drift, review it, and only promote it to a convention when the same issue keeps mattering.

## The point

The goal is not to make a codebase uniform. The goal is to make surprise visible early enough that someone can decide whether it belongs.
