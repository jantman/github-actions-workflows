# `claude-pr-review`

Reviews a pull request with Claude Code and posts the findings as **one** GitHub review,
with the run's duration, turn count and cost in the body. Fails the check when a tool
call was denied.

```yaml
      - name: Checkout repository
        uses: actions/checkout@v7
        with:
          fetch-depth: 1

      - uses: jantman/github-actions-workflows/claude-pr-review@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

Full caller workflow: [`examples/claude-pr-review.yml`](../examples/claude-pr-review.yml).
It is the same in every repository — copy it verbatim.

## The problem this solves

**A green check is not evidence that a review happened.**

The action denies a tool call, the agent improvises around the gap, ends its turn and
exits 0. With `display_report` off, the only trace is a `permission_denials_count` in a
streamed result object that never reaches the log.

That is how `/code-review:code-review` ran for months without reviewing anything: it is
a *Skill*, and `Skill` was missing from the allowlist. Four repositories were in that
state. Everything here that looks like belt-and-braces is there because of it:

- `Skill`, `Task` and `Write` in the default allowlist, with a lint check in this repo
  that fails if anyone removes them.
- **No enumerated `Bash(cmd:*)` allowlist.** The review plugin fans out subagents that
  reach for `sed`/`awk`/`jq`/`diff`, and every unlisted command is a *silent denial*.
- `display_report` and `show_full_output` on, plus the transcript and session logs
  uploaded as `claude-pr-review-logs-*` — the action otherwise discards both with the
  runner. Reach for that artifact first when a run is green but posted nothing.
- Duration, turns, cost and `permission_denials_count` in the job summary and the review
  footer, and **a denial or an `is_error` result fails the check.**

> [!IMPORTANT]
> **Agent mode does not merge a base tool set into `--allowedTools`.** The list in
> `allowed_tools` is the entire allowlist; anything absent from it is absent from the
> run. (Tag mode *does* merge, which is why [`claude-mention`](../claude-mention/)'s
> default is much shorter. Do not copy one default to the other.)

## The agent does not post to GitHub. This action does.

The plugin's `--comment` mode posts each finding as a standalone inline comment, and a
summary comment only when it finds nothing — upstream's inline-comment MCP server exists
precisely so the agent cannot reach the review API. That leaves findings as N ungrouped
comments with no top-level body to hang a summary or a cost footer off.

So `--comment` is deliberately **not** passed. The agent writes its findings to a JSON
file and this action turns them into a single `POST /pulls/N/reviews`. Every run posts
exactly one review — including a run that decided to *skip*, because a skip that should
not have happened is otherwise invisible.

Reviews are authored by `github-actions[bot]`, not `claude[bot]`: claude-code-action
revokes its own GitHub App token in an `if: always()` step before the posting step runs.
Hence the attribution line in every review body, and the hidden
`<!-- claude-code-review -->` marker that later runs use to find their own previous
reviews. **That marker is a contract — keep it byte-identical across repositories and
versions.**

### Failure paths

Each of these has happened at least once:

| situation | what the action does |
|---|---|
| Findings file missing, unparseable, or the wrong shape | Posts the agent's raw final message from the transcript, then **fails** |
| GitHub 422s the batched review (a finding on a line the diff doesn't touch) | Retries once with the findings folded into the body |
| Any non-422 rejection | Does **not** retry — it may have created the review server-side, and a retry would double-post |
| Review body over 60 KB | Trims, with a pointer to the log artifact |
| A finding with no usable `path`/`line`/`body` | Goes in the review body rather than taking the whole batch down with it |

## What the caller is responsible for

Two of these cannot move into a composite action: by the time one runs, the runner is up
and the token is minted.

1. **The fork guard.**
   `if: github.event.pull_request.head.repo.full_name == github.repository`
   A fork pull request gets no secrets, so `CLAUDE_CODE_OAUTH_TOKEN` is empty and the
   action fails red on somebody else's contribution. Always true on a repository that
   takes no fork PRs.
   > Do **not** switch to `pull_request_target` to "fix" fork PRs — that runs with
   > secrets against untrusted code.
2. **`permissions`.** Notably `id-token: write` for the OIDC exchange, and
   `actions: read` — without it the CI MCP server is skipped with only a warning in the
   log, and the reviewer silently loses the ability to read CI results.
3. **`concurrency`**, so a new push supersedes the review it interrupted rather than
   burning a full agent run on code that no longer exists.
4. **`actions/checkout`** — see [Conventions](../README.md#conventions).

## Testing a change

**A change to the *caller workflow* cannot be tested by the pull request that makes it.**
claude-code-action refuses to run when the workflow file differs from the copy on the
default branch — that guard is what stops a pull request from rewriting the workflow to
steal the OAuth token — and it exits 0 when it skips. Merge first; the next pull request
gets a real review. Look for `Exiting due to workflow validation skip`.

A validation skip finishes in ~10s and uploads no logs. A real run takes a minute or
more and uploads `execution-output.json`.

**A change to *this action* does not have that problem**, which is most of the reason
this repository exists: the workflow file is unchanged, so validation passes, and in
this repository `./claude-pr-review` resolves to the pull request's checked-out copy.

The flip side is the same hole the validation guard exists to close: in *this*
repository a pull request can change what the workflow does without changing the
workflow file. Fork PRs get no secrets and are skipped, so it is bounded to people with
push access — the same trust boundary as the mention job. Consumers pinned at `@v1` are
unaffected, since a pull request in their repository cannot reach this action.

## Inputs

| input | required | default | |
|---|---|---|---|
| `claude_code_oauth_token` | yes | — | Generate with `claude setup-token`. |
| `github_token` | no | `${{ github.token }}` | Posts the review; needs `pull-requests: write`. |
| `allowed_tools` | no | see [`action.yml`](action.yml) | The **complete** `--allowedTools` list. Override only to add — removing `Skill`, `Task` or `Write` breaks the review in ways that still exit 0. |
| `prompt_extra` | no | `''` | Extra review instructions. Inserted after the findings-file contract, but **before** the do-not-background block, which must stay last. |
| `retention_days` | no | `14` | Log artifact retention. World-readable on a public repository. |
| `claude_avatar` | no | github.com/claude's avatar | Used in the review heading and job summary. |

## Outputs

| output | |
|---|---|
| `ran` | `"true"` when the agent actually started (produced an execution file). |
| `cost` | Total USD to 4dp. Empty when the agent did not run. |
| `denials` | `permission_denials_count` from the result block. |
| `review_url` | `html_url` of the review that was posted, when one was. |
