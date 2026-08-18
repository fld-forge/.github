# .github

The official settings repository for the **fld-forge** organization. It has two
distinct roles.

## 1. Organization profile and shared defaults

GitHub reads certain paths in this repository and applies them organization
wide:

| Path | Effect |
| --- | --- |
| `profile/README.md` | The public page rendered at <https://github.com/fld-forge> |
| `SECURITY.md` | Default security policy for every repository that has none of its own |
| `CONTRIBUTING.md` | Default contribution guide, same inheritance rule |
| `CODE_OF_CONDUCT.md` | Default code of conduct, same inheritance rule |
| `.github/PULL_REQUEST_TEMPLATE.md` | Default pull request template |
| `.github/ISSUE_TEMPLATE/` | Default issue forms |

Inheritance is a fallback, not an override: a repository that ships its own
version of one of these files keeps it. Editing a file here changes the default
for every repository that relies on it, so treat this content as organization
policy rather than as documentation.

## 2. Versioned governance policy

| Path | Contents |
| --- | --- |
| `rulesets/` | Organization ruleset definitions as JSON, reviewed here before being applied through the API |
| `docs/org-settings.md` | Inventory of every organization level setting, with the exact command to re-apply it |

Rulesets are kept as files so that a change to the organization's protection
rules is proposed, reviewed and recorded like any other change, instead of
being an untracked click in a settings page.

The per-repository desired state and the tooling that audits the fleet against
it live in a separate repository:
[governance](https://github.com/FrancoisLDaigneault/governance).
