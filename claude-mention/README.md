# `claude-mention`

Runs Claude Code in **tag mode** when someone writes `@claude` on an issue, pull request
or review. Collects the transcript, and fails the check when a tool call was denied.

```yaml
      - name: Checkout repository
        uses: actions/checkout@v7
        with:
          fetch-depth: 1

      - uses: jantman/github-actions-workflows/claude-mention@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

Full caller workflows — **pick the one that matches the repository's visibility**:

- [`examples/claude-mention-public.yml`](../examples/claude-mention-public.yml)
- [`examples/claude-mention-private.yml`](../examples/claude-mention-private.yml)

## ⚠️ The `author_association` guard is load-bearing on a public repository

This job runs with `contents: write` and blanket `Bash`. Without a guard, **any passer-by
who types `@claude` in a comment gets an arbitrary-code-execution agent run billed to
your account.**

The guard cannot live in this action. By the time a composite action runs, the runner is
already up and the OAuth token already minted — so it has to be a job-level `if:` in the
caller:

```yaml
    if: |
      (github.event_name == 'issue_comment' &&
       contains(github.event.comment.body, '@claude') &&
       contains(fromJSON('["OWNER","MEMBER","COLLABORATOR"]'), github.event.comment.author_association)) ||
      ...
```

> [!WARNING]
> **Read `author_association` per-event.** On `issue_comment`,
> `github.event.issue.author_association` describes the issue's *opener*, not the
> commenter. Using that field here lets a stranger trigger a run merely by commenting on
> one of your own issues. The public example gets each event's pairing right; copy it
> rather than reconstructing it.

`OWNER` covers the repository owner; `COLLABORATOR`/`MEMBER` cover anyone later given
push access.

## Why this is not merged with `claude-pr-review`

They look redundant. They are not, and the difference is one input.

| | `claude-mention` | `claude-pr-review` |
|---|---|---|
| `prompt` supplied? | **No** | Yes |
| claude-code-action mode | **tag** | agent |
| Triggered by | `@claude` in a comment/issue/review | every `pull_request` |
| `contents:` | `write` — it can push fixes | `read` |

Supplying a `prompt` puts claude-code-action in agent mode **for every event it sees**.
A single job carrying both sets of triggers would answer `@claude` comments in agent
mode: no PR context, no tracking comment, and nothing posted back. Supplying no prompt
is the entire mechanism that keeps this in tag mode, where Claude performs the
instructions in the comment that tagged it.

Deleting one as a duplicate of the other has silently broken `@claude` before.

## Tag mode merges the tool list; agent mode does not

`allowed_tools` here is only what to **add** to the base set claude-code-action already
supplies in tag mode: `Read`/`Glob`/`Grep`/`LS`, the comment-update tool, the three
`mcp__github_ci__*` tools, and `git add`/`commit`/`push`/`rm`. That is the opposite of
[`claude-pr-review`](../claude-pr-review/), where the list you pass is the whole
allowlist. **Do not copy one default to the other.**

Two consequences worth knowing:

- **`WebSearch` and `WebFetch` are disallowed by default in tag mode.** Naming them in
  `allowed_tools` is what lifts that. They are in the default here — `@claude why does
  this library do X` is most of what this job gets asked — and the trigger guard above
  is what makes that an acceptable trade.
- **Naming an `mcp__github_*` tool is what installs that MCP server.** The
  inline-comment tool is in the default for that reason, even though this job rarely
  reaches for it.
- `additional_permissions: actions: read` is set by this action, which is what makes the
  `mcp__github_ci__*` tools available. The caller still needs `actions: read` in its
  `permissions` block.

## Do not go back to an enumerated command allowlist

Earlier versions of these workflows listed individual Bash commands — `puppet-lint`,
`nox`, `poetry`, `./bin/run_tests.sh`. Anything unlisted is a **silent denial**, not an
error: the run goes green having done nothing. `Skill` and `Task` matter for the same
reason — a skill or a subagent that cannot be invoked is not an error, just an absence.

One repository had 64 committed skills under `.claude/skills/` and no `Skill` in its
allowlist, so every `@claude run /some-skill` was a no-op that exited 0.

## Editing the caller has no effect until it is merged

`issue_comment` workflows **always run from the default branch.** Changes to the caller
workflow on a branch do nothing at all until they land on the default branch — this is a
stronger constraint than the validation skip that affects
[`claude-pr-review`](../claude-pr-review/#testing-a-change), and it applies to every
trigger this workflow uses.

The 👀 reaction on an `@claude` comment comes from the GitHub App acknowledging the
mention. It is **not** evidence that anything ran — the work happens in this job, and if
the workflow is missing or not yet on the default branch, the eyes are all you get.

## Inputs

| input | required | default | |
|---|---|---|---|
| `claude_code_oauth_token` | yes | — | Generate with `claude setup-token`. |
| `allowed_tools` | no | see [`action.yml`](action.yml) | Tools to **add** to tag mode's base set. |
| `retention_days` | no | `14` | Log artifact retention. World-readable on a public repository. |

## Outputs

| output | |
|---|---|
| `ran` | `"true"` when the agent actually started (produced an execution file). |
