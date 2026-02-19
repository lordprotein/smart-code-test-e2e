# Waiting and Synchronization

## The Cardinal Sin: sleep()

`sleep()` / `wait(ms)` / hardcoded delays are ALWAYS wrong in E2E tests. Too short = flaky, too long = slow, no correct value exists.

```
# BAD
page.click("submit")
sleep(3000)
assert page.text("Success")

# GOOD — wait for condition
page.click("submit")
wait_for_text("Success")

# GOOD — wait for network
page.click("submit")
wait_for_response("/api/order")
assert page.text("Success")
```

If you write `sleep(N)` where N > 0, the test is wrong.

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

**When auto-waiting is NOT sufficient** (explicit waits required): specific API response, animations, multi-step state transitions, third-party scripts, loading indicators.

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
wait_for_text("Order confirmed")
wait_for_selector("[data-testid='order-summary']", state="visible")
wait_for_url("/order/confirmation")
wait_for_response(url="/api/orders", status=200)
wait_for_selector(".loading-spinner", state="hidden")
wait_for(() => page.locator(".item").count() >= 10)
```

---

## Retry Strategies

Framework-level assertion retry is the correct approach. Manual retry loops with sleep are not.

```
# BAD — manual retry with sleep
for i in range(3):
    try: assert page.text("Success")
    except: sleep(1000)

# GOOD — let the framework handle retry
expect(page.locator(".result")).to_have_text("Success")  # Auto-retries until timeout
expect(page.locator(".report")).to_have_text("Generated", timeout=15000)  # Custom timeout
```

Rules:
- Trust framework auto-retry for assertions
- Increase timeout for genuinely slow operations, do not add manual retries
- If a test needs manual retry logic, the test design is wrong

---

## Network Idle Detection

`networkidle` waits until no requests are in flight. Use sparingly — **not** with analytics, WebSocket, or polling.

| Situation | Use networkidle? |
|-----------|-----------------|
| Analytics pings / WebSocket / polling | **No** — never idle |
| Known finite API calls | **No** — wait for specific response |
| Unknown parallel fetches / initial page load | **Yes** — last resort |

Prefer `wait_for_response("/api/specific-endpoint")` over `networkidle`.

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

When animations cannot be disabled: use `wait_for_selector(".modal", state="stable")` or `wait_for(() => element.computed_style("animation-name") == "none")`.

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
| Wait for element >10s | Wrong selector | Verify selector targets correct element |
| Wait for response >10s | Slow API or wrong URL pattern | Check API performance; verify URL match |
| Network idle never resolves | Polling/WebSocket/analytics | Switch to specific response wait |
| Full test >30s | Multiple slow waits | Profile each wait; fix slowest first |

Ask: is this a test problem or an app performance problem? If the app is slow for users too, file a performance bug.
