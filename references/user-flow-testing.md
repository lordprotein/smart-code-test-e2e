# User Flow Testing

## Identifying Critical Paths

Three heuristics for finding which flows deserve E2E coverage:

### 1. Follow the money

Any flow where money changes hands or financial state changes:
- Checkout and payment
- Subscription creation, upgrade, downgrade, cancellation
- Refund and credit processing
- Invoice generation
- Free trial to paid conversion

### 2. Follow the user

Core user lifecycle and retention flows:
- Registration and onboarding
- Login and session management
- Core feature usage (the "job to be done")
- Profile management
- Password reset and account recovery

### 3. Follow the data

Flows where data is created, transformed, or destroyed:
- Record creation (especially without undo)
- Bulk operations (import, export, mass update)
- Data deletion (especially irreversible)
- File upload and processing
- Cross-system data synchronization

**Apply all three heuristics**, then deduplicate. The union of these lists is your candidate set for E2E coverage.

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

### Test the complete submission, not field-by-field validation

```
# BAD -- testing individual field validation (this is a unit test concern)
test_email_field_shows_error_for_invalid_email():
    form.fill_email("not-an-email")
    form.tab_to_next_field()
    assert form.email_error == "Invalid email"

# GOOD -- test the complete form submission journey
test_registration_with_invalid_data_shows_errors():
    reg_page = RegistrationPage(page)
    reg_page.fill_form(name="", email="bad", password="123")
    reg_page.submit()

    assert reg_page.has_errors()
    assert reg_page.is_on_registration_page()  # Did not navigate away
```

### E2E form testing strategy

| Test | What to verify |
|------|---------------|
| **Happy path** | Fill all fields correctly -> submit -> verify success |
| **All required missing** | Submit empty -> verify error state shown |
| **Fix and resubmit** | Submit invalid -> see errors -> fix -> resubmit -> verify success |
| **Long form preservation** | Fill many fields -> fail on one -> other fields preserved |

**Field-level validation** (specific format, min/max length, regex) belongs in unit tests. E2E verifies that the form works as a whole.

---

## CRUD Lifecycle Testing

Test the full lifecycle of a resource in a single test:

```
test_product_crud_lifecycle():
    products = ProductsPage(page)

    # Create
    new_product = products.create(name="Widget", price=29.99)
    assert new_product.name == "Widget"
    assert new_product.price == "$29.99"

    # Read (verify in list)
    products.navigate_to_list()
    assert products.has_product("Widget")

    # Update
    edit_page = products.open_product("Widget")
    edit_page.update_price(39.99)
    edit_page.save()
    assert edit_page.price == "$39.99"

    # Delete
    edit_page.delete()
    products.navigate_to_list()
    assert not products.has_product("Widget")
```

### CRUD test guidelines

| Guideline | Rationale |
|-----------|-----------|
| **Lifecycle as one test** | Tests the full resource journey, catches state transition bugs |
| **OR separate per operation** | When operations are independently complex. Create via API for Read/Update/Delete tests |
| **Verify visible results** | After create: item appears in list. After delete: item gone from list |
| **Test both detail and list views** | Create on form, verify on list. Edit on detail, verify on list |

---

## Search and Filter Flows

```
test_product_search_returns_relevant_results():
    # Arrange -- create known data
    api.create_product(name="Blue Widget")
    api.create_product(name="Red Widget")
    api.create_product(name="Green Gadget")

    # Act -- search
    products = ProductsPage(page)
    products.search("Widget")

    # Assert -- only relevant results
    assert products.result_count == 2
    assert products.has_product("Blue Widget")
    assert products.has_product("Red Widget")
    assert not products.has_product("Green Gadget")
```

### What to cover

| Scenario | What to verify |
|----------|---------------|
| **Search with results** | Results are relevant, count is correct |
| **Search with no results** | Empty state message shown, not a blank page |
| **Filter application** | Results match filter criteria |
| **Filter combination** | Multiple filters narrow results correctly (AND/OR) |
| **Clear filters** | All results return after clearing |
| **Search + filter together** | Both applied simultaneously |

---

## Empty States

Test what the user sees when there is no data:

```
test_new_user_sees_empty_order_history():
    # Fresh user with no orders
    login_as(new_user)
    orders = navigate_to("/orders")

    assert orders.is_empty()
    assert orders.empty_message == "You have no orders yet"
    assert orders.has_cta("Browse Products")  # Helpful next action
```

### Common empty states to test
- New account: no orders, no saved items, no notifications
- Empty cart: message + CTA to browse
- No search results: helpful message + suggestions
- Empty dashboard: onboarding prompts or getting-started guide
- Empty admin list: "No users match your filters" + clear filters CTA

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

---

## Flow Mapping Template

Before writing tests, map the flow:

```
Flow: Checkout
  Entry point: Cart page with items
  Steps:
    1. Cart review -> click "Checkout"
    2. Shipping address form -> fill and continue
    3. Payment method -> fill and continue
    4. Order review -> place order
  Success state: Order confirmation page
  Error states:
    - Payment declined -> error message, stay on payment step
    - Out of stock (during checkout) -> error, link to cart
    - Session expired -> redirect to login, preserve cart
  Prerequisite data:
    - User account (programmatic auth)
    - Product in cart (API setup)
  Risk tags: PAYMENT
```

Use this template to plan tests before writing them. The map directly translates into test scenarios.
