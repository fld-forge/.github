# Security Policy

This is the default security policy for repositories in the **fld-forge**
organization. A repository that ships its own `SECURITY.md` overrides it.

## Supported versions

For a repository that publishes releases, only the latest GitHub release is
supported. `main` is the development branch: fixes land there first and ship
with the next release.

## Reporting a vulnerability

Do not open a public issue. Use GitHub's private reporting on the affected
repository: **Security** tab -> **Report a vulnerability**. For this repository,
[open a private vulnerability report directly](https://github.com/fld-forge/.github/security/advisories/new).

You will receive an initial response within 7 business days.

## Automated controls

Every repository in the organization inherits an enforced security
configuration: dependency graph, Dependabot alerts and security updates, CodeQL
default setup, secret scanning with push protection, and private vulnerability
reporting. Organization rulesets require pull requests and signed commits on
default branches and protect release tags. Actions must be pinned to a full
commit SHA, and the default `GITHUB_TOKEN` permission is read-only.

Individual repositories add their own gates on top: secret scanning of the full
git history, dependency auditing, workflow auditing, and static analysis.

## Verifying release assets

Releases that ship build artifacts also ship SHA-256 checksums
(`SHA256SUMS`), an SPDX SBOM (`sbom.spdx.json`) and GitHub build provenance
attestations. To verify a downloaded asset:

```bash
gh attestation verify <asset> --repo fld-forge/<repository>
sha256sum --check SHA256SUMS   # inside the folder holding the downloaded assets
```
