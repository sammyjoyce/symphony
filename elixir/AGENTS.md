# Symphony Elixir

This directory contains the Elixir agent orchestration service that polls Linear, creates per-issue workspaces, and runs Codex in app-server mode.

## Environment

- Elixir: `1.19.x` (OTP 28) via `mise`.
- Install deps: `mix setup`.
- Main quality gate: `make all` (format check, lint, coverage, dialyzer).


## Codebase-Specific Conventions

- Runtime config is loaded from `WORKFLOW.md` front matter via `SymphonyElixir.Workflow` and `SymphonyElixir.Config`.
- Keep the implementation aligned with [`../SPEC.md`](../SPEC.md) where practical.
  - The implementation may be a superset of the spec.
  - The implementation must not conflict with the spec.
  - If implementation changes meaningfully alter the intended behavior, update the spec in the same
    change where practical so the spec stays current.
- Prefer adding config access through `SymphonyElixir.Config` instead of ad-hoc env reads.
- Workspace safety is critical:
  - Never run Codex turn cwd in source repo.
  - Workspaces must stay under configured workspace root.
- Orchestrator behavior is stateful and concurrency-sensitive; preserve retry, reconciliation, and cleanup semantics.
- Follow `docs/logging.md` for logging conventions and required issue/session context fields.

## Tests and Validation

Run targeted tests while iterating, then run full gates before handoff.

```bash
make all
```

## Required Rules

- Public functions (`def`) in `lib/` must have an adjacent `@spec`.
- `defp` specs are optional.
- `@impl` callback implementations are exempt from local `@spec` requirement.
- Keep changes narrowly scoped; avoid unrelated refactors.
- Follow existing module/style patterns in `lib/symphony_elixir/*`.

Validation command:

```bash
mix specs.check
```

## PR Requirements

- PR body must follow `../.github/pull_request_template.md` exactly.
- Validate PR body locally when needed:

```bash
mix pr_body.check --file /path/to/pr_body.md
```

## Nix Fallback for Toolchain

When `mise install` fails (e.g., missing system build dependencies for Erlang
compilation), use Nix with exact version attributes:

```bash
nix shell nixpkgs#elixir_1_19 nixpkgs#erlang_28
```

Avoid generic `nixpkgs#elixir` / `nixpkgs#erlang` — they resolve to older versions
that don't match `mise.toml` pins.

## Known Lint Debt

- `lib/symphony_elixir/codex/dynamic_tool.ex` — `normalize_sync_workpad_args/1`
  exceeds Credo's cyclomatic complexity limit (11 > 9). Refactor into smaller
  helpers when touching this module.

## Staying Synced with Upstream openai/symphony

This is a fork of [openai/symphony](https://github.com/openai/symphony). Before
merging upstream, verify fork parity with targeted greps — faster than a full diff:

```bash
grep -r "ssh\.ex" lib/                           # SSH support
grep -r "PathSafety\|canonicalize" lib/           # Path safety module
grep -r "protocol_message_candidate" lib/         # JSON filtering
grep -r "tick_token\|retry_token" lib/            # Token reconciliation
```

- Use `librarian` (or `git log` on a shallow clone) to read upstream commit history
  and per-commit diffs without maintaining a full second checkout.
- Build a commit-level comparison table (upstream SHA → present / absent / N/A)
  before any cherry-pick or merge.
- Upstream CI-only changes (e.g., pinned GitHub Actions SHAs) are N/A when the fork
  uses its own `.github/` workflows — skip without merging.

## Dispatch Pipeline Invariants (Learned 2026-03-30)

`should_dispatch_issue?` in `orchestrator.ex` gates every dispatch. An issue
dispatches only when **all** of these hold:

1. **`candidate_issue?`** — issue state is in `active_states` and not terminal;
   issue is assigned to a routable worker (assignee filter).
2. **`!todo_issue_blocked_by_non_terminal?`** — for Todo issues, every
   `blocked_by` relation must reference an issue in a `terminal_states` state.
   A single non-terminal blocker blocks dispatch.
3. **Not already `claimed` or `running`** in orchestrator state.
4. **Slots available** — global `max_concurrent_agents`, per-state limits
   (`max_concurrent_agents_for_state`), and worker-level limits all checked.

### Project slug filter

`fetch_candidate_issues` in `linear/client.ex` filters by `project_slug` from
WORKFLOW.md's `tracker` config. Issues that lack a project assignment (even if
they belong to the right team) are **invisible** to the poll — always assign
the project explicitly on child issues, not just the parent epic.

### Blocker state is the only dispatch gate (upstream)

This upstream repo checks blocker **state** only (`non_terminal?`). Fork
variants (e.g., HAR) may add PR-merge verification gates
(`terminal_blocker_merge_verified?` / `PullRequestVerifier`). When porting
fork-specific gates back upstream, add them as composable verifier functions
passed to `todo_issue_blocked_by_non_terminal?`, not as inline conditionals.

## Docs Update Policy

If behavior/config changes, update docs in the same PR:

- `../README.md` for project concept and goals.
- `README.md` for Elixir implementation and run instructions.
- `WORKFLOW.md` for workflow/config contract changes.
