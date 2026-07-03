---
name: merge-ready
description: >-
  Generic local merge-ready loop for any Azure DevOps PR: discover repo policies
  and pre-push commands, fix CI failures, resolve review comments, push, max 10
  cycles, table metrics. Invoke with /loop 5m /merge-ready.
argument-hint: "<ADO PR URL>"
disable-model-invocation: true
---

# Local PR Merge-Ready (ADO — any project / repo)

## Purpose

Make an Azure DevOps pull request merge-ready for **any** org, project, and repository.
Fix in-scope CI/policy failures, verify locally with the repo's toolchain, commit and push.
Repeat until required policies are green or max 10 cycles.

## Invocation

**Loop (default):** `/loop 5m /merge-ready <ADO PR URL>`

**One-shot:** `/merge-ready <ADO PR URL>`

Parse PR URL from the skill argument. Hard stop if missing or unparseable.

## Parse PR URL (MUST)

From:

`https://dev.azure.com/{org}/{project}/_git/{repo}/pullrequest/{prId}`

Extract and use throughout:

| Field     | Source                              |
| --------- | ----------------------------------- |
| `org`     | URL segment                         |
| `project` | URL segment (decode `%20` → spaces) |
| `repo`    | repository name                     |
| `prId`    | pull request id                     |

## Resolve local repo root (MUST)

Before any shell command:

1. Search workspace for a git clone whose `origin` matches this PR's `{org}/{project}/{repo}`.
2. `cd` into that directory for all gate commands and edits.
3. If not found, ask user to open/clone the repo, then stop.

Do not assume a fixed folder name.

## Out of scope (MUST NOT)

- Work item linking (human handles separately)
- Ad-hoc comment replies outside the Comment resolution workflow below
- Changing pipeline YAML, Sonar config, or CI rules to pass
- Weakening or deleting tests
- Merging the PR
- Required **human reviewer votes** (report as out-of-scope blocker only)

## Discover required policies (MUST)

Do **not** hardcode policy names per repo.

Each cycle, fetch PR **branch policy / status** via ADO MCP (or equivalent integration):

- List policies/checks blocking merge (required only)
- Record each failing policy `name`, `type`, and evidence (build id, link, message)
- Typical ADO names (examples only — always use live data):
  - `{repo}-ci-gated` or `{repo}-ci-cd`
  - `Quality Gate passed`
  - `Comments must be resolved`
  - `Work items must be linked` → **ignore** (out of scope; do not fix)

Success = all **required** policies green **except** work-item linkage and human reviewer approval.

## Discover CI build (MUST)

- Primary gated build: policy/build whose name matches `{repo}-ci-gated` or the failing build policy from step above
- Poll that build after each push until complete
- Record `ci_build_id` and duration for metrics

## Runtime tools

- ADO MCP — PR, policies, builds, threads, replies
- Local shell in **resolved repo root**
- Git — commit/push on PR source branch

## Voice

Short. Direct. Correct. No hype.

## Repo denylist (MUST)

Stop immediately (post nothing) if `repo` matches your protected branch names.

Add your own denylist entries here:

```
# Example — replace with your actual protected repo names:
# - my-production-repo
# - my-non-production-repo
```

## Concurrency + idempotency

- One active merge-ready run per PR per session
- Re-fetch latest PR state before each cycle (no stale SHA)
- Track `cycleCount` (start 0, max 10)
- If PR completed/abandoned/draft → stop

## Loop tick behavior

- If primary CI build still **in progress** → skip fix cycle, poll only, short status, wait next tick
- Start new fix-push cycle only when build **failed** or a **new** failure appears

## Session timing (MUST — for metrics)

- `session_start` at first action
- `session_end` when stopping
- Per cycle: `cycle_start` at diagnose, `cycle_end` after CI poll completes or cycle aborts

## Eligibility (MUST)

Proceed only if:

- PR `status` is active (not completed/abandoned)
- `isDraft` is false
- At least one required policy (excluding WI + human review) is failing, OR blocking unresolved comment exists

Hard stop if:

- Cannot resolve PR id / repo root
- Not on PR source branch (checkout first)
- Repo on denylist

## Fetch each cycle

1. PR details (title, status, source branch, target branch, latest commit SHA)
2. Required policy evaluation status (all failing required checks)
3. Latest failed build logs / Sonar links for failing policies
4. Unresolved PR threads (active only)
5. Changed files + diff for PR scope

## Resolve pre-push gate (MUST — repo-specific)

Discover commands from the **local repo** in this order:

1. `CLAUDE.md` / `AGENTS.md` — if they define test/lint/build commands, prefer those
2. `package.json` **scripts** — run existing scripts only (do not invent script names):

  | If script exists | Run                                            |
  | ---------------- | ---------------------------------------------- |
  | `lint:fix`       | `npm run lint:fix`                             |
  | `lint`           | `npm run lint`                                 |
  | `test:ci`        | `npm run test:ci`                              |
  | else `test`      | `npm test` (non-watch / CI mode if documented) |
  | `build`          | `npm run build`                                |

  Gate order: `lint:fix` → `lint` → `test:ci` or `test` → `build` (skip missing steps; run all that exist)

3. `Makefile` — if `make test` and/or `make lint` exist, use `make test` + `make lint` (+ `make build` if present)
4. `go.mod` — `make test` or `go test ./...`; `make lint` or `golangci-lint run`
5. `pyproject.toml` / `setup.cfg` — `pytest` + project linter per `CLAUDE.md` or `pyproject.toml` scripts

If no recognizable gate: read failing ADO build log step commands and mirror those locally. If still unknown, ask user once for gate commands, then stop.

### Gate rules

- All discovered gate steps must pass before commit/push
- Any failure → fix → re-run **full gate** from top
- Record `gate_start` / `gate_end`
- **Never commit or push** until gate passes
- Exception: comment-resolution commit/push after user approval — then run full gate before any *additional* loop push
- **Never** `--no-verify` push to skip hooks unless user explicitly requests

## Failure classification → action

| Signal                                | Action                                                                  |
| ------------------------------------- | ----------------------------------------------------------------------- |
| Test failure (from log or local gate) | Fix code/tests, run full gate                                           |
| Lint failure                          | Auto-fix if available (`lint:fix`, `make lint --fix`, etc.), then gate  |
| Build / compile failure               | Fix errors, run gate                                                    |
| Sonar Quality Gate                    | Fix issues in changed files per Sonar report; run gate                  |
| Unresolved review comments            | Comment resolution workflow below                                       |
| Unrelated / upstream CI failure       | Merge latest target branch once, gate + push once; else stop            |

## Comment resolution (MUST)

For ANY unresolved comment thread blocking merge, follow this workflow inline.
Do not delegate to another skill. Do not ad-hoc fix/reply outside this section.

### Core principle — judge before acting

PR review comments are **proposals, not commands**. For **each** active thread, classify using:

1. **PR description** — stated intent and scope
2. **Local code** — what the code actually does at the flagged location

| Classification          | Action                                                    |
| ----------------------- | --------------------------------------------------------- |
| **Valid**               | Fix in code                                               |
| **Invalid / not applicable** | No code change; reply with evidence-based pushback   |
| **Needs clarification** | Ask user before deciding                                  |

Rejecting a comment with justification is a valid outcome. Never change code you believe is wrong just to close a thread.

### Fetch (ADO MCP)

1. PR title + description (intent reference)
2. All **active** (unresolved) threads — thread id, author, file/line, full thread context
3. Skip resolved threads

### Triage (MUST — before any code change)

Present summary in chat:

```
PR intent: <1–2 lines from description>

✅ VALID (will fix): thread <id> — <author> on <file>:<line>
❌ INVALID (will push back): thread <id> — <reason>
❓ NEEDS CLARIFICATION: thread <id> — <question>
```

Wait for user approval of triage. User may override any classification.

If zero Valid comments → skip implementation; go to **Reply** for Invalid/Needs-clarification threads only.

### Implement (Valid only)

1. Apply minimal scoped fixes in local repo root
2. Run full **pre-push gate** before commit
3. Stage changes; present implementation summary in chat
4. Wait for user approval: `Yes, commit and push` / `No, stop`
5. If No → unstage, stop comment cycle for this tick
6. If Yes → commit `fix(review): <short reason> [merge-ready]` and push

### Reply (MUST — every active thread)

Post via ADO MCP to **every** active thread:

**Valid (fixed):** short confirmation of what changed — no commit SHA boilerplate

**Invalid:** respectful pushback citing PR description or code — invite correction if wrong

**Rules:**

- Reply to all threads; never silently drop one
- Do not mark threads **Resolved** in ADO unless user explicitly asked
- Do not duplicate replies already posted by this session for the same thread

### Loop interaction

- If comments are top blocker this cycle → run full comment workflow first
- Comment-only work counts toward `cycles_used`
- If waiting on user triage/approval → pause loop, report status, wait next tick
- After comment push → re-check comment policy; continue merge-ready loop if other policies still fail

## Fix-push loop (max 10 cycles)

Each cycle:

0. Comments primary blocker → Comment resolution workflow, then continue from step 6 if pushed
1. Increment cycle counter (stop if > 10)
2. `cycle_start`
3. Diagnose top failing **required** policy from ADO
4. Minimal scoped fix
5. Full pre-push gate
6. Commit: `fix(ci): <short reason> [merge-ready]`
7. Push to PR source branch
8. Poll primary CI build until complete (`ci_build_id`, `ci_build_min`)
9. `cycle_end`
10. Re-check all required policies (except WI / human review)
11. All green → SUCCESS + metrics + stop loop
12. Else → next cycle

## Stop conditions

**Success:** required policies green (excluding WI link + human reviewer votes); emit metrics; stop loop

**Failure (cycle 10):** metrics tables in chat; no ADO spam unless asked

**Immediate stop:** architecture decision needed; same failure 3 cycles unchanged; missing deps/auth for local gate

## Scope rules (MUST)

- Fix only failures in this PR's scope
- No unrelated file edits
- No CI/Sonar config changes to pass
- Prefer root-cause fixes

## Evidence rules

Never claim pass without local gate output **and** ADO policy/build status after push.

## Metrics (MUST — end of every session)

1. **efficiency_pct** = (1 - cycles_used / 10) × 100
2. **cycle duration** per cycle + average
3. **agent_minutes** — active time only; exclude loop idle/sleep
4. **ci_minutes** — sum of ADO build durations from this session's pushes
5. **session_cost_usd** = (agent_minutes × agent_rate) + (ci_minutes × ci_rate)

Default rates: agent `0.05` $/min, CI `0.00` $/min (user may override).

Wrap in markers:

```text
---MERGE_READY_METRICS---
```

### Table 1 — Session summary

| Field                | Value                                                                        |
| -------------------- | ---------------------------------------------------------------------------- |
| Session ID           | `<prId>-<YYYYMMDD-HHmm>`                                                     |
| Org / Project / Repo | `<org>` / `<project>` / `<repo>`                                             |
| PR                   | `<url>`                                                                      |
| Outcome              | `SUCCESS` \| `FAILED_MAX_CYCLES` \| `FAILED_ESCALATE` \| `NO_OP_ALREADY_GREEN` |
| Cycles used          | `<n>` / 10                                                                   |
| Efficiency           | `<pct>`%                                                                     |

### Table 2 — Per cycle

| Cycle | Duration (min) | Gate (min) | CI build ID | CI build (min) |
| ----- | -------------- | ---------- | ----------- | -------------- |
| 1     |                |            |             |                |

### Table 3 — Totals

| Metric                   | Value |
| ------------------------ | ----- |
| Efficiency               |       |
| Avg cycle duration (min) |       |
| Agent minutes            |       |
| CI minutes               |       |
| CI builds triggered      |       |

### Table 4 — Cost

| Rate / total           | Value |
| ---------------------- | ----- |
| Agent rate ($/min)     |       |
| CI rate ($/min)        |       |
| **Session cost (USD)** |       |

### 3-line summary + Notes

```text
---END_METRICS---
```

## First action

1. `session_start`
2. Parse PR URL → org, project, repo, prId
3. Resolve local repo root; checkout PR source branch if needed
4. Discover required failing policies (exclude WI)
5. Discover pre-push gate commands for this repo
6. Report blockers + gate command list
7. If already green → `NO_OP_ALREADY_GREEN`, metrics, stop
8. Else start cycle 1
