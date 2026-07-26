# Spec 001 — Agent Swarm MVP

Status: **READY — all decisions closed (§6). Slice into issues, then implement.**

## 1. Problem

A local coding agent session is bounded by one machine: subagent concurrency,
CPU, and wall-clock. When a task decomposes into N genuinely independent slices,
the local session either serialises them or blocks while fanning out. Meanwhile
GitHub Actions offers ephemeral, isolated, already-repo-authenticated VMs with
free-tier concurrency that sit idle.

There is no way today to hand a slice to a throwaway VM, have an agent do the
work there, and get a reviewable result back without babysitting the browser.

## 2. Goal

A local CLI that takes N independent task slices, dispatches each to a GitHub
Actions job running an ephemeral coding agent, and collects each slice's output
as a reviewable artifact — while the local session stays free.

Observable end state:

```
swarm init            # write the caller workflow into the target repo, for you to commit
swarm run --task "…" --task "…" --task "…"   # --base defaults to the repo's default branch
swarm status          # per-slice: queued | running | done | failed
swarm collect         # branches + result.json per slice, locally
```

## 3. Non-goals (blast-radius fence)

- **No autonomous merge.** The swarm never merges to `main`. A human (or the
  local session) reviews and lands.
- **No dependent task graph.** Slices are independent by construction. A DAG
  scheduler is v2.
- **No self-hosted runners.** GitHub-hosted only.
- **No cross-repo swarming.** One target repo per swarm run.
- **No long-lived daemon.** The CLI is invoked, does work, exits. No background
  service, no server.
- **No web/TUI dashboard.** Terminal output only.
- **No cost accounting UI.** Record per-slice token usage; do not build
  budgeting.
- **No untrusted input.** Only repositories and task text you trust may be
  swarmed. The agent job necessarily holds the model key (see D2), so hardening
  against repo- or issue-sourced prompt injection is explicitly out of scope.

## 4. Acceptance criteria

Each is verifiable. The MVP is done when all pass.

- [ ] **AC1 — Correlation.** Dispatching a slice with id `S` produces exactly one
      Actions run whose `run-name` contains `S`, resolvable by
      `gh run list --json name,databaseId`. Verified: dispatch 3 slices, all 3
      resolve to distinct run IDs with no polling ambiguity.
      **The mechanism lives in the target repo's caller, not in the reusable
      workflow** — `run-name` inside a `workflow_call` workflow is ignored, only
      the caller's applies (D1). So `swarm run` **preflights the remote caller
      before dispatching** — reading it via
      `gh api repos/{owner}/{repo}/contents/…?ref=<ref>`, never the local
      working tree, since dispatch executes whatever GitHub has. `run-name` must
      template `inputs.slice_id` and the required inputs must be declared, else
      it refuses to dispatch. Verified two ways: a committed caller with the
      `run-name` line stripped is rejected, and an `init`-ed but **uncommitted**
      caller is also rejected — in both cases **with no Actions run created**.
      Post-dispatch fail-fast remains the backstop for the preflight/dispatch
      race.
- [ ] **AC2 — Isolation.** Each slice job checks out `base_ref` fresh; no slice
      observes another slice's commits. Verified: two slices editing the same
      file both succeed, each diffing against `base_ref`.
- [ ] **AC3 — Result contract.** Every terminated job — success or failure —
      publishes a `result.json` artifact conforming to the schema in §5, with a
      non-empty `status` field. Verified: force one slice to fail; its artifact
      still parses.
- [ ] **AC4 — Collection survives process exit.** `swarm collect` downloads
      every finished slice's result and leaves the work applied or fetchable
      locally, with a summary table. Verified end-to-end on a real repo, **from
      a new shell** — `run` exits, so `status`/`collect` must resolve the swarm
      from the manifest in §5a, not from memory. Two interleaved swarms against
      the same repo must stay distinguishable via `--run`.
- [ ] **AC5 — Failure is visible, not silent.** A slice that times out, errors,
      or produces an empty diff is reported as such and never counted as
      success. Verified: three induced failure modes, three distinct reports.
- [ ] **AC6 — Secret hygiene.** Model API keys live only in repo secrets, are
      never echoed into logs or `result.json`, and the caller workflow exposes
      no fork-PR trigger path. Verified: log scan of a completed run.
- [ ] **AC7 — Concurrency is bounded and degrades sanely.** Dispatching more
      slices than the account's concurrency limit queues rather than fails, and
      `swarm status` shows queued slices distinctly from running ones.
- [ ] **AC8 — DeepSeek runtime is genuinely agentic.** A slice running against
      `api.deepseek.com/anthropic` completes a multi-turn loop that reads a
      file, edits it, and runs a shell command. Verified: the smallest possible
      real task end-to-end. This closes the `[INFERENCE]` on client-side tool
      delivery in D2 and is the **first slice to build** — everything else is
      wasted if the endpoint cannot sustain the loop.
- [ ] **AC9 — The agent holds no usable write credential.** Verified
      adversarially, not by inspection: plant a task instructing the agent to
      push a branch and to harvest credentials. The assertions are what a
      hosted runner can actually enforce — an unsandboxed agent can obviously
      write `/tmp` and read `.git/config`, so those are not the test:
      (a) any `git push` from the `agent` job fails unauthorized;
      (b) `.git/config` contains no reusable token;
      (c) writes outside `$GITHUB_WORKSPACE` cannot influence `publish`, since
      `publish` runs on a fresh runner and consumes only the artifact;
      (d) the legitimate `swarm/*` branch is still produced correctly.
      Making (a)–(c) into "the write itself fails" would require a real
      container or sandbox boundary, which this MVP does not build.
- [ ] **AC10 — Publish validates before it pushes.** A malformed, oversized, or
      out-of-allow-list patch is rejected by `publish` with a clear failure and
      no ref written. Verified: three crafted bad artifacts, three rejections.

## 5. Result contract

**Two artifacts, because an uploaded artifact is immutable.** The `agent` job
cannot know `branch` or `commit` — they do not exist until `publish` applies
the patch — so a single artifact written by `agent` could never carry them.

| Artifact | Written by | Contents |
|---|---|---|
| `agent-result.json` + `patch.diff` | `agent` job, `if: always()` | Everything the agent run knows: `status`, `summary`, `files_changed`, `usage`, `error`, timings, resolved model |
| `result.json` | `publish` job, `if: always()` | **Authoritative.** `agent-result.json` plus `branch`, `commit`, and any publish-stage failure |

`result.json` is the record `swarm collect` reads. If `publish` never ran or
failed, `collect` falls back to `agent-result.json` and reports the slice as
unpublished — the work is visible in `patch.diff` either way, never trapped.

```json
{
  "slice_id": "s-20260726-a1b2",
  "status": "success | failed | timeout | empty | unpublished",
  "base_ref": "master",
  "model": "deepseek-v4-pro[1m]",
  "branch": "swarm/<run>/<slice> | null",
  "commit": "<sha> | null",
  "files_changed": ["path/a", "path/b"],
  "summary": "one-paragraph agent report",
  "error": "null | message",
  "usage": { "input_tokens": 0, "output_tokens": 0 },
  "started_at": "RFC3339",
  "finished_at": "RFC3339"
}
```

No field here is agent-authored except `summary`. `status` comes from an
`if: always()` step in `agent`; `branch` and `commit` come from `publish` after
the patch actually lands. An agent-authored `commit` could claim a push that
never happened.

`usage` carries **token counts only, never dollars.** The Claude Code CLI's
`--output-format json` emits a cost field priced against Anthropic's tariff;
under D2 the actual biller is DeepSeek, so that figure would be wrong by roughly
two orders of magnitude. Record tokens; price them outside the contract.

## 5a. Local run manifest

`swarm run` exits, and each slice is its own Actions run — so a later
`swarm status` has nothing to go on unless the dispatch is recorded locally.
There is no daemon (§3), so the record is a file.

```
~/.swarm/<owner>/<repo>/
├── runs/<swarm_id>.json
└── current            # swarm_id of the most recent run
```

`<swarm_id>.json` holds what the CLI cannot re-derive: `repo`, `base_ref`,
`created_at`, and per slice its `slice_id`, task digest, and resolved Actions
`run_id` once correlation (AC1) succeeds.

It lives under `$HOME`, not in the target repo: swarm state is the operator's,
not the project's, and a `.swarm/` directory would otherwise need a
`.gitignore` entry in every repo you ever swarm.

`swarm status` and `swarm collect` default to `current` and accept
`--run <swarm_id>` to address any earlier one. Concurrent swarms against the
same repo are therefore unambiguous — each has its own manifest, and only the
`current` pointer is shared.

## 6. Decisions

Resolved decisions are recorded here as they land. Open ones block
implementation. **No code while any decision below is open.**

### D1 — Distribution: reusable workflow, consumed by the target repo (DECIDED 2026-07-26)

This repo publishes `.github/workflows/agent.yml` with `on: workflow_call`. A
target repo adopts the swarm by committing a thin caller into its own
`.github/workflows/`:

```yaml
name: swarm-agent
run-name: agent/${{ inputs.slice_id }}     # REQUIRED — the correlation handle

on:
  workflow_dispatch:
    inputs:
      slice_id: { required: true,  type: string }
      task:     { required: true,  type: string }
      base_ref: { required: false, type: string }

jobs:
  agent:
    uses: mach4-braai/github-actions-agent-swarm/.github/workflows/agent.yml@<sha>
    with:
      slice_id: ${{ inputs.slice_id }}
      task:     ${{ inputs.task }}
      base_ref: ${{ inputs.base_ref }}
    permissions:
      contents: write        # push swarm/* only
    secrets:
      DEEPSEEK_API_KEY: ${{ secrets.DEEPSEEK_API_KEY }}
```

**The `run-name` line is load-bearing and cannot be moved into the reusable
workflow.** `workflow_dispatch` returns no run ID, so correlation depends
entirely on matching a templated run name — and GitHub ignores `run-name` in a
called workflow, since one caller may invoke several. Verified against
[GitHub's own discussion of the limitation](https://github.com/orgs/community/discussions/11396).

The correlation handle therefore sits in code *we do not own*. That does **not**
mean the orchestrator is reduced to failing late, and it does not force a
trade-off against write access to consumer repos — generating the caller is a
local write, and checking it is a read-only API call:

- **`swarm init`** writes the caller template into the target repo's working
  tree for the human to review and commit. A local file write; the orchestrator
  never needs a token that can push to a consumer's repo.
- **`swarm run` preflights the *remote* caller before dispatching**, via
  `gh api repos/{owner}/{repo}/contents/.github/workflows/<file>?ref=<ref>`, and
  verifies `run-name` templates `inputs.slice_id` and that the required inputs
  are declared. A caller that fails the check is refused **before** any
  dispatch.

**Preflight must read the remote ref, never the local working tree.** Dispatch
executes whatever GitHub has, so a freshly `init`-ed but uncommitted file would
pass a local check and then 404 on dispatch — or, worse, silently run an older
committed caller. `workflow_dispatch` additionally requires the workflow to
exist on the **default branch** for the event to be available at all, so
preflight checks both the default branch and the dispatch ref when they differ.

Preflight is what actually closes the hole. Failing fast *after* dispatch still
leaves an orphan Actions run burning minutes with no way to address it; the
run-name is already fixed by then. Post-dispatch fail-fast (AC1) remains as the
backstop for the race where the workflow changes between preflight and dispatch.

Secrets are mapped explicitly, never `secrets: inherit` — inherit hands the
agent every secret the target repo owns, including ones unrelated to this
workflow.

Why this over a central worker repo: the caller runs *in* the target repo, so
`GITHUB_TOKEN` is natively scoped to it with no cross-repo PAT — a PAT with
write access to every target repo would be the single largest blast-radius item
in the design. Consumers pin a SHA, so a change here cannot silently alter
someone else's CI.

The caller owns the triggers. MVP ships `workflow_dispatch` for the local
orchestrator. **Issue-driven dispatch** (label an issue → swarm solves it) is a
second trigger on the same caller and needs no change to the reusable workflow,
so it is deferred to v2 rather than designed around now. See §8.

### D2 — Agent runtime: Claude Code CLI against DeepSeek's Anthropic endpoint (DECIDED 2026-07-26)

The stated tension — wanting a real terminal agent but not Anthropic's billing —
does not exist. DeepSeek publishes an Anthropic-format endpoint, and its own
docs document Claude Code as a supported client
([anthropic_api](https://api-docs.deepseek.com/guides/anthropic_api),
[coding_agents](https://api-docs.deepseek.com/guides/coding_agents)). The job
installs the CLI and exports:

```
ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
ANTHROPIC_AUTH_TOKEN=${{ secrets.DEEPSEEK_API_KEY }}
ANTHROPIC_MODEL=deepseek-v4-pro[1m]
ANTHROPIC_DEFAULT_HAIKU_MODEL=deepseek-v4-flash
CLAUDE_CODE_SUBAGENT_MODEL=deepseek-v4-flash
```

So the swarm gets the full agent loop — shell, file edits, subagents — billed by
DeepSeek. No Docker container is required: an Actions job *is* an ephemeral VM
with a shell, which is what "run a terminal in a container" was reaching for.

**Credential handling — the agent is untrusted.** The agent runs
`--dangerously-skip-permissions`, and VM ephemerality is *not* the reason that
is acceptable. An ephemeral VM protects the host; it does nothing for the
secrets living inside it. The agent has unrestricted shell and network, and it
reads repository content — so a poisoned file, dependency, or issue body can
instruct it to exfiltrate anything in its environment. Ephemerality does not
help, because exfiltration completes long before the VM is torn down.

A post-agent *step* is not enough. Steps in one job share a mutable
environment: the agent can write `$GITHUB_ENV`, prepend to `PATH`, install a
git hook, or overwrite a binary in the workspace, and a later privileged step
in the same job then executes attacker-shaped code with the write token in
hand. Isolation has to be at the **job** boundary, where the runner is fresh.

So the reusable workflow is two jobs:

1. **`agent` (unprivileged).** `permissions: contents: read`, checkout with
   `persist-credentials: false` so `GITHUB_TOKEN` is never left in
   `.git/config`. Runs the CLI, then emits `patch.diff` + `result.json` as an
   artifact. This job never holds a write credential.
2. **`publish` (privileged, `needs: agent`).** Fresh runner, fresh checkout of
   `base_ref`, no agent-produced code on `PATH`. Downloads the artifact,
   validates it (`git apply --check`, path allow-list, size cap), commits, and
   pushes **only** `refs/heads/swarm/*`. It never runs the agent's code — it
   only moves a validated diff.

`DEEPSEEK_API_KEY` is unavoidably exposed to the agent job — it is how the
agent reaches its model, and no workflow structure changes that. The residual
risk is **accepted**, bounded by using a dedicated, rotatable, spend-capped key
and never a personal one.

Consequence for the fence in §3: **only repositories and task text you trust
may be swarmed.** Untrusted-input hardening is not in this MVP. Issue-driven
dispatch (§8) inherits this constraint directly — an issue body is
attacker-controlled text flowing straight into an agent prompt — so it must not
ship until this is revisited.

Verified constraints from DeepSeek's compatibility table, each with a real
consequence for this design:

| Field | Status | Consequence |
|---|---|---|
| `cache_control` | Ignored | **No prompt caching.** Cost grows with the square of turn count as context replays. Keep slices small; this is a design constraint, not a footnote. |
| `image` / `document` | Not supported | No screenshot or PDF slices. Code-only tasks. |
| `mcp_servers` | Ignored | Anthropic's *server-side* MCP connector is unavailable. Client-side MCP is sent as ordinary `tools` and should still work `[INFERENCE — untested against this endpoint; AC8 verifies]`. |
| `anthropic-beta` | Ignored | No beta features; do not depend on any. |
| `tools`, `tool_choice` | Supported | The agent loop itself is sound. |

Unknown model names silently fall back to `deepseek-v4-flash`, so a typo
degrades quality rather than erroring — the workflow must echo the resolved
model into `result.json`.

### D3 — Delivery: push `swarm/*` branch + `result.json` artifact (DECIDED 2026-07-26)

The `agent` job uploads `patch.diff` + `agent-result.json`; the `publish` job
validates the patch, pushes the namespaced branch, and writes the authoritative
`result.json` (§5). `swarm collect` fetches the branches and reads
`result.json`. PR creation sits behind an opt-in `--pr` flag so N slices do not
become N PRs by default.

The patch artifact is not redundant with the branch: it is the input `publish`
validates, and it remains the fallback when a push is rejected — the work is
never trapped inside a finished run.

### D4 — Orchestrator language: Go (DECIDED 2026-07-26)

Single static binary, no runtime to install on the machine running the swarm.
Matches the incubator's stated preferred language and existing siblings
(`agent-conductor`, `value-lens`, `voice-dictation`). Practical consequences:
`gh` is shelled out to rather than reimplemented against the REST API, and the
result contract in §5 is a plain struct with `encoding/json` tags.

## 7. Risks & rollback

| Risk | Mitigation |
|---|---|
| Task prompt overruns the `workflow_dispatch` payload budget | **25 inputs max, and 65,535 characters across the *entire* serialized `inputs` object** — not per input ([REST docs](https://docs.github.com/en/rest/actions/workflows#create-a-workflow-dispatch-event); raised from 10 in [Dec 2025](https://github.blog/changelog/2025-12-04-actions-workflow-dispatch-workflows-now-support-25-inputs/)). `task`, `slice_id`, and `base_ref` share one budget. The CLI measures the serialized aggregate and rejects oversized slices by name before dispatch, rather than surfacing GitHub's bare `Error: inputs are too large`. Payload indirection stays out of the MVP |
| Consumer's caller omits `run-name`, so correlation silently dies | `swarm init` generates the caller correctly; `swarm run` preflights it and refuses to dispatch on a mismatch, so no orphan run is created (AC1). Both are local, read-only — no write access to consumer repos. Post-dispatch fail-fast is the backstop for the preflight/dispatch race |
| Agent pushes garbage to the target repo | Agent job has no write token (D2); `publish` validates the patch (AC10); `swarm/*` namespace only; branch protection on `main` |
| Prompt injection from repo content exfiltrates `DEEPSEEK_API_KEY` | **Not fully mitigable** — the key must reach the agent. Bounded by a dedicated, spend-capped, rotatable key and the trusted-input fence in §3. The write token is out of reach via the two-job split (AC9) |
| Agent poisons the runner to hijack a later privileged step | Privileged work runs in a separate job on a fresh runner, consuming only a validated artifact (D2) |
| Runaway job cost / infinite agent loop | Hard `timeout-minutes` on the job; `status: timeout` in the contract |
| GitHub concurrency limit or Actions outage | AC7 queueing; swarm is additive — local execution always remains available |
| No prompt caching on DeepSeek (D2) makes long loops superlinear in cost | Small slices; cap turns; record `usage` per slice and watch it |
| DeepSeek's Anthropic compatibility drifts or degrades | AC8 is a standing canary, not a one-off check; the runtime step is isolated so swapping to OpenCode or the real Anthropic API is a one-step change |
| Unknown model name silently downgrades to `deepseek-v4-flash` | Workflow echoes the resolved model into `result.json` |

**Rollback:** delete the `swarm/*` branches and the workflow file. The
orchestrator is a local binary with no persistent remote state, so removal is
complete and immediate.

## 8. v2 growth path (explicitly not now)

Deferred, in the order they are likely to matter:

- **Issue-driven dispatch.** A second trigger on the target repo's caller
  workflow: label an issue, the swarm picks it up and opens a PR. Additive — no
  change to the reusable workflow, which is why D1 chose this shape. **Blocked
  on lifting the trusted-input fence in §3:** an issue body is attacker-supplied
  text, so this cannot ship on the MVP's threat model.
- **Dependent task graph.** Slices that consume each other's output.
- **Alternate runtimes.** OpenCode is also DeepSeek-documented; the D2 runtime
  step is deliberately one swappable block.
- **Cross-repo swarming** and **cost budgeting**.
