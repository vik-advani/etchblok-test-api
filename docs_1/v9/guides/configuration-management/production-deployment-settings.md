---
title: Production Deployment Settings
description: A guide to production-ready configurations, focusing on security through environment variables and high-performance caching.
code_symbols: [SYM#d2d22b366491d800917a8a1043152349a435eed9]
section_id: 42c20c37-42e6-4b78-89a5-bf99f4a43629_production_deployment_settings
doc_type: guide
section_type: guide
---
The `ProductionConfig` class in `app/config.py` provides a specialized configuration for production environments. It prioritizes security by enforcing the presence of environment variables and optimizes performance through tuned caching and pagination settings.

## Security Enforcement

In production, the application requires a strict security posture. Unlike `BaseConfig` or `DevelopmentConfig`, which provide default values for sensitive settings, `ProductionConfig` mandates that critical secrets be provided by the environment.

### Mandatory Secret Key
The `SECRET_KEY` is used for cryptographic signing. In `ProductionConfig`, this is implemented using a `default_factory` that calls `os.environ["SECRET_KEY"]`.

```python
@dataclass
class ProductionConfig(BaseConfig):
    """Configuration for production deployments."""

    SECRET_KEY: str = field(default_factory=lambda: os.environ["SECRET_KEY"])
    # ...
```

Because it uses `os.environ` directly (rather than `os.environ.get`), the application will raise a `KeyError` and fail to start if the `SECRET_KEY` environment variable is missing. This prevents the application from running in an insecure state with a default or empty key.

## Performance Tuning

`ProductionConfig` overrides default settings to handle higher traffic volumes and larger datasets more efficiently.

### High-Performance Caching
The configuration uses the internal `_build_cache_config` helper to define optimized cache parameters. Compared to the development environment (which uses a 30-second TTL and 128 entries), the production settings are significantly more robust:

*   **TTL (Time-To-Live):** 600 seconds (10 minutes).
*   **Max Size:** 4096 entries.
*   **Eviction Policy:** LRU (Least Recently Used).

```python
def get_cache_config(self) -> Dict[str, Any]:
    return _build_cache_config(ttl=600, max_size=4096)
```

### Pagination Limits
The `PAGE_SIZE` is set to `DEFAULT_PAGE_SIZE` (25), ensuring consistent response sizes. While `BaseConfig` includes a `_validate()` method to ensure `PAGE_SIZE` does not exceed `MAX_PAGE_SIZE` (100), this validation must be manually invoked if configuration is modified at runtime.

## Application Integration

To deploy with production settings, the `ProductionConfig` class must be passed to the `create_app` factory in `app/__init__.py`.

```python
from app import create_app
from app.config import ProductionConfig

# Initialize the app with production settings
app = create_app(config_class=ProductionConfig)
```

The factory uses `app.config.from_object(config_class)` to load these settings into the Flask application's configuration dictionary.

## Implementation Notes and Gotchas

### Cache Configuration Discrepancy
While `ProductionConfig.get_cache_config()` defines a capacity of 4096 entries, the current implementation of `BookmarkService` (found in `app/services/bookmark_service.py`) does not consume this configuration. Instead, it initializes its internal `LRUCache` with a hardcoded size:

```python
# app/services/bookmark_service.py
def _init_services(self) -> None:
    """Bootstrap repository, cache, and search index."""
    self._repo = BookmarkRepository()
    self._cache: LRUCache[Bookmark] = LRUCache(max_size=256) # Hardcoded size
    self._search = SearchIndex(self._repo)
```

Developers should be aware that increasing the cache size in `ProductionConfig` will not affect the `BookmarkService` until the service is updated to read from the application configuration.

### Internal Validation
The `BaseConfig` class provides a `_validate` method that checks if the `SECRET_KEY` is present and if the `PAGE_SIZE` is within the allowed bounds (`MAX_PAGE_SIZE`).

```python
def _validate(self) -> bool:
    """Check internal invariants. Not part of the public API."""
    return bool(self.SECRET_KEY) and self.PAGE_SIZE <= MAX_PAGE_SIZE
```

This method is intended for internal sanity checks and is not automatically executed by Flask during the `from_object` initialization.