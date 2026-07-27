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

A local CLI that takes N independent task slices and runs them **concurrently**
as ephemeral coding agents on GitHub Actions, collecting each slice's output as
a reviewable artifact — while the local session stays free.

**Parallelism is the point, not a side effect.** A swarm that dispatches N
slices and executes them one after another has no reason to exist: the local
machine could do that. The wall-clock for N slices must approach the time of the
slowest single slice, not the sum. Everything below that would serialise
execution is therefore a defect, not a tuning issue.

Ceilings, measured rather than assumed:

| Limit | Value | Binding? |
|---|---|---|
| **Swarm fan-out cap** | **5 concurrent slices — hard, not configurable** | **Yes — the one that governs** |
| GitHub Actions concurrent jobs | 20 (this account is a Free org) | No, while the cap is 5 |
| DeepSeek concurrent requests | 500 (`deepseek-v4-pro`), 2500 (flash), [account-level](https://api-docs.deepseek.com/quick_start/rate_limit) | No, by a wide margin |

A slice occupies one job at a time (`agent`, then `publish`). Five sits below
GitHub's 20 so the swarm never consumes the whole account allowance: the target
repo's ordinary CI keeps running, and a bad swarm burns a bounded amount before
you notice.

Two keys in the caller compose into a genuine repo-wide ceiling, both enforced
by GitHub rather than by the CLI:

| Key | Bounds |
|---|---|
| `concurrency: {group: swarm, queue: max}` | one swarm run at a time per repo; further swarms queue FIFO instead of being cancelled |
| `strategy.max-parallel: 5` | five slices at a time within the active swarm |

Without the first, two overlapping swarms would run ten legs and the cap would
be per-run only. Enforcing that client-side would not hold — two shells or two
machines can race — so it belongs on the server.

Neither is a flag. Raising the cap means editing the caller workflow: a visible,
reviewable change rather than a forgotten shell argument.

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

- [ ] **AC1 — Correlation.** `swarm run` dispatches **one** run (D5) and takes
      its ID **directly from the dispatch response**, never by polling run
      names. Per-slice state comes from
      `GET /repos/{o}/{r}/actions/runs/{id}/jobs`, keyed by each matrix leg's
      `slice_id`. The dispatch response is gated on the API version:

      | `X-GitHub-Api-Version` | Response |
      |---|---|
      | `2022-11-28` — **`gh`'s default** | `204 No Content`, no run ID |
      | `2026-03-10` | `200 OK` with `workflow_run_id`, `run_url`, `html_url` |

      Both measured against this repo. The CLI MUST send
      `X-GitHub-Api-Version: 2026-03-10`; the default loses the run ID entirely.
      Verified: with the header, `{"workflow_run_id":30203984480,…}`; without
      it, `204`. Remaining for S2: one swarm of 3 slices yielding one
      `workflow_run_id` and 3 job-level `slice_id`s in the manifest (§5a). A
      response lacking the ID fails fast rather than guessing.
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
- [ ] **AC11 — Slices run concurrently, and never more than 5.** Not "N legs
      exist" — **overlapping execution** plus a respected cap. Dispatch 8 slices
      that each sleep long enough to observe, then from
      `GET /repos/{o}/{r}/actions/runs/{id}/jobs` assert:
      (a) at some instant **exactly 5** legs are `in_progress`, proving both real
      parallelism and that `max-parallel: 5` binds;
      (b) the other 3 are `queued`, never `cancelled` — this is what separates
      `max-parallel` from a default `concurrency` group, which discards (D1);
      (c) all 8 reach a terminal state **unattended**, with no CLI invocation
      after `swarm run` exits;
      (d) `swarm status` distinguishes queued from running;
      (e) wall-clock is roughly two waves, not eight serial slices.
- [ ] **AC12 — One failing slice does not harm its siblings.** Force one slice
      to fail; every other slice still completes and publishes. Verified
      adversarially, because GitHub's default `fail-fast: true` would cancel all
      in-progress and queued legs, converting one bad task into total data loss.
- [ ] **AC13 — The ceiling holds repo-wide, and CI is not starved.** Dispatch
      two swarms back to back and assert the second **queues** rather than
      running alongside — at no instant do more than 5 slice legs run in the
      repo, and the first swarm is never cancelled. Then confirm an ordinary
      workflow in the target repo still starts while a swarm is in flight. This
      is what `concurrency: {group: swarm, queue: max}` buys beyond
      `max-parallel` (§2); testing a single swarm would not catch its absence.
- [x] **AC8 — DeepSeek sustains a multi-turn agentic loop.** *Met by run
      [30203984480](https://github.com/mach4-braai/github-actions-agent-swarm/actions/runs/30203984480).*
      Shell execution is **observed, not inferred**. The workflow seeded a
      random nonce into `spike/target.txt` at runtime — never present in the
      prompt — so the expected hash could not be precomputed by anyone, and the
      agent could only learn the nonce by reading the file. The model's own
      `stream-json` output then showed the tool calls directly:

      ```
      2 Bash
      1 Read
      1 Write
      ```

      5 turns, `is_error: false`, `expected=b497bd3f2433 actual=b497bd3f2433`
      against nonce `nonce-515bd4bd6736b25f0ebae72d8c7d28e4`.
      Usage: 22,023 input / 813 output / 87,808 cached-read tokens.
      **Still out of scope:** MCP. No MCP server was configured or invoked, so
      client-side MCP over this endpoint remains untested (see D2).
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

**Artifact names are namespaced by `slice_id`.** Under D5 every slice is a
matrix leg of *one* run, and `upload-artifact@v4` names must be unique within a
run — generic names would collide across parallel legs.

| Artifact | Written by | Contents |
|---|---|---|
| `agent-<slice_id>` (`agent-result.json` + `patch.diff`) | `agent` job, `if: always()` | Everything the agent run knows: `status`, `summary`, `files_changed`, `usage`, `error`, timings, resolved model |
| `result-<slice_id>` (`result.json`) | `publish` job, `if: always()` | **Authoritative.** The above plus `branch`, `commit`, and any publish-stage failure |

`result-<slice_id>` is the record `swarm collect` reads. If `publish` never ran
or failed, `collect` falls back to `agent-<slice_id>` and reports the slice as
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

`swarm run` exits, so a later `swarm status` has nothing to go on unless the
dispatch is recorded locally. There is no daemon (§3), so the record is a file.

```
~/.swarm/<owner>/<repo>/
├── runs/<swarm_id>.json
└── current
```

`<swarm_id>.json` holds what the CLI cannot re-derive: `repo`, `base_ref`,
`created_at`, the single parent `run_id` from the dispatch response, and the
slice list (`slice_id` plus task digest). Per-slice state is not stored — it is
read live from the run's jobs (AC1), so the manifest never goes stale.

It lives under `$HOME`, not in the target repo: swarm state is the operator's,
not the project's, and a `.swarm/` directory would otherwise need a
`.gitignore` entry in every repo you ever swarm.

`swarm status` and `swarm collect` default to `current` and accept
`--run <swarm_id>` to address a finished one.

## 6. Decisions

Resolved decisions are recorded here as they land. Open ones block
implementation. **No code while any decision below is open.**

### D1 — Distribution: reusable workflow, consumed by the target repo (DECIDED 2026-07-26)

This repo publishes `.github/workflows/agent.yml` with `on: workflow_call`. A
target repo adopts the swarm by committing a thin caller into its own
`.github/workflows/`:

```yaml
name: swarm
run-name: swarm/${{ inputs.swarm_id }}

on:
  workflow_dispatch:
    inputs:
      swarm_id: { required: true,  type: string }
      slices:   { required: true,  type: string }
      base_ref: { required: false, type: string }

concurrency:
  group: swarm
  queue: max

jobs:
  slices:
    name: slice/${{ matrix.slice_id }}
    strategy:
      matrix:
        include: ${{ fromJSON(inputs.slices) }}
      max-parallel: 5
      fail-fast: false
    uses: mach4-braai/github-actions-agent-swarm/.github/workflows/agent.yml@<sha>
    with:
      slice_id: ${{ matrix.slice_id }}
      task:     ${{ matrix.task }}
      base_ref: ${{ inputs.base_ref }}
    permissions:
      contents: write
    secrets:
      DEEPSEEK_API_KEY: ${{ secrets.DEEPSEEK_API_KEY }}
```

`slices` is a JSON array of `{slice_id, task}`. The two concurrency keys are
explained in §2; `fail-fast: false` is mandatory (D5).

The explicit `name:` is required, not cosmetic. GitHub's default label for a
matrix leg interpolates every matrix value, so without it each job would be
titled with the **entire task prompt** — unreadable in the UI, and a brittle
correlation key. `slice_id` is constrained to `[a-z0-9-]{1,32}` so the job name
stays a clean lookup handle.

**`run-name` is display-only.** An earlier revision of this spec made it the
correlation handle, on the premise that `workflow_dispatch` returns no run ID.
That premise is obsolete: the dispatch endpoint now returns
`workflow_run_id` directly (AC1). The line stays because `agent/<slice_id>` is
far easier to scan in the Actions UI than five identical `swarm-agent` rows —
but a consumer who deletes it degrades nothing but readability.

This retires the design's sharpest edge. Correlation no longer depends on
consumer-owned code being exactly right, so the whole class of "the caller
dropped a line and runs became unfindable" failures disappears rather than
being mitigated.

Two supports remain, both still worth their keep:

- **`swarm init`** writes the caller into the target repo's working tree for the
  human to review and commit. A local write; the orchestrator never needs a
  token that can push to a consumer's repo.
- **`swarm run` preflights the *remote* caller before dispatching**, via
  `gh api repos/{owner}/{repo}/contents/.github/workflows/<file>?ref=<ref>`. It
  no longer checks `run-name`. It verifies two things:
  1. the **declared inputs** match what the CLI is about to send — GitHub would
     reject a mismatch anyway, but preflight turns that into a clear local
     error and catches the uncommitted-caller case;
  2. the concurrency keys are exactly as required (below) — the workflow-level
     group present with `queue: max`, and no job-level group.

Preflight must still read the **remote** ref, never the local working tree:
dispatch executes whatever GitHub has, so a freshly `init`-ed but uncommitted
file would pass a local check and then 404. `workflow_dispatch` also requires
the workflow on the **default branch** for the event to exist at all, so
preflight checks both the default branch and the dispatch ref when they differ.

**The two concurrency levels do opposite things, so preflight treats them
differently.** Per the
[workflow syntax reference](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#concurrency),
a group permits one running run and — under the default `queue: single` — one
pending, where "any existing `pending` job or workflow in the same concurrency
group will be canceled." The default discards; `queue: max` queues up to 100.

| Placement | Verdict |
|---|---|
| workflow-level `concurrency: {group: swarm, queue: max}` | **Required.** Serialises whole swarms without loss, giving the repo-wide half of the ceiling (§2) |
| workflow-level with default `queue: single` | **Rejected.** Dispatching a second swarm cancels the first while it is pending |
| any `cancel-in-progress: true` | **Rejected.** Kills a running swarm. Also invalid combined with `queue: max` |
| `jobs.slices.concurrency` | **Rejected.** Gates the matrix legs, so slices serialise or cancel each other inside one swarm |

Groups are repository-scoped, so the blast radius is one consumer's own swarms.
The reusable workflow declares no `concurrency` key — capping belongs in the
caller, where the consumer can see it.

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
| `cache_control` | Ignored | The field is ignored, but caching still happens: DeepSeek runs [context caching on disk](https://api-docs.deepseek.com/guides/kv_cache) by default, no code change needed. Observed once, in run [30202976198](https://github.com/mach4-braai/github-actions-agent-swarm/actions/runs/30202976198): `cache_read_input_tokens: 107904` vs `cache_creation_input_tokens: 0` (a translation of DeepSeek's native `prompt_cache_hit_tokens`). The spec's original "no prompt caching" claim is refuted. But DeepSeek states it is **best-effort with no guaranteed hit rate**, hits need a **full match** of a persisted prefix unit, and unused caches clear "within a few hours to a few days" — so this is one measured run, not a cost curve. |
| `image` / `document` | Not supported | No screenshot or PDF slices. Code-only tasks. |
| `mcp_servers` | Ignored | Anthropic's *server-side* MCP connector is unavailable. Whether **client-side MCP** works is `[UNTESTED]` — AC8 exercised only Claude Code's built-in Read/Edit/Bash tools, not an MCP server. Ordinary tool-calling is confirmed; do not extrapolate from that to MCP. No slice in this MVP needs MCP. |
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

### D5 — Fan-out: one caller run, matrix of slices, `max-parallel: 5` (DECIDED 2026-07-26)

`swarm run` dispatches **one** workflow run whose single job is a matrix over
the slices, each leg calling the reusable workflow. The caller template in D1 is
the authoritative form; the load-bearing keys are `max-parallel: 5` and
`fail-fast: false`.

Supported explicitly: ["Jobs using the matrix strategy can call a reusable
workflow."](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows#using-a-matrix-strategy-with-a-reusable-workflow)
Nesting allows ten levels; this uses two.

**Why not a client-side cap.** The obvious alternative — the CLI dispatches five
runs, then dispatches more as they finish — is broken by §3's no-daemon rule.
`swarm run` exits, so nothing is left to drain the queue: slices six and beyond
would stall until the operator happened to run another command. That
contradicts the whole "local session stays free" premise. `max-parallel` hands
the queue to Actions, which drains it unattended.

It is also the *only* GitHub mechanism that queues without discarding. A
`concurrency` group cancels pending runs by default (D1), and `queue: max`
serialises to one at a time. `max-parallel: 5` runs five and queues the rest.

**`fail-fast: false` is mandatory, and the default is wrong for us.** It
[defaults to `true`](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idstrategyfail-fast),
which "will cancel all in-progress and queued jobs in the matrix if any job in
the matrix fails." One slice hitting a syntax error would destroy every other
slice's work. This is the same shape of trap as the `concurrency` key: a
sensible default for ordinary CI, catastrophic for a swarm.

**What this costs.** Slices are now jobs inside one run, not independent runs:

- Correlation is *simpler*: one `workflow_run_id` from the dispatch response,
  then per-slice status from `GET /repos/{o}/{r}/actions/runs/{id}/jobs`, keyed
  by the matrix leg's `slice_id`.
- Cancelling the parent run cancels every slice. Acceptable — that is usually
  what you want, and it makes "stop the swarm" one call.
- Re-running one slice is `gh run rerun --job <job_id>`, not a fresh dispatch.
- All slices travel in one `slices` JSON input, so the **65,535-character
  aggregate budget is now shared across every slice at once** — the payload
  limit binds much sooner than it did with one dispatch per slice.
- Matrix is capped at 256 jobs, far above the fan-out cap.

## 7. Risks & rollback

| Risk | Mitigation |
|---|---|
| Task prompts overrun the dispatch payload budget | **65,535 characters across the entire serialized `inputs` object**, and under D5 *every slice travels in one `slices` input* — so the budget is shared by the whole swarm at once, and binds far sooner than with one dispatch per slice ([REST docs](https://docs.github.com/en/rest/actions/workflows#create-a-workflow-dispatch-event); the 25-property cap is not the constraint here). The CLI measures the serialized aggregate and names the offending slices before dispatch, rather than surfacing GitHub's bare `Error: inputs are too large`. Payload indirection stays out of the MVP |
| Caller in the consumer's repo is wrong or uncommitted | No longer a correlation risk — the run ID comes from the dispatch response (AC1), so `run-name` is display-only. Residual: a caller whose declared inputs mismatch. `swarm init` generates it; `swarm run` preflights the **remote** ref and refuses to dispatch, so no orphan run is created |
| Agent pushes garbage to the target repo | Agent job has no write token (D2); `publish` validates the patch (AC10); `swarm/*` namespace only; branch protection on `main` |
| Prompt injection from repo content exfiltrates `DEEPSEEK_API_KEY` | **Not fully mitigable** — the key must reach the agent. Bounded by a dedicated, spend-capped, rotatable key and the trusted-input fence in §3. The write token is out of reach via the two-job split (AC9) |
| Agent poisons the runner to hijack a later privileged step | Privileged work runs in a separate job on a fresh runner, consuming only a validated artifact (D2) |
| Runaway job cost / infinite agent loop | Hard `timeout-minutes` on the job; `status: timeout` in the contract |
| GitHub concurrency limit or Actions outage | The swarm caps itself at 5 (§2), below the account's 20, so it queues via `max-parallel` rather than contending. Swarm is additive — local execution always remains available |
| Swarm silently serialises and nobody notices | Presents as "it worked, just slowly", so it needs an assertion rather than vigilance: AC11 requires exactly 5 legs `in_progress` at once, and the rest `queued` not `cancelled` |
| A `concurrency` key or `fail-fast: true` destroys slices | Both cancel rather than delay, and both look like ordinary CI hygiene. Preflight rejects any `concurrency` key at either level (D1); `fail-fast: false` is mandatory and asserted by AC12 |
| Long agent loops cost more than expected | Measured once (AC8, a *trivial* 3-step task): 6 turns, 21,509 input + 1,141 output, 107,904 cached-read tokens — fixed system-prompt and tool-definition overhead dominates a small slice. Cross-slice behaviour is **unmeasured and not guaranteed**: caching is best-effort, needs a full prefix match, and expires when idle. Slices sharing one system prompt plausibly hit the shared prefix, but do not budget on it. Record `usage` per slice and measure against DeepSeek's own billing before treating cost as solved |
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
