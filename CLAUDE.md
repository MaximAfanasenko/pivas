# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**Project knowledge (architecture, build system, conventions):** see [docs/KNOWLEDGE.md](docs/KNOWLEDGE.md)

## Instructions

- **POSIX sh only** — no `[[`, no bash arrays, no process substitution. All conditionals use `[`.
- Follow existing function naming conventions (`cmd_*`, `get_*`, `is_*`, `setup_*`, `update_*`, `ip4_*`).
- Use the output helpers (`ready`, `when_ok`, `when_bad`, `error`, `print_line`) for user-facing messages.
- There is no formal test suite. Manual testing uses `kvas test` and `kvas debug` on the target router.
