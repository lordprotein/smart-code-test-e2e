# E2E Antipatterns Checklist

12 antipatterns ordered by severity. Each entry: description, signals, BAD/GOOD pseudo-code, fix, severity.

---

## 1. The Sleeper (🟠 High)

**What:** Tests use `sleep()`, `wait(ms)`, or hardcoded delays instead of waiting for conditions.

**Signals:** `sleep(3000)`, `wait(5000)`, `cy.wait(2000)`, `page.waitForTimeout(1000)`, any numeric wait.

```
# BAD — The Sleeper
test_order_submission():
    page.click("Submit")
    sleep(3000)
    assert page.text("Order confirmed")

# GOOD — wait for condition
test_order_submission():
    page.click("Submit")
    wait_for_text("Order confirmed")
```

**Fix:** Replace every `sleep()` with wait-for-condition: text appeared, element visible, URL changed, or network response received.

**Why it matters:** Hardcoded waits are either too short (flaky) or too long (slow). Conditions are both faster and reliable.

---

## 2. The Ice Cream Cone (🟠 High)

**What:** Too many E2E tests, too few unit/integration tests. The test pyramid is inverted.

**Signals:** E2E suite > 30 min, >15% of total tests are E2E, simple validation logic tested via browser, developers avoid running E2E locally.

```
# BAD — testing validation logic in E2E
test_email_validation():
    page.fill("email", "invalid")
    page.click("submit")
    assert page.text("Invalid email")            # This should be a unit test!

# GOOD — unit test for validation, E2E for the flow
# Unit: test_email_validator_rejects_invalid_format()
# E2E:  test_registration_happy_path()           # Tests the complete flow
```

**Fix:** Move logic testing to unit tests. E2E tests user journeys, not individual validations. Ask: "Does this test require a browser?" If no, it is not an E2E test.

---

## 3. The Chain Gang (🔴 Critical / 🟠 High)

**What:** Tests depend on execution order or data created by previous tests.

**Signals:** Tests fail when run individually but pass in sequence. Test B uses data created by Test A. Suite fails if one test in the middle fails.

```
# BAD — tests are chained
test_1_create_product():
    create_product("Widget")

test_2_edit_product():
    # Depends on test_1 having created "Widget"!
    edit_product("Widget", price=39.99)

test_3_delete_product():
    # Depends on test_1 and test_2!
    delete_product("Widget")

# GOOD — each test is independent
test_edit_product():
    product = api.create_product("Widget", price=29.99)    # Own setup
    navigate_to(product.url)
    edit_product(price=39.99)
    assert product_price() == "$39.99"
```

**Fix:** Each test creates its own data in setup and cleans up in teardown. Tests must pass when run alone, in reverse order, or in parallel.

**Severity note:** 🔴 Critical when tests completely break if run out of order. 🟠 High when there are soft/implicit dependencies.

---

## 4. The Greedy Test (🟠 High)

**What:** One test verifies too many unrelated things. Covers multiple flows in a single test function.

**Signals:** 20+ assertions, multiple unrelated user flows, test name is vague ("test_everything", "test_app"), test takes >2 minutes alone.

```
# BAD — The Greedy Test
test_application():
    # Tests registration AND login AND profile AND settings AND logout
    register(user)
    login(user)
    update_profile(user)
    change_settings(user)
    logout()
    # 15 assertions across unrelated flows

# GOOD — focused on one flow
test_user_can_update_profile():
    login_as(user)
    navigate_to("/profile")
    update_name("New Name")
    assert page.has_text("New Name")
```

**Fix:** One user flow per test. If the test name needs "and", it is testing too much. Split into separate tests with shared setup.

---

## 5. The CSS Sniper (🟠 High)

**What:** Tests use fragile CSS selectors, XPath, or implementation-coupled locators.

**Signals:** `.btn-primary`, `#submit-form`, `div > span:nth-child(2)`, `//div[@class="header"]`, any selector that breaks when CSS classes or DOM structure change.

```
# BAD — fragile selectors
page.click(".btn-primary.submit-order")
page.fill("#email-input-field", "test@test.com")
page.click("div.modal > div.footer > button:last-child")

# GOOD — accessible selectors
page.click(role="button", name="Submit order")
page.fill(label="Email", value="test@test.com")
page.click(role="button", name="Confirm")
```

**Fix:** Use selector priority:

| Priority | Selector Type | Example                                |
|----------|---------------|----------------------------------------|
| 1        | Role          | `role="button", name="Submit"`         |
| 2        | Label         | `label="Email"`                        |
| 3        | Text          | `text="Submit order"`                  |
| 4        | Placeholder   | `placeholder="Enter email"`            |
| 5        | test-id       | `data-testid="submit-btn"`             |
| NEVER    | CSS/XPath     | `.btn-primary`, `//div[@class="..."]`  |

---

## 6. The UI Typist (🟠 High)

**What:** Every test logs in through the UI, wasting time on repeated form filling.

**Signals:** Every test file starts with `fill("username"), fill("password"), click("login")`. Login takes 3-5 seconds per test. 50 tests = 3+ minutes just logging in.

```
# BAD — UI login in every test
test_dashboard():
    page.fill("username", "admin")
    page.fill("password", "pass")
    page.click("Login")
    wait_for_url("/dashboard")
    # Now test actually begins...

# GOOD — programmatic auth
test_dashboard():
    login_via_api(admin_user)                    # <100ms, no UI
    navigate_to("/dashboard")
    # Test begins immediately
```

**Fix:** One dedicated test for the UI login flow. All other tests use programmatic authentication (API call, cookie injection, storage state reuse).

---

## 7. The Retry Masker (🟠 High)

**What:** Automatic retries hide real flakiness without fixing the root cause.

**Signals:** `retries: 3` in config, test passes on 2nd/3rd attempt regularly, team says "just re-run it", CI dashboard shows retry patterns.

```
# BAD — masking flakiness with retries
config:
    retries: 3                                   # "It usually passes on the second try"

# GOOD — fix the root cause
# Instead of retrying, investigate WHY the test is flaky:
# - Missing wait?     -> Add proper wait-for-condition
# - Race condition?   -> Wait for network response
# - Shared data?      -> Isolate test data
# - Unstable selector? -> Use accessible selector
```

**Fix:** Retries are a band-aid. Investigate and fix the root cause. If retries are used in CI, log which tests needed them and track flakiness rate. A test that needs retries is a broken test.

---

## 8. The Data Leaker (🔴 Critical)

**What:** Test data leaks between tests, causing non-deterministic results.

**Signals:** Tests fail randomly, pass individually but fail in suite, results change based on test order, "works on my machine" syndrome.

```
# BAD — shared mutable data
global_user = create_user()                      # Created once, used by all tests

test_1():
    global_user.update(name="Changed")           # Mutates shared data!

test_2():
    assert global_user.name == "Original"        # FAILS — test_1 changed it!

# GOOD — isolated data
test_1():
    user = create_user()                         # Own data
    user.update(name="Changed")
    assert user.name == "Changed"

test_2():
    user = create_user()                         # Own data
    assert user.name == "Original"
```

**Fix:** Each test creates and owns its data. No shared mutable state. Clean up in afterEach/teardown. Use unique identifiers (UUID suffix) to prevent collision.

---

## 9. The False Prophet (🔴 Critical)

**What:** Test passes but does not actually verify the intended outcome. Gives false confidence.

**Signals:** No assertions on visible outcome, asserting on setup data, asserting on mock return values, test body has no `assert` at all (just actions).

```
# BAD — no real assertion
test_order_placement():
    fill_checkout_form()
    click_submit()
    # Test "passes" because nothing throws... but did the order actually go through?

# BAD — asserting on what you set up
test_order_placement():
    mock_api("/orders", response={status: "created"})
    click_submit()
    assert response.status == "created"          # Asserting on your own mock!

# GOOD — assert on visible user outcome
test_order_placement():
    fill_checkout_form()
    click_submit()
    assert page.has_text("Order confirmed")
    assert page.has_text("Order #")
    assert page.url.includes("/confirmation")
```

**Fix:** Every test must assert on a user-visible outcome: text displayed, page navigated, element state changed, download started. If you remove the feature, the test must fail.

---

## 10. The Environment Prisoner (🟡 Medium)

**What:** Test only works in one specific environment. Hardcoded values tie it to localhost or a specific machine.

**Signals:** Hardcoded URLs (`http://localhost:3000`), hardcoded ports, environment-specific test data, works on local but fails in CI.

```
# BAD — hardcoded environment
test_homepage():
    navigate_to("http://localhost:3000")
    page.fill("email", "admin@mylocal.dev")

# GOOD — configurable
test_homepage():
    navigate_to(BASE_URL)                        # Set via environment variable
    page.fill("email", TEST_ADMIN_EMAIL)         # From config
```

**Fix:** Use environment variables for URLs, credentials, and configuration. Tests should work in local, CI, and staging without code changes.

---

## 11. The Pixel Watcher (🟡 Medium)

**What:** Using visual regression screenshots as functional assertions. Any CSS change breaks functional tests.

**Signals:** Functional tests rely on screenshot comparison, font rendering differences cause failures, functional tests break after CSS refactor with no behavior change.

```
# BAD — screenshot as functional assertion
test_checkout_works():
    fill_form()
    click_submit()
    compare_screenshot("checkout-success.png")   # Fails if font changes!

# GOOD — assert on behavior, not pixels
test_checkout_works():
    fill_form()
    click_submit()
    assert page.has_text("Order confirmed")
```

**Fix:** Use visual regression for dedicated visual tests (separate suite). Use text/element/state assertions for functional tests. These are two different concerns.

---

## 12. The Network Optimist (🟠 High)

**What:** Tests only cover the happy path for network interactions. No error or timeout scenarios tested.

**Signals:** Only sunny-day API tests, no error simulation, no timeout handling, app breaks on 500 but no test catches it.

```
# BAD — only tests the sunny day
test_payment():
    fill_payment_form()
    click_pay()
    assert page.has_text("Payment successful")
    # What happens when payment fails? Timeout? 500 error?

# GOOD — tests failure scenarios too
test_payment_api_failure():
    mock_api("/payment", status=500)
    fill_payment_form()
    click_pay()
    assert page.has_text("Payment failed. Please try again.")
    assert payment_form.is_still_visible()       # User can retry

test_payment_timeout():
    mock_api("/payment", delay=30000)            # Simulate timeout
    fill_payment_form()
    click_pay()
    assert page.has_text("Request timed out")
```

**Fix:** For every critical external API integration, test at least: success, server error (500), and timeout. Users will encounter these scenarios.

---

## Quick Reference Detection Table

| Antipattern              | Key Signal                           | Severity        |
|--------------------------|--------------------------------------|-----------------|
| The Sleeper              | `sleep()` / `wait(ms)` in test       | 🟠 High         |
| The Ice Cream Cone       | >15% E2E, simple logic in E2E       | 🟠 High         |
| The Chain Gang           | Tests depend on execution order      | 🔴 Critical / 🟠 High |
| The Greedy Test          | 20+ assertions, vague name           | 🟠 High         |
| The CSS Sniper           | CSS/XPath selectors                  | 🟠 High         |
| The UI Typist            | UI login in every test               | 🟠 High         |
| The Retry Masker         | `retries: 3` hiding flakiness        | 🟠 High         |
| The Data Leaker          | Shared mutable test data             | 🔴 Critical     |
| The False Prophet        | No meaningful assertion              | 🔴 Critical     |
| The Environment Prisoner | Hardcoded URLs/ports                 | 🟡 Medium       |
| The Pixel Watcher        | Screenshot for functional test       | 🟡 Medium       |
| The Network Optimist     | No error/timeout scenarios           | 🟠 High         |
