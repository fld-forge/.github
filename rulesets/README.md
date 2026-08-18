# Organization rulesets

Versioned definitions of the organization rulesets and of the custom property
that targets them. They are **not applied automatically**: a change here is
applied deliberately with the commands below, so that protection rules are
reviewed before they take effect.

| File | Object | Targets |
| --- | --- | --- |
| `tier-property.json` | `tier` custom property | Every repository, default `sandbox` |
| `floor-no-destruction.json` | `floor-no-destruction` ruleset | Default branch of every repository |
| `floor-release-tags.json` | `floor-release-tags` ruleset | `refs/tags/v*` in every repository |
| `mature-discipline.json` | `mature-discipline` ruleset | Default branch of repositories where `tier = mature` |

## The two-tier design, and why

A single strict ruleset over `~ALL` would cover every repository created or
transferred in the future, which is the point of organization rulesets. It
would also impose pull requests and signed commits on scratch and
experimentation repositories, where that friction buys nothing.

The rules are therefore split along one line: **the floor prevents
destruction, the tier adds process.**

- **Floor**, applied to `~ALL`. Force-pushing and deleting the default branch
  are blocked, and `v*` tags cannot be deleted, moved or force-updated. None of
  this impedes ordinary work: normal pushes are untouched and tag *creation*
  stays free, which is what release automation needs. What it prevents is
  irreversible loss, in any repository, including ones that do not exist yet.
- **Tier**, applied where `tier = mature`. Adds the contribution discipline:
  changes land through a pull request, and every commit on the branch must be
  signed.

Tag protection sits in the floor rather than in the tier on purpose. A
published version tag is a supply-chain artifact: checksums and attestations
only mean something if the tag they refer to cannot move afterwards. That holds
regardless of how casual the repository is, and creation being free means a
sandbox repository never meets the rule during normal use.

## The `tier` property

`tier` is a single-select organization custom property with values `mature` and
`sandbox`.

Two deliberate choices:

- **The default is `sandbox`.** The property is `required: true` with
  `default_value: "sandbox"`, so every repository carries a value from the
  moment it exists. A new repository is therefore protected by the floor
  immediately and is never strict by accident. Promotion to `mature` is an
  explicit act.
- **Only organization actors can set it** (`values_editable_by: org_actors`).
  A repository administrator cannot move their own repository between tiers,
  which is what stops the strict ruleset from being opt-out.

## How an organization ruleset differs from a repository ruleset

1. **A repository condition is mandatory.** Repository rulesets only carry
   `conditions.ref_name`. Organization rulesets must also say which
   repositories they apply to. Three forms exist: `repository_name` (used here
   with `~ALL` for the floor), `repository_property` (used here to select
   `tier = mature`), and `repository_id` (a fixed list, not used — it would have
   to be edited every time a repository is added).
2. **The endpoint differs**: `orgs/{org}/rulesets` instead of
   `repos/{owner}/{repo}/rulesets`.
3. **Rules aggregate rather than replace.** When an organization ruleset and a
   repository ruleset both apply, the most restrictive result wins, and a
   repository administrator cannot weaken an organization rule. Repository-level
   rulesets are left in place: they are redundant with the organization rules on
   `mature` repositories, and harmless, since aggregation cannot loosen anything.

Two fields returned by the API are deliberately omitted from these payloads:
`allowed_merge_methods` and `required_reviewers`. They are server-side defaults
rather than governed state, and omitting them keeps the definitions equal to the
desired state instead of to an API response.

## Applying

The property must exist before the ruleset that reads it, otherwise the
`repository_property` condition has nothing to match.

```bash
# 1. The property schema
gh api orgs/fld-forge/properties/schema/tier -X PUT --input rulesets/tier-property.json

# 2. Tier assignment (only organization actors may do this)
gh api orgs/fld-forge/properties/values -X PATCH --input - <<'EOF'
{"repository_names": [".github", "governance", "pi-config"],
 "properties": [{"property_name": "tier", "value": "mature"}]}
EOF

# 3. The rulesets, in evaluate mode first
jq '.enforcement = "evaluate"' rulesets/floor-no-destruction.json |
  gh api orgs/fld-forge/rulesets -X POST --input -
```

Then promote to the committed state:

```bash
gh api orgs/fld-forge/rulesets/<id> -X PUT --input rulesets/floor-no-destruction.json
```

Verify what is live at any time:

```bash
gh api orgs/fld-forge/rulesets --jq '.[] | {id, name, target, enforcement}'
gh api orgs/fld-forge/properties/values --jq '.[] | {repo: .repository_name, props: .properties}'

# Everything that applies to one repository, organization rules included
gh api 'repos/fld-forge/<repo>/rulesets?includes_parents=true' \
  --jq '.[] | {name, source_type, source, enforcement}'
```

### Evaluate mode is blind on the Team plan

`evaluate` reports what a rule *would* have blocked without blocking it, and
those reports are read through Rule insights. That endpoint
(`orgs/{org}/rulesets/rule-suites`) answers **403, "Upgrade to GitHub
Enterprise"** on this plan, and the organization Rules settings page offers no
equivalent view.

So an evaluate phase here produces no readable signal. It is still worth using
as a staging step — a ruleset sitting in `evaluate` enforces nothing, so it can
be created and inspected with a `GET` before it can affect anyone — but the
decision to promote has to rest on reasoning about the rules and on the state of
the repositories they will match, not on collected telemetry.

## Ordering constraint

`required_signatures` blocks every pull request whose branch commits are not
verified. Register the signing key on the account **before** the rule reaches
`active`, otherwise all pull requests are blocked, including the one that would
fix the situation. The rule gates *every commit on the pull request branch*, not
only the merge commit, and the block is retroactively cleared once the key is
registered, because the commits were signed all along and only the key was
unknown.

A second consequence, once the rule is active organization-wide: a **new
machine** pushing a branch to any `mature` repository is blocked until that
machine's signing key is registered on the account.
