---
title: Setting up the Test Environment
description: Instructions for configuring the application to run automated tests with isolated settings.
code_symbols: [SYM#2019837532635e08a2c17173935df96d18248622]
section_id: ef663746-57a7-46c0-817d-564e615d97be_setting_up_the_test_environment
doc_type: how_to
section_type: guide
---
To configure the application for automated testing with isolated settings, use the `TestingConfig` class when initializing the application factory.

### Initialize the App with Testing Configuration

The most direct way to set up the test environment is to pass the `TestingConfig` class to the `create_app` factory function. This ensures that the application runs with `TESTING=True` and a reduced `PAGE_SIZE` for easier pagination testing.

```python
from app import create_app
from app.config import TestingConfig

def test_example():
    # Initialize the app with the specialized testing config
    app = create_app(config_class=TestingConfig)
    
    # Create a test client for making requests
    with app.test_client() as client:
        response = client.get("/_internal/health")
        assert response.status_code == 200
```

### Configuration Details

The `TestingConfig` class in `app/config.py` provides specific overrides for the test environment:

*   **`TESTING`**: Set to `True`. This enables Flask's debug mode for tests, allowing for better error reporting.
*   **`PAGE_SIZE`**: Set to `5`. This is significantly lower than the `DevelopmentConfig` (10) or `ProductionConfig` (25), making it easier to test pagination logic with a small number of records.

```python
@dataclass
class TestingConfig(BaseConfig):
    """Configuration for test runs."""

    TESTING: bool = True
    PAGE_SIZE: int = 5
```

### Using Pytest Fixtures

For a standard test suite, it is recommended to define a pytest fixture in a `conftest.py` file to reuse the application instance across tests.

```python
import pytest
from app import create_app
from app.config import TestingConfig

@pytest.fixture
def app():
    """Create and configure a new app instance for each test."""
    app = create_app(TestingConfig)
    yield app

@pytest.fixture
def client(app):
    """A test client for the app."""
    return app.test_client()

def test_bookmark_pagination(client):
    # With PAGE_SIZE=5, we only need 6 items to test pagination
    response = client.get("/api/bookmarks")
    assert response.status_code == 200
```

### Overriding Settings for Specific Tests

If you need to modify a specific setting for a single test (e.g., testing a different `PAGE_SIZE`), you can subclass `TestingConfig` or modify the `app.config` object directly after creation.

```python
from app import create_app
from app.config import TestingConfig
from dataclasses import dataclass

@dataclass
class LargePageConfig(TestingConfig):
    PAGE_SIZE: int = 50

def test_large_page_size():
    app = create_app(LargePageConfig)
    assert app.config["PAGE_SIZE"] == 50
```

### Troubleshooting

**Secret Key Warnings**
By default, `TestingConfig` inherits `SECRET_KEY` from `BaseConfig`, which defaults to the string `"change-me"` if the `SECRET_KEY` environment variable is not set. While this is sufficient for local testing, ensure your CI environment does not accidentally use `ProductionConfig`, which will raise a `KeyError` if the environment variable is missing:

```python
# From app/config.py - ProductionConfig requires the env var
@dataclass
class ProductionConfig(BaseConfig):
    SECRET_KEY: str = field(default_factory=lambda: os.environ["SECRET_KEY"])
```

**Pagination Logic**
If your tests are failing because they expect more than 5 items per page, remember that `TestingConfig` explicitly sets `PAGE_SIZE` to 5. Verify your test assertions against this value or use a custom configuration class as shown in the variations section.