# github-actions-workflows

Shared GitHub Actions for my repositories. Currently: the two Claude Code actions that
run across ten repos.

| | |
|---|---|
| [`claude-pr-review`](claude-pr-review/action.yml) | Reviews every PR unprompted. Posts **one** GitHub review with the run's cost in the body. |
| [`claude-mention`](claude-mention/action.yml) | Runs Claude in tag mode when someone writes `@claude`. |

Copy the matching file from [`examples/`](examples/) into a consumer repo's
`.github/workflows/`. The caller stays thin — triggers, `concurrency`, `permissions`,
the guards, the secret — and everything else lives here.

```yaml
      - name: Checkout repository
        uses: actions/checkout@v7
        with:
          fetch-depth: 1

      - uses: jantman/github-actions-workflows/claude-pr-review@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

Consumers need a `CLAUDE_CODE_OAUTH_TOKEN` secret (`claude setup-token`) and the Claude
GitHub App installed on the repo.

## The problem these solve

**A green check is not evidence that a review happened.** The action denies a tool call,
the agent improvises around the gap, ends its turn and exits 0. With `display_report`
off, the only trace is a `permission_denials_count` in a streamed result object that
never reaches the log.

That is how `/code-review:code-review` ran for months without reviewing anything: it is
a *Skill*, and `Skill` was missing from the allowlist. Four repositories were in that
state. Everything here that looks like belt-and-braces is there because of it:

- `Skill`, `Task` and `Write` in the review allowlist, and a lint check that fails if
  anyone removes them. **Agent mode does not merge a base tool set into `--allowedTools`
  — the list you pass is the whole list.** (Tag mode *does* merge, which is why
  `claude-mention`'s default is much shorter. See `docs`-free note in each action.)
- No enumerated `Bash(cmd:*)` allowlist. The review plugin fans out subagents that reach
  for `sed`/`awk`/`jq`/`diff`, and every unlisted command is a *silent denial*.
- `display_report` and `show_full_output` on, plus the transcript and session logs
  uploaded as an artifact — the action otherwise discards both with the runner.
- Duration, turns, cost and `permission_denials_count` in the job summary and the review
  footer, and **a denial or an `is_error` result fails the check.**

## The agent does not post to GitHub. The action does.

The plugin's `--comment` mode posts each finding as a standalone inline comment and a
summary only when it finds nothing — upstream's inline-comment MCP server exists
precisely so the agent cannot reach the review API. That leaves findings as N ungrouped
comments with no top-level body to hang a cost footer off.

So `--comment` is deliberately not passed. The agent writes findings to a JSON file and
the action turns them into one `POST /pulls/N/reviews`. Every run posts exactly one
review — including a run that decided to *skip*, because a skip that should not have
happened is otherwise invisible.

Reviews are authored by `github-actions[bot]`, not `claude[bot]`: claude-code-action
revokes its own App token in an `if: always()` step before the posting step runs. Hence
the attribution line in every review body, and the hidden `<!-- claude-code-review -->`
marker that later runs use to find their own previous reviews.

Failure paths, because each of these has happened: a findings file that is missing or
the wrong shape posts the agent's raw final message and then fails; a batched review
GitHub rejects with a 422 is retried once with the findings folded into the body;
a body over 60 KB is trimmed with a pointer to the artifact. Anything other than a 422
is *not* retried — it may have created the review server-side, and a retry would
double-post.

## What must stay in the caller

Three things cannot move into a composite action, because by the time one runs the
runner is already up and the token already minted:

1. **The fork guard.** `if: github.event.pull_request.head.repo.full_name == github.repository`.
   A fork PR gets no secrets, so without it the action fails red on someone else's
   contribution. Do **not** switch to `pull_request_target` to "fix" fork PRs — that
   runs with secrets against untrusted code.
2. **The `author_association` guard, on public repos.** `claude-mention` runs with
   `contents: write` and blanket `Bash`. Without the guard, any passer-by who types
   `@claude` gets an agent run billed to your account. Read the field **per-event**: on
   `issue_comment`, `github.event.issue.author_association` is the issue's *opener*, not
   the commenter, so using it there lets a stranger trigger a run by commenting on your
   own issue.
3. **`permissions`.** Notably `id-token: write` (the OIDC exchange) and `actions: read`
   (without it the CI MCP server is skipped with only a warning).

The caller also does its own `actions/checkout` — see the note in
`claude-pr-review/action.yml` for why.

## Testing a change

**A change to a *caller workflow* cannot be tested by the PR that makes it.**
claude-code-action refuses to run when the workflow file differs from the copy on the
default branch — that guard is what stops a PR from rewriting the workflow to steal the
OAuth token — and it exits 0 when it skips. Merge first; the next PR gets a real review.
Look for `Exiting due to workflow validation skip`. A validation skip finishes in ~10s
and uploads no logs; a real run takes a minute or more and uploads
`execution-output.json`.

**A change to an *action here* does not have that problem**, which is most of the reason
for this repo. This repository runs its own actions by local path (`./claude-pr-review`)
rather than `@v1`, so the workflow file is unchanged, validation passes, and `./`
resolves to the PR's checked-out copy. Changes are testable on their own PR.

The flip side is the same hole the validation guard exists to close: in *this*
repository a PR can change what the workflow does without changing the workflow file.
Fork PRs get no secrets and are skipped, so it is bounded to people with push access —
the same trust boundary as the mention job. Consumers pinned at `@v1` are unaffected,
since a PR in their repo cannot reach the action.

## Versioning

`@v1` is a moving tag. Moving it updates every consumer at once, which is the point —
and also means a bad change reaches ten repos immediately. Pin a commit SHA instead if
you would rather take updates deliberately.

```bash
git tag -f v1 && git push -f origin v1
```

## Repos using these

privatepuppet · biweeklybudget · workshop-inventory-tracking · robot-army ·
RPiRFIDWiegandReader · dm-puppet · dm-network-docs · machine-access-control ·
kiosk-show-replacement · equipment-status-board
