# Test Data Management

## The Golden Rule: Test Data Isolation

Every E2E test must own its data lifecycle completely:

| Principle | Meaning |
|-----------|---------|
| **Each test creates its own data** | No reliance on pre-existing state or other tests' output |
| **Each test cleans up its own data** | Teardown is mandatory, not optional |
| **Tests NEVER share mutable data** | Read-only reference data is OK; writable data is not |
| **Tests can run in parallel without conflicts** | UUID-based uniqueness guarantees this |

Violating any of these principles produces order-dependent, flaky tests that fail unpredictably in CI.

---

## API-Based Setup (Preferred)

Create test data via API calls in beforeEach/setup, not through the UI.

Why:
- 10-50x faster than clicking through forms
- More reliable than database seeding (uses real validation)
- Bypasses UI interactions (which are tested separately)
- Closer to how production systems create data

```
# BAD — creating data through UI in every test
setup():
    navigate("/signup")
    fill("name", "Test User")
    fill("email", "test@example.com")
    click("Register")
    wait_for_url("/dashboard")

# GOOD — API-based setup
setup():
    user = api.post("/users", {name: "Test User", email: unique_email()})
    product = api.post("/products", {name: "Widget", price: 29.99})
    return {user, product}

test_user_can_purchase(setup_data):
    login_as(setup_data.user)
    navigate("/products")
    # ... test the actual purchase flow with pre-created data
```

---

## Database Seeding

| Use For | Do NOT Use For |
|---------|----------------|
| Initial schema setup | Per-test mutable data |
| Reference/lookup data (countries, categories, roles) | User accounts that tests will modify |
| Large read-only datasets for performance tests | Any data that a test writes to or deletes |

Rules:
- Seed scripts must be idempotent (running twice produces the same result)
- Keep seeds minimal -- only what the application requires to boot
- Version seed scripts alongside application migrations
- Never seed test-specific data -- that belongs in test setup

```
# GOOD — idempotent seed for reference data
seed_reference_data():
    for category in ["Electronics", "Clothing", "Books"]:
        api.post("/categories", {name: category}, ignore_conflict=true)

# BAD — seed that creates test-specific data
seed():
    api.post("/users", {name: "TestAdmin", role: "admin"})  # Will conflict on second run
```

---

## Factories / Builder Pattern

Factories produce unique, valid test entities with sensible defaults and per-test overrides.

```
# Test data factory
class UserFactory:
    @staticmethod
    def create(**overrides):
        defaults = {
            name: "Test User " + uuid(),
            email: uuid() + "@test.com",
            role: "user",
            active: true
        }
        return api.post("/users", {**defaults, **overrides})

# Usage — override only what matters for the test
test_admin_dashboard():
    admin = UserFactory.create(role="admin")
    regular = UserFactory.create(role="user")
    login_as(admin)
    navigate("/admin/users")
    assert page.text(regular.name)

test_inactive_user_blocked():
    user = UserFactory.create(active=false)
    login_as(user)
    assert page.url == "/account-suspended"
```

Benefits:
- Tests declare intent (what differs), not boilerplate (what's the same)
- Defaults stay in one place -- update once, all tests benefit
- UUID-based fields guarantee uniqueness across parallel runs

---

## Data Isolation Strategies

| Strategy | How | Best For | Trade-off |
|----------|-----|----------|-----------|
| **UUID-based** | Unique identifiers in every created entity | Default -- always use | Minimal overhead, strong isolation |
| **Transaction rollback** | Wrap each test in a DB transaction, rollback after | Fast CI runs with direct DB access | Requires DB connection from test runner |
| **Dedicated DB per worker** | Each parallel worker gets its own DB instance | CI with heavy parallelism | Higher resource cost, setup complexity |
| **Container-per-suite** | Spin up a fresh DB container per test suite | Maximum isolation, clean-room runs | Slowest startup, highest resource cost |

**Recommendation**: UUID-based as the universal default. Add transaction rollback for speed in CI when DB access is available.

---

## Cleanup Strategies

```
# GOOD — targeted API cleanup in afterEach
teardown(setup_data):
    api.delete("/users/" + setup_data.user.id)
    api.delete("/products/" + setup_data.product.id)

# GOOD — bulk cleanup by test run prefix
teardown():
    api.delete("/users?prefix=test-" + test_run_id)

# BAD — no cleanup at all
teardown():
    pass  # "Someone else will clean it up" — no, they won't
```

Rules:
- Always clean up in afterEach/teardown, even if the test fails
- Tag test data with a test-run ID for bulk identification
- Cleanup must be idempotent (don't fail if data was already deleted)
- Order of deletion matters -- delete dependent entities first (orders before users)
- Consider: truncate test tables between suites (not between individual tests)

---

## Sensitive Data Rules

| Rule | Rationale |
|------|-----------|
| NEVER hardcode real credentials in test files | Source code is not a secret store |
| NEVER use production data or real user accounts | Legal risk, data corruption risk |
| Use sandbox/test environments only | Isolation from real systems |
| Store test credentials in env vars or secure vaults | Rotation, access control, audit |
| Create dedicated test accounts with limited permissions | Principle of least privilege |

```
# BAD — hardcoded real credentials
test_login():
    login("admin@company.com", "RealP@ssw0rd!")

# BAD — credentials in test file, even if "test" ones
test_login():
    login("test@test.com", "Test123!")

# GOOD — environment variables
test_login():
    login(env.TEST_USER_EMAIL, env.TEST_USER_PASSWORD)
```

---

## Large Data Sets

When tests require hundreds or thousands of records (pagination, search, filtering):

- Use factories with loops for bulk creation
- Create bulk data in globalSetup (once per suite), clean in globalTeardown
- Test pagination with exact, known record counts
- Test search/filtering with controlled, predictable datasets

```
# GOOD — controlled dataset for pagination test
global_setup():
    test_run = uuid()
    for i in range(55):
        ProductFactory.create(name=f"Product-{test_run}-{i}")
    return {test_run, total: 55}

test_pagination_shows_correct_pages(global_data):
    navigate("/products?search=" + global_data.test_run)
    assert page.text("Page 1 of 3")  # 55 items / 20 per page = 3 pages
    click("Next")
    assert page.text("Page 2 of 3")

global_teardown(global_data):
    api.delete("/products?prefix=Product-" + global_data.test_run)
```

- Keep bulk datasets tagged with the test run ID for reliable cleanup
- Avoid creating bulk data in beforeEach -- it's too slow per test
- If bulk creation takes >10s, consider DB seeding for that specific suite
