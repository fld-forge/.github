# NORTHSTAR - .github

Steering KPIs for this repository (organization profile, shared defaults and
versioned governance policy for **fld-forge**). One North Star KPI per axis.
Every value is measured; an unmeasured value is written as unmeasured, never
invented. Updated whenever a measurement changes category.

A `Current` value that a gate verifies on every run is written plain: it cannot
drift without turning something red. A value read by hand carries the date it
was read, because nothing keeps it current afterwards - the `Measurement`
column is how to refresh it, and a dated reading stays true even once stale.

## Speed

| KPI | Current | Target | Measurement |
| --- | --- | --- | --- |
| Validation CI duration | 5 s, median of the last 11 successful `main` runs, read 2026-08-22 | < 1 min | `validation` job start-to-completion time, taken as a median over the recent successful `main` runs - 3 to 6 s observed across 11 |

This repository holds policy, profile and documentation, not code, so it has no
test suite: the fleet's `Median cost per test` and `Test suite duration`
indicators have nothing to divide here and are deliberately absent rather than
forced. What does carry over is the discipline they were changed for - a single
reading of a noisy figure is not a measurement, so the reading above is a median
and says how many runs it spans.

## Security

| KPI | Current | Target | Measurement |
| --- | --- | --- | --- |
| Rulesets active without bypass | 3 of 3 | 100% | The `validation` job fails unless every ruleset has `enforcement: active` and `bypass_actors: []` |

Supporting indicator: OpenSSF Scorecard. The target is a successful published
scan every week, measured by the latest `OpenSSF Scorecard` workflow run and
the public `api.scorecard.dev` result.

## Maintainability

| KPI | Current | Target | Measurement |
| --- | --- | --- | --- |
| Age of the org-settings inventory | 0 days, read 2026-08-22 | < 90 days since last verification | The `Last verified` date at the top of `docs/org-settings.md` against the UTC calendar date |

## Scalability

| KPI | Current | Target | Measurement |
| --- | --- | --- | --- |
| Undocumented manual steps to onboard a repository | 0 known | 0 | Every organization-level step needed by a new repository has its exact command in `docs/org-settings.md` or the [governance](https://github.com/fld-forge/governance) baseline |

Measurement cadence: the `validation` job runs on every push and pull request
to `main`. The dated inventory in `docs/org-settings.md` is refreshed whenever
an organization setting changes, and its age is reviewed when this file is
touched.
