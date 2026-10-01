# NeonCodex Agent for GitHub Actions

Mention `@neoncodex` in an issue, a pull request or a review comment, and the NeonCodex coding agent makes the change on a branch and opens a pull request. On a pull request it pushes to that pull request's branch instead. Every run reports back in one comment with the agent's summary and the files it changed.

## Set up

1. Create an API key at [neoncodex.io/dashboard/developer](https://neoncodex.io/dashboard/developer) and save it as a repository secret named `NEONCODEX_API_KEY` (Settings, Secrets and variables, Actions).
2. Add `.github/workflows/neoncodex.yml`:

```yaml
name: NeonCodex
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  workflow_dispatch:
    inputs:
      prompt:
        description: What should the agent do?
        required: true

permissions:
  contents: write
  pull-requests: write
  issues: write

jobs:
  neoncodex:
    if: >-
      github.event_name == 'workflow_dispatch' ||
      contains(github.event.comment.body, '@neoncodex') ||
      contains(github.event.issue.body, '@neoncodex') ||
      contains(github.event.issue.title, '@neoncodex')
    runs-on: ubuntu-latest
    timeout-minutes: 45
    concurrency:
      group: neoncodex-${{ github.event.issue.number || github.event.pull_request.number || github.run_id }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - uses: BrowserlessAPI/neoncodex-action@v1
        with:
          api-key: ${{ secrets.NEONCODEX_API_KEY }}
          prompt: ${{ github.event.inputs.prompt }}
```

3. To let the workflow open pull requests, turn on "Allow GitHub Actions to create and approve pull requests" in Settings, Actions, General. Without it the agent still pushes its branch and the comment links to a page where you open the pull request yourself.

Then comment on any issue:

> @neoncodex the signup form accepts empty emails. Add validation and a test.

## What starts it

| Event | When |
|---|---|
| `issue_comment` | A new comment that mentions the trigger phrase, on an issue or a pull request |
| `pull_request_review_comment` | A new review comment on a line of code that mentions it (the file, line and diff are passed to the agent) |
| `pull_request_review` | A submitted review whose text mentions it |
| `issues` | An issue opened, assigned, labeled or edited whose title or text mentions it |
| `workflow_dispatch`, `schedule`, anything else | Always, with the `prompt` input as the task |

Only people with a role in `allowed-roles` (default admin, maintain, write) can start it from comments and issues. Comments from bots, and the action's own comments, are ignored.

## Inputs

| Input | Default | What it does |
|---|---|---|
| `api-key` | | Your NeonCodex API key (required). |
| `github-token` | `github.token` | Token for comments, branches and pull requests. |
| `prompt` | | The task for workflow_dispatch and scheduled runs. For comment runs it is added as standing instructions. |
| `trigger-phrase` | `@neoncodex` | The word that starts the agent. |
| `mode` | `agent` | `agent` changes code; `ask` and `plan` only answer; or the id of a custom mode in `.neoncodex/modes`. |
| `model` | auto | A model id, for example `claude-sonnet-5-5`. |
| `allowed-roles` | `admin,maintain,write` | Who may start it. |
| `trust-project` | `false` | Run the repository's `.neoncodex/hooks.json` and project MCP servers. |
| `browser` | `false` | Let the agent use a headless browser (needs Chrome or Chromium on the runner). |
| `branch-prefix` | `neoncodex/` | Prefix for new branches. |
| `base-branch` | checked-out branch | Branch new pull requests target. |
| `draft-pr` | `false` | Open pull requests as drafts. |
| `commit-name`, `commit-email` | `github-actions[bot]` | Author of the agent's commits. |
| `cli-version` | `^1.6.0` | Version of the `neoncodex` npm package to run. |
| `timeout-minutes` | `30` | Stop the agent after this long. |
| `isolation` | `auto` | `auto`/`user`: the agent runs as a separate user on a copy of the checkout (Linux runners with passwordless sudo, like GitHub-hosted ones). `none`: it runs as the runner user, which is required on macOS, Windows and most self-hosted runners; it could then read the job's token. |
| `api-url` | | Only if your organization runs its own NeonCodex server. |

## Outputs

`status` (completed, failed, cancelled or skipped), `branch`, `pr-url`, `files-changed`, `output` (the agent's answer).

## How a run works

1. The action checks who asked, reacts to the comment and posts a "working on this" comment with a link to the run.
2. It creates a branch (`neoncodex/issue-12-a1b2c3`), or checks out the pull request's branch when the request is on a pull request from the same repository.
3. It copies the checkout (with a copy of `.git` that has no credentials) and runs `npx neoncodex agent --json --yes` there as a separate user, with your request on standard input. The project's rules (AGENTS.md, CLAUDE.md, .neoncodex/rules) and custom modes apply as they do on your computer.
4. It commits what changed, pushes, opens a pull request (or updates the existing one), and edits its comment with the summary and the list of changed files.

## Security

- **A separate user and a copy.** The agent runs as its own `neoncodex-agent` user on a copy of the checkout. It can't read the action's environment (which holds the GitHub token), the real `.git` folder, or credentials saved by `actions/checkout`, which are locked to the runner user while it runs. Its leftover processes are stopped when it finishes. Use `persist-credentials: false` on `actions/checkout` anyway, so no credentials are on disk at all.
- **Changes come back as files only.** When the agent finishes, its file changes are copied into the real checkout (symlinks stay links, nothing inside `.git` is copied) and this action commits and pushes them, ignoring git hooks and global config. Nothing the agent does to git settings in its copy reaches the commit or the push, and the push goes to your repository's URL.
- **Who can start it.** Only roles in `allowed-roles`. For issue events the issue's author needs a role too, so labeling or assigning someone else's issue does not turn its text into an instruction. Bots and the action's own comments are ignored.
- **Text from GitHub** reaches the agent on standard input, never through a shell or workflow expression. Issue and pull request descriptions are marked as information, not instructions.
- **Forks.** Pull requests from forks are answered in ask mode: no code changes, nothing pushed, and the fork's hooks and MCP servers never run, even with `trust-project: true`.
- **Project hooks and MCP servers** run only with `trust-project: true`.
- **Output.** The agent's log lines are printed with workflow commands switched off. Its answer is posted with @mentions and closing keywords ("fixes", "closes", "resolves") neutralized, in comments, commit messages and pull requests.
- **What remains.** The agent has your NeonCodex API key and can reach the network, as on your computer. Review its pull requests before merging and don't give the job secrets it doesn't need. With `isolation: none` (macOS, Windows, most self-hosted runners) it runs as the runner user and could read the job's GitHub token: keep the job's permissions to the three above.

Repositories that use Git LFS: the commit step does not run LFS filters, so large files the agent changes are committed as regular files.

## Develop

`dist/index.js` has no dependencies. Tests: `node test/action_test.cjs` in the NeonCodex clients repository runs the action against a fake GitHub API, a local git server and the CLI.
