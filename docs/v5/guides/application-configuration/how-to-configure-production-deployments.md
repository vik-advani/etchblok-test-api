---
title: How to Configure Production Deployments
description: Practical instructions for setting up production environments, including mandatory environment variables like SECRET_KEY.
code_symbols: [SYM#d2d22b366491d800917a8a1043152349a435eed9]
section_id: 58daf313-7164-463a-83f7-0c9b5b140780_how_to_configure_production_deployments
doc_type: how_to
section_type: guide
---
To prepare the application for a production environment, you must use the `ProductionConfig` class and ensure that mandatory environment variables are set. This configuration optimizes caching and enforces security requirements that are relaxed in development.

### Basic Production Setup

To run the application in production, initialize the application factory with the `ProductionConfig` class and provide the required `SECRET_KEY` via the environment.

```python
import os
from app import create_app
from app.config import ProductionConfig

# Ensure SECRET_KEY is set in the environment
# os.environ["SECRET_KEY"] = "your-secure-random-string"

# Initialize the app with ProductionConfig
app = create_app(config_class=ProductionConfig)
```

### Mandatory Configuration: SECRET_KEY

Unlike `BaseConfig` or `DevelopmentConfig`, which provide a default value of `"change-me"`, the `ProductionConfig` class in `app/config.py` forces the use of an environment variable. 

```python
@dataclass
class ProductionConfig(BaseConfig):
    """Configuration for production deployments."""

    SECRET_KEY: str = field(default_factory=lambda: os.environ["SECRET_KEY"])
    # ...
```

If `SECRET_KEY` is not present in the environment when the application starts, the `default_factory` will raise a `KeyError`, preventing the application from running with insecure defaults.

### Production Defaults and Caching

The `ProductionConfig` overrides several settings to optimize performance for high-traffic environments:

1.  **Pagination**: Sets `PAGE_SIZE` to `DEFAULT_PAGE_SIZE` (25), whereas `DevelopmentConfig` reduces this to 10.
2.  **Caching**: The `get_cache_config()` method returns a configuration with a longer TTL and a larger entry limit.

```python
def get_cache_config(self) -> Dict[str, Any]:
    # Production uses 600s TTL and 4096 max entries
    return _build_cache_config(ttl=600, max_size=4096)
```

The internal helper `_build_cache_config` (found in `app/config.py`) converts these parameters into a dictionary used by the application's internal services:
`{"ttl_seconds": 600, "max_entries": 4096, "eviction": "lru"}`.

### Switching Environments Dynamically

A common pattern in this codebase is to toggle the configuration class based on an environment variable like `FLASK_ENV`. You can implement this in your entry point (e.g., `run.py` or a WSGI wrapper):

```python
import os
from app import create_app
from app.config import ProductionConfig, DevelopmentConfig

env = os.environ.get("FLASK_ENV", "development")

if env == "production":
    app = create_app(config_class=ProductionConfig)
else:
    app = create_app(config_class=DevelopmentConfig)
```

### Troubleshooting

#### KeyError: 'SECRET_KEY'
If you see `KeyError: 'SECRET_KEY'` during startup, it means you are using `ProductionConfig` but have not exported the environment variable. 
**Solution**: Run `export SECRET_KEY='your-key'` in your shell or include it in your `.env` file before starting the server.

#### Cache Size Discrepancies
Note that while `ProductionConfig` defines a cache size of 4096 via `get_cache_config()`, the `BookmarkService` in `app/services/bookmark_service.py` currently initializes its internal `LRUCache` with a hardcoded limit of 256:

```python
# app/services/bookmark_service.py
def _init_services(self) -> None:
    self._repo = BookmarkRepository()
    self._cache: LRUCache[Bookmark] = LRUCache(max_size=256) # Hardcoded limit
    self._search = SearchIndex(self._repo)
```

If you notice unexpected cache misses in production despite increasing the limit in `ProductionConfig`, be aware that the service layer may need to be updated to consume the configuration value.

#### Validation Failures
The `BaseConfig` includes a `_validate()` method that checks if the `PAGE_SIZE` exceeds `MAX_PAGE_SIZE` (100). If you customize the `ProductionConfig` and set a `PAGE_SIZE` higher than 100, internal validation will fail.

```python
# app/config.py
def _validate(self) -> bool:
    return bool(self.SECRET_KEY) and self.PAGE_SIZE <= MAX_PAGE_SIZE
```