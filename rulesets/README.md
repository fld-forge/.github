# Organization rulesets

Versioned definitions of the organization rulesets. They are **not applied
automatically**: a change here is applied deliberately with the commands below,
so that protection rules are reviewed before they take effect.

| File | Ruleset | Targets |
| --- | --- | --- |
| `branch-protection.json` | `branch-protection` | Default branch of every repository |
| `release-tags.json` | `release-tags` | `refs/tags/v*` in every repository |

## How an organization ruleset differs from a repository ruleset

The payloads are modelled on the live pi-config rulesets, with three
differences that are specific to the organization endpoint:

1. **A repository condition is mandatory.** Repository rulesets only carry
   `conditions.ref_name`. Organization rulesets must also say which
   repositories they apply to, here `repository_name.include: ["~ALL"]`, which
   covers every current and future repository. The alternatives are
   `repository_id` (a fixed list) and `repository_property` (a custom property
   filter).
2. **The endpoint differs**: `orgs/{org}/rulesets` instead of
   `repos/{owner}/{repo}/rulesets`.
3. **Rules aggregate rather than replace.** When an organization ruleset and a
   repository ruleset both apply, the most restrictive result wins, and a
   repository administrator cannot weaken an organization rule.

Two fields present in the live pi-config rulesets are deliberately omitted:
`allowed_merge_methods` and `required_reviewers`. They are server-side defaults
rather than governed state, and omitting them keeps the definitions equal to
the desired state instead of to an API response.

## Applying

Create in `evaluate` mode first. In that mode the rules are reported but not
enforced, which surfaces every repository that would be blocked before anything
actually breaks:

```bash
jq '.enforcement = "evaluate"' rulesets/branch-protection.json |
  gh api orgs/fld-forge/rulesets -X POST --input -
```

Review the insights (organization **Settings** -> **Rules** -> **Rule
insights**), then promote to the committed state:

```bash
gh api orgs/fld-forge/rulesets/<id> -X PUT --input rulesets/branch-protection.json
```

Verify what is live at any time:

```bash
gh api orgs/fld-forge/rulesets --jq '.[] | {id, name, target, enforcement}'
```

## Ordering constraint

`required_signatures` blocks every pull request whose branch commits are not
verified. Register the signing key on the account **before** the rule reaches
`active`, otherwise all pull requests are blocked, including the one that would
fix the situation. This failure mode was reproduced on pi-config and is
recorded in its ADR-0010.
