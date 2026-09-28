# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## Claude Code specifics

- The `devdb` skill for *using* the environment lives in `skills/devdb/` and is exposed to Claude Code through the
  `.claude/skills/devdb` symlink. `./devdb install --skill` also links it into `~/.claude/skills/` so it works from
  any project. Keep `SKILL.md` in sync when you change CLI commands or flags.
- User-facing docs (`README.md`, `init/README.md`, CLI messages, `SKILL.md`) are written in Spanish. Keep them in Spanish.
- A new or changed CLI flag touches four places: `cmd_help` in `devdb`, the "Referencia del CLI" table in `README.md`,
  the quick-reference table in `SKILL.md`, and (if it should be remembered between runs) `load_state`/`save_state`.

## Testing changes

- The Bash tool has no TTY, so `devdb` never enters the interactive wizard there. To exercise the wizard, drive it
  through a pseudo-terminal (e.g. Python's `pty.fork()`); `script` may not be installed.
- `.devdb.state` persists the last selection (`DBS`, `MODE`, `UI`, `PGVECTOR`) and is reused by a bare `devdb up`.
  Delete it before tests that depend on defaults, and clean up afterwards with `./devdb down --volumes && rm -f .devdb.state`.
- Default host ports may already be taken by other projects on the machine. Export alternates for the whole test run
  (e.g. `POSTGRES_PORT=15432 QDRANT_PORT=16333 ADMINER_PORT=18080`); the CLI and Compose both read them.
- Test in both storage modes: behavior that only shows up with existing volumes (init scripts not re-running,
  pgvector image switches) needs `--persist`.
- Never touch containers outside the `devdb` Compose project.

## Git workflow

Work on `develop` and open PRs into `main` on `p4uty/local-database-environment-docker`. PRs are merged on GitHub, so
fetch `origin/main` before opening a new one; a previously opened PR may already be merged.
