# E2E Test Review Checklist

Structured checklist for reviewing E2E tests. Use the Quick-Start Review Sequence at the bottom for the recommended order.

---

## 1. Four Pillars Assessment for E2E

> Apply Four Pillars from `e2e-testing-principles.md`. Key checks per pillar:

- [ ] **Confidence**: Critical journeys covered, happy + sad paths, risk-tagged flows have E2E
- [ ] **Resistance to Refactoring**: Accessible selectors, user-visible outcomes, Page Objects encapsulate details
- [ ] **Speed**: No sleep(), programmatic auth, API-based setup, parallel-ready, suite < 15 min
- [ ] **Determinism**: 100% pass rate on reruns, proper waits, isolated data, no external service dependency

---

## 2. Antipattern Detection

### 🔴 Critical — Must fix immediately

- [ ] **The False Prophet** — tests without meaningful visible-outcome assertions
- [ ] **The Data Leaker** — shared mutable test data across tests
- [ ] **The Chain Gang (severe)** — tests completely dependent on execution order

### 🟠 High — Should fix before merge

- [ ] **The Sleeper** — `sleep()` / hardcoded waits
- [ ] **The Chain Gang (moderate)** — soft dependencies between tests
- [ ] **The CSS Sniper** — CSS/XPath selectors instead of accessible ones
- [ ] **The UI Typist** — UI login repeated in every test
- [ ] **The Retry Masker** — retries hiding real flakiness
- [ ] **The Greedy Test** — one test covers too many unrelated flows
- [ ] **The Network Optimist** — no error or timeout scenario tests
- [ ] **The Ice Cream Cone** — too many E2E tests, too few unit tests

### 🟡 Medium — Fix in this PR or follow-up

- [ ] **The Environment Prisoner** — hardcoded environment values
- [ ] **The Pixel Watcher** — screenshot comparison for functional assertions

---

## 3. Flow Coverage Quality

- [ ] Critical flows (revenue, auth, data) have E2E tests
- [ ] Happy paths covered for all critical and important flows
- [ ] Primary sad paths covered for critical flows
- [ ] Risk-tagged features (PAYMENT, AUTH, PII) have coverage
- [ ] Empty states and edge conditions considered
- [ ] Multi-step workflows tested end-to-end (not just individual steps)

---

## 4. Wait Strategy Assessment

- [ ] No `sleep()` / `wait(ms)` / hardcoded delays
- [ ] Auto-waiting used where sufficient (framework default)
- [ ] Explicit wait-for-condition where auto-waiting is not enough
- [ ] Wait for network response before network-dependent assertions
- [ ] Animations disabled or properly waited for
- [ ] Timeouts are reasonable (not too short causing flakes, not too long masking issues)

---

## 5. Selector Strategy Assessment

- [ ] Accessible selectors preferred (role, label, text)
- [ ] `test-id` used only when accessible selectors are not possible
- [ ] No CSS class selectors
- [ ] No XPath selectors
- [ ] No positional selectors (`nth-child`, `first`, `last`)
- [ ] Selectors encapsulated in Page Objects (not scattered in test bodies)

---

## 6. Auth Efficiency

- [ ] One dedicated UI login test exists (verifies the login flow itself)
- [ ] All other tests use programmatic auth (API, cookie, token injection)
- [ ] Storage state reused where possible (login once, share session)
- [ ] Multi-role tests use role-based fixtures
- [ ] Auth setup is fast (<1s per test)

---

## 7. Test Data Management

- [ ] API-based data setup (not clicking through UI to create data)
- [ ] Each test owns its data (created in setup, cleaned in teardown)
- [ ] Unique identifiers (UUID suffix) prevent data collision in parallel runs
- [ ] Cleanup in afterEach/teardown (not relying on test order)
- [ ] No hardcoded credentials in test code
- [ ] Tests run against sandbox/test environments only

---

## 8. Test Isolation

- [ ] Tests can run in any order and produce the same result
- [ ] Tests can run in parallel without interference
- [ ] No shared mutable state between tests
- [ ] No dependency on data created by a previous test
- [ ] Database/application state clean between tests

---

## 9. CI Readiness

- [ ] Tests work in headless mode
- [ ] No hardcoded URLs or ports (use environment variables)
- [ ] Environment variables used for all configuration
- [ ] Reasonable timeouts for CI (CI is slower than local)
- [ ] Retry policy documented and justified (not hiding flakiness)
- [ ] Screenshots/traces captured on failure for debugging

---

## 10. Naming and Readability

- [ ] Test names describe user journeys in plain language
- [ ] No technical jargon in test names (user perspective, not developer perspective)
- [ ] AAA structure (Arrange, Act, Assert) is clear in each test
- [ ] Page Objects used for UI abstraction (selectors and actions encapsulated)
- [ ] Comments explain non-obvious scenarios or workarounds
- [ ] Test files organized by feature or flow (not by page or component)

---

## Quick-Start Review Sequence

When reviewing E2E tests, follow this exact order. Stop and report as soon as 🔴 Critical issues are found.

| Step | Check                    | Severity     | What to look for                                   |
|------|--------------------------|--------------|-----------------------------------------------------|
| 1    | False Prophets           | 🔴 Critical  | Meaningful visible-outcome assertions?              |
| 2    | Flakiness                | 🔴 Critical  | Non-deterministic elements? Race conditions? Shared data? |
| 3    | Isolation                | 🔴 Critical  | Tests independent? Own data? No order dependency?   |
| 4    | Waits                    | 🟠 High      | `sleep()`? Proper wait-for-condition strategy?      |
| 5    | Selectors                | 🟠 High      | CSS/XPath? Accessible selectors used?               |
| 6    | Coverage                 | 🟠 High      | Critical user journeys tested?                      |
| 7    | Abstraction              | 🟠 High      | Page Objects? Code duplication?                     |
| 8    | Data management          | 🟠 High      | API setup? Cleanup? Data isolation?                 |
| 9    | Readability              | 🟡 Medium    | Clear names? AAA structure? Comments?               |
| 10   | Auth efficiency          | 🟠 High      | Programmatic auth? Storage state reuse?             |
