# Waiting and Synchronization

## The Cardinal Sin: sleep()

`sleep()` / `wait(ms)` / hardcoded delays are ALWAYS wrong in E2E tests.

| Problem | Consequence |
|---------|-------------|
| Value too short | Test is flaky -- fails on slow CI, passes locally |
| Value too long | Test suite is unnecessarily slow |
| No correct value exists | Timing is non-deterministic across environments |

```
# BAD — the sleeper
page.click("submit")
sleep(3000)  # Hope the API responds in 3 seconds
assert page.text("Success")

# BAD — still a sleeper, just dressed up
page.click("submit")
wait(5000)
assert page.text("Success")

# GOOD — wait for condition
page.click("submit")
wait_for_text("Success")  # Returns as soon as text appears, up to timeout

# GOOD — wait for network
page.click("submit")
wait_for_response("/api/order")
assert page.text("Success")
```

The only acceptable use of a timed wait is `wait_for_time(0)` to yield the event loop in rare edge cases. If you write `sleep(N)` where N > 0, the test is wrong.

---

## Auto-Waiting

Modern E2E frameworks have built-in auto-waiting on actions and assertions. This covers ~80% of synchronization needs automatically.

**When auto-waiting is sufficient** (no explicit waits needed):

| Action | Framework Behavior |
|--------|-------------------|
| Clicking a button | Waits for element to be visible, enabled, and stable |
| Typing in an input | Waits for element to be editable |
| Asserting text is visible | Auto-retries until text appears or timeout |
| Asserting element count | Auto-retries until count matches or timeout |
| Selecting a dropdown option | Waits for select to be interactive |

```
# These all auto-wait — no explicit synchronization needed
page.click("[data-testid='submit']")        # Waits for button to be actionable
page.fill("[data-testid='email']", "a@b.c") # Waits for input to be editable
expect(locator(".result")).to_have_text("OK") # Retries until text matches
```

**When auto-waiting is NOT sufficient** (explicit waits required):

- Waiting for a specific API response before asserting derived state
- Waiting for animations or transitions to complete
- Waiting for complex multi-step state transitions
- Waiting for third-party scripts to load and initialize
- Waiting for a loading indicator to disappear before interacting

---

## Explicit Wait-for-Condition

When auto-waiting is not enough, use explicit condition-based waits.

| Wait For | When to Use | Example |
|----------|-------------|---------|
| **Text visible** | After action that displays a message | `wait_for_text("Order confirmed")` |
| **Element visible** | After action that reveals content | `wait_for_selector(".modal", state="visible")` |
| **Element hidden** | After action that hides content (modal close, loading done) | `wait_for_selector(".spinner", state="hidden")` |
| **URL change** | After navigation (form submit, redirect) | `wait_for_url("/confirmation")` |
| **Network response** | After action triggering a specific API call | `wait_for_response("/api/orders")` |
| **Network idle** | After action triggering multiple unknown API calls | `wait_for_load_state("networkidle")` |
| **Function/predicate** | Complex conditions not covered above | `wait_for(() => get_count(".items") > 5)` |

```
# Wait for text
wait_for_text("Order confirmed")

# Wait for element to appear
wait_for_selector("[data-testid='order-summary']", state="visible")

# Wait for URL change
wait_for_url("/order/confirmation")

# Wait for specific network response
wait_for_response(url="/api/orders", status=200)

# Wait for loading to finish
wait_for_selector(".loading-spinner", state="hidden")

# Wait for custom condition
wait_for(() => page.locator(".item").count() >= 10)
```

---

## Retry Strategies

Framework-level assertion retry is the correct approach. Manual retry loops with sleep are not.

```
# BAD — manual retry with sleep
for i in range(3):
    try:
        assert page.text("Success")
        break
    except:
        sleep(1000)

# BAD — retry with increasing delay
attempt = 0
while attempt < 5:
    if page.has_text("Success"):
        break
    sleep(attempt * 500)
    attempt += 1

# GOOD — let the framework handle retry
expect(page.locator(".result")).to_have_text("Success")  # Auto-retries until timeout

# GOOD — custom timeout for slow operations
expect(page.locator(".report")).to_have_text("Generated", timeout=15000)
```

Rules:
- Trust framework auto-retry for assertions
- Increase timeout for genuinely slow operations, do not add manual retries
- If a test needs manual retry logic, the test design is wrong

---

## Network Idle Detection

`wait_for_load_state("networkidle")` waits until no network requests are in flight for a period.

Use sparingly:

| Situation | Use networkidle? |
|-----------|-----------------|
| Page with analytics pings | **No** -- analytics will keep firing |
| Page with WebSocket/polling | **No** -- never idle |
| Page with known finite API calls | **No** -- wait for the specific response |
| Unknown page with many parallel fetches | **Yes** -- as last resort |
| Initial page load in smoke test | **Yes** -- acceptable |

```
# BAD — using networkidle when a specific wait is possible
page.click("submit")
wait_for_load_state("networkidle")  # Unreliable, may wait for analytics
assert page.text("Success")

# GOOD — wait for the response that matters
page.click("submit")
wait_for_response("/api/orders")
assert page.text("Success")
```

---

## Animation and Transition Waits

Animations cause flakiness when assertions fire mid-transition.

**Best approach**: Disable animations in the test environment entirely.

```
# Disable CSS animations and transitions for deterministic tests
inject_css(`
    *, *::before, *::after {
        animation-duration: 0s !important;
        animation-delay: 0s !important;
        transition-duration: 0s !important;
        transition-delay: 0s !important;
    }
`)
```

When animations cannot be disabled (e.g., testing animation itself):

```
# Wait for element to reach stable position
wait_for_selector(".modal", state="stable")

# Wait for CSS animation to end
wait_for(() => element.computed_style("animation-name") == "none")

# Wait for specific visual state
wait_for_selector(".fade-in", state="visible")
```

---

## Timeout Tuning

| Scope | Default | When to Adjust |
|-------|---------|----------------|
| **Global default** | 30s | Lower if app is fast (5-10s), raise for slow CI (45s) |
| **Per-action** | 5-10s | Raise for: file upload, PDF generation, report building |
| **Per-assertion** | 5s | Raise for: eventual consistency, async processing |
| **Navigation** | 30s | Adjust for slow page loads, heavy SSR |

```
# GOOD — raise timeout for a known slow operation
page.click("generate-report")
wait_for_response("/api/reports", timeout=30000)  # Report generation is slow
assert page.text("Report ready")

# BAD — raising global timeout to cover one slow test
config.default_timeout = 60000  # Now ALL tests wait up to 60s — suite is slow
```

Rules:
- Start with framework defaults -- they are tuned for common cases
- Only increase timeouts with a documented reason (comment in code)
- Never decrease below 5s for CI environments
- If a test needs >60s timeout, either the feature is too slow or the test is wrong
- Per-test timeout overrides are better than global increases

---

## Debugging Slow Waits

When a wait consistently takes >10s, treat it as a signal, not normal behavior.

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Wait for element takes >10s | Wrong selector, element never appears | Verify selector targets the right element |
| Wait for response takes >10s | Slow API, wrong URL pattern in wait | Check API performance; verify URL match |
| Wait for text takes >10s | Text rendered differently than expected | Log actual text; check for whitespace/casing |
| Network idle never resolves | Polling, WebSocket, analytics | Switch to specific response wait |
| Full test takes >30s | Multiple slow waits compounding | Profile each wait; fix the slowest first |

```
# Debugging: add trace to identify which wait is slow
console.time("wait-for-submit-response")
wait_for_response("/api/orders")
console.timeEnd("wait-for-submit-response")  # Shows actual wait duration

console.time("wait-for-confirmation-text")
wait_for_text("Order confirmed")
console.timeEnd("wait-for-confirmation-text")
```

Ask: is this a test problem or an application performance problem? If the real application is slow for users too, file a performance bug rather than increasing the test timeout.
