<!-- Copilot instructions for the Microsoft Fabric to Gateway migration project -->

# Project: Microsoft Fabric to Gateway Migration

Automated migration of Microsoft Fabric artifacts to Gateway format.

## Architecture — Pipeline

```
Microsoft Fabric → Gateway
```

## Project Structure

- **Source / Extraction**: `src/`
- **Target / Generation**: (generated output)
- **Tests**: `tests/` (0 test files)
- **Docs**: `docs/`

## Key Modules

- See source directories for module listing

## Hard Constraints

1. **Read before write** — never assume file contents from memory
2. **Test after every change** — run `pytest tests/ --tb=short -q`
3. **No duplicate functions** — always search for an existing name before creating one
4. **Git hygiene** — commit only when tests pass, conventional messages (`feat:`, `fix:`, `test:`, `docs:`)

## Multi-Agent Architecture

This project uses a specialized agent architecture. See `docs/AGENTS.md` for the full
architecture diagram and `.github/agents/` for per-agent definitions.

## Workflow Rules

### 1. Plan Before Build
- For multi-step work, create a plan before starting
- If something goes sideways, STOP and re-plan

### 2. Read Before Write
- **Always read target code before editing**
- Read `copilot-instructions.md` at session start for project rules

### 3. Testing Contract
- Run `pytest tests/ --tb=short -q` after EVERY implementation change
- If tests fail → fix them before reporting completion
- New features **require** new tests
- Never weaken test assertions to make tests pass

### 4. Scope Discipline
- Only modify files directly related to the task
- No drive-by refactors
- Prefer the smallest change that solves the problem
