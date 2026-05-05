---
title: Configuring for Local Development
description: How to set up your local environment with debug mode enabled and optimized cache settings for rapid iteration.
code_symbols: [SYM#de19a9e9116dcd6a24f5962096a9ebb6f40d5857]
section_id: a00ef9a2-ffaa-4772-9119-e16f9337d693_configuring_for_local_development
doc_type: how_to
section_type: guide
---
To set up your local environment for rapid iteration, use the `DevelopmentConfig` class. This configuration enables auto-reloading, detailed error pages, and optimized cache settings tailored for development.

## Starting the Application Locally

The simplest way to start the application with development settings is to execute the `run.py` script. This script initializes the application using the default `DevelopmentConfig`.

```python
# run.py
from app import create_app

# create_app defaults to DevelopmentConfig
app = create_app()

if __name__ == "__main__":
    # Explicitly enables debug mode for the Flask development server
    app.run(debug=True, port=5000)
```

## Development Configuration Settings

The `DevelopmentConfig` class in `app/config.py` inherits from `BaseConfig` and overrides several settings to improve the developer experience.

```python
# app/config.py
@dataclass
class DevelopmentConfig(BaseConfig):
    """Configuration for local development."""

    DEBUG: bool = True
    PAGE_SIZE: int = 10

    def get_cache_config(self) -> Dict[str, Any]:
        return _build_cache_config(ttl=30, max_size=128)
```

### Key Features
- **Debug Mode**: `DEBUG` is set to `True`, which enables Flask's interactive debugger and the auto-reloader (restarts the server when code changes).
- **Pagination**: `PAGE_SIZE` is reduced to `10` (from the default `25`). This makes it easier to test pagination logic with smaller datasets.
- **Cache Optimization**: The `get_cache_config()` method returns a shorter TTL (30 seconds) and a smaller capacity (128 entries) to ensure that stale data doesn't persist long during testing.

## Customizing Environment Variables

While `DevelopmentConfig` provides sensible defaults, you can still override core settings like the `SECRET_KEY` using environment variables. `BaseConfig` (the parent of `DevelopmentConfig`) handles these lookups.

```python
# app/config.py snippet
@dataclass
class BaseConfig:
    SECRET_KEY: str = field(default_factory=lambda: os.environ.get("SECRET_KEY", "change-me"))
    # ...
```

To use a custom secret key locally, set the environment variable before running the app:

```bash
export SECRET_KEY="your-local-dev-key"
python run.py
```

## Troubleshooting

### Cache Settings Not Reflecting
You may notice that changes to `get_cache_config()` in `DevelopmentConfig` do not affect the `BookmarkService`. This is because the `BookmarkService` currently hardcodes its internal `LRUCache` size during initialization.

```python
# app/services/bookmark_service.py
def _init_services(self) -> None:
    """Bootstrap repository, cache, and search index."""
    self._repo = BookmarkRepository()
    # Note: max_size is hardcoded to 256, ignoring DevelopmentConfig
    self._cache: LRUCache[Bookmark] = LRUCache(max_size=256)
    self._search = SearchIndex(self._repo)
```

If you need to test specific cache capacities, you must temporarily modify the `max_size` parameter in `app/services/bookmark_service.py`.

### Debug Mode Conflicts
The `run.py` script calls `app.run(debug=True)`. This will override the `DEBUG` setting in your config object if they differ. Ensure both are set to `True` for a consistent development experience.