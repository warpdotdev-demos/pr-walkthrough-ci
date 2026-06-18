# PR Walkthrough CI

Sample GitHub Actions workflows for generating and publishing a live PR walkthrough visualizer using [the /pr-walkthrough skill](https://github.com/warpdotdev/common-skills/blob/main/.agents/skills/pr-walkthrough/SKILL.md).

Copy the workflow files in `.github/workflows/` into your GitHub repository to add PR walkthrough previews:

- `.github/workflows/pr-walkthrough.yml` generates a static PR walkthrough with Oz and uploads it as an Actions artifact.
- `.github/workflows/pr-walkthrough-publish.yml` downloads that artifact, publishes it to GitHub Pages, and posts a sticky PR comment with the live URL.

The workflows are split so the agentic generation job can run with read-only repository permissions. Only the publish/comment workflow receives write permissions.

## What the PR comment links to

The generated site is published to the `gh-pages` branch under a PR-specific path:

```text
https://<owner>.github.io/<repo>/pr-walkthrough/pr-<number>/
```

## Required setup

### 1. Add a Warp API key secret

This example uses [the Oz platform](https://docs.warp.dev/agent-platform/cloud-agents/integrations/github-actions/) to run a coding agent on a GitHub action. To use this, you'll need [a Warp API key](https://docs.warp.dev/reference/cli/api-keys/) with some available tokens.

Create [a repository secret](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-a-repository) named `WARP_API_KEY`, and the generation workflow passes this secret to `warpdotdev/oz-agent-action`:

```yaml
warp_api_key: ${{ secrets.WARP_API_KEY }}
```

Oz action:

- [`warpdotdev/oz-agent-action`](https://github.com/warpdotdev/oz-agent-action)

### 2. Enable GitHub Pages

Enable GitHub Pages for the repository using the `gh-pages` branch and `/` root.

In the GitHub UI:

1. Open repository settings.
2. Go to **Pages**.
3. Set **Build and deployment** source to **Deploy from a branch**.
4. Select branch `gh-pages` and folder `/`.
5. Save.

GitHub docs:

- [Configuring a publishing source for your GitHub Pages site](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site)
- [GitHub Pages quickstart](https://docs.github.com/en/pages/quickstart)

You can also create the Pages site with the GitHub API once the `gh-pages` branch exists:

```bash
pages_body=$(mktemp)
printf '{"source":{"branch":"gh-pages","path":"/"}}' > "$pages_body"
gh api --method POST repos/OWNER/REPO/pages --input "$pages_body"
```

### 3. Allow GitHub Actions to write

The publish workflow needs to push to `gh-pages` and comment on PRs. Ensure workflow permissions allow this.

Repository setting:

1. Open **Settings → Actions → General**.
2. Under **Workflow permissions**, choose **Read and write permissions**.
3. Enable **Allow GitHub Actions to create and approve pull requests** only if your repository requires it for other workflows. These sample workflows do not create PRs.

GitHub docs:

- [Assigning permissions to jobs](https://docs.github.com/en/actions/using-jobs/assigning-permissions-to-jobs)
- [`GITHUB_TOKEN` permissions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication)

## Workflow behavior

### `pr-walkthrough.yml`

Triggers:

- `pull_request` on open, synchronize, reopen, and ready-for-review.
- `workflow_dispatch` with an optional `pr_number`.

Permissions:

```yaml
permissions:
  contents: read
  pull-requests: read
  issues: read
  actions: read
```

Main steps:

1. Resolve the PR number.
2. Check out the PR head.
3. Run `warpdotdev/oz-agent-action` with the public skill `warpdotdev/common-skills:pr-walkthrough`.
4. Verify `.warp/pr-walkthrough/index.html` exists.
5. Upload `.warp/pr-walkthrough/` as an artifact named `pr-walkthrough-<number>`.

### `pr-walkthrough-publish.yml`

Triggers:

- `workflow_run` after `PR Walkthrough` completes successfully.
- `workflow_dispatch` with `run_id` and `pr_number`, useful for retrying publish/comment without rerunning Oz.

Permissions:

```yaml
permissions:
  contents: write
  pull-requests: write
  issues: write
  actions: read
```

Main steps:

1. Resolve the source workflow run and PR number.
2. Download artifact `pr-walkthrough-<number>` from the source run.
3. Publish the artifact contents to `gh-pages` at `pr-walkthrough/pr-<number>/`.
4. Post or update a sticky PR comment containing the live URL.
