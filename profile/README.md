# fld-forge

Personal engineering organization of Francois L. Daigneault.

Repositories here are governed by a single versioned policy: every change lands
through a pull request with signed commits, quality and security gates run on
every push, and releases are automated, immutable and cryptographically
attested.

## What lives here

| Repository | Purpose |
| --- | --- |
| [.github](https://github.com/fld-forge/.github) | Organization profile, shared community health defaults, and the versioned governance policy applied across the organization |

Other repositories are added as they are transferred into the organization.

## Governance

The organization enforces its policy at the platform level rather than by
convention:

- **Actions** - commit SHA pinning is required for every action; the default
  `GITHUB_TOKEN` permission is read-only.
- **Security** - a security configuration named `fld-forge-baseline` is
  enforced and applied by default to every new repository: dependency graph,
  Dependabot alerts and security updates, CodeQL default setup, secret scanning
  with push protection, and private vulnerability reporting.
- **Branches and tags** - organization rulesets require pull requests and
  signed commits on default branches, and protect release tags from deletion,
  update and force-push.

The policy definitions and the full settings inventory live in
[.github](https://github.com/fld-forge/.github).
