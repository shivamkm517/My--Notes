---
tags: [python, testing, unittest, pytest, notes]
title: Unit Testing in Python
---

# Unit Testing in Python

A deep-dive reference on unit testing — concepts, the built-in `unittest` framework, `pytest`, mocking, fixtures, TDD, and best practices.

---

## Table of Contents

- [[#1. What is Unit Testing]]
- [[#2. The Testing Pyramid]]
- [[#3. Anatomy of a Good Unit Test (AAA Pattern)]]
- [[#4. The `unittest` Module]]
- [[#5. Assertion Methods in `unittest`]]
- [[#6. Test Fixtures (setUp / tearDown)]]
- [[#7. Skipping Tests & Expected Failures]]
- [[#8. Running `unittest` Tests]]
- [[#9. `pytest` Framework]]
- [[#10. `pytest` Fixtures]]
- [[#11. Parametrized Tests]]
- [[#12. Mocking & Patching]]
- [[#13. Testing Exceptions]]
- [[#14. Testing Async Code]]
- [[#15. Code Coverage]]
- [[#16. Test Organization & Naming Conventions]]
- [[#17. Test Doubles: Mocks, Stubs, Fakes, Spies]]
- [[#18. Test-Driven Development (TDD)]]
- [[#19. Common Pitfalls]]
- [[#20. Best Practices]]
- [[#21. Quick Reference Cheat Sheet]]

---

## 1. What is Unit Testing?

**Unit testing** is the practice of testing the smallest testable pieces of code — typically individual functions or methods — in **isolation** from the rest of the system, to verify they behave correctly.

### Why Unit Test?
- **Catches bugs early**, before they reach production.
- **Enables safe refactoring** — tests act as a safety net confirming behavior didn't break.
- **Serves as living documentation** — tests show how code is meant to be used.
- **Speeds up debugging** — a failing test narrows down exactly what broke and where.
- **Builds confidence** for continuous integration/deployment pipelines.

### Key Characteristics of a Good Unit Test
| Property | Meaning |
|---|---|
| **Fast** | Runs in milliseconds; you should be able to run thousands of tests in seconds |
| **Isolated** | Doesn't depend on other tests, external services, databases, or the network |
| **Repeatable** | Produces the same result every time, regardless of environment |
| **Self-validating** | Automatically reports pass/fail — no manual inspection needed |
| **Timely** | Written close to (ideally before/alongside) the code it tests |

---

## 2. The Testing Pyramid

A common mental model for how much of each test type to write:

```
        ▲
       /E2E\          Few — slow, expensive, test the whole system end-to-end
      /------\
     /  Integ. \      Some — test how components work together
    /------------\
   /   Unit Tests  \  Many — fast, cheap, test individual functions/classes in isolation
  /------------------\
```

| Test Type | Scope | Speed | Example |
|---|---|---|---|
| **Unit** | Single function/class, isolated (dependencies mocked) | Very fast | Test that `calculate_discount()` returns correct value |
| **Integration** | Multiple components working together | Medium | Test that the service correctly saves to a real (test) database |
| **End-to-End (E2E)** | Entire application, from user's perspective | Slow | Test that a user can sign up, log in, and place an order via the UI |

---

## 3. Anatomy of a Good Unit Test (AAA Pattern)

Most well-structured tests follow the **Arrange–Act–Assert** pattern:

```python
def test_addition():
    # Arrange — set up inputs and expected outcomes
    a, b = 2, 3
    expected = 5

    # Act — call the code under test
    result = a + b

    # Assert — verify the outcome
    assert result == expected
```

| Step | Purpose |
|---|---|
| **Arrange** | Set up test data, inputs, mocks, and any preconditions |
| **Act** | Execute the specific function/method being tested |
| **Assert** | Check that the actual result matches the expected result |

> [!tip]
> A unit test should ideally have **one clear reason to fail**. If a single test checks many unrelated things, a failure won't tell you precisely what went wrong — split it into multiple focused tests instead.

---

## 4. The `unittest` Module

`unittest` is Python's **built-in** testing framework (inspired by Java's JUnit), requiring no external installation.

### Basic Structure

**`calculator.py`**
```python
def add(a, b):
    return a + b

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b
```

**`test_calculator.py`**
```python
import unittest
from calculator import add, divide

class TestCalculator(unittest.TestCase):

    def test_add_positive_numbers(self):
        self.assertEqual(add(2, 3), 5)

    def test_add_negative_numbers(self):
        self.assertEqual(add(-1, -1), -2)

    def test_divide_by_zero_raises_error(self):
        with self.assertRaises(ValueError):
            divide(10, 0)

if __name__ == "__main__":
    unittest.main()
```

### Key Rules
- Test classes must inherit from `unittest.TestCase`.
- Test method names **must start with `test_`** — this is how `unittest` discovers them.
- Each test method should be independent and test one specific behavior.

---

## 5. Assertion Methods in `unittest`

| Method | Checks |
|---|---|
| `assertEqual(a, b)` | `a == b` |
| `assertNotEqual(a, b)` | `a != b` |
| `assertTrue(x)` | `bool(x) is True` |
| `assertFalse(x)` | `bool(x) is False` |
| `assertIs(a, b)` | `a is b` (identity) |
| `assertIsNone(x)` | `x is None` |
| `assertIsNotNone(x)` | `x is not None` |
| `assertIn(a, b)` | `a in b` |
| `assertNotIn(a, b)` | `a not in b` |
| `assertIsInstance(a, cls)` | `isinstance(a, cls)` |
| `assertRaises(Exception)` | Code inside block raises the given exception |
| `assertAlmostEqual(a, b)` | `a` and `b` are equal to a given number of decimal places (for floats) |
| `assertGreater(a, b)` / `assertLess(a, b)` | `a > b` / `a < b` |
| `assertListEqual(a, b)` | Two lists are equal (with a clearer diff on failure) |
| `assertDictEqual(a, b)` | Two dicts are equal (with a clearer diff on failure) |

```python
class TestExamples(unittest.TestCase):
    def test_various_assertions(self):
        self.assertEqual(2 + 2, 4)
        self.assertTrue(isinstance("hello", str))
        self.assertIn(3, [1, 2, 3])
        self.assertAlmostEqual(0.1 + 0.2, 0.3, places=7)

        with self.assertRaises(ZeroDivisionError):
            1 / 0
```

> [!tip]
> Prefer `assertAlmostEqual` over `assertEqual` for floating-point comparisons — due to floating-point precision, `0.1 + 0.2 == 0.3` is actually `False` in Python.

---

## 6. Test Fixtures (setUp / tearDown)

**Fixtures** are the fixed baseline state/environment a test needs before it runs (and cleans up afterward). `unittest.TestCase` provides hooks for this.

```python
class TestDatabaseOperations(unittest.TestCase):

    def setUp(self):
        """Runs before EACH test method."""
        self.db = create_test_database()
        self.db.add_user("Amit")

    def tearDown(self):
        """Runs after EACH test method, even if it failed."""
        self.db.close()

    def test_user_exists(self):
        self.assertTrue(self.db.user_exists("Amit"))

    def test_user_count(self):
        self.assertEqual(self.db.user_count(), 1)
```

### Class-Level Fixtures (Run Once for the Whole Class)

```python
class TestExpensiveSetup(unittest.TestCase):

    @classmethod
    def setUpClass(cls):
        """Runs ONCE before all tests in this class."""
        cls.shared_resource = load_expensive_resource()

    @classmethod
    def tearDownClass(cls):
        """Runs ONCE after all tests in this class."""
        cls.shared_resource.close()
```

| Method | Runs |
|---|---|
| `setUp` | Before every test method |
| `tearDown` | After every test method (even on failure) |
| `setUpClass` | Once, before any test in the class |
| `tearDownClass` | Once, after all tests in the class |

---

## 7. Skipping Tests & Expected Failures

```python
import unittest
import sys

class TestPlatformSpecific(unittest.TestCase):

    @unittest.skip("Not implemented yet")
    def test_future_feature(self):
        pass

    @unittest.skipIf(sys.platform == "win32", "Doesn't work on Windows")
    def test_unix_only_feature(self):
        pass

    @unittest.skipUnless(sys.platform == "linux", "Requires Linux")
    def test_linux_only_feature(self):
        pass

    @unittest.expectedFailure
    def test_known_bug(self):
        self.assertEqual(1, 2)  # expected to fail; won't count as a failure in results
```

---

## 8. Running `unittest` Tests

```bash
python -m unittest test_calculator.py           # run a specific file
python -m unittest test_calculator.TestCalculator.test_add_positive_numbers  # run one test
python -m unittest discover                      # auto-discover all test_*.py files
python -m unittest discover -s tests -p "test_*.py"  # discover in a specific directory
python -m unittest -v test_calculator.py         # verbose output (shows each test name)
```

---

## 9. `pytest` Framework

`pytest` is the most popular **third-party** testing framework — not built-in, requires `pip install pytest`, but offers a simpler syntax, powerful fixtures, and rich plugin ecosystem.

```bash
pip install pytest
```

### Basic Test (No Class Required!)

```python
# test_calculator.py
from calculator import add, divide
import pytest

def test_add_positive_numbers():
    assert add(2, 3) == 5

def test_add_negative_numbers():
    assert add(-1, -1) == -2

def test_divide_by_zero_raises_error():
    with pytest.raises(ValueError):
        divide(10, 0)
```

### Running Tests

```bash
pytest                          # auto-discovers and runs all test_*.py / *_test.py files
pytest test_calculator.py       # run a specific file
pytest -v                       # verbose output
pytest -k "add"                 # run only tests whose name contains "add"
pytest -x                       # stop after the first failure
pytest --tb=short               # shorter traceback output on failure
```

### `pytest` vs `unittest`

| Aspect | `unittest` | `pytest` |
|---|---|---|
| Installation | Built-in | `pip install pytest` |
| Test style | Class-based (`TestCase` subclass) | Plain functions (classes optional) |
| Assertions | Special methods (`assertEqual`, etc.) | Plain `assert` statements |
| Fixtures | `setUp`/`tearDown` | `@pytest.fixture` (more flexible, reusable) |
| Parametrized tests | Verbose, manual looping | Built-in `@pytest.mark.parametrize` |
| Plugin ecosystem | Limited | Extensive (coverage, mocking, async, etc.) |
| Can run `unittest` tests? | N/A | Yes — `pytest` can run existing `unittest` test suites too |

> [!tip]
> `pytest` can run your existing `unittest`-style tests without modification — making it easy to adopt `pytest` gradually in an existing codebase.

---

## 10. `pytest` Fixtures

Fixtures in `pytest` are reusable setup/teardown functions, injected into tests as **arguments** — far more flexible and composable than `unittest`'s `setUp`.

```python
import pytest

@pytest.fixture
def sample_data():
    return {"name": "Amit", "age": 25}

def test_name(sample_data):
    assert sample_data["name"] == "Amit"

def test_age(sample_data):
    assert sample_data["age"] == 25
```

### Fixtures with Setup AND Teardown (using `yield`)

```python
@pytest.fixture
def db_connection():
    conn = create_test_db_connection()   # setup
    yield conn                            # provided to the test
    conn.close()                          # teardown, runs after the test finishes
```

### Fixture Scopes

```python
@pytest.fixture(scope="function")   # default: new instance per test function
@pytest.fixture(scope="class")      # shared across all tests in a class
@pytest.fixture(scope="module")     # shared across all tests in a file
@pytest.fixture(scope="session")    # shared across the ENTIRE test run
def expensive_resource():
    resource = load_expensive_resource()
    yield resource
    resource.cleanup()
```

| Scope | Created |
|---|---|
| `function` (default) | Once per test function |
| `class` | Once per test class |
| `module` | Once per test file |
| `session` | Once for the entire test run |

### Sharing Fixtures Across Files: `conftest.py`

Fixtures defined in a `conftest.py` file are automatically available to all tests in that directory (and subdirectories) — no import needed.

```
tests/
├── conftest.py      ← shared fixtures live here
├── test_users.py
└── test_orders.py
```

```python
# conftest.py
import pytest

@pytest.fixture
def api_client():
    client = create_test_api_client()
    yield client
    client.close()
```

```python
# test_users.py
def test_get_user(api_client):   # fixture auto-available, no import needed
    response = api_client.get("/users/1")
    assert response.status_code == 200
```

---

## 11. Parametrized Tests

Running the **same test logic** against multiple sets of inputs, without duplicating code.

### With `pytest` (Clean & Built-in)

```python
import pytest

@pytest.mark.parametrize("a, b, expected", [
    (2, 3, 5),
    (-1, 1, 0),
    (0, 0, 0),
    (100, 200, 300),
])
def test_add(a, b, expected):
    assert add(a, b) == expected
```

This runs `test_add` **4 times**, once per tuple — each shown individually in the test output.

### With `unittest` (More Verbose)

```python
class TestAdd(unittest.TestCase):
    def test_add_multiple_cases(self):
        cases = [
            (2, 3, 5),
            (-1, 1, 0),
            (0, 0, 0),
        ]
        for a, b, expected in cases:
            with self.subTest(a=a, b=b):
                self.assertEqual(add(a, b), expected)
```

> [!note]
> `subTest()` in `unittest` lets you continue running remaining cases even if one fails, and reports which specific case failed — without it, a loop-based test stops at the first failure and loses information about how many cases actually passed.

---

## 12. Mocking & Patching

**Mocking** replaces real objects/dependencies (databases, APIs, file systems) with fake, controllable stand-ins — so tests stay fast, isolated, and don't depend on external systems.

### The `unittest.mock` Module (Built-in, works with both `unittest` and `pytest`)

```python
from unittest.mock import Mock, MagicMock, patch
```

### Basic Mock Object

```python
from unittest.mock import Mock

mock_obj = Mock()
mock_obj.some_method.return_value = 42

print(mock_obj.some_method())   # 42
mock_obj.some_method.assert_called_once()
```

### Patching a Function/Dependency

**`weather.py`**
```python
import requests

def get_temperature(city):
    response = requests.get(f"https://api.weather.com/{city}")
    return response.json()["temp"]
```

**Test with `patch`:**

```python
from unittest.mock import patch
from weather import get_temperature

@patch("weather.requests.get")   # patch WHERE it's used, not where it's defined
def test_get_temperature(mock_get):
    mock_get.return_value.json.return_value = {"temp": 25}

    result = get_temperature("London")

    assert result == 25
    mock_get.assert_called_once_with("https://api.weather.com/London")
```

> [!warning] Critical Rule
> Always patch the dependency **where it is used/imported**, not where it's originally defined. `@patch("weather.requests.get")` patches the `requests.get` reference as seen **inside the `weather` module**, which is what actually matters — patching `"requests.get"` directly usually won't have the intended effect if `weather.py` did `import requests`.

### Using `patch` as a Context Manager

```python
def test_with_context_manager():
    with patch("weather.requests.get") as mock_get:
        mock_get.return_value.json.return_value = {"temp": 30}
        result = get_temperature("Paris")
        assert result == 30
```

### Common Mock Assertions

```python
mock_obj.assert_called()               # was called at least once
mock_obj.assert_called_once()          # was called exactly once
mock_obj.assert_called_with(1, 2)      # last call had these exact arguments
mock_obj.assert_called_once_with(1, 2) # called exactly once, with these arguments
mock_obj.assert_not_called()           # was never called
print(mock_obj.call_count)              # number of times called
print(mock_obj.call_args)               # arguments of the most recent call
print(mock_obj.call_args_list)          # arguments of ALL calls
```

### `MagicMock` vs `Mock`
`MagicMock` is a subclass of `Mock` that additionally implements Python's "magic methods" (`__len__`, `__iter__`, `__enter__`, etc.), so it behaves more like a real object in more contexts. `patch()` uses `MagicMock` by default.

### `side_effect` — Simulating Exceptions or Dynamic Behavior

```python
mock_obj = Mock(side_effect=ValueError("Something went wrong"))

with pytest.raises(ValueError):
    mock_obj()

# side_effect can also be a function or a list of return values (one per call)
mock_obj2 = Mock(side_effect=[1, 2, 3])
print(mock_obj2())  # 1
print(mock_obj2())  # 2
print(mock_obj2())  # 3
```

### Mocking with `pytest-mock` (Cleaner Syntax, Third-Party Plugin)

```bash
pip install pytest-mock
```

```python
def test_get_temperature(mocker):
    mock_get = mocker.patch("weather.requests.get")
    mock_get.return_value.json.return_value = {"temp": 25}

    result = get_temperature("London")
    assert result == 25
```

> [!tip]
> `pytest-mock`'s `mocker` fixture automatically handles cleanup (unpatching) after each test — no need for `with` blocks or decorators.

---

## 13. Testing Exceptions

### `unittest`

```python
class TestDivide(unittest.TestCase):
    def test_divide_by_zero(self):
        with self.assertRaises(ValueError):
            divide(10, 0)

    def test_exception_message(self):
        with self.assertRaises(ValueError) as context:
            divide(10, 0)
        self.assertEqual(str(context.exception), "Cannot divide by zero")
```

### `pytest`

```python
import pytest

def test_divide_by_zero():
    with pytest.raises(ValueError):
        divide(10, 0)

def test_exception_message():
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        divide(10, 0)
```

> [!note]
> `pytest.raises(..., match=...)` uses a **regular expression** to check the exception message — remember to escape special regex characters if your expected message contains them.

---

## 14. Testing Async Code

### Using `unittest.IsolatedAsyncioTestCase` (Python 3.8+)

```python
import unittest

async def fetch_data():
    return {"status": "ok"}

class TestAsync(unittest.IsolatedAsyncioTestCase):
    async def test_fetch_data(self):
        result = await fetch_data()
        self.assertEqual(result, {"status": "ok"})
```

### Using `pytest-asyncio`

```bash
pip install pytest-asyncio
```

```python
import pytest

@pytest.mark.asyncio
async def test_fetch_data():
    result = await fetch_data()
    assert result == {"status": "ok"}
```

---

## 15. Code Coverage

**Code coverage** measures what percentage of your code is actually executed by your test suite — helping identify untested paths (though 100% coverage does NOT guarantee correctness, since it doesn't check assertion quality).

```bash
pip install coverage pytest-cov
```

### With `coverage` directly

```bash
coverage run -m pytest
coverage report          # text summary in terminal
coverage html             # generates an interactive HTML report in htmlcov/
```

### With `pytest-cov` Plugin

```bash
pytest --cov=mypackage --cov-report=term-missing
```

Example output:
```
Name                 Stmts   Miss  Cover   Missing
--------------------------------------------------
calculator.py            8      1    88%   12
--------------------------------------------------
TOTAL                     8      1    88%
```

> [!warning]
> High coverage percentage is not the same as good tests. A test that calls a function but doesn't assert anything meaningful still counts toward "coverage" while providing near-zero actual protection against bugs. Treat coverage as a tool to **find untested code**, not as a quality metric on its own.

---

## 16. Test Organization & Naming Conventions

### Recommended Project Structure

```
myproject/
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── calculator.py
│       └── weather.py
└── tests/
    ├── conftest.py
    ├── test_calculator.py
    └── test_weather.py
```

### Naming Conventions
- Test files: `test_<module>.py` or `<module>_test.py`
- Test functions (`pytest`): `test_<what_it_tests>`
- Test methods (`unittest`): `def test_<what_it_tests>(self):`
- Test classes (`unittest`): `Test<ClassBeingTested>`

### Descriptive Test Names

```python
# Vague — unclear what's actually being verified
def test_divide():
    assert divide(10, 2) == 5

# Descriptive — clearly states the scenario and expected outcome
def test_divide_returns_correct_quotient_for_positive_numbers():
    assert divide(10, 2) == 5

def test_divide_raises_value_error_when_dividing_by_zero():
    with pytest.raises(ValueError):
        divide(10, 0)
```

> [!tip]
> A good test name should let you understand **what broke** just by reading the name in a CI failure report, without opening the test file.

---

## 17. Test Doubles: Mocks, Stubs, Fakes, Spies

"Test double" is the umbrella term (like "stunt double") for any object that stands in for a real dependency during testing.

| Type | Description | Example |
|---|---|---|
| **Dummy** | Passed around but never actually used, just to satisfy a parameter list | `def test(_unused_logger): ...` |
| **Stub** | Returns hardcoded/canned answers, no real logic | A fake API client that always returns `{"status": "ok"}` |
| **Fake** | A working, simplified implementation (not production-ready but functionally real) | An in-memory database used instead of a real one |
| **Mock** | Records how it was called, and lets you assert on those interactions | `mock.assert_called_once_with(...)` |
| **Spy** | Wraps a real object, recording calls while still delegating to the real implementation | `mocker.spy(real_obj, "method")` |

```python
# Fake example — a simplified in-memory "database"
class FakeUserRepository:
    def __init__(self):
        self.users = {}

    def add(self, user_id, name):
        self.users[user_id] = name

    def get(self, user_id):
        return self.users.get(user_id)

def test_user_service_with_fake_repo():
    fake_repo = FakeUserRepository()
    service = UserService(fake_repo)   # dependency injection

    service.create_user(1, "Amit")
    assert service.get_user_name(1) == "Amit"
```

> [!tip]
> Designing code with **dependency injection** (passing dependencies in, rather than hardcoding them inside a function/class) makes it far easier to substitute test doubles without needing to patch internals.

---

## 18. Test-Driven Development (TDD)

**TDD** is a development workflow where you write the **test first**, watch it fail, then write just enough code to make it pass, then refactor.

### The Red-Green-Refactor Cycle

```
1. RED      → Write a failing test for a feature that doesn't exist yet
2. GREEN    → Write the minimal code needed to make the test pass
3. REFACTOR → Clean up the code (and/or test) while keeping tests green
        │
        └──────────────► repeat for the next small piece of behavior
```

### Example Walkthrough

**Step 1 — Write a failing test (RED):**
```python
def test_add_returns_sum_of_two_numbers():
    assert add(2, 3) == 5
```
Running this fails immediately — `add` doesn't exist yet (`NameError`).

**Step 2 — Write minimal code to pass (GREEN):**
```python
def add(a, b):
    return a + b
```
Test now passes.

**Step 3 — Refactor (still GREEN):**
Clean up implementation/naming if needed, re-run tests to confirm nothing broke.

### Benefits of TDD
- Forces you to think about the **API/interface** before implementation details.
- Naturally produces high test coverage, since code only exists to satisfy a test.
- Provides fast feedback loops and prevents over-engineering (you write only what's needed to pass).

> [!note]
> TDD is a discipline, not a mandatory law — many teams use a hybrid approach (writing tests alongside or shortly after code) and still get most of the benefits. What matters most is that tests exist and are maintained, however they were written.

---

## 19. Common Pitfalls

| Pitfall | Why It's a Problem | Fix |
|---|---|---|
| Tests depend on execution order | Fragile — passes/fails depending on which tests ran first | Make each test fully independent; don't share mutable state across tests |
| Testing implementation details, not behavior | Tests break on harmless refactors, even when behavior is unchanged | Test observable outputs/behavior, not internal implementation |
| Overusing mocks | Tests pass even when real integration is broken; over-mocked tests validate almost nothing | Mock only true external dependencies (network, DB, filesystem), not your own logic |
| Slow tests due to real I/O (network, DB, sleep) | Slows down the whole suite, discourages running tests often | Mock I/O, or move such tests to a separate integration test suite |
| One giant test checking many things | Failure doesn't clearly indicate what broke | Split into focused, single-purpose tests |
| Not testing edge cases (empty input, None, zero, negative numbers) | Bugs slip through in untested boundary conditions | Explicitly write tests for edge cases, not just the "happy path" |
| Chasing 100% coverage without meaningful assertions | False sense of security; coverage ≠ correctness | Focus on test quality, use coverage only to find gaps |
| Patching the wrong location | `patch()` silently has no effect, test passes for the wrong reason | Patch where the dependency is **used/imported**, not where it's defined |

---

## 20. Best Practices

- ✅ Follow the **Arrange–Act–Assert** structure for clarity.
- ✅ Keep tests **fast, isolated, and repeatable** — mock external dependencies (network, DB, filesystem, time).
- ✅ Use **descriptive test names** that state the scenario and expected outcome.
- ✅ Test **behavior/outputs**, not internal implementation details.
- ✅ Cover **edge cases** explicitly: empty inputs, `None`, zero, negative numbers, large inputs, invalid types.
- ✅ Use fixtures (`pytest` fixtures or `setUp`/`tearDown`) to avoid duplicated setup code.
- ✅ Use **parametrized tests** to cover multiple input combinations without duplicating logic.
- ✅ Run tests automatically in CI on every commit/PR.
- ✅ Use code coverage to **find gaps**, not as a vanity metric.
- ✅ Prefer dependency injection in your code design — it makes testing (and mocking) far easier.

---

## 21. Quick Reference Cheat Sheet

```python
# ── unittest ──
import unittest

class TestSomething(unittest.TestCase):
    def setUp(self): ...
    def tearDown(self): ...
    def test_case(self):
        self.assertEqual(actual, expected)
        with self.assertRaises(ValueError):
            risky_call()

if __name__ == "__main__":
    unittest.main()

# Run:
# python -m unittest discover


# ── pytest ──
import pytest

def test_case():
    assert actual == expected
    with pytest.raises(ValueError, match="message"):
        risky_call()

@pytest.fixture
def resource():
    r = setup()
    yield r
    teardown(r)

@pytest.mark.parametrize("a,b,expected", [(1,2,3), (2,2,4)])
def test_add(a, b, expected):
    assert add(a, b) == expected

# Run:
# pytest -v


# ── Mocking ──
from unittest.mock import patch, Mock

@patch("module.dependency")
def test_x(mock_dep):
    mock_dep.return_value = "fake result"
    ...
    mock_dep.assert_called_once_with(expected_args)


# ── Coverage ──
# pytest --cov=mypackage --cov-report=term-missing
```

| Concept | Summary |
|---|---|
| **Unit test** | Tests one function/method in isolation |
| **Integration test** | Tests multiple components together |
| **AAA pattern** | Arrange, Act, Assert |
| **Fixture** | Reusable setup/teardown logic for tests |
| **Mock** | Fake object standing in for a real dependency |
| **Parametrize** | Run the same test logic across multiple inputs |
| **Coverage** | % of code executed by the test suite |
| **TDD** | Write test → fail → implement → pass → refactor |

---

*Tags:* #python #testing #unittest #pytest #mocking #tdd