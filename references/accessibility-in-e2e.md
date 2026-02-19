# Accessibility in E2E Tests

## Why E2E is Natural for Accessibility Testing

- E2E tests interact with the real rendered DOM — the same DOM assistive technologies use
- Accessible selectors (role, label, text) ARE the default selector strategy in modern E2E frameworks
- A11y checks catch real issues in the real browser context, not in synthetic/unit environments
- If your E2E test can't find an element by role/label, your app has an a11y issue
- Accessibility and testability are two sides of the same coin: well-structured accessible UI is easy to test

---

## axe-core Integration

axe-core is the industry standard for automated accessibility testing. It can be injected into any E2E test regardless of framework.

**What it does:**
- Checks the current page against WCAG rules automatically
- Returns violations with impact level, description, and remediation guidance
- Covers ~57% of WCAG 2.1 rules (the automatable subset)

```
# Add a11y check at key points in any E2E test
test_checkout_is_accessible():
    navigate_to("/checkout")
    results = run_axe()                          # Run axe-core on current page
    assert results.violations.count == 0

    # Check again after significant UI change
    fill_form(shipping_info)
    results = run_axe()
    assert results.violations.count == 0
```

```
# Scope axe-core to a specific region
test_modal_is_accessible():
    open_modal("Confirm order")
    results = run_axe(scope="[role='dialog']")   # Only check the modal
    assert results.violations.count == 0
```

---

## WCAG Levels

| Level    | Description                        | Typical Requirement                              |
|----------|------------------------------------|--------------------------------------------------|
| **A**    | Minimum — basic accessibility      | Legal minimum in most jurisdictions               |
| **AA**   | Standard — good accessibility      | Most organizations target this                    |
| **AAA**  | Enhanced — best accessibility      | Specialized contexts (government, healthcare)     |

Default to **AA** for most projects. Configure axe-core to check the appropriate level:

```
# Configure axe-core for AA
results = run_axe(
    run_only=["wcag2a", "wcag2aa"]               # Skip AAA rules
)
```

---

## What to Check

### 1. Keyboard Navigation

- Tab order is logical
- All interactive elements are keyboard-accessible
- Focus is visible (focus ring/outline)
- No keyboard traps (user can always tab away)

```
# GOOD — test keyboard navigation
test_form_keyboard_navigation():
    navigate_to("/contact")
    page.keyboard.press("Tab")
    assert focused_element() == "Name input"
    page.keyboard.press("Tab")
    assert focused_element() == "Email input"
    page.keyboard.press("Tab")
    assert focused_element() == "Submit button"
    page.keyboard.press("Enter")
    assert page.has_text("Message sent")
```

### 2. Focus Management

- Focus moves to modal when opened
- Focus returns to trigger element when modal closed
- Focus moves to error message on validation failure
- Focus trapped inside modal (cannot tab to background content)

```
# GOOD — test focus management in modal
test_modal_focus_trap():
    page.click(role="button", name="Open settings")
    assert focused_element().role == "dialog"

    # Tab through all focusable elements in modal
    page.keyboard.press("Tab")                   # First element
    page.keyboard.press("Tab")                   # Second element
    page.keyboard.press("Tab")                   # Should wrap back, not escape

    assert focused_element().closest("[role='dialog']") is not None

    page.keyboard.press("Escape")
    assert focused_element().name == "Open settings"    # Focus returned
```

### 3. Screen Reader Compatibility

- All images have alt text (or are marked decorative)
- Form inputs have associated labels
- Headings are hierarchical (h1 -> h2 -> h3, no skipping)
- ARIA roles are correct and necessary
- Live regions announce dynamic content

### 4. Color Contrast

- Text meets minimum contrast ratio: 4.5:1 for normal text, 3:1 for large text
- axe-core checks this automatically
- Non-text content (icons, borders) meets 3:1

### 5. Form Labels

- Every input has an associated label
- Error messages are linked to inputs (aria-describedby)
- Required fields are indicated (not just by color)
- Group labels exist for related inputs (fieldset/legend)

---

## When to Add A11y Checks in E2E Flow

Run axe-core at these points — not on every single action:

| When                                | Why                                        |
|-------------------------------------|--------------------------------------------|
| After initial page load             | Catch baseline violations                  |
| After modal/dialog opens            | Modals often have focus/ARIA issues        |
| After form validation (error state) | Error messages need ARIA links             |
| After dynamic content loads         | Injected content may lack semantics        |
| After significant UI state changes  | Toggled panels, expanded sections          |

---

## A11y-Specific Assertions

```
# Check element has correct role
assert element.role == "button"

# Check element has accessible name
assert element.accessible_name == "Submit order"

# Check element is in tab order
assert element.is_focusable()

# Check ARIA attributes
assert element.aria_expanded == "true"
assert element.aria_label == "Close menu"

# Check live region exists for dynamic updates
assert page.locator("[aria-live='polite']").is_visible()
```

---

## Common A11y Problems Found in E2E

| Problem                                | Impact                                     | Detection Method            |
|----------------------------------------|--------------------------------------------|-----------------------------|
| Missing alt text on images             | Screen readers can't describe images       | axe-core rule: image-alt    |
| No form labels                         | Screen readers can't identify inputs       | axe-core rule: label        |
| Low color contrast                     | Text unreadable for low-vision users       | axe-core rule: color-contrast |
| Missing skip-to-content link           | Keyboard users must tab through everything | Manual check in E2E         |
| Focus not managed in modals            | Keyboard users lost in background          | Test focus after modal open |
| Non-descriptive link text ("click here")| Screen readers can't convey purpose       | axe-core rule: link-name    |
| Missing heading hierarchy             | Navigation confusing for screen readers    | axe-core rule: heading-order |
| Auto-playing media                     | Disruptive for all users                   | Manual check in E2E         |

---

## Integrating A11y into Existing E2E Tests

You do not need a separate a11y test suite. Add axe-core checks into existing flow tests:

```
# BAD — separate, disconnected a11y test
test_checkout_a11y():
    navigate_to("/checkout")
    results = run_axe()
    assert results.violations.count == 0
    # Doesn't test any real flow — just a static page scan

# GOOD — a11y check embedded in flow test
test_checkout_flow():
    navigate_to("/checkout")
    results = run_axe()
    assert results.violations.count == 0

    fill_shipping_form(address)
    click_continue()
    results = run_axe()                          # Check after state change
    assert results.violations.count == 0

    fill_payment_form(card)
    click_place_order()
    assert page.has_text("Order confirmed")
    results = run_axe()                          # Check final state
    assert results.violations.count == 0
```

This way, a11y coverage grows with your E2E suite automatically.
