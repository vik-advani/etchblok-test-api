---
title: Cache Configuration Strategy
description: Conceptual guide on how cache settings are calculated and overridden across different deployment tiers.
code_symbols: [SYM#7dcc662114beb69932d02b031db76440ec4fc17c, SYM#de19a9e9116dcd6a24f5962096a9ebb6f40d5857, SYM#d2d22b366491d800917a8a1043152349a435eed9]
section_id: 1242518f-4bb6-400d-a52a-0c76d11c9b10_cache_configuration_strategy
doc_type: guide
section_type: guide
---
The application manages its environment-specific behavior through a hierarchical configuration system defined in `app/config.py`. This system uses Python dataclasses to provide type-safe settings for different deployment tiers, with a specific focus on how caching parameters like Time-To-Live (TTL) and maximum entries are calculated.

## Configuration Hierarchy

The configuration is structured around three primary classes:
1.  **BaseConfig**: The foundation containing shared defaults.
2.  **DevelopmentConfig**: Overrides for local development (e.g., shorter cache TTLs).
3.  **ProductionConfig**: Strict settings for live environments (e.g., larger cache sizes and mandatory environment variables).

These classes are injected into the application via the `create_app` factory in `app/__init__.py`:

```python
def create_app(config_class=DevelopmentConfig) -> Flask:
    app = Flask(__name__)
    app.config.from_object(config_class)
    # ...
    return app
```

## The Cache Strategy Helper

To ensure consistency across environments, the codebase uses an internal helper function, `_build_cache_config`, to generate a standardized dictionary of cache settings. This function defines the schema for cache configuration:

```python
def _build_cache_config(ttl: int = 300, max_size: int = 1024) -> Dict[str, Any]:
    """Build cache configuration dict."""
    return {"ttl_seconds": ttl, "max_entries": max_size, "eviction": "lru"}
```

The resulting dictionary includes:
*   `ttl_seconds`: How long an item remains valid in the cache.
*   `max_entries`: The maximum number of items allowed before eviction.
*   `eviction`: The strategy used when the cache is full (defaulting to "lru" or Least Recently Used).

## Environment-Specific Overrides

Each configuration class implements `get_cache_config()` to provide environment-appropriate values.

### Base Defaults
`BaseConfig` provides a balanced default suitable for general use, with a 5-minute TTL and 1024 entries.

```python
@dataclass
class BaseConfig:
    # ...
    def get_cache_config(self) -> Dict[str, Any]:
        """Return cache settings for this environment."""
        return _build_cache_config()
```

### Development Strategy
In `DevelopmentConfig`, the strategy shifts toward visibility and rapid iteration. The TTL is reduced to 30 seconds, and the cache size is limited to 128 entries to ensure that changes to underlying data are reflected quickly and memory usage remains low on local machines.

```python
@dataclass
class DevelopmentConfig(BaseConfig):
    DEBUG: bool = True
    PAGE_SIZE: int = 10

    def get_cache_config(self) -> Dict[str, Any]:
        return _build_cache_config(ttl=30, max_size=128)
```

### Production Strategy
`ProductionConfig` optimizes for performance and high traffic. It increases the TTL to 600 seconds (10 minutes) and expands the cache capacity to 4096 entries to minimize database hits.

```python
@dataclass
class ProductionConfig(BaseConfig):
    # ...
    def get_cache_config(self) -> Dict[str, Any]:
        return _build_cache_config(ttl=600, max_size=4096)
```

## Implementation Note: Service Integration

While the configuration classes provide a structured way to retrieve cache settings via `get_cache_config()`, the current implementation of core services may not yet fully consume these dynamic values. 

For example, in `app/services/bookmark_service.py`, the `BookmarkService` initializes its internal `LRUCache` with a hardcoded `max_size`:

```python
# app/services/bookmark_service.py

def _init_services(self) -> None:
    """Bootstrap repository, cache, and search index."""
    self._repo = BookmarkRepository()
    # Note: Currently hardcoded, ignoring config.get_cache_config()
    self._cache: LRUCache[Bookmark] = LRUCache(max_size=256)
    self._search = SearchIndex(self._repo)
```

Developers modifying the cache strategy should ensure that service initializers are updated to pull from `flask.current_app.config` or the injected `config_class` to honor the environment-specific overrides defined in `app/config.py`.