# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

**lp-smart-code-e2e-test** — AI skill (Claude Code skill) for writing and reviewing E2E tests. Framework-agnostic (Playwright, Cypress, Selenium), focused exclusively on end-to-end testing — full user journeys through a real browser. Based on practices from Beck, Fowler, Cohn, Rauch, and Google.

Two modes:
- **Generate** (default) — analyze features, classify flows, produce E2E tests with Page Objects
- **Review** — evaluate existing E2E tests for antipatterns, flakiness, and quality

## Architecture

```
SKILL.md              ← Main skill definition: workflow, rules, output formats
agents/agent.yaml     ← Agent interface config (display name, default prompt)
references/           ← Knowledge base loaded on-demand during skill execution
```

### Skill Execution Flow

The skill is invoked via `/lp-smart-code-e2e-test`. SKILL.md defines the entire workflow:

**Generate mode** (7 steps): Determine scope → Preflight → Classify flows (E2E Flow Matrix) → Determine scenarios → Generate tests → Self-check antipatterns → Output

**Review mode** (4 steps): Preflight → Evaluate against checklist → Assess severity → Output

Steps marked MANDATORY in SKILL.md must load specific reference files before execution — this is enforced by the workflow, not optional.

### References (Knowledge Base)

Each reference file is loaded at specific steps, not all at once:

| File | Loaded When |
|------|-------------|
| `e2e-testing-principles.md` | Step 2 (Classify flows) |
| `user-flow-testing.md` | Step 3 (Determine scenarios) |
| `e2e-test-design-patterns.md` | Step 4 (Generate tests) |
| `network-and-api-handling.md` | Step 4 (Generate tests) |
| `waiting-and-synchronization.md` | Step 4 (Generate) / Step 2 (Review) |
| `test-data-management.md` | Step 4, only if data setup needed |
| `accessibility-in-e2e.md` | Step 4, only if a11y checks selected |
| `e2e-antipatterns-checklist.md` | Step 5 (Self-check) / Step 2 (Review) |
| `e2e-test-review-checklist.md` | Step 2 (Review mode only) |

### Key Concepts

- **E2E Flow Classification Matrix** (criticality × complexity + risk tags) drives which flows get tested and how
- **Risk tags** (PAYMENT, AUTH, DATA_INTEGRITY, PII, COMPLIANCE) override the matrix — always require E2E
- **Severity levels**: 🔴 Critical = false confidence/flakiness/shared data, 🟠 High = sleep waits/CSS selectors/UI auth, 🟡 Medium = readability, 🟢 Low = style
- **Programmatic auth by default**: one UI login test, all others via API
- **Determinism first**: no sleep(), proper waits, isolated data
- **Confirmation-first**: never write test files to disk without user confirmation

## Editing Guidelines

- `SKILL.md` is the single source of truth for the skill workflow. All mandatory steps, output formats, and rules live here.
- `references/` files are the knowledge base. They are self-contained documents — each covers one topic fully.
- `agents/agent.yaml` defines the agent interface only (display name, description, default prompt). Keep it minimal.
- When adding a new E2E testing topic, create a new file in `references/` and add a loading instruction to the relevant step in SKILL.md.

## Важно
- Не делай вывод слишком громоздким, чтобы экономить токены на выводе
