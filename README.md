# LP Smart Code E2E Test

A comprehensive **E2E testing** skill for AI agents. Focused exclusively on end-to-end tests — full user journeys through a real browser. Framework-agnostic (Playwright, Cypress, Selenium). Generates ideal E2E tests and reviews existing ones using best practices from Kent Beck, Martin Fowler, Mike Cohn, Guillermo Rauch, and Google.

## Installation

```bash
npx skills add lordprotein/lp-smart-code-e2e-test
```

Or via Agent Skills:

```bash
npx agent-skills-cli install @lordprotein/lp-smart-code-e2e-test
```

## Features

- **Two Modes** — Generate new E2E tests or review existing ones
- **Flow Classification** — E2E Flow Matrix: criticality × complexity + risk tags (PAYMENT, AUTH, PII)
- **Page Object Model** — Auto-generates Page Objects for UI abstraction
- **Smart Authentication** — Programmatic login by default, one UI login test only
- **Wait Strategy** — No sleep(), proper auto-waiting and explicit conditions
- **Antipattern Detection** — 12 E2E-specific antipatterns with severity levels and fixes
- **Accessibility Integration** — Optional axe-core checks and keyboard navigation testing
- **Framework Agnostic** — Works with Playwright, Cypress, Selenium, or any E2E framework
- **Project-Aware** — Discovers existing Page Objects, fixtures, auth patterns, and conventions

## Usage

After installation, run:

```
/lp-smart-code-e2e-test
```

### Mode 1: Generate E2E Tests

The skill analyzes your application features and generates comprehensive E2E tests:

1. Determines test scope — asks which flow types to generate (critical journeys, core features, workflows)
2. Classifies flows by criticality × complexity using E2E Flow Matrix + risk tags
3. Identifies test scenarios (happy paths, critical sad paths, state transitions) — 3-8 per flow
4. Generates tests with Page Objects, proper waits, and programmatic auth
5. Self-checks against 12 E2E antipatterns before output

### Mode 2: Review E2E Tests

The skill reviews existing E2E tests for quality issues:

1. Evaluates flakiness risk, wait strategies, selector quality
2. Detects antipatterns (The Sleeper, The Chain Gang, The False Prophet, etc.)
3. Assesses flow coverage and auth efficiency
4. Reports findings by severity (🔴 Critical / 🟠 High / 🟡 Medium / 🟢 Low)
5. Asks for confirmation before implementing fixes

## Workflow

### Generate Mode
1. **Scope** — Determine flow types: critical journeys, core features, cross-page workflows
2. **Preflight** — Analyze app, detect framework, find Page Objects, auth patterns
3. **Classify** — E2E Flow Matrix (criticality × complexity + risk tags)
4. **Scenarios** — Happy paths, sad paths, state transitions, edge conditions (3-8 per flow)
5. **Generate** — Page Objects, AAA pattern, programmatic auth, proper waits
6. **Self-check** — Verify against 12 E2E antipatterns
7. **Output** — Tests with classification, Page Objects, and coverage notes

### Review Mode
1. **Preflight** — Collect E2E test files, read Page Objects, production code
2. **Evaluate** — 10-step procedure: flakiness → assertions → isolation → waits → selectors → coverage → abstraction → data → readability → auth
3. **Severity** — Assign 🔴/🟠/🟡/🟢 to each finding
4. **Output** — Findings with flakiness risk assessment and suggested fixes
5. **Confirm** — Ask user before implementing changes

## Severity Levels

| Badge | Level | Action |
|-------|-------|--------|
| 🔴 | **Critical** | Must fix — false confidence, flaky tests, shared data |
| 🟠 | **High** | Should fix — sleep waits, CSS selectors, UI auth, missing coverage |
| 🟡 | **Medium** | Fix or follow-up — readability, naming, organization |
| 🟢 | **Low** | Optional — style, minor suggestions |

## Structure

```
lp-smart-code-e2e-test/
├── SKILL.md                              # Main skill definition
├── agents/
│   └── agent.yaml                        # Agent interface config
└── references/
    ├── e2e-testing-principles.md         # Four pillars for E2E, flow matrix, test pyramid
    ├── e2e-test-design-patterns.md       # Page Object Model, Screenplay, selectors, auth
    ├── user-flow-testing.md              # Critical paths, happy/sad, multi-step flows
    ├── test-data-management.md           # API setup, factories, isolation, cleanup
    ├── network-and-api-handling.md       # Network mocking, interception, error simulation
    ├── waiting-and-synchronization.md    # No sleep(), auto-waiting, explicit waits
    ├── accessibility-in-e2e.md           # axe-core, WCAG, keyboard nav, focus
    ├── e2e-antipatterns-checklist.md     # 12 antipatterns with detection and fixes
    └── e2e-test-review-checklist.md      # Structured review checklist by severity
```

## Knowledge Base

Based on:
- **Kent Beck** — Test-Driven Development: By Example
- **Martin Fowler** — TestPyramid, PageObject pattern, Refactoring
- **Mike Cohn** — Succeeding with Agile (Test Pyramid)
- **Guillermo Rauch** — Testing Trophy
- **Google** — Testing Blog, Software Engineering at Google
- **Microsoft** — Playwright documentation, E2E testing best practices
- **Deque Systems** — axe-core, accessibility testing

## License

MIT
