---
title: Application Configuration
description: Manage environment-specific settings, including security keys, debugging flags, and cache parameters.
code_symbols: [SYM#2a6650db2a04b7eb03cfe02be64ee94b0e0e0e18]
section_id: 0c56b5cd-e472-4b59-98e7-dd605d79efba_application_configuration
doc_type: how_to
section_type: guide
---
Manage environment-specific settings in this project using the configuration classes defined in `app/config.py`. These classes handle security keys, debugging flags, and pagination limits for different deployment stages.

## Configure the Application Environment

To apply a specific configuration, pass the desired class to the `create_app` factory in `app/__init__.py`. By default, the application uses `DevelopmentConfig`.

```python
from app import create_app
from app.config import ProductionConfig, TestingConfig

# For Production
app = create_app(config_class=ProductionConfig)

# For Testing
app = create_app(config_class=TestingConfig)
```

The application factory uses `app.config.from_object(config_class)` to load these settings into the Flask `app.config` dictionary.

## Configuration Classes

The project uses a hierarchy of dataclasses to manage settings:

### Base Configuration
`BaseConfig` defines the default values shared across all environments.

*   **SECRET_KEY**: Defaults to `"change-me"` (via `os.environ.get`).
*   **DEBUG**: Defaults to `False`.
*   **TESTING**: Defaults to `False`.
*   **PAGE_SIZE**: Defaults to `25` (defined by `DEFAULT_PAGE_SIZE`).

### Development Configuration
`DevelopmentConfig` inherits from `BaseConfig` and overrides settings for local work:
*   `DEBUG = True`
*   `PAGE_SIZE = 10`
*   Cache TTL: 30s, Max Size: 128

### Production Configuration
`ProductionConfig` is strictly for live environments and enforces security:
*   **SECRET_KEY**: Must be provided via the `SECRET_KEY` environment variable. It will raise a `KeyError` if missing.
*   `PAGE_SIZE = 25`
*   Cache TTL: 600s, Max Size: 4096

### Testing Configuration
`TestingConfig` is optimized for automated test suites:
*   `TESTING = True`
*   `PAGE_SIZE = 5`

## Accessing Cache Settings

Each configuration class provides a `get_cache_config()` method that returns a dictionary of cache parameters (TTL, max entries, and eviction policy).

```python
from app.config import DevelopmentConfig

config = DevelopmentConfig()
cache_settings = config.get_cache_config()
# Returns: {'ttl_seconds': 30, 'max_entries': 128, 'eviction': 'lru'}
```

## Production Deployment Setup

When deploying to production, you must set the `SECRET_KEY` environment variable. The `ProductionConfig` class uses a `default_factory` that accesses `os.environ` directly without a fallback.

```bash
# Example: Setting the environment variable before running the app
export SECRET_KEY="your-highly-secure-random-string"
python run.py
```

## Troubleshooting

### Missing SECRET_KEY in Production
If you attempt to start the application with `ProductionConfig` and the `SECRET_KEY` environment variable is not set, the application will fail to initialize with a `KeyError`. Ensure your deployment environment (Docker, Heroku, etc.) has this variable defined.

### Cache Size Inconsistency
Note that while `BaseConfig` and its subclasses define cache settings via `get_cache_config()`, the `BookmarkService` currently initializes its internal `LRUCache` with a hardcoded `max_size=256` in `app/services/bookmark_service.py`:

```python
# app/services/bookmark_service.py
def _init_services(self) -> None:
    """Bootstrap repository, cache, and search index."""
    self._repo = BookmarkRepository()
    self._cache: LRUCache[Bookmark] = LRUCache(max_size=256) # Hardcoded value
    self._search = SearchIndex(self._repo)
```

To align the service with your configuration, you must manually update this value or refactor the service to accept configuration parameters.

### Validation
The `BaseConfig` class includes an internal `_validate()` method that checks if `SECRET_KEY` is present and `PAGE_SIZE` is within the `MAX_PAGE_SIZE` (100) limit. This method is not automatically called by Flask; if you need to enforce these invariants at startup, call it manually in your entry point.