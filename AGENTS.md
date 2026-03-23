# Symphony

Top-level guidance for agents working across this repository.

## Repository Layout

- `elixir/` — Elixir orchestration service (see `elixir/AGENTS.md` for subsystem rules).
- `.agents/skills/` — Agent skills (commit, debug, land, linear, pull, push, symphony-setup).
- `SPEC.md` — Canonical specification; implementation must not conflict with it.

## Environment Bootstrapping

The Elixir service requires Erlang/OTP 28 and Elixir 1.19.x. The repo pins versions
in `elixir/mise.toml`. When `mise` is unavailable or fails to compile (missing system
build deps like curses/autoconf), use Nix as a fallback:

```bash
nix shell nixpkgs#elixir_1_19 nixpkgs#erlang_28
```

**Use exact Nix package attributes** (`elixir_1_19`, `erlang_28`) — generic attrs
(`nixpkgs#elixir`, `nixpkgs#erlang`) pull older versions that won't match `mise.toml`.

## Push Safety on `main`

When pushing directly to `main`:

1. Fetch first: `git fetch origin main`
2. Verify sync: `git rev-list --left-right --count origin/main...HEAD`
3. Stage precisely: `git add <specific files>` — never `git add -A` or `git add .`
4. Use conventional commit prefixes (`docs:`, `fix:`, `feat:`, etc.)

## Validation Discipline

Always run the full quality gate (`make -C elixir all`) before pushing, even for
docs-only changes. This catches pre-existing failures early and proves your change
doesn't introduce new ones. If a gate fails on code you didn't touch, note it
explicitly in the commit/PR but don't let it block unrelated work.

## Known Issues

- `lib/symphony_elixir/codex/dynamic_tool.ex:114` — `normalize_sync_workpad_args/1`
  exceeds Credo's cyclomatic complexity limit (11 > max 9). Pre-existing; needs
  refactoring into smaller helper functions.
