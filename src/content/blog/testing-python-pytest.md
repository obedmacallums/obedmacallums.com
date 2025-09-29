---
author: Obed Macallums
pubDatetime: 2025-09-29T00:00:00Z
title: "Testing in Python with Pytest: A Complete Guide"
slug: testing-python-pytest
featured: true
draft: false
tags:
  - python
  - testing
  - pytest
  - tdd
  - quality-assurance
description:
  A comprehensive guide to testing Python applications with Pytest, covering fixtures, parametrization, mocking, async testing, and best practices for building robust test suites.
---

Testing is a fundamental part of professional software development, ensuring code reliability, catching bugs early, and providing confidence when refactoring. **Pytest** has become the de facto testing framework in the Python ecosystem, offering a simple yet powerful approach to writing tests. This comprehensive guide will take you from basic concepts to advanced testing techniques.

## Table of Contents

## Why Pytest?

While Python includes the built-in `unittest` module, Pytest has gained massive adoption due to several compelling advantages:

- **Simple and Pythonic**: Write tests as simple functions with assert statements
- **Powerful fixtures**: Elegant dependency injection system for test setup/teardown
- **Detailed failure reports**: Clear and informative error messages
- **Plugin ecosystem**: Over 800+ plugins for extended functionality
- **Parametrization**: Run the same test with different inputs effortlessly
- **Backward compatible**: Can run existing unittest and nose tests
- **Active development**: Large community and regular updates

## Installation and Setup

### Basic Installation

```bash
# Install pytest
pip install pytest

# Install with coverage support
pip install pytest pytest-cov

# Verify installation
pytest --version
```

### Project Structure

A well-organized test structure is crucial for maintainability:

```
myproject/
├── src/
│   ├── __init__.py
│   ├── calculator.py
│   └── user.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py          # Shared fixtures
│   ├── test_calculator.py
│   └── test_user.py
├── pytest.ini                # Pytest configuration
└── requirements.txt
```

### Configuration

Create a `pytest.ini` file in your project root:

```ini
[pytest]
# Minimum version required
minversion = 6.0

# Directories to search for tests
testpaths = tests

# Pattern for test files
python_files = test_*.py *_test.py

# Pattern for test functions/classes
python_functions = test_*
python_classes = Test*

# Options always used
addopts =
    -v
    --strict-markers
    --cov=src
    --cov-report=html
    --cov-report=term-missing

# Custom markers
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
    unit: marks tests as unit tests
    smoke: marks tests for smoke testing
```

## Writing Your First Test

Let's start with a simple example. Create `src/calculator.py`:

```python
# src/calculator.py
def add(a, b):
    """Add two numbers."""
    return a + b

def subtract(a, b):
    """Subtract b from a."""
    return a - b

def multiply(a, b):
    """Multiply two numbers."""
    return a * b

def divide(a, b):
    """Divide a by b."""
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b
```

Now create `tests/test_calculator.py`:

```python
# tests/test_calculator.py
from src.calculator import add, subtract, multiply, divide
import pytest

def test_add():
    """Test addition function."""
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
    assert add(0, 0) == 0

def test_subtract():
    """Test subtraction function."""
    assert subtract(5, 3) == 2
    assert subtract(1, 1) == 0
    assert subtract(0, 5) == -5

def test_multiply():
    """Test multiplication function."""
    assert multiply(3, 4) == 12
    assert multiply(-2, 3) == -6
    assert multiply(0, 100) == 0

def test_divide():
    """Test division function."""
    assert divide(10, 2) == 5
    assert divide(9, 3) == 3
    assert divide(1, 2) == 0.5
```

Run the tests:

```bash
pytest tests/test_calculator.py -v
```

## Testing Exceptions

Testing error conditions is just as important as testing happy paths:

```python
# tests/test_calculator.py (continued)
def test_divide_by_zero():
    """Test that dividing by zero raises ValueError."""
    with pytest.raises(ValueError) as excinfo:
        divide(10, 0)

    assert "Cannot divide by zero" in str(excinfo.value)

def test_divide_by_zero_alternative():
    """Alternative syntax for testing exceptions."""
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        divide(5, 0)
```

## Fixtures: The Heart of Pytest

Fixtures provide a way to set up test preconditions and share common test data. They're one of Pytest's most powerful features.

### Basic Fixtures

```python
# tests/conftest.py
import pytest

@pytest.fixture
def sample_numbers():
    """Provide sample numbers for testing."""
    return [1, 2, 3, 4, 5]

@pytest.fixture
def calculator_operations():
    """Provide a dictionary of calculator functions."""
    from src.calculator import add, subtract, multiply, divide
    return {
        'add': add,
        'subtract': subtract,
        'multiply': multiply,
        'divide': divide
    }
```

Using fixtures in tests:

```python
# tests/test_calculator.py
def test_operations_with_fixture(calculator_operations):
    """Test calculator operations using fixture."""
    ops = calculator_operations
    assert ops['add'](2, 3) == 5
    assert ops['subtract'](5, 2) == 3
    assert ops['multiply'](3, 4) == 12
    assert ops['divide'](10, 2) == 5

def test_sum_sample_numbers(sample_numbers):
    """Test sum of sample numbers."""
    assert sum(sample_numbers) == 15
```

### Fixture Scopes

Fixtures can have different scopes to control when they're created and destroyed:

```python
# tests/conftest.py
@pytest.fixture(scope="function")  # Default: recreated for each test
def function_scope_fixture():
    print("Setup function scope")
    yield "function data"
    print("Teardown function scope")

@pytest.fixture(scope="class")  # Shared across test class
def class_scope_fixture():
    print("Setup class scope")
    yield "class data"
    print("Teardown class scope")

@pytest.fixture(scope="module")  # Shared across test module
def module_scope_fixture():
    print("Setup module scope")
    database = {"users": []}
    yield database
    print("Teardown module scope")

@pytest.fixture(scope="session")  # Shared across entire test session
def session_scope_fixture():
    print("Setup session scope")
    config = {"api_url": "https://api.example.com"}
    yield config
    print("Teardown session scope")
```

### Fixtures with Setup and Teardown

```python
# tests/conftest.py
import tempfile
import os

@pytest.fixture
def temp_file():
    """Create a temporary file for testing."""
    # Setup
    fd, path = tempfile.mkstemp()
    os.write(fd, b"Hello, World!")
    os.close(fd)

    # Provide the fixture value
    yield path

    # Teardown
    os.unlink(path)

# Using the fixture
def test_read_temp_file(temp_file):
    """Test reading from temporary file."""
    with open(temp_file, 'rb') as f:
        content = f.read()
    assert content == b"Hello, World!"
```

### Parametrized Fixtures

```python
# tests/conftest.py
@pytest.fixture(params=["sqlite", "postgres", "mysql"])
def database_type(request):
    """Parametrized fixture for different database types."""
    return request.param

def test_database_connection(database_type):
    """This test will run 3 times, once for each database type."""
    print(f"Testing with {database_type}")
    assert database_type in ["sqlite", "postgres", "mysql"]
```

## Parametrization: Testing Multiple Scenarios

Parametrization allows you to run the same test with different input values:

### Basic Parametrization

```python
# tests/test_calculator.py
@pytest.mark.parametrize("a, b, expected", [
    (2, 3, 5),
    (0, 0, 0),
    (-1, 1, 0),
    (100, 200, 300),
    (-5, -5, -10),
])
def test_add_parametrized(a, b, expected):
    """Test addition with multiple input combinations."""
    assert add(a, b) == expected

@pytest.mark.parametrize("a, b, expected", [
    (10, 2, 5),
    (9, 3, 3),
    (100, 4, 25),
    (1, 2, 0.5),
])
def test_divide_parametrized(a, b, expected):
    """Test division with multiple input combinations."""
    assert divide(a, b) == expected
```

### Multiple Parameter Sets

```python
@pytest.mark.parametrize("operation", [add, subtract, multiply])
@pytest.mark.parametrize("a, b", [(1, 2), (3, 4), (5, 6)])
def test_operations_combinations(operation, a, b):
    """Test multiple operations with multiple inputs (9 test cases)."""
    result = operation(a, b)
    assert isinstance(result, (int, float))
```

### Parametrization with IDs

```python
@pytest.mark.parametrize(
    "dividend, divisor, expected",
    [
        (10, 2, 5),
        (9, 3, 3),
        (1, 2, 0.5),
    ],
    ids=["ten_divided_by_two", "nine_divided_by_three", "one_divided_by_two"]
)
def test_divide_with_ids(dividend, divisor, expected):
    """Test division with custom test IDs."""
    assert divide(dividend, divisor) == expected
```

## Markers: Organizing and Controlling Tests

Markers allow you to categorize and selectively run tests:

### Built-in Markers

```python
# tests/test_advanced.py
import pytest
import time

@pytest.mark.skip(reason="Not implemented yet")
def test_future_feature():
    """This test will be skipped."""
    assert False

@pytest.mark.skipif(sys.version_info < (3, 8), reason="Requires Python 3.8+")
def test_python38_feature():
    """Skip on Python versions below 3.8."""
    # Use walrus operator (Python 3.8+)
    if (n := 10) > 5:
        assert True

@pytest.mark.xfail(reason="Known bug in production")
def test_known_bug():
    """Expected to fail."""
    assert 1 == 2

@pytest.mark.xfail(strict=True)
def test_strict_xfail():
    """Must fail, otherwise test fails."""
    assert False
```

### Custom Markers

```python
# tests/test_performance.py
@pytest.mark.slow
def test_slow_operation():
    """Mark tests that take a long time."""
    time.sleep(2)
    assert True

@pytest.mark.integration
def test_api_integration():
    """Mark integration tests."""
    # API call simulation
    assert True

@pytest.mark.unit
def test_unit_logic():
    """Mark unit tests."""
    assert add(1, 1) == 2
```

Run specific markers:

```bash
# Run only slow tests
pytest -m slow

# Run everything except slow tests
pytest -m "not slow"

# Run unit or integration tests
pytest -m "unit or integration"

# Run integration tests that are not slow
pytest -m "integration and not slow"
```

## Mocking and Patching

Testing often requires isolating code from external dependencies like APIs, databases, or file systems.

### Using unittest.mock

```python
# src/user.py
import requests

class UserService:
    """Service for managing users."""

    def __init__(self, api_url):
        self.api_url = api_url

    def get_user(self, user_id):
        """Fetch user from API."""
        response = requests.get(f"{self.api_url}/users/{user_id}")
        response.raise_for_status()
        return response.json()

    def create_user(self, username, email):
        """Create a new user."""
        response = requests.post(
            f"{self.api_url}/users",
            json={"username": username, "email": email}
        )
        response.raise_for_status()
        return response.json()
```

```python
# tests/test_user.py
from unittest.mock import Mock, patch, MagicMock
from src.user import UserService
import pytest

@pytest.fixture
def user_service():
    """Create a UserService instance."""
    return UserService("https://api.example.com")

def test_get_user_success(user_service):
    """Test successful user retrieval."""
    mock_response = Mock()
    mock_response.json.return_value = {
        "id": 1,
        "username": "johndoe",
        "email": "john@example.com"
    }
    mock_response.raise_for_status.return_value = None

    with patch('requests.get', return_value=mock_response) as mock_get:
        user = user_service.get_user(1)

        # Verify the mock was called correctly
        mock_get.assert_called_once_with("https://api.example.com/users/1")

        # Verify the result
        assert user["username"] == "johndoe"
        assert user["email"] == "john@example.com"

def test_create_user(user_service):
    """Test user creation."""
    mock_response = Mock()
    mock_response.json.return_value = {
        "id": 2,
        "username": "janedoe",
        "email": "jane@example.com"
    }
    mock_response.raise_for_status.return_value = None

    with patch('requests.post', return_value=mock_response) as mock_post:
        user = user_service.create_user("janedoe", "jane@example.com")

        # Verify call arguments
        mock_post.assert_called_once()
        call_args = mock_post.call_args
        assert call_args[0][0] == "https://api.example.com/users"
        assert call_args[1]["json"]["username"] == "janedoe"

        assert user["id"] == 2
```

### Using pytest-mock

Install the plugin: `pip install pytest-mock`

```python
# tests/test_user_mocker.py
def test_get_user_with_mocker(user_service, mocker):
    """Test using pytest-mock's mocker fixture."""
    mock_get = mocker.patch('requests.get')
    mock_get.return_value.json.return_value = {
        "id": 1,
        "username": "testuser"
    }

    user = user_service.get_user(1)
    assert user["username"] == "testuser"
    mock_get.assert_called_once()
```

## Testing Async Code

Modern Python applications often use async/await. Pytest can test async code with the `pytest-asyncio` plugin.

### Installation

```bash
pip install pytest-asyncio
```

### Testing Async Functions

```python
# src/async_ops.py
import asyncio
import aiohttp

async def fetch_data(url):
    """Fetch data from URL asynchronously."""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

async def process_multiple_urls(urls):
    """Process multiple URLs concurrently."""
    tasks = [fetch_data(url) for url in urls]
    return await asyncio.gather(*tasks)
```

```python
# tests/test_async_ops.py
import pytest
from src.async_ops import fetch_data, process_multiple_urls

@pytest.mark.asyncio
async def test_fetch_data(mocker):
    """Test async data fetching."""
    mock_response = mocker.Mock()
    mock_response.json = mocker.AsyncMock(return_value={"data": "test"})

    mock_session = mocker.Mock()
    mock_session.get.return_value.__aenter__.return_value = mock_response

    mocker.patch('aiohttp.ClientSession', return_value=mock_session)

    result = await fetch_data("https://api.example.com/data")
    assert result["data"] == "test"

@pytest.mark.asyncio
async def test_process_multiple_urls(mocker):
    """Test processing multiple URLs concurrently."""
    mocker.patch(
        'src.async_ops.fetch_data',
        side_effect=[
            {"id": 1, "data": "first"},
            {"id": 2, "data": "second"},
            {"id": 3, "data": "third"},
        ]
    )

    urls = ["url1", "url2", "url3"]
    results = await process_multiple_urls(urls)

    assert len(results) == 3
    assert results[0]["id"] == 1
    assert results[2]["data"] == "third"
```

### Async Fixtures

```python
# tests/conftest.py
@pytest.fixture
async def async_database():
    """Async fixture for database connection."""
    # Setup
    db = await connect_to_database()
    yield db
    # Teardown
    await db.close()
```

## Testing with Databases

### Using pytest-postgresql

```bash
pip install pytest-postgresql
```

```python
# tests/test_database.py
def test_database_operations(postgresql):
    """Test database operations."""
    cursor = postgresql.cursor()
    cursor.execute("CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR(100))")
    cursor.execute("INSERT INTO users (name) VALUES (%s)", ("John Doe",))
    cursor.execute("SELECT name FROM users WHERE id = 1")
    result = cursor.fetchone()
    assert result[0] == "John Doe"
```

### Using SQLite with Fixtures

```python
# tests/conftest.py
import sqlite3
import pytest

@pytest.fixture
def sqlite_db():
    """Create an in-memory SQLite database."""
    conn = sqlite3.connect(":memory:")
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE products (
            id INTEGER PRIMARY KEY,
            name TEXT NOT NULL,
            price REAL NOT NULL
        )
    """)
    conn.commit()
    yield conn
    conn.close()

def test_insert_product(sqlite_db):
    """Test inserting a product."""
    cursor = sqlite_db.cursor()
    cursor.execute(
        "INSERT INTO products (name, price) VALUES (?, ?)",
        ("Laptop", 999.99)
    )
    sqlite_db.commit()

    cursor.execute("SELECT name, price FROM products WHERE id = 1")
    result = cursor.fetchone()
    assert result[0] == "Laptop"
    assert result[1] == 999.99
```

## Integration with FastAPI

Testing FastAPI applications with Pytest is straightforward:

```python
# src/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

items_db = {}

@app.post("/items/")
def create_item(item: Item):
    """Create a new item."""
    item_id = len(items_db) + 1
    items_db[item_id] = item
    return {"id": item_id, **item.dict()}

@app.get("/items/{item_id}")
def get_item(item_id: int):
    """Get an item by ID."""
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    return items_db[item_id]
```

```python
# tests/test_fastapi_app.py
from fastapi.testclient import TestClient
from src.main import app
import pytest

@pytest.fixture
def client():
    """Create a test client."""
    return TestClient(app)

def test_create_item(client):
    """Test item creation endpoint."""
    response = client.post(
        "/items/",
        json={"name": "Laptop", "price": 999.99}
    )
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "Laptop"
    assert data["price"] == 999.99
    assert "id" in data

def test_get_item(client):
    """Test getting an item."""
    # First create an item
    create_response = client.post(
        "/items/",
        json={"name": "Mouse", "price": 29.99}
    )
    item_id = create_response.json()["id"]

    # Then retrieve it
    response = client.get(f"/items/{item_id}")
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "Mouse"

def test_get_nonexistent_item(client):
    """Test getting an item that doesn't exist."""
    response = client.get("/items/9999")
    assert response.status_code == 404
    assert response.json()["detail"] == "Item not found"
```

## Integration with Django

Testing Django applications with Pytest using `pytest-django`:

```bash
pip install pytest-django
```

Create `pytest.ini`:

```ini
[pytest]
DJANGO_SETTINGS_MODULE = myproject.settings
python_files = tests.py test_*.py *_tests.py
```

```python
# tests/test_django_models.py
import pytest
from django.contrib.auth.models import User

@pytest.mark.django_db
def test_create_user():
    """Test user creation."""
    user = User.objects.create_user(
        username="testuser",
        email="test@example.com",
        password="testpass123"
    )
    assert user.username == "testuser"
    assert user.email == "test@example.com"
    assert user.check_password("testpass123")

@pytest.mark.django_db
def test_user_count():
    """Test user count."""
    User.objects.create_user("user1", "user1@test.com", "pass1")
    User.objects.create_user("user2", "user2@test.com", "pass2")
    assert User.objects.count() == 2
```

## Code Coverage

Measuring test coverage helps identify untested code:

```bash
# Install coverage support
pip install pytest-cov

# Run tests with coverage
pytest --cov=src

# Generate HTML report
pytest --cov=src --cov-report=html

# Show missing lines
pytest --cov=src --cov-report=term-missing

# Set minimum coverage threshold
pytest --cov=src --cov-fail-under=80
```

Coverage configuration in `pytest.ini`:

```ini
[pytest]
addopts =
    --cov=src
    --cov-report=html
    --cov-report=term-missing
    --cov-fail-under=80
```

## Useful Plugins

Pytest has a rich plugin ecosystem:

### pytest-xdist (Parallel Testing)

```bash
pip install pytest-xdist

# Run tests in parallel (automatic CPU detection)
pytest -n auto

# Run tests on 4 CPUs
pytest -n 4
```

### pytest-benchmark

```bash
pip install pytest-benchmark
```

```python
def test_calculation_performance(benchmark):
    """Benchmark a calculation."""
    result = benchmark(lambda: sum(range(10000)))
    assert result == 49995000
```

### pytest-timeout

```bash
pip install pytest-timeout
```

```python
@pytest.mark.timeout(5)
def test_with_timeout():
    """This test must complete within 5 seconds."""
    # Long-running operation
    pass
```

### pytest-randomly

```bash
pip install pytest-randomly

# Runs tests in random order to detect order dependencies
pytest
```

### pytest-sugar

```bash
pip install pytest-sugar

# Provides a better-looking test output
pytest
```

## Best Practices

### 1. Test Organization

```python
# Group related tests in classes
class TestCalculator:
    """Tests for calculator operations."""

    def test_addition(self):
        assert add(2, 3) == 5

    def test_subtraction(self):
        assert subtract(5, 3) == 2

class TestAdvancedCalculator:
    """Tests for advanced calculator operations."""

    def test_power(self):
        assert pow(2, 3) == 8
```

### 2. Use Descriptive Test Names

```python
# Good
def test_divide_by_zero_raises_value_error():
    with pytest.raises(ValueError):
        divide(10, 0)

# Bad
def test_divide():
    with pytest.raises(ValueError):
        divide(10, 0)
```

### 3. Follow the AAA Pattern

```python
def test_user_creation():
    # Arrange: Set up test data
    username = "testuser"
    email = "test@example.com"

    # Act: Execute the code under test
    user = create_user(username, email)

    # Assert: Verify the results
    assert user.username == username
    assert user.email == email
```

### 4. Test One Thing at a Time

```python
# Good: Separate tests for separate concerns
def test_user_has_correct_username():
    user = create_user("john", "john@example.com")
    assert user.username == "john"

def test_user_has_correct_email():
    user = create_user("john", "john@example.com")
    assert user.email == "john@example.com"

# Bad: Testing multiple things
def test_user_creation():
    user = create_user("john", "john@example.com")
    assert user.username == "john"
    assert user.email == "john@example.com"
    assert user.is_active == True
    assert user.date_joined is not None
```

### 5. Use Fixtures for Common Setup

```python
# Good: Reusable fixture
@pytest.fixture
def authenticated_client():
    client = TestClient(app)
    token = get_auth_token("testuser", "password")
    client.headers = {"Authorization": f"Bearer {token}"}
    return client

def test_protected_endpoint(authenticated_client):
    response = authenticated_client.get("/protected")
    assert response.status_code == 200

# Bad: Repeated setup
def test_protected_endpoint():
    client = TestClient(app)
    token = get_auth_token("testuser", "password")
    client.headers = {"Authorization": f"Bearer {token}"}
    response = client.get("/protected")
    assert response.status_code == 200
```

### 6. Mock External Dependencies

```python
# Good: Isolated test
def test_send_email(mocker):
    mock_smtp = mocker.patch('smtplib.SMTP')
    send_email("test@example.com", "Hello")
    mock_smtp.assert_called_once()

# Bad: Requires real SMTP server
def test_send_email():
    send_email("test@example.com", "Hello")
    # This will fail without a real SMTP server
```

### 7. Use Parametrization for Similar Tests

```python
# Good: Parametrized test
@pytest.mark.parametrize("input,expected", [
    ("hello", "HELLO"),
    ("World", "WORLD"),
    ("123", "123"),
])
def test_uppercase(input, expected):
    assert input.upper() == expected

# Bad: Repetitive tests
def test_uppercase_hello():
    assert "hello".upper() == "HELLO"

def test_uppercase_world():
    assert "World".upper() == "WORLD"
```

## Debugging Tests

### Using pytest options

```bash
# Show print statements
pytest -s

# Drop into debugger on failure
pytest --pdb

# Show local variables in traceback
pytest -l

# Only run failed tests from last run
pytest --lf

# Run failed tests first, then others
pytest --ff

# Stop after first failure
pytest -x

# Stop after N failures
pytest --maxfail=3

# Increase verbosity
pytest -vv
```

### Using breakpoint()

```python
def test_complex_logic():
    data = complex_calculation()
    breakpoint()  # Python 3.7+ debugger
    assert data["result"] == expected
```

## Continuous Integration

### GitHub Actions Example

```yaml
# .github/workflows/tests.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: [3.8, 3.9, 3.10, 3.11]

    steps:
    - uses: actions/checkout@v3

    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install pytest pytest-cov

    - name: Run tests with coverage
      run: |
        pytest --cov=src --cov-report=xml

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
```

## Real-World Example: Testing a Complete Module

Let's put it all together with a complete example:

```python
# src/shopping_cart.py
from typing import List, Dict
from dataclasses import dataclass

@dataclass
class Product:
    """Product data class."""
    id: int
    name: str
    price: float

class ShoppingCart:
    """Shopping cart implementation."""

    def __init__(self):
        self.items: Dict[int, int] = {}  # product_id: quantity
        self.products: Dict[int, Product] = {}

    def add_product(self, product: Product, quantity: int = 1):
        """Add a product to the cart."""
        if quantity <= 0:
            raise ValueError("Quantity must be positive")

        self.products[product.id] = product
        if product.id in self.items:
            self.items[product.id] += quantity
        else:
            self.items[product.id] = quantity

    def remove_product(self, product_id: int):
        """Remove a product from the cart."""
        if product_id not in self.items:
            raise KeyError(f"Product {product_id} not in cart")

        del self.items[product_id]
        del self.products[product_id]

    def update_quantity(self, product_id: int, quantity: int):
        """Update product quantity."""
        if product_id not in self.items:
            raise KeyError(f"Product {product_id} not in cart")
        if quantity <= 0:
            raise ValueError("Quantity must be positive")

        self.items[product_id] = quantity

    def get_total(self) -> float:
        """Calculate total cart value."""
        total = 0.0
        for product_id, quantity in self.items.items():
            product = self.products[product_id]
            total += product.price * quantity
        return round(total, 2)

    def get_item_count(self) -> int:
        """Get total number of items."""
        return sum(self.items.values())

    def clear(self):
        """Clear the cart."""
        self.items.clear()
        self.products.clear()
```

```python
# tests/test_shopping_cart.py
import pytest
from src.shopping_cart import ShoppingCart, Product

@pytest.fixture
def empty_cart():
    """Provide an empty shopping cart."""
    return ShoppingCart()

@pytest.fixture
def sample_products():
    """Provide sample products."""
    return [
        Product(1, "Laptop", 999.99),
        Product(2, "Mouse", 29.99),
        Product(3, "Keyboard", 79.99),
    ]

@pytest.fixture
def cart_with_items(empty_cart, sample_products):
    """Provide a cart with some items."""
    empty_cart.add_product(sample_products[0], 1)  # Laptop
    empty_cart.add_product(sample_products[1], 2)  # 2x Mouse
    return empty_cart

class TestShoppingCart:
    """Test suite for ShoppingCart class."""

    def test_empty_cart_initialization(self, empty_cart):
        """Test that new cart is empty."""
        assert empty_cart.get_item_count() == 0
        assert empty_cart.get_total() == 0.0

    def test_add_single_product(self, empty_cart, sample_products):
        """Test adding a single product."""
        laptop = sample_products[0]
        empty_cart.add_product(laptop)

        assert empty_cart.get_item_count() == 1
        assert empty_cart.get_total() == 999.99

    def test_add_product_with_quantity(self, empty_cart, sample_products):
        """Test adding product with specific quantity."""
        mouse = sample_products[1]
        empty_cart.add_product(mouse, 3)

        assert empty_cart.get_item_count() == 3
        assert empty_cart.get_total() == 89.97

    def test_add_same_product_twice(self, empty_cart, sample_products):
        """Test adding the same product multiple times."""
        keyboard = sample_products[2]
        empty_cart.add_product(keyboard, 1)
        empty_cart.add_product(keyboard, 2)

        assert empty_cart.get_item_count() == 3
        assert empty_cart.get_total() == 239.97

    @pytest.mark.parametrize("quantity", [0, -1, -10])
    def test_add_product_invalid_quantity(self, empty_cart, sample_products, quantity):
        """Test that adding product with invalid quantity raises error."""
        with pytest.raises(ValueError, match="Quantity must be positive"):
            empty_cart.add_product(sample_products[0], quantity)

    def test_remove_product(self, cart_with_items):
        """Test removing a product from cart."""
        initial_count = cart_with_items.get_item_count()
        cart_with_items.remove_product(1)  # Remove laptop

        assert cart_with_items.get_item_count() == initial_count - 1
        assert 1 not in cart_with_items.items

    def test_remove_nonexistent_product(self, empty_cart):
        """Test removing a product that's not in cart."""
        with pytest.raises(KeyError, match="Product 999 not in cart"):
            empty_cart.remove_product(999)

    def test_update_quantity(self, cart_with_items):
        """Test updating product quantity."""
        cart_with_items.update_quantity(2, 5)  # Update mouse quantity
        assert cart_with_items.items[2] == 5

    def test_update_quantity_nonexistent_product(self, empty_cart):
        """Test updating quantity of non-existent product."""
        with pytest.raises(KeyError):
            empty_cart.update_quantity(999, 5)

    def test_update_quantity_invalid(self, cart_with_items):
        """Test updating to invalid quantity."""
        with pytest.raises(ValueError, match="Quantity must be positive"):
            cart_with_items.update_quantity(1, -1)

    def test_calculate_total_multiple_items(self, cart_with_items):
        """Test total calculation with multiple items."""
        # Cart has: 1x Laptop (999.99) + 2x Mouse (29.99)
        expected_total = 999.99 + (2 * 29.99)
        assert cart_with_items.get_total() == round(expected_total, 2)

    def test_clear_cart(self, cart_with_items):
        """Test clearing the cart."""
        cart_with_items.clear()
        assert cart_with_items.get_item_count() == 0
        assert cart_with_items.get_total() == 0.0
        assert len(cart_with_items.items) == 0

    @pytest.mark.parametrize("products,quantities,expected_total", [
        ([(1, "Item1", 10.00)], [1], 10.00),
        ([(1, "Item1", 10.00)], [5], 50.00),
        ([(1, "Item1", 10.00), (2, "Item2", 20.00)], [1, 1], 30.00),
        ([(1, "Item1", 9.99), (2, "Item2", 19.99)], [2, 3], 79.95),
    ])
    def test_various_totals(self, empty_cart, products, quantities, expected_total):
        """Test total calculation with various product combinations."""
        for (id, name, price), qty in zip(products, quantities):
            product = Product(id, name, price)
            empty_cart.add_product(product, qty)

        assert empty_cart.get_total() == expected_total
```

Run the complete test suite:

```bash
# Run all tests
pytest tests/test_shopping_cart.py -v

# Run with coverage
pytest tests/test_shopping_cart.py --cov=src.shopping_cart --cov-report=term-missing

# Run specific test class
pytest tests/test_shopping_cart.py::TestShoppingCart -v

# Run specific test
pytest tests/test_shopping_cart.py::TestShoppingCart::test_add_single_product -v
```

## Conclusion

Testing with Pytest provides a powerful, flexible, and elegant way to ensure code quality in Python applications. Key takeaways:

- **Start simple**: Basic assert statements are often enough
- **Use fixtures**: They make tests cleaner and more maintainable
- **Parametrize**: Avoid repetitive test code
- **Mock external dependencies**: Keep tests fast and isolated
- **Measure coverage**: But don't obsess over 100% coverage
- **Organize tests logically**: Use classes, modules, and markers
- **Integrate with CI/CD**: Automate testing in your workflow
- **Keep learning**: Explore plugins and advanced features

Pytest's ecosystem continues to grow, with plugins for almost every testing need. Whether you're testing a small utility function or a large web application, Pytest provides the tools to write comprehensive, maintainable tests that give you confidence in your code.

Remember: good tests are not just about finding bugs—they're about documenting behavior, enabling refactoring, and facilitating collaboration. Invest time in writing quality tests, and your future self (and teammates) will thank you.

Happy testing!