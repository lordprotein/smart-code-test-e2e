# E2E Test Design Patterns

## Page Object Model (POM)

Encapsulate page interactions behind high-level methods. A Page Object represents a page or significant component of the UI.

### Core rules

| Rule | Rationale |
|------|-----------|
| **Methods describe user actions** | `add_to_cart()` not `click_button()` |
| **Properties expose page state** | `cart.total`, `confirmation.success_message` |
| **Never expose selectors** | Selectors stay inside the Page Object, never leak to tests |
| **Methods return Page Objects** | Navigation returns the next Page Object: `cart.checkout()` returns `CheckoutPage` |
| **Waits are encapsulated** | Page Object methods handle waiting internally, tests never wait explicitly |
| **No assertions in Page Objects** | Page Objects expose state, tests make assertions (see note below) |

### Example

```
# BAD -- raw selectors in test
test_checkout():
    page.click("[data-testid='add-to-cart']")
    page.click(".cart-icon")
    page.fill("#email", "test@test.com")
    page.click("button.submit")
    assert page.text_content(".success-msg") == "Order placed"

# GOOD -- Page Object Model
test_checkout():
    product_page = ProductPage(page)
    cart_page = product_page.add_to_cart()
    checkout_page = cart_page.proceed_to_checkout()
    confirmation = checkout_page.complete_order(email="test@test.com")
    assert confirmation.success_message == "Order placed"
```

### Page Object structure

```
class ProductPage:
    def __init__(page):
        self.page = page

    # --- Actions (return self or next Page Object) ---

    def add_to_cart() -> CartPage:
        self.page.get_by_role("button", name="Add to cart").click()
        return CartPage(self.page)

    def search(query) -> ProductPage:
        self.page.get_by_role("searchbox").fill(query)
        self.page.get_by_role("button", name="Search").press("Enter")
        return self

    # --- State (expose visible information) ---

    @property
    def product_name() -> str:
        return self.page.get_by_role("heading", level=1).text_content()

    @property
    def price() -> str:
        return self.page.get_by_test_id("product-price").text_content()

    @property
    def is_in_stock() -> bool:
        return self.page.get_by_text("In Stock").is_visible()
```

### Assertions in Page Objects -- the debate

| Approach | When to use |
|----------|-------------|
| **No assertions in POM** (recommended) | Tests own all assertions. POM only exposes state. Clearer separation of concerns |
| **Assertions in POM** (alternative) | Convenience methods like `has_error_message()` returning bool. Acceptable for complex visibility checks |

**Recommendation**: Keep assertions in tests. Page Objects return data, tests verify it.

---

## Screenplay Pattern

Actor-centric pattern for complex multi-page workflows:

```
# Actor has Abilities, performs Tasks, checks Questions
actor = Actor.named("Customer")
    .who_can(BrowseTheWeb.using(browser))
    .who_can(CallAPI.at(base_url))

actor.attempts_to(
    Navigate.to("/products"),
    AddToCart.product("Widget"),
    Checkout.with_shipping("123 Main St"),
    Pay.with_card("4111111111111111")
)

actor.should_see(
    that(OrderConfirmation.message(), equals("Order placed")),
    that(OrderConfirmation.total(), equals("$29.99"))
)
```

### When to use Screenplay vs POM

| Criteria | POM | Screenplay |
|----------|-----|------------|
| Team size | Small-medium | Large, multiple teams |
| Flow complexity | Simple-medium page flows | Multi-actor, multi-system |
| Learning curve | Low | Higher |
| Composition | Inheritance-based | Composition-based |
| Multi-role testing | Awkward (multiple POM instances) | Natural (multiple actors) |

**Default choice**: POM for most projects. Screenplay when you need multi-actor scenarios or highly composable test steps.

---

## AAA Pattern Adapted for E2E

```
test_user_can_place_order():
    # Arrange -- set up data via API, authenticate, navigate
    product = api.create_product(name="Widget", price=29.99)
    login_as(user)
    navigate_to(product.url)

    # Act -- user interactions through Page Objects
    product_page = ProductPage(page)
    cart = product_page.add_to_cart()
    checkout = cart.proceed_to_checkout()
    confirmation = checkout.place_order()

    # Assert -- visible outcome
    assert confirmation.has_success_message()
    assert confirmation.order_total == "$29.99"
```

### E2E-specific AAA rules

| Phase | Unit test | E2E test |
|-------|-----------|----------|
| **Arrange** | Create objects in memory | API calls to seed data, programmatic auth, navigate to starting page |
| **Act** | Call one method | Multi-step user interactions through Page Objects |
| **Assert** | Verify return value / state | Verify visible outcomes: text, navigation, element state |

**Key difference**: In E2E, the Act phase often involves multiple user interactions forming a complete journey. This is acceptable -- the "one act" in E2E means "one user journey," not "one click."

---

## Given-When-Then for E2E

```
test_returning_customer_sees_saved_address():
    # Given a customer with a saved shipping address
    customer = api.create_customer(address="123 Main St")
    login_as(customer)

    # When they proceed to checkout
    cart = CartPage(page)
    checkout = cart.proceed_to_checkout()

    # Then their saved address is pre-filled
    assert checkout.shipping_address == "123 Main St"
```

Use Given-When-Then when tests serve as living documentation for stakeholders. Functionally equivalent to AAA.

---

## Selector Priority

Use the most stable, user-centric selector available:

| Priority | Selector type | Example | Stability |
|----------|--------------|---------|-----------|
| 1 | **Role** | `get_by_role("button", name="Submit")` | Best -- accessible, semantic |
| 2 | **Label** | `get_by_label("Email address")` | Great -- tied to visible label |
| 3 | **Text** | `get_by_text("Add to cart")` | Great -- visible to user |
| 4 | **Placeholder** | `get_by_placeholder("Search...")` | Acceptable -- visible hint |
| 5 | **Test ID** | `get_by_test_id("checkout-btn")` | Stable -- but invisible to user |
| 6 | **NEVER use** | `.cart-icon`, `#submit`, `//div[3]/span` | Fragile -- breaks on any refactor |

### Why this order matters

```
# BAD -- CSS selector: breaks when class name changes
page.click(".btn-primary.submit-form")

# BAD -- XPath: breaks when DOM structure changes
page.click("//div[@class='form']/button[2]")

# GOOD -- role selector: stable, accessible, user-centric
page.get_by_role("button", name="Place order").click()
```

- Role/label/text selectors reflect the real user perspective
- CSS/XPath selectors couple tests to implementation (DOM structure, class names)
- `data-testid` is stable but does not verify accessibility -- use as last resort
- If you cannot select an element by role or label, the UI likely has an accessibility issue

---

## Test Naming Convention

### Describe the user journey and outcome

```
# GOOD -- describes who, what, outcome
test_user_completes_checkout_with_credit_card
test_admin_can_delete_inactive_user
test_guest_is_redirected_to_login_on_checkout
test_user_sees_error_when_payment_fails

# BAD -- describes technical actions
test_click_submit_button_then_redirect
test_fill_form_and_check_response
test_api_returns_200
```

### Naming rules

| Rule | Example |
|------|---------|
| **Include user role if relevant** | `test_admin_...`, `test_guest_...` |
| **Describe the journey, not the clicks** | `test_user_registers` not `test_fill_name_email_password_click` |
| **Include the outcome** | `test_user_sees_confirmation_after_order` |
| **Use domain language** | `test_subscriber_upgrades_plan` not `test_click_upgrade_button` |

---

## Test Organization

### Organize by feature/flow, not by page

```
# GOOD -- organized by user flow
e2e/
  auth/
    login.e2e.ts
    registration.e2e.ts
    password-reset.e2e.ts
  checkout/
    checkout-happy-path.e2e.ts
    checkout-errors.e2e.ts
  products/
    product-search.e2e.ts
    product-crud.e2e.ts
  settings/
    profile.e2e.ts
    notifications.e2e.ts

# BAD -- organized by page (mirrors UI structure)
e2e/
  header.e2e.ts
  homepage.e2e.ts
  sidebar.e2e.ts
  modal.e2e.ts
```

### File guidelines

| Guideline | Rationale |
|-----------|-----------|
| One file per user flow or feature area | Easy to find, run selectively |
| Page Objects in separate directory | `e2e/pages/` or `e2e/support/pages/` |
| Shared fixtures in dedicated directory | `e2e/fixtures/` for test data, auth states |
| Helpers for common operations | `e2e/helpers/` for API shortcuts, data factories |

---

## Authentication in E2E Tests

### The "One Login Test" Rule

Write exactly ONE test that verifies the UI login flow:

```
test_user_can_login_via_ui():
    login_page = LoginPage(page)
    login_page.enter_email("user@test.com")
    login_page.enter_password("password123")
    dashboard = login_page.submit()

    assert dashboard.welcome_message == "Welcome back!"
    assert dashboard.is_logged_in()
```

**ALL other tests**: use programmatic authentication. Never go through the login UI.

### Programmatic login

```
# Pattern: API login -> inject token -> skip UI
setup():
    response = api.post("/auth/login", {email, password})
    token = response.token
    storage.set("auth_token", token)
    # OR: set cookie directly
    page.context.add_cookies([{name: "session", value: token}])
```

### Storage state reuse

```
# Save auth state once (in global setup)
authenticated_state = login_via_api(user)
save_storage_state("auth-state.json")

# Reuse in all tests (in test setup)
test_setup():
    load_storage_state("auth-state.json")
    # User is already logged in -- no login needed
```

### Multi-role fixtures

```
# Create fixtures per role
admin_state = login_via_api(admin_user)
regular_state = login_via_api(regular_user)
guest_state = empty_state()

# Use in tests
test_admin_sees_user_management(use_auth=admin_state):
    dashboard = DashboardPage(page)
    assert dashboard.has_menu_item("User Management")

test_user_cannot_see_user_management(use_auth=regular_state):
    dashboard = DashboardPage(page)
    assert not dashboard.has_menu_item("User Management")
```

### Why programmatic auth matters

| Approach | Time per test | 50 tests | Flakiness risk |
|----------|--------------|----------|----------------|
| UI login every test | 2-5 seconds | 100-250 seconds wasted | High -- login UI can break |
| Programmatic auth | < 100ms | < 5 seconds total | Low -- API is stable |
| Storage state reuse | ~0ms | ~0 seconds | Lowest -- no auth call at all |

---

## Data Setup Patterns

### API-first setup

```
# BAD -- creating data through UI is slow and fragile
test_user_can_edit_product():
    # 15 clicks to create a product through UI...
    products_page.click_new()
    products_page.fill_name("Widget")
    products_page.fill_price("29.99")
    products_page.save()
    # NOW test the actual edit...

# GOOD -- create via API, test only the edit journey
test_user_can_edit_product():
    product = api.create_product(name="Widget", price=29.99)
    navigate_to(product.edit_url)

    edit_page = ProductEditPage(page)
    edit_page.update_price("39.99")
    edit_page.save()

    assert edit_page.success_message == "Product updated"
```

### Data isolation

| Strategy | How | When |
|----------|-----|------|
| **Create per test** | Each test creates its own data via API | Default -- safest |
| **Unique identifiers** | Append timestamp/UUID to names | When testing search, lists |
| **Cleanup after test** | Delete created data in teardown | When data persists across runs |
| **Ephemeral environment** | Fresh database per test run | CI pipeline, most reliable |
