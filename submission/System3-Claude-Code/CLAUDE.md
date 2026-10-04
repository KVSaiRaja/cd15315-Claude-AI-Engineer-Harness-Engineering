# E-Commerce Platform — Shared Claude Code Conventions

This file is the **project-level** entry point for the e-commerce monorepo. Every teammate's Claude session loads it automatically.

## Scope: what belongs here vs. elsewhere

| Scope | Location | Purpose |
|---|---|---|
| Project-level | `./CLAUDE.md`, `.claude/` | Shared repository instructions and team conventions committed to version control. |
| User-level | `~/.claude/` | Personal preferences and settings for an individual developer. User-level settings are **NOT shared with teammates via version control**. Example: preferred commit message style. |
| Directory-level | `CLAUDE.md` inside a subdirectory | Instructions specific to that directory and its files. |

The team keeps shared conventions at project level so all teammates receive the same repository guidance, while personal preferences remain user-level and directory-specific guidance stays close to the code it affects.

## Shared standards (modular via @-imports)

The actual conventions live in focused files so this entry point stays scannable:

@.claude/standards/frontend.md
@.claude/standards/api.md
@.claude/standards/database.md
@.claude/standards/testing.md

Path-scoped rules in [.claude/rules/](.claude/rules/) layer on top of these standards and activate only when Claude is editing matching files (React components, API handlers, test files).

## Repository layout

```text
src/components/   React components (functional + hooks)
src/pages/        Top-level routes
src/api/          Node.js API handlers
src/db/           PostgreSQL repository-pattern modules

### 2. Replace `.claude/rules/api.md`

```bash
cat > .claude/rules/api.md <<'EOF'
---
description: Conventions for Node.js API handler files
paths:
  - "src/api/**/*"
---

# API handler rules

- Keep API handlers thin: validate the request, call the appropriate repository/service, and return the response; do not put database queries directly in the handler.
- Use the shared API error type for expected client and domain errors so handlers return consistent error responses.


## Troubleshooting

Run `/memory` to diagnose which Claude Code configuration files are loaded for the current session.
