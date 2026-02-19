# User Flow Testing

## Identifying Critical Paths

Three heuristics — apply all, then deduplicate:

| Heuristic | What to cover |
|-----------|--------------|
| **Follow the money** | Checkout, payment, subscriptions, refunds, trial-to-paid conversion |
| **Follow the user** | Registration, login, core feature usage, password reset, account recovery |
| **Follow the data** | Record creation (no undo), bulk ops, irreversible deletion, file upload, cross-system sync |

The union of these lists is your candidate set for E2E coverage.

---

## Happy Path -- Always First

The primary successful journey through a flow. Every critical flow MUST have at least one happy path E2E test before any sad path or edge case.

```
test_user_registers_successfully():
    # Complete registration journey: start to finish
    reg_page = navigate_to("/register")
    reg_page.fill_form(name="John", email="john@test.com", password="Str0ng!Pass")
    dashboard = reg_page.submit()

    assert dashboard.welcome_message == "Welcome, John!"
    assert dashboard.is_logged_in()
```

### Happy path rules

| Rule | Rationale |
|------|-----------|
| **Cover the complete flow** | Start at the entry point, end at the success state |
| **Use realistic data** | Not "test123" -- use data that resembles production |
| **Verify the final outcome** | Assert on what the user sees at the end, not intermediate states |
| **One happy path per critical flow** | Mandatory minimum -- this is the test you write first |

---

## Sad Path Prioritization

Not all error scenarios deserve E2E coverage. Prioritize by impact:

### Priority 1: Data loss or corruption
- Payment failure mid-transaction (money charged, order not created)
- Form data lost on navigation error
- Partial save corrupts existing data

```
test_payment_failure_does_not_create_order():
    checkout = CheckoutPage(page)
    checkout.fill_shipping(address="123 Main St")
    checkout.fill_payment(card="4000000000000002")  # Decline card
    checkout.place_order()

    assert checkout.error_message == "Payment declined"
    # Verify no order was created
    orders_page = navigate_to("/orders")
    assert orders_page.order_count == 0
```

### Priority 2: User blocked
- Cannot proceed in critical flow (stuck state)
- Account locked after failed attempts
- Required action impossible due to bug

### Priority 3: High probability errors
- Invalid form input (most common user error)
- Expired session during long flow
- Network timeout on submission

### For each sad path, verify three things
1. The error is shown to the user clearly
2. No data is corrupted or lost
3. The user can recover (retry, go back, correct input)

---

## Multi-Step Workflows

Wizard and stepper flows need specific test coverage:

### Forward progression

```
test_checkout_wizard_completes_all_steps():
    checkout = CheckoutPage(page)

    # Step 1: Shipping
    checkout.fill_shipping(address="123 Main St")
    checkout.next_step()
    assert checkout.current_step == "Payment"

    # Step 2: Payment
    checkout.fill_payment(card="4111111111111111")
    checkout.next_step()
    assert checkout.current_step == "Review"

    # Step 3: Review and confirm
    confirmation = checkout.place_order()
    assert confirmation.success_message == "Order placed!"
```

### Back navigation and state preservation

```
test_checkout_preserves_state_on_back_navigation():
    checkout = CheckoutPage(page)
    checkout.fill_shipping(address="123 Main St")
    checkout.next_step()
    checkout.go_back()

    # State must be preserved
    assert checkout.shipping_address == "123 Main St"
```

### What to test in multi-step flows

| Scenario | What to verify |
|----------|---------------|
| **Complete forward path** | All steps complete, final state is correct |
| **Back navigation** | Previous step data is preserved |
| **Resume after leaving** | If flow supports it: return to page, state is restored |
| **Direct URL to middle step** | Redirect to step 1 or show error (depends on design) |
| **Skip step (if allowed)** | Optional steps can be skipped, required steps cannot |

---

## Cross-Page State Transitions

Test that state changes on one page are reflected on another:

```
test_adding_item_updates_cart_badge():
    product_page = ProductPage(page)
    assert product_page.cart_badge_count == 0

    product_page.add_to_cart()
    assert product_page.cart_badge_count == 1

    # Navigate to different page -- badge persists
    home = navigate_to("/")
    assert home.cart_badge_count == 1
```

### Common cross-page transitions

| Action on Page A | Expected on Page B |
|-----------------|-------------------|
| Add item to cart | Cart page shows item, header badge updates |
| Change language in settings | All pages render in new language |
| Update profile name | Header shows new name on all pages |
| Mark notification as read | Notification count decreases everywhere |
| Create new record | List page includes the new record |

---

## Forms in E2E

Test complete form submission, not field-by-field validation. Field-level validation belongs in unit tests.

| Test | What to verify |
|------|---------------|
| **Happy path** | Fill all fields -> submit -> verify success |
| **All required missing** | Submit empty -> verify error state shown |
| **Fix and resubmit** | Submit invalid -> see errors -> fix -> resubmit -> success |
| **Long form preservation** | Fill many fields -> fail on one -> other fields preserved |

---

## CRUD Lifecycle Testing

Test full lifecycle in one test (Create -> Read -> Update -> Delete) or separate per operation when each is independently complex. Always create via API for Read/Update/Delete tests. Verify visible results: item appears in list after create, gone after delete.

```
test_product_crud_lifecycle():
    products = ProductsPage(page)
    new_product = products.create(name="Widget", price=29.99)
    assert new_product.name == "Widget"

    products.navigate_to_list()
    assert products.has_product("Widget")

    edit_page = products.open_product("Widget")
    edit_page.update_price(39.99)
    edit_page.save()
    assert edit_page.price == "$39.99"

    edit_page.delete()
    products.navigate_to_list()
    assert not products.has_product("Widget")
```

---

## Search and Filter Flows

| Scenario | What to verify |
|----------|---------------|
| **Search with results** | Results are relevant, count is correct |
| **Search with no results** | Empty state message shown |
| **Filter application** | Results match filter criteria |
| **Filter combination** | Multiple filters narrow correctly (AND/OR) |
| **Clear filters** | All results return after clearing |
| **Search + filter together** | Both applied simultaneously |

Setup: create known data via API, then search/filter against it. Assert on result count and specific items.

---

## Empty States

Test what users see with no data. Common empty states:
- New account: no orders, no saved items, no notifications
- Empty cart: message + CTA to browse
- No search results: helpful message + suggestions
- Empty dashboard: onboarding prompts
- Empty admin list: "No matches" + clear filters CTA

Verify: empty state message is shown, helpful CTA exists, page is not blank.

---

## Scenario Budget: 3-8 Per Flow

E2E tests are expensive. For each flow, allocate a fixed scenario budget:

| Scenario type | Count | Notes |
|---------------|-------|-------|
| **Happy path** | 1 | Mandatory. Always first |
| **Critical sad paths** | 1-3 | Data loss, user blocked, payment failure |
| **Edge cases** | 1-2 | Empty state, maximum data, boundary values |
| **State transitions** | 0-2 | Only for multi-step flows |
| **Total** | **3-8** | Hard maximum per flow |

### If you need more coverage

```
Flow: User Registration
  E2E (4 tests):
    1. Happy path: complete registration
    2. Sad path: duplicate email
    3. Sad path: submit with errors, fix, resubmit
    4. Edge: registration with all optional fields empty

  Push to unit/integration:
    - Email format validation (unit)
    - Password strength rules (unit)
    - Rate limiting (integration)
    - Email uniqueness check (integration)
    - CAPTCHA verification (integration)
```

**Rule**: If your E2E test count for a single flow exceeds 8, you are testing too much at the E2E level. Push detailed scenarios down to cheaper test types.

