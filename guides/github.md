# How to secure GitHub repositories

It is recommended to turn on 2FA for most repositories, especially if building
a package that is a dependency of multiple other packages. Note that this is
not a complete security guarantee, as it can easily be circumvented.

At least for critical packages, administrators should enable branch
protection: allow only repository owners to merge PRs, and only after CI
passes. Contributors should contribute via forks. It is recommended to
contribute via PRs instead of direct pushes to main branch, even for
repository owners, as this prevents accidental pushes of bad code. Only
repository owners should be able to change the main branch, by merging PRs.

> [!WARNING]
> It is recommended to run workflows only after the PR has been reviewed. The
> "Require approval for all outside collaborators" setting is the recommended
> one. However, this can cause friction in repositories with a high number of
> PRs but small number of contributors. In this case, if all contributors are
> trusted, "Require approval for first-time contributors" is a valid option,
> but dedicate more efforts to separate sensitive steps into workflows that
> can only run after approval.

It is ideal to require PR reviews before merging. For security critical
packages, having at least two reviewers is ideal, as this alleviates the risk
of [sock-puppet accounts][sock].

To keep dependencies up to date, it is advisable to enable tools such as
[Dependabot][dependabot] or [Renovate][renovate]. These create PRs to update
dependencies and alert on vulnerabilities in dependencies. Both of these tools
can be configured to send a PR at regular intervals (e.g., once a week).

> [!NOTE]
> Neither [Dependabot][dependabot-2745] nor [Renovate][renovate-8187]
> currently support Haskell dependencies. Use these tools to update
> dependencies from other ecosystems (in multi-language projects).

For Haskell dependencies, one way to keep them up to date is to use
[haskell-bounds-bump-action][haskell-dep-bump] as a GitHub Action.

It is recommended to install [Scorecards action][scorecard] for public
repositories and trying to improve the score as high as possible. This is a
scanner for security best practices, most of which are already discussed in
this document.

## Securing GitHub Actions and GitHub Apps

It is preferable to use the hosted runners whenever possible. The large
runners supported by GitHub come with usage costs, but are better from the
point of view of supply chain security than hosting your own infrastructure
for CI.

> [!NOTE]
> If you have the ability to secure your own hosting infrastructure, it is
> feasible to use that instead of GitHub's large hosted runners.

All GitHub Actions workflows should restrict [permissions][gha-permissions] to
the minimum scope needed. Scope the permissions at job level, instead of at
workflow level.

```
  check-tests:
    permissions:
      actions: read
```

Minimize usage of actions that create PRs or push code to branch, and review
those that are indispensable for the repository. Thoroughly inspect actions
that can approve PRs and workflows that are triggered after a PR has been
approved (time-of-check-vs-time-of-use type of concerns). Similarly,
thoroughly review any GitHub application (bot, tool) that you use to maintain
the repository (e.g., [mergify][mergify]).

If using actions which are defined outside of your organisation (that is,
using the `uses` syntax), these should be pinned by commit hash. Don't pin by
version tag as these tags can be forced pushed. The [Scorecards Pinned
Dependencies workflow][scorecard] can help identify which actions should still
be pinned. [Dependabot][dependabot] is able to upgrade the versions of these
actions periodically:

```
      - name: Upload sdist as an artifact for later jobs in workflow
        uses: actions/upload-artifact@a8a3f3ad30e3422c9c7b888a15615d19a852ae32 # v3.1.3
        with:
          ...
```

If secrets are used as arguments to commands for the workflows, pass them via
environment variables. The action log can leak command line arguments!

If using secrets, these should be scrubbed as soon as they are no longer
needed. Make sure they don't show up in logs, including debug runs of the
workflow!

Be extra careful of GitHub context variables (e.g.,
`github.event.issue.title`) which are vulnerable to command injection attacks.
Don't use these directly in commands. Prefer passing them via environment
variables. Use [Scorecards][scorecard] to identify dangerous workflows.

Audit carefully all workflows of type `pull_request_target` and
`workflow_run`. For these, don't use secrets at all and separate the sensitive
parts into separate runs. Validate all inputs and only run these workflows on
protected branches. Never use `pull_request_head.ref` as the branch head
reference since this can be updated between the time the PR gets approved and
the time the CI workflow starts executing.

### Using self-hosted runners

If using self-hosted runners, create 2 separate pools: one pool to be used
only for CI for PRs and another one to be used for releases and other
sensitive jobs. Ideally, don't share these pools across multiple projects in
the same organisation.

> [!WARNING]
> Repository-level self-hosted runners are accessible to any workflow in the
> repository. Never trigger a CI job before the PR is reviewed as a PR can
> maliciously alter the workflow to attack the hosted runner.

Don't allow any secrets in jobs that run in the CI pool. Only use secrets on
the release pool and never trigger jobs on the release pool on code that is
not reviewed.

Ideally, the self-hosted runners should be ephemeral. Destroy them as soon as
the CI run finishes (see [Github documentation for scripting
this][ephemeral-runner]). If caching between jobs is required, periodically
recycle all runners (e.g., destroy them every week).

Do not allow writing to the repository from the workflow actions if using
self-hosted runners. A compromise of the runner can result in compromise of
the repository otherwise.

All CI runners should have a timeout. All workflows that run on CI runners
should not run on forks of the repository. Gate workflow runs by checking the
value of the `github.repository` variable:

```
jobs:
  build:
    if: github.repository == 'haskell/haskell-language-server' # Don't do this in forks
```

You can also use `pull_request.author_association` to differentiate between
owners of the project and collaborators added to the project that also have
write access.

Monitor all activity on the runners and have separate infrastructure to scan
for malicious activity.

[dependabot-2745]: https://github.com/dependabot/dependabot-core/issues/2745
[dependabot]: https://github.com/dependabot
[ephemeral-runner]: https://docs.github.com/en/rest/actions/self-hosted-runners?apiVersion=2022-11-28#create-configuration-for-a-just-in-time-runner-for-an-organization
[gha-permissions]: https://docs.github.com/en/actions/using-jobs/assigning-permissions-to-jobs
[haskell-dep-bump]: https://github.com/nomeata/haskell-bounds-bump-action
[mergify]: https://github.com/marketplace/mergify
[renovate-8187]: https://github.com/renovatebot/renovate/issues/8187
[renovate]: https://github.com/renovatebot/renovate
[scorecard]: https://github.com/ossf/scorecard-action
[sock]: https://en.wikipedia.org/wiki/Sock_puppet_account
