---
name: lp-smart-code-e2e-test
description: "Write ideal E2E tests and review existing ones. Focused exclusively on end-to-end testing — full user journeys through a real browser. Not for unit or integration tests. Two modes: generate new E2E tests from feature analysis, or review existing E2E tests for antipatterns, flakiness, and quality issues. Framework-agnostic, based on best practices from Beck, Fowler, Cohn, Rauch, and Google."
---

# E2E Test Expert

## Overview

Generate high-quality E2E tests or review existing tests using industry best practices. Works in two modes:

- **Mode 1: Generate E2E Tests** — analyze application flows and produce comprehensive end-to-end tests
- **Mode 2: Review E2E Tests** — evaluate existing E2E tests for quality, flakiness, and antipatterns

Default to **Generate** mode unless the user explicitly asks to review existing tests or the input is clearly test code.

### Scope

This skill is focused **exclusively on E2E testing**:
- Tests **full user journeys** through a real browser
- Verifies **critical business flows** end-to-end
- External services mocked at **network level only**
- No unit tests, no component tests, no pure API tests
- "Integration" within E2E scope is acceptable: e.g., testing a multi-step checkout that spans multiple pages

## Severity Levels

| Level | Name | Description | Action |
|-------|------|-------------|--------|
| **P0** | Critical | False confidence (test passes but doesn't verify outcome), flaky tests (non-deterministic pass/fail), shared test data (tests pollute each other) | Must fix immediately |
| **P1** | High | sleep()/hardcoded waits, test order dependency, UI login in every test, fragile selectors (CSS/XPath), missing critical flow coverage | Should fix before merge |
| **P2** | Medium | Readability, naming, poor test organization, environment-specific tests | Fix in this PR or follow-up |
| **P3** | Low | Style, minor suggestions | Optional improvement |

---

## Mode 1: Generate E2E Tests

### 0) Determine test scope

**MANDATORY** — you MUST perform this step BEFORE preflight. Do NOT skip it.

- Analyze the application and determine which flow categories are present:
  - **Critical user journeys** — auth, checkout, payment, data creation/deletion
  - **Core feature flows** — main CRUD operations, search, navigation
  - **Cross-page workflows** — multi-step wizards, onboarding, settings
- **Ask the user** (multiselect) which types of E2E tests to generate:
  - E2E tests for critical user journeys
  - E2E tests for core feature flows
  - E2E tests for cross-page workflows
- If the application clearly belongs to only one category, still confirm with the user before proceeding.
- Generate only the selected categories — do NOT generate tests for unselected categories.

### 1) Preflight context

- Use `git status -sb`, `git diff --stat`, and `git diff` to scope changes. If user specifies features/pages, analyze those instead.
- Identify the E2E test framework (Playwright, Cypress, Selenium, etc.) and configuration.
- Search for existing E2E tests (`rg -g "*e2e*" -g "*spec*" -g "*test*" --files`) to discover:
  - Test directory structure and file naming conventions
  - Page Objects or abstractions in use
  - Fixtures, helpers, auth setup patterns
  - Base URL, environment config
- Search for existing Page Objects/abstractions.
- Identify authentication mechanism (API-based, UI-based, storage state).

**Edge cases:**
- **No E2E framework found**: Ask user about preferred framework or propose Playwright.
- **No changes**: If `git diff` is empty, ask which features/flows to test.
- **Large scope**: Summarize features first, ask user to prioritize.
- **Test files in diff**: Switch to Mode 2 for test files, Mode 1 for production files.

### 2) Classify flows (E2E Flow Matrix)

**MANDATORY** — you MUST load and apply the classification matrix. Do NOT skip this step, do NOT generate tests without classifying first.

- Load `references/e2e-testing-principles.md` for the E2E Flow Classification Matrix.
- Categorize each user flow using two axes: **business criticality** x **flow complexity** + **risk tags**.

| | Low complexity (single page CRUD) | Medium complexity (multi-step, cross-page) | High complexity (cross-service, external) |
|---|---|---|---|
| **Critical** (revenue, auth, data integrity) | Core CRUD — few focused tests | Critical Journeys — sweet spot, test thoroughly | System integration — key paths only |
| **Important** (frequently used features) | Happy path only | Happy + primary sad paths | Happy path + monitoring |
| **Nice-to-have** (cosmetic, rare features) | Skip E2E | Skip or smoke test only | Contract tests instead of E2E |

Risk tags: `PAYMENT`, `AUTH`, `DATA_INTEGRITY`, `PII`, `COMPLIANCE` — override matrix, always require E2E regardless of position.

- Report classification to user before generating tests.
- For nice-to-have + low complexity: recommend skipping E2E, suggest unit/integration instead.
- For overcomplicated cross-service flows: recommend contract tests + selective E2E.

### 3) Determine scenarios

**MANDATORY** — you MUST load scenario identification techniques before determining scenarios. Do NOT rely on general knowledge alone.

- Load `references/user-flow-testing.md` for flow scenario identification.
- For each classified flow, identify (3-8 scenarios per flow — E2E tests are expensive):
  - **Happy path**: The primary successful user journey — always first
  - **Critical sad paths**: Payment failure, auth rejection, data validation failure
  - **State transitions**: Multi-step navigation, back/forward, resume after interruption
  - **Edge conditions**: Empty states, maximum data, concurrent access
- Prioritize: revenue-impacting first, data-integrity second, user-blocking third.
- Limit: 3-8 scenarios per flow. E2E tests are expensive — be selective.

### 4) Generate tests

**MANDATORY** — you MUST load the reference files listed below before generating any test code.

- Load `references/e2e-test-design-patterns.md` for patterns (POM, selectors, auth).
- Load `references/network-and-api-handling.md` for network mocking strategy.
- Load `references/waiting-and-synchronization.md` for wait strategies.
- **If test data setup needed** → also load `references/test-data-management.md` for data setup patterns.
- **If accessibility checks selected** → also load `references/accessibility-in-e2e.md` for a11y integration.
- Apply these rules:

**Structure:**
- Follow AAA pattern adapted for E2E: Arrange (navigate + API setup) → Act (UI interactions) → Assert (visible outcome)
- Use Page Object Model for UI abstraction — tests read as user stories
- One user flow per test — never mix unrelated journeys
- No conditional logic in tests (no if/for/try-catch)
- Group tests by feature/flow using describe/context

**Selectors (priority order):**
- role > label > text > placeholder > test-id > NEVER css/xpath
- Document selector strategy choice in Page Objects

**Naming:**
- Use project's existing naming convention if detected
- Otherwise: describe the user journey, not implementation
- Format: `test_[user_action]_[expected_outcome]` or `should [describe user journey]`
- Use domain language: "checkout", "register", not "click button then fill form"

**Authentication:**
- ONE UI login test — verifies the login flow itself
- ALL other tests: programmatic login (API call → token → inject into storage/cookies)
- Use storage state reuse across tests in same auth context
- Multi-role: create fixtures per role (admin, user, guest)

**Network handling:**
- Mock ONLY external third-party services at network level
- NEVER mock your own backend — E2E means end-to-end
- Wait for network responses before assertions — never assume timing

**Wait strategy:**
- NEVER use sleep()/wait(ms)/hardcoded delays
- Use framework's auto-waiting capabilities
- Use explicit wait-for-condition: wait for text, element, URL change, network idle
- Set reasonable timeouts (not too short, not too long)

**Data setup:**
- Prefer API-based setup in beforeEach (create test data via API)
- Each test manages its own data — no shared mutable test data
- Clean up in afterEach via API calls
- Use unique identifiers (UUID) to prevent data collision

### 5) Self-check against antipatterns

**MANDATORY** — you MUST load the antipatterns checklist and run EVERY check below for EACH generated test. Do NOT output tests without completing this verification.

- Load `references/e2e-antipatterns-checklist.md`.
- Before outputting, verify each generated test against:
  - [ ] Not a False Prophet — has meaningful visible-outcome assertions
  - [ ] Not a Sleeper — no sleep()/hardcoded waits
  - [ ] Not a Chain Gang — tests are independent, run in any order
  - [ ] Not a Greedy Test — focused on one flow
  - [ ] Not a CSS Sniper — uses accessible selectors
  - [ ] Not a UI Typist — programmatic auth (except the one login test)
  - [ ] Not a Data Leaker — isolated test data
  - [ ] Not a Network Optimist — handles error scenarios
- If any check fails, fix the test before outputting.

### 6) Output format

```markdown
## E2E Test Generation Summary

**Scope**: X features/flows analyzed
**Classification**: Critical journeys: A, Important flows: B, Skipped (nice-to-have): C
**Risk tags applied**: [list if any]
**Tests generated**: Z test cases across N flows

---

## Flow Classification

> This section is **required**. You MUST classify every analyzed flow before generating tests.

| Flow | Criticality x Complexity | Risk Tags | Strategy |
|------|--------------------------|-----------|----------|
| `Checkout flow` | Critical x Medium | PAYMENT, DATA_INTEGRITY | Full E2E — happy + sad paths |
| `User registration` | Critical x Low | AUTH, PII | Core CRUD tests |
| `Profile settings` | Important x Low | — | Happy path only |
| `Theme toggle` | Nice-to-have x Low | — | Skip E2E |

---

## Page Objects

### [PageName]Page

[Page Object code with high-level methods]

---

## Generated Tests

### [flow_name.e2e.ext]

[Test code with comments explaining the scenario for non-obvious cases]

---

## Coverage Notes

- **Tested (critical journeys)**: [List of critical flows covered]
- **Tested (important flows)**: [List of important flows — happy path]
- **Skipped (nice-to-have)**: [List with rationale]
- **Needs monitoring instead**: [High-complexity flows better served by monitoring]
- **Risk tags applied**: [Which risk tags overrode the matrix]

---

## Next Steps

1. **Run tests** — Shall I run the generated E2E tests?
2. **Add Page Objects** — Shall I extract more Page Objects for reuse?
3. **Add accessibility checks** — Shall I add axe-core checks to the flows?
4. **Add visual regression** — Shall I add screenshot comparison for key states?
```

**Important**: Do NOT write test files to disk until user explicitly confirms. Present the generated tests in the output first, then ask how to proceed.

---

## Mode 2: Review E2E Tests

### 1) Preflight context

- Collect existing E2E test files: from `git diff` or user-specified files.
- Identify E2E framework, configuration, and project conventions.
- Read the corresponding Page Objects and production code to understand what's being tested.
- Identify auth setup pattern in use.

**Edge cases:**
- **No E2E test files found**: Inform user and offer to switch to Mode 1 (Generate).
- **Mixed test and production code in diff**: Review tests (Mode 2), generate tests for uncovered flows (Mode 1).

### 2) Evaluate against checklist

**MANDATORY** — you MUST load ALL THREE reference files below. Do NOT evaluate from memory alone.

- Load `references/e2e-test-review-checklist.md` for the full checklist.
- Load `references/e2e-antipatterns-checklist.md` for antipattern detection.
- Load `references/waiting-and-synchronization.md` for wait strategy assessment.

**MANDATORY procedure** — follow this exact sequence for every review. Do NOT skip steps or merge findings:
1. **Scan for False Prophets** (P0) — meaningful visible-outcome assertions?
2. **Check flakiness risk** (P0) — non-deterministic elements? race conditions?
3. **Check test isolation** (P0) — shared test data? order dependency?
4. **Check wait strategy** (P1) — sleep()? hardcoded delays? proper auto-waiting?
5. **Check selectors** (P1) — CSS/XPath? accessible selectors?
6. **Check flow coverage** (P1) — critical journeys tested? happy + sad paths?
7. **Check abstraction** (P1) — Page Objects used? code duplication?
8. **Check test data** (P1) — API-based setup? proper isolation? cleanup?
9. **Check readability** (P2) — names describe user journeys? AAA structure?
10. **Check auth efficiency** (P1) — programmatic login? storage state reuse?

### 3) Assess severity

Assign severity to each finding:
- **P0**: False confidence (no real assertion), flaky tests, shared mutable test data
- **P1**: sleep()/hardcoded waits, CSS/XPath selectors, UI login everywhere, test order dependency, missing critical flows, no Page Objects
- **P2**: Readability, naming, poor organization, environment-specific code
- **P3**: Style, minor suggestions

### 4) Output format

```markdown
## E2E Test Review Summary

**Files reviewed**: X test files, Y test cases
**Overall assessment**: [APPROVE / REQUEST_CHANGES / COMMENT]
**Flakiness risk**: [Low / Medium / High]
**Flow coverage**: [Good / Adequate / Insufficient]

---

## Findings

### P0 - Critical
(none or list)

### P1 - High
1. **[test_file:line]** Brief title
   - **Antipattern**: Which antipattern
   - **Problem**: What's wrong
   - **Impact**: Why it matters (flakiness, false confidence, slow CI)
   - **Fix**: Suggested correction with code example

### P2 - Medium
2. (continue numbering)

### P3 - Low
...

---

## Flakiness Risk Assessment

- [Sources of non-determinism identified]
- [Wait strategy issues]
- [Data isolation concerns]

## Coverage Gaps

- [Critical user journeys not covered by any E2E test]

## Positive Observations

- [What's done well — reinforce good practices]

---

## Next Steps

I found X issues (P0: _, P1: _, P2: _, P3: _).

**How would you like to proceed?**

1. **Fix all** — I'll implement all suggested fixes
2. **Fix P0/P1 only** — Address critical and high priority issues
3. **Fix specific items** — Tell me which issues to fix
4. **No changes** — Review complete, no implementation needed

Please choose an option or provide specific instructions.
```

**Important**: Do NOT implement any changes until user explicitly confirms. This is a review-first workflow.

---

## Resources

### references/

| File | Purpose |
|------|---------|
| `e2e-testing-principles.md` | Four pillars for E2E, FIRST adapted, test pyramid, E2E Flow Classification Matrix |
| `e2e-test-design-patterns.md` | Page Object Model, Screenplay, AAA for E2E, selectors, auth strategy |
| `user-flow-testing.md` | Critical paths, happy/sad, multi-step workflows, CRUD lifecycle |
| `test-data-management.md` | API-based setup, factories, isolation, cleanup |
| `network-and-api-handling.md` | Network mocking, request interception, MSW, error simulation |
| `waiting-and-synchronization.md` | No sleep(), auto-waiting, explicit waits, retry, timeouts |
| `accessibility-in-e2e.md` | axe-core, WCAG, keyboard nav, focus management |
| `e2e-antipatterns-checklist.md` | 12 antipatterns with examples, detection, and fixes |
| `e2e-test-review-checklist.md` | Structured review checklist for E2E tests by severity |
