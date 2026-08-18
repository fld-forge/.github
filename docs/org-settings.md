# Organization platform settings

Inventory of the **fld-forge** settings that are not versioned as files. If the
organization ever has to be rebuilt, re-apply them with the commands below.

> Maintained by hand: organization state is not reachable from a test suite.
> Every value below was verified with a `GET` immediately after it was written.
> Last verified: 2026-08-18.

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
by the [governance](https://github.com/FrancoisLDaigneault/governance) baseline,
which deliberately preserves the live per repository value instead of forcing
it.

```bash
gh api orgs/fld-forge/actions/permissions/workflow -X PUT --input - <<'EOF'
{"default_workflow_permissions": "read", "can_approve_pull_request_reviews": true}
EOF
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

The last two stay off to mirror the verified pi-config posture.

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
 "description": "Baseline security posture for fld-forge repositories, mirroring the verified pi-config reference state (see the governance repository).",
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

## Rulesets

Not applied yet. The definitions live in [`rulesets/`](../rulesets/) with the
apply procedure and the ordering constraint on `required_signatures`.

## Membership and billing

Plan `team`, one seat, single owner. Agents operate with the owner's token and
are not organization members, so no additional seat is billed.
