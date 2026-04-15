---
title: Configuring the Test Runner
description: How to apply specific settings for automated testing to ensure isolation and appropriate resource limits.
code_symbols: [SYM#2019837532635e08a2c17173935df96d18248622]
section_id: f3be8890-69ce-47df-b3df-cb182aa7c062_configuring_the_test_runner
doc_type: how_to
section_type: guide
---
To configure the test runner for this application, you must inject the `TestingConfig` class into the application factory and use the service-level reset utility to ensure state isolation between test cases.

### Initializing the App with Testing Configuration

Use the `create_app` factory from `app/__init__.py` to initialize the Flask application with `TestingConfig`. This ensures the `TESTING` flag is active and resource limits like `PAGE_SIZE` are reduced for easier assertions.

```python
from app import create_app
from app.config import TestingConfig

def test_setup():
    # Inject TestingConfig into the factory
    app = create_app(config_class=TestingConfig)
    app.testing = True
    
    with app.test_client() as client:
        # Run your tests here
        response = client.get('/_internal/health')
        assert response.status_code == 200
```

### Ensuring Test Isolation

The `BookmarkService` is a singleton that maintains an in-memory repository, cache, and search index. To prevent state leakage between tests, call the `_reset()` method in your test setup or teardown.

```python
from app.services.bookmark_service import BookmarkService

def setup_function():
    """Reset the singleton service state before every test."""
    service = BookmarkService()
    service._reset()

def test_create_bookmark():
    service = BookmarkService()
    bookmark, error = service.create_bookmark({"url": "https://example.com", "title": "Example"})
    assert bookmark is not None
    # Subsequent tests will start with a fresh repository due to setup_function
```

### Key Testing Settings

The `TestingConfig` class in `app/config.py` modifies the application behavior specifically for automated environments:

*   **`TESTING = True`**: Enables Flask's testing mode, which improves error reporting.
*   **`PAGE_SIZE = 5`**: Significantly lower than the default (25) or development (10) limits. This allows you to test pagination logic with fewer records.

```python
# app/config.py

@dataclass
class TestingConfig(BaseConfig):
    """Configuration for test runs."""

    TESTING: bool = True
    PAGE_SIZE: int = 5
```

### Handling Pagination in Tests

Because `TestingConfig` overrides the default `PAGE_SIZE`, your tests should account for a limit of 5 items when verifying paginated results from the `BookmarkService.list_bookmarks` method.

```python
from app.services.bookmark_service import BookmarkService
from app.config import TestingConfig

def test_pagination_limit():
    service = BookmarkService()
    # TestingConfig.PAGE_SIZE is 5
    bookmarks, total = service.list_bookmarks(page=1, per_page=TestingConfig.PAGE_SIZE)
    
    # Assertions should be based on the TestingConfig limit
    assert len(bookmarks) <= 5
```

### Troubleshooting: State Persistence

If you notice that bookmarks or tags created in one test are appearing in another, ensure that:
1.  You are not manually instantiating `BookmarkRepository` or `SearchIndex` in your tests, as `BookmarkService` manages these internally.
2.  You are explicitly calling `BookmarkService()._reset()` between tests. Since the database is in-memory, it will persist for the lifetime of the process unless re-initialized via `_reset()`.