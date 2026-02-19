# Network and API Handling

## When to Mock at Network Level

E2E tests exercise the full stack. Mock only what you do not own or cannot control.

| Dependency | Mock? | Rationale |
|------------|-------|-----------|
| Own backend API | **NO** | E2E means testing the full stack including your server |
| External payment (Stripe, PayPal) | **YES** | Cannot charge real cards; unreliable in CI |
| External email (SendGrid, SES) | **YES** | Side effects; cannot verify delivery in test |
| External auth (OAuth, SAML) | **YES** | Cannot control external provider state |
| CDN / static assets | **NO** | Part of the real user experience |
| Analytics (Segment, GA) | **YES or ignore** | Side effect, irrelevant to flow correctness |
| External maps / geocoding | **YES** | Rate-limited, unreliable, costly |
| Internal microservices (your team) | **NO** | Still your system -- test the real integration |

**Rule: Mock external third-party services. NEVER mock your own backend.**

---

## Request Interception Pattern

Intercept outgoing requests and return controlled responses. Use for any external dependency that cannot be called in tests.

```
# GOOD — mock an external payment API
intercept("POST", "https://api.stripe.com/charges", {
    status: 200,
    body: {id: "ch_test_123", status: "succeeded"}
})

# Test proceeds with deterministic payment result
checkout.fill_card_details(test_card)
checkout.place_order()
assert confirmation.is_visible()
assert confirmation.text("Order confirmed")
```

```
# GOOD — mock with dynamic response based on request
intercept("POST", "https://api.stripe.com/charges", (request) => {
    amount = request.body.amount
    if amount > 100000:
        return {status: 402, body: {error: "Amount exceeds limit"}}
    return {status: 200, body: {id: "ch_test", status: "succeeded"}}
})
```

```
# BAD — mocking your own API
intercept("POST", "/api/orders", {status: 200, body: {id: "1"}})  # Defeats the purpose of E2E
```

---

## Mock Service Worker (MSW) Pattern

MSW intercepts at the network level, transparent to the application code. Useful when the same mocking strategy is needed across unit, integration, and E2E tests.

```
# Define handlers for external services
handlers = [
    rest.post("https://api.stripe.com/*", (req, res, ctx) => {
        return res(ctx.json({id: "ch_test", status: "succeeded"}))
    }),
    rest.get("https://api.maps.com/geocode", (req, res, ctx) => {
        return res(ctx.json({results: [mock_address]}))
    }),
    rest.post("https://api.sendgrid.com/send", (req, res, ctx) => {
        return res(ctx.status(202))
    })
]
```

Benefits:
- Application sees real-looking network responses
- Handlers are reusable across test types
- Easy to override per-test for error scenarios
- Does not require modifying application code

---

## Waiting for Network Responses

**CRITICAL**: Never assert immediately after an action that triggers a network call. The response may not have arrived yet.

```
# BAD — race condition
page.click("submit")
assert page.text("Order confirmed")  # API might not have responded yet

# GOOD — wait for the specific network response
page.click("submit")
wait_for_response("/api/orders", status=200)
assert page.text("Order confirmed")

# ALSO GOOD — wait for a visible outcome (implies network completed)
page.click("submit")
wait_for_text("Order confirmed")  # Framework auto-waits for text to appear
```

Choosing between strategies:

| Strategy | When to Use |
|----------|-------------|
| **Wait for response** | When you need to verify the API was called or check response data |
| **Wait for visible outcome** | When you only care that the UI updated correctly |
| **Wait for URL change** | After form submissions that redirect |

---

## Clock Control

For time-dependent features: countdown timers, session expiry, scheduled actions, date-based logic.

```
# Test session expiry
set_clock(now)
login_as(user)
advance_clock(31, "minutes")  # Past 30-min session timeout
navigate("/dashboard")
assert page.url == "/login"  # Redirected to login

# Test countdown timer
set_clock("2025-12-31T23:59:50Z")
navigate("/new-year-countdown")
assert page.text("10 seconds")
advance_clock(5, "seconds")
assert page.text("5 seconds")
```

Rules:
- Never depend on real wall-clock time in tests
- Use the framework's clock/time API to freeze or advance time
- Reset the clock in teardown to avoid leaking into other tests
- Clock control is essential for testing: expiry, scheduling, rate limiting, animations

---

## Error Simulation

Test how the application handles failures from external services. Users will encounter these errors -- the UI must handle them gracefully.

```
# Simulate server error (500)
intercept("POST", "https://api.stripe.com/charges", {status: 500})
checkout.place_order()
assert checkout.error_message == "Payment failed. Please try again."

# Simulate timeout
intercept("POST", "https://api.stripe.com/charges", {delay: 30000})
checkout.place_order()
assert checkout.error_message == "Request timed out"

# Simulate network error (connection refused)
intercept("POST", "https://api.stripe.com/charges", {error: "net::ERR_CONNECTION_REFUSED"})
checkout.place_order()
assert checkout.error_message == "Unable to connect. Check your internet."

# Simulate slow but successful response (test loading state)
intercept("POST", "https://api.stripe.com/charges", {
    delay: 5000,
    status: 200,
    body: {id: "ch_test", status: "succeeded"}
})
checkout.place_order()
assert checkout.loading_spinner.is_visible()  # Shows loading during delay
wait_for_response("https://api.stripe.com/charges")
assert confirmation.is_visible()  # Eventually succeeds
```

Test matrix for external dependencies:

| Scenario | What to Verify |
|----------|---------------|
| **Success (200)** | Happy path works end-to-end |
| **Client error (4xx)** | Meaningful error message shown, form state preserved |
| **Server error (5xx)** | Retry prompt or graceful error, no data corruption |
| **Timeout** | Loading state shown, timeout message, option to retry |
| **Network failure** | Offline-friendly message, no unhandled exceptions |

---

## GraphQL Specifics

GraphQL sends all requests to the same URL. Intercept by operation name, not by endpoint.

```
# Mock a specific mutation
intercept_graphql("CreateOrder", {
    data: {createOrder: {id: "order_123", status: "created"}}
})

# Mock a specific query
intercept_graphql("GetUserProfile", {
    data: {user: {id: "1", name: "Test User", email: "test@test.com"}}
})

# Mock an error response
intercept_graphql("CreateOrder", {
    errors: [{message: "Insufficient inventory", code: "OUT_OF_STOCK"}]
})
```

- Match on `operationName` in the request body
- Test loading states, error states, and partial data responses
- Test optimistic updates by delaying the mock response

---

## WebSocket Specifics

WebSocket connections require separate handling from HTTP requests.

```
# Test real-time notification
mock_ws = create_mock_websocket("/ws/notifications")
login_as(user)
navigate("/dashboard")

# Simulate incoming message
mock_ws.send({type: "notification", text: "New order received"})
assert page.text("New order received")

# Test reconnection
mock_ws.close()
wait_for_text("Connection lost")
mock_ws.reopen()
wait_for_text_gone("Connection lost")
```

Test scenarios for WebSocket:
- Connection established successfully
- Message received and UI updates
- Connection lost -- reconnection attempt shown
- Reconnection succeeds -- state resynchronized
- Server sends malformed message -- no crash
