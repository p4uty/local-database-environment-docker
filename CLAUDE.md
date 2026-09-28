# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## Claude Code specifics

- The `devdb` skill for *using* the environment lives in `skills/devdb/` and is exposed to Claude Code through the
  `.claude/skills/devdb` symlink. `./devdb install --skill` also links it into `~/.claude/skills/` so it works from
  any project. Keep `SKILL.md` in sync when you change CLI commands or flags.
- User-facing docs (`README.md`, `init/README.md`, CLI messages, `SKILL.md`) are written in Spanish. Keep them in Spanish.
