# Dependency updates and the merge queue

Dependabot proposes dependency updates using `.github/dependabot.yml`.
The `Dependabot auto-merge` workflow requests automatic merging for verified
patch and minor updates targeting `main`. A grouped update qualifies only when
every dependency has a patch or minor update type. Major updates and updates without
recognized semver metadata need a maintainer to review and enqueue them.
Nix input revisions may fall into this manual category.

The queue tests each PR with the latest `main` and earlier queued changes.
It builds and merges one entry at a time, using squash merges. Required checks
are `server`, `web`, `flake check`, and both CodeQL language checks. These
workflows run on every PR to `main` and on `merge_group` events, so path filters
cannot leave a required check pending. Failed entries leave the queue and need
attention before they can be queued again.

This setup covers `main`. Updates targeting `v0.x` retain that branch's checks
and manual merge process.

## Set up the GitHub App

An organization administrator completes these steps. Keep the private key out
of commits, issues, and chat.

1. Under the `sidereal-io` organization's **Settings → Developer settings →
   GitHub Apps**, create an App for dependency merging. Disable webhooks;
   the workflow starts the App's work.
2. Grant repository permissions **Contents: Read and write** and **Pull requests:
   Read and write**. GitHub supplies read access to metadata automatically.
   No organization permissions or ruleset bypass permissions are needed.
3. Install the App on **only `sidereal-io/sidereal`**.
4. Copy its Client ID into the repository's Actions variable
   `DEPENDABOT_MERGE_APP_CLIENT_ID`.
5. Generate a private key and upload its PEM contents as the repository's
   Actions secret `DEPENDABOT_MERGE_APP_PRIVATE_KEY`.

The workflow uses `pull_request_target` to read the base branch's workflow and
access these Actions credentials. It checks the PR author and repository,
verifies Dependabot metadata, and never checks out or executes PR code. It
creates an installation token scoped to this repository only for eligible
updates. The token is revoked when the job finishes.

## Activate after the workflow changes reach main

The ruleset payload in `.github/main-merge-queue.json` starts disabled so it can
be imported before the new CI triggers reach `main`. Enabling it too early
would leave queued PRs waiting for checks that cannot run.

1. Merge the setup PR and configure the App credentials above.
2. In **Settings → Rules → Rulesets**, open **main merge queue** and set
   enforcement to **Active**. If it has not been imported, create it with:

   ```bash
   gh api --method POST repos/sidereal-io/sidereal/rulesets \
     --input .github/main-merge-queue.json
   ```

3. Confirm repository **Settings → General → Pull Requests → Allow auto-merge**
   is enabled.
4. On an eligible Dependabot PR, reopen it or let Dependabot push an update to
   trigger the workflow. Existing PRs are not enrolled just by merging this setup.
5. Confirm the workflow requests auto-merge, the queue starts the required
   checks, and the PR merges after they pass. A second queued PR should be
   checked with the first change included.

The workflow refuses to request a merge while the queue rule is inactive.
There are no bypass actors or mandatory review approvals in this ruleset;
reviews remain available, and unresolved review threads block merging.
Maintainers enqueue reviewed major updates using **Merge when ready**.
The queue applies to all PRs targeting `main`, including human-authored PRs.

To pause automatic enrollment, disable the **Dependabot auto-merge** workflow
in the Actions UI. Also cancel auto-merge or remove queue entries already
requested if they should stop. Keep the branch's required checks active.

See GitHub's [merge queue documentation](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue)
and [Dependabot automation guide](https://docs.github.com/en/code-security/tutorials/secure-your-dependencies/automate-dependabot-with-actions).
