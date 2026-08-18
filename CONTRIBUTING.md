# Contributing

This is the default contribution guide for repositories in the **fld-forge**
organization. A repository that ships its own `CONTRIBUTING.md` overrides it,
and its instructions win wherever the two differ.

## Contribution flow

Default branches are protected by an organization ruleset: direct pushes are
rejected and every change lands through a pull request. Work on a branch, push
it, open a pull request, wait for the checks to be green, and merge with a
squash commit.

## Signed commits

Every commit on a pull request branch must carry a verified signature. Configure
signing once per clone:

```bash
git config gpg.format ssh
git config user.signingkey ~/.ssh/<your key>.pub
git config commit.gpgsign true
```

The public key must be registered on your GitHub account as a **Signing Key**
(<https://github.com/settings/ssh/new>), not as an authentication key. An
unregistered key produces commits that GitHub reports as unverified, and the
ruleset blocks the merge.

## Quality gates

Each repository defines its own gates and runs them both in a versioned
pre-commit hook and in continuous integration. Run them locally before opening
a pull request; the repository's own `CONTRIBUTING.md` or `AGENTS.md` lists the
exact commands.

## Commit messages

[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) are
mandatory. In repositories with automated releases the type drives the version:
`feat:` bumps the minor, `fix:` and `docs:` bump the patch, and a breaking
change bumps the major; `ci:`, `chore:`, `refactor:` and `test:` do not trigger
a release on their own.

Because pull requests are merged with a squash commit, the **pull request
title** becomes the commit message: it must follow the same convention.

## Actions and dependencies

Third party actions must be pinned to a full commit SHA, with the version in a
trailing comment. This is enforced at the organization level; a tag or branch
reference is rejected.
