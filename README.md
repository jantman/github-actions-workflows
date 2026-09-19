# github-actions-workflows

Shared GitHub Actions for my repositories.

## What's here

| action | what it does |
|---|---|
| [`claude-pr-review`](claude-pr-review/) | Reviews every pull request unprompted with Claude Code, and posts the findings as **one** GitHub review with the run's cost in the body. |
| [`claude-mention`](claude-mention/) | Runs Claude Code in tag mode when someone writes `@claude` on an issue, PR or review. |

Each directory has its own README with inputs, outputs, and what the calling
workflow is responsible for. Ready-made caller workflows are in
[`examples/`](examples/) — copy the matching one into a consumer repo's
`.github/workflows/`.

## Versioning

Consumers pin `@v1`, a moving tag:

```yaml
- uses: jantman/github-actions-workflows/claude-pr-review@v1
```

Moving the tag updates every consumer at once, which is the point — and also means a
bad change reaches every one of them immediately. Pin a commit SHA instead if you would
rather take updates deliberately.

```bash
git tag -f v1 && git push -f origin v1
```

## Conventions

**The caller checks out.** No action here runs `actions/checkout` for you. `uses: ./...`
only resolves once the workspace exists, so an action that checked itself out could
never be referenced by a local path — and this repository would not be able to run its
own actions against a pull request's copy of them. It also leaves `fetch-depth`,
submodules and LFS to the caller, which knows what the repository needs.

**This repository runs its own actions by local path** (`./claude-pr-review`), not
`@v1`. That is what makes a change to an action testable by the pull request that makes
it, rather than only after merge — see each action's README for why that distinction
matters and what it costs.

**Lint before pushing.** `actionlint.yml` checks the workflows and, with shellcheck
preinstalled on the runner, every `run:` block — including those inside the composite
actions, which actionlint reads from `action.yml`. An invalid workflow file does not
fail loudly on GitHub: it registers no triggers, so you get a red zero-second run
against the *push*, with no jobs and no log, and it never appears in `gh pr checks`.

## Repositories using these

privatepuppet · biweeklybudget · workshop-inventory-tracking · robot-army ·
RPiRFIDWiegandReader · dm-puppet · dm-network-docs · machine-access-control ·
kiosk-show-replacement · equipment-status-board
