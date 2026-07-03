## What is loop engineering?

Most developers interact with AI coding agents the same way they interact with a search engine: one prompt, one answer, repeat. Loop engineering is a different model entirely.

Instead of you prompting the agent turn by turn, you design a **condition** — a verifiable, yes-or-no statement of what "done" looks like — and let the agent work autonomously until that condition is met. The agent doesn't stop because it thinks it's finished. It stops because something external confirmed it actually is.

Boris Cherny, who leads Claude Code at Anthropic, put it plainly: he doesn't prompt Claude anymore — he writes loops.

The idea has roots in how reliable software systems are built: a generator (the agent doing the work), an evaluator (something that checks the output against a rubric), and a loop that feeds the evaluator's result back to the generator until the rubric passes. The one rule that makes it work: **the generator must not grade its own homework.**

For a PR, the condition is concrete and checkable:

- Edge cases covered, coverage above 80%
- All tests passing
- Lint clean
- Every review comment resolved

`merge-ready` is that loop, applied to an Azure DevOps PR. You give it a PR URL. It works until the conditions are met or it tells you clearly why they aren't.

---

# merge-ready — Claude Code skill for Azure DevOps PRs

A Claude Code skill that makes an Azure DevOps pull request merge-ready autonomously. It resolves PR review comments, fixes lint and test failures, commits, pushes, and polls CI — on a loop — until all required policies are green or it hits 10 cycles.

Built and tested on a real ADO monorepo at work. First clean run: 2 review comments resolved, lint + tests fixed and pushed, CI passed. ~17 minutes end-to-end, $1.40 session cost.

---

## What it does

Each cycle, the skill:

1. Fetches the live policy/check status from ADO (no hardcoded policy names — works across repos)
2. Classifies any unresolved PR review comments as valid, invalid, or needs-clarification — and asks you before touching code
3. Runs your repo's own lint/test/build gate locally before every push
4. Commits and pushes to the PR source branch
5. Polls CI until the build completes
6. Repeats until all required policies are green, or stops at 10 cycles with a full metrics report

It does **not** merge the PR, link work items, change CI config to pass, or require human reviewer votes.

---

## Requirements

- **Claude Code** (Claude Desktop with the Code tab, or claude.ai with Claude Code enabled)
- **ADO MCP connected** — the skill talks to Azure DevOps via MCP for PR state, policies, builds, and thread replies
- **Repo open in your workspace** — the skill finds the local clone by matching `origin` to the PR URL; no fixed folder name assumed

---

## Setup

### 1. Install the skill

Copy `SKILL.md` into your Claude Code skills directory:

```
~/.claude/skills/merge-ready/SKILL.md
```

The directory name (`merge-ready`) becomes the skill name.

### 2. Connect ADO MCP

In Claude Code, connect the Azure DevOps MCP server so the skill can read and write PR state. If your org uses a self-hosted ADO MCP, point Claude Code at that server URL.

### 3. Add your repo denylist

Open `~/.claude/skills/merge-ready/SKILL.md` and update the denylist section with any repo names the skill should never touch:

```markdown
## Repo denylist (MUST)
- my-production-repo
- my-release-repo
```

The skill stops immediately if the PR's repo matches any entry on this list.

---

## Usage

```
# Run on a loop (checks every 5 minutes)
/loop 5m /merge-ready https://dev.azure.com/{org}/{project}/_git/{repo}/pullrequest/{id}

# Run once
/merge-ready https://dev.azure.com/{org}/{project}/_git/{repo}/pullrequest/{id}
```

The loop mode is the main pattern: start it, step away, come back to a green PR or a clear report of what it couldn't fix.

---

## How it handles PR review comments

This part is worth understanding before you trust it with a real PR.

The skill reads every unresolved thread and classifies each one **before touching any code**:

- **Valid** — reviewer is right; skill fixes the code
- **Invalid / not applicable** — skill pushes back in the thread with evidence, no code change
- **Needs clarification** — skill asks you before deciding

It presents the full triage in chat and waits for your approval before implementing anything. You can override any classification. A comment being rejected with a reasoned reply is a valid outcome — the skill won't change code it believes is correct just to close a thread.

---

## How it discovers your gate commands

The skill doesn't hardcode lint/test/build commands. It reads your repo in this order:

1. `CLAUDE.md` or `AGENTS.md` — preferred if present
2. `package.json` scripts — runs `lint:fix`, `lint`, `test:ci`/`test`, `build` if they exist
3. `Makefile` — `make lint`, `make test`, `make build`
4. `go.mod` — `go test ./...`, `golangci-lint run`
5. `pyproject.toml` / `setup.cfg` — `pytest` + configured linter

It never commits or pushes until every discovered gate step passes.

---

## Metrics

Every session ends with a structured metrics report:

| What               | Example         |
| ------------------ | --------------- |
| Outcome            | SUCCESS         |
| Cycles used        | 1 / 10          |
| Efficiency         | 90%             |
| Agent minutes      | 7 min           |
| CI minutes         | 10 min          |
| Session cost (USD) | $0.35           |

Cycle-level detail (gate time, CI build ID, CI build duration) is included per run so you can see exactly where time went.

---

## What it will not do

- Merge the PR
- Link work items
- Change CI pipeline YAML or Sonar config to make checks pass
- Weaken or delete tests
- Push with `--no-verify` unless you explicitly ask
- Reply to or resolve threads it didn't open in this session

---

## Adapting for other CI systems

The skill is ADO-specific for PR/policy/thread operations but the local gate logic is toolchain-agnostic. If you're on GitHub Actions or another CI provider, the gate discovery section transfers directly — the main thing you'd need to swap is the MCP integration (ADO MCP → GitHub MCP or equivalent) and the policy-fetch logic.
