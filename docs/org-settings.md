# Organization platform settings

Inventory of the **fld-forge** settings that are not versioned as files. If the
organization ever has to be rebuilt, re-apply them with the commands below.

> Maintained by hand: organization state is not reachable from a test suite.
> Every value below was verified with a `GET` immediately after it was written.
> Last verified: 2026-08-22. Repository control C1 is live on all three
> repositories, and the organization `mature-discipline` ruleset now matches
> its versioned definition — see the Rulesets section.

## Actions: allowed actions and SHA pinning

`allowed_actions` is `all`, and **SHA pinning is required**: an action
referenced by tag or branch is rejected, only a full 40 character commit SHA is
accepted.

Restricting `allowed_actions` to `selected` was considered and rejected. The
repositories to be governed use fourteen distinct third party actions
(`astral-sh/setup-uv`, `googleapis/release-please-action`,
`ossf/scorecard-action`, `anchore/sbom-action`, `gitleaks/gitleaks-action`,
`release-drafter/release-drafter`, `sigstore/cosign-installer`,
`Swatinem/rust-cache`, `dtolnay/rust-toolchain`, `softprops/action-gh-release`,
`taiki-e/install-action`, `extractions/setup-just`, `pnpm/setup`,
`zizmorcore/zizmor-action`). Enumerating them would break workflows on transfer
and would require an organization settings change for every new action, while
protecting less than SHA pinning does: an allow-listed action referenced by a
mutable tag is still vulnerable to that tag being repointed, which is the
attack SHA pinning actually prevents. Every repository in the fleet is already
100 percent SHA pinned, so the requirement is enforceable today without
breaking anything.

```bash
gh api orgs/fld-forge/actions/permissions -X PUT --input - <<'EOF'
{"enabled_repositories": "all", "allowed_actions": "all", "sha_pinning_required": true}
EOF
```

## Actions: default workflow permissions

Default `GITHUB_TOKEN` permission is `read`. `can_approve_pull_request_reviews`
is `true`.

The organization value is a **ceiling**, not a preference: a repository can be
more restrictive than the organization, never less. Leaving
`can_approve_pull_request_reviews` at `false` here would make it impossible for
any repository to enable it, and release automation running with `GITHUB_TOKEN`
would fail with "GitHub Actions is not permitted to create or approve pull
requests". The permissive value at the organization level is what allows each
repository to decide; the per repository desired state is governed separately
by the [governance](https://github.com/fld-forge/governance) baseline,
which deliberately preserves the live per repository value instead of forcing
it.

```bash
gh api orgs/fld-forge/actions/permissions/workflow -X PUT --input - <<'EOF'
{"default_workflow_permissions": "read", "can_approve_pull_request_reviews": true}
EOF
```

## Actions: artifact and log retention

The organization default is **30 days**, lowered from GitHub's default of 90.

The organization value is a **cap**, not a suggestion: once it is set to 30, a
repository can choose less but never more (`maximum_allowed_days` on every
repository drops to 30 to match). Thirty was chosen because the workflows in
the fleet already declare `retention-days: 30` on the artifacts they upload, so
the organization default and the workflow intent become the same number instead
of one silently overriding the other. It also shortens the window in which
build logs, which can incidentally contain more than intended, are retained.

The trade-off accepted: a supply-chain problem discovered more than 30 days
after the fact loses its workflow-log forensic trail. The other forensic
sources are unaffected because they do not expire with logs, namely immutable
releases with their SLSA attestations, signed commits, and code scanning and
Dependabot alert history. Raising the value again is a single `PUT`.

```bash
gh api orgs/fld-forge/actions/permissions/artifact-and-log-retention \
  -X PUT -F days=30

# Verify, at organization scope and then as seen by a repository
gh api orgs/fld-forge/actions/permissions/artifact-and-log-retention
gh api repos/fld-forge/<repo>/actions/permissions/artifact-and-log-retention
```

## Security configuration: fld-forge-baseline (id 267493)

Enforced (`enforcement: enforced`), and the default for **all** new
repositories. Enforced means a repository administrator cannot turn off the
features it controls.

| Feature | State |
| --- | --- |
| `code_security` / `secret_protection` | enabled (required by the API before any of the features below can be enabled) |
| `dependency_graph` | enabled |
| `dependabot_alerts` | enabled |
| `dependabot_security_updates` | enabled |
| `code_scanning_default_setup` | enabled |
| `secret_scanning` | enabled |
| `secret_scanning_push_protection` | enabled |
| `private_vulnerability_reporting` | enabled |
| `secret_scanning_non_provider_patterns` | disabled |
| `secret_scanning_validity_checks` | disabled |

The last two stay off: they are the noisier detections, and neither was part of
the verified reference posture this baseline was frozen from.

### Plan boundary

On the Team plan, code scanning, secret scanning, push protection and private
vulnerability reporting are free on **public** repositories only. Private
repositories need the paid Code Security and Secret Protection add-ons, and
private vulnerability reporting does not exist for them at all. Dependency
graph, Dependabot alerts and Dependabot security updates work everywhere.

`default_for_new_repos` is set to `all` because the API accepts it and because
every repository currently planned for the organization is public. The
consequence to watch: the first **private** repository created or transferred
here will attach this configuration and may report a failed attachment for the
licensed features. Check it with the command below and, if it fails, either
create a second configuration limited to the universally available features or
buy the add-on.

```bash
# Recreate the configuration
gh api orgs/fld-forge/code-security/configurations -X POST --input - <<'EOF'
{"name": "fld-forge-baseline",
 "description": "Baseline security posture for fld-forge repositories.",
 "code_security": "enabled", "secret_protection": "enabled",
 "dependency_graph": "enabled", "dependabot_alerts": "enabled",
 "dependabot_security_updates": "enabled", "code_scanning_default_setup": "enabled",
 "secret_scanning": "enabled", "secret_scanning_push_protection": "enabled",
 "private_vulnerability_reporting": "enabled", "enforcement": "enforced"}
EOF

# Make it the default for new repositories
gh api orgs/fld-forge/code-security/configurations/<id>/defaults \
  -X PUT -f default_for_new_repos=all

# Verify (the /defaults path is write only on a single configuration;
# this plural endpoint is the one that reads back)
gh api orgs/fld-forge/code-security/configurations/defaults \
  --jq '.[] | {default_for_new_repos, id: .configuration.id, name: .configuration.name}'

# Attachment status per repository, once repositories exist
gh api orgs/fld-forge/code-security/configurations/<id>/repositories \
  --jq '.[] | {repo: .repository.full_name, status}'
```

Existing repositories are **not** attached by making a configuration the
default: the default only applies to repositories created or transferred
afterwards. A transferred repository must be attached explicitly:

```bash
gh api orgs/fld-forge/code-security/configurations/<id>/attach \
  -X POST -f scope=selected -F 'selected_repository_ids[]=<repo_id>'
```

## Custom repository properties

| Property | Type | Values | Default | Editable by |
| --- | --- | --- | --- | --- |
| `tier` | single select | `mature`, `sandbox` | `sandbox` | organization actors only |

`tier` selects which repositories the strict ruleset applies to. It is
`required: true` with a default, so every repository carries a value from the
moment it exists, and that value is `sandbox` unless someone deliberately
promotes the repository. `values_editable_by: org_actors` prevents a repository
administrator from moving their own repository out of the strict tier.

Current assignment: `.github`, `governance` and `pi-config` are all `mature`.

```bash
# Read the schema and every assignment
gh api orgs/fld-forge/properties/schema/tier
gh api orgs/fld-forge/properties/values \
  --jq '.[] | {repo: .repository_name, props: .properties}'

# Promote a repository into the strict tier
gh api orgs/fld-forge/properties/values -X PATCH --input - <<'EOF'
{"repository_names": ["<repo>"],
 "properties": [{"property_name": "tier", "value": "mature"}]}
EOF
```

## Rulesets

Three organization rulesets are live, all `active`, all with no bypass actors.
The definitions live in [`rulesets/`](../rulesets/) together with the design
rationale, the apply procedure and the ordering constraint on
`required_signatures`.

| Ruleset | Target | Applies to | Rules |
| --- | --- | --- | --- |
| `floor-no-destruction` | branch | every repository | `non_fast_forward`, `deletion` |
| `floor-release-tags` | tag | every repository | `deletion`, `update`, `non_fast_forward` on `refs/tags/v*` |
| `mature-discipline` | branch | `tier = mature` | `pull_request` (0 approvals), `required_signatures` |

The split is deliberate: the floor prevents irreversible loss everywhere,
including in repositories that do not exist yet, while the contribution
discipline is reserved for repositories deliberately promoted to `mature`. A
new repository therefore starts protected but unencumbered.

Organization and repository rules **aggregate**, most restrictive wins, so the
repository-level rulesets that already exist on the transferred repositories
stay in place and cannot weaken anything.

### Live repository checks and the organization ruleset

Repository control C1 is live on the Python repositories and requires exactly
eight strict status contexts: `CodeQL`, `dependency-review`, `pip-audit`,
`quality`, `secrets-scan`, `semgrep`, `uv-audit`, and `zizmor`. The definitions
live in the `ruleset-main-protection` control of
[`fld-forge/governance`](https://github.com/fld-forge/governance)
(`src/governance_tools/baseline.json`, ADR-0009). Required checks remain
**per repository, never org-wide**.

As of 2026-08-20, the `fld-forge/.github` repository ruleset
`main-protection` (ID `20970461`) requires strict status checks for exactly
`CodeQL` and `validation`, the contexts this repository actually produces. It
has no bypass actors. The live `GET` returned
`updated_at: 2026-08-20T03:28:24.113-04:00`; re-check the evidence with:

```bash
gh api repos/fld-forge/.github/rulesets/20970461 \
  --jq '{id, enforcement, bypass_actors, rules}'
```

No Python-only context is required here because that would block every merge.

This live C1 enforcement is independent of the organization
`mature-discipline` ruleset (ID `20969835`), whose versioned definition pins
`allowed_merge_methods: ["squash"]` on the `pull_request` rule. The live
ruleset matches that definition: a `GET` on 2026-08-22 returned
`allowed_merge_methods: ["squash"]`, `enforcement: active` and no bypass
actors, on a ruleset whose `updated_at` is `2026-08-20T20:01:33.103-04:00`.
A full fleet audit the same day reported 42 of 42 repository cells and 12 of
12 organization cells `OK`, with zero drift. Re-check the ruleset with:

```bash
gh api orgs/fld-forge/rulesets/20969835 \
  --jq '{id, enforcement, bypass_actors, rules}'
```

## Two-factor authentication

**Not enforced yet, and it cannot be enforced from the REST API.**

`two_factor_requirement_enabled` is returned by `GET /orgs/fld-forge` but is
**read only**: a `PATCH` that sets it returns `HTTP 200 OK` with the field
echoed back unchanged. There is no error to react to, so an automated run that
trusts the exit code alone will report success while nothing happened. It has
to be set in the web interface:

> Organization settings -> Authentication security ->
> "Require two-factor authentication for everyone in the fld-forge organization"

### Verify before turning it on

Enabling organization 2FA **removes** every member who does not have 2FA on
their personal account, and it does not spare the owner. With a single member
that member is the owner, so the check below is the difference between a
hardening step and losing access to the organization:

```bash
# The set of members who would be removed. Must be 0.
gh api 'orgs/fld-forge/members?filter=2fa_disabled' --jq '. | length'

# Sanity check that the filter is discriminating rather than empty:
# this must be larger than the count above.
gh api 'orgs/fld-forge/members?filter=all' --jq '. | length'
```

Verified 2026-08-18: `filter=all` returns 1, `filter=2fa_disabled` returns 0.
The sole member already has 2FA, so enabling the requirement removes nobody.

Note that `GET /user --jq .two_factor_authentication` is **not** a usable
substitute: it returns `null` unless the token carries the `user` scope, and a
`null` there says nothing about whether 2FA is on.

## Member privileges

Restrictive values, so that the day a second member is invited they cannot
create repositories outside the governed baseline. All of it is inert while the
organization has one member, which is exactly why it is worth setting now
rather than remembering later.

| Setting | Value | Settable by API |
| --- | --- | --- |
| `members_can_create_repositories` | `false` | yes |
| `members_can_create_public_repositories` | `false` | yes (cascades from the line above) |
| `members_can_create_private_repositories` | `false` | yes (cascades from the line above) |
| `members_can_create_teams` | `false` | yes |
| `default_repository_permission` | `read` | yes (already correct) |
| `members_can_fork_private_repositories` | `false` | yes (already correct) |
| `members_can_delete_issues` | `false` | yes (already correct) |
| `deploy_keys_enabled_for_repositories` | `false` | yes (already correct) |
| `members_can_delete_repositories` | `true` | **no, web UI only** |
| `members_can_change_repo_visibility` | `true` | **no, web UI only** |
| `members_can_invite_outside_collaborators` | `true` | **no, web UI only** |

Organization **owners are exempt** from `members_can_create_repositories`, so
turning it off does not stop the owner from creating repositories. It applies
to the `member` role only.

The last three rows behave exactly like `two_factor_requirement_enabled`: the
`PATCH` returns `HTTP 200 OK` and echoes the old value back. They are part of
the same web-only group and are listed with 2FA in the manual checklist below.

```bash
gh api orgs/fld-forge -X PATCH \
  -F members_can_create_repositories=false \
  -F members_can_create_public_repositories=false \
  -F members_can_create_private_repositories=false \
  -F members_can_create_teams=false \
  -F members_can_fork_private_repositories=false \
  -f default_repository_permission=read

# Always read back: this endpoint ignores read-only fields without complaining.
gh api orgs/fld-forge --jq '{members_can_create_repositories,
  members_can_create_public_repositories, members_can_create_private_repositories,
  members_can_create_teams, members_can_delete_repositories,
  members_can_change_repo_visibility, members_can_invite_outside_collaborators,
  default_repository_permission, members_can_fork_private_repositories}'
```

## Web commit sign-off

`web_commit_signoff_required` is `true`: a commit authored through the GitHub
web interface must carry a `Signed-off-by` trailer.

It affects browser-made commits only. Commits pushed from a clone are unaffected
because they are already signed with SSH, and it does not interact with the
`required_signatures` ruleset rule, which checks a cryptographic signature
rather than a trailer. The point is that the occasional edit made in the browser
is explicit rather than anonymous.

```bash
gh api orgs/fld-forge -X PATCH -F web_commit_signoff_required=true
gh api orgs/fld-forge --jq '{web_commit_signoff_required}'
```

## Manual checklist: settings the API cannot write

Four organization settings are read only over REST and silently ignore a
`PATCH`. They must be set in the web interface, and an automated audit cannot
fix them, only report them.

| Setting | Current | Wanted | Where |
| --- | --- | --- | --- |
| `two_factor_requirement_enabled` | `false` | `true` | Settings -> Authentication security |
| `members_can_delete_repositories` | `true` | `false` | Settings -> Member privileges -> Repository deletion and transfer |
| `members_can_change_repo_visibility` | `true` | `false` | Settings -> Member privileges -> Repository visibility change |
| `members_can_invite_outside_collaborators` | `true` | `false` | Settings -> Member privileges -> Repository invitations |

Re-read them after any web change, because this is the only way to know they
landed:

```bash
gh api orgs/fld-forge --jq '{two_factor_requirement_enabled,
  members_can_delete_repositories, members_can_change_repo_visibility,
  members_can_invite_outside_collaborators}'
```

## Membership and billing

Plan `team`, one seat, single owner. Agents operate with the owner's token and
are not organization members, so no additional seat is billed.
