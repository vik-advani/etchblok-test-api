---
title: Configuration Architecture
description: An overview of the inheritance-based configuration system, explaining how environment-specific settings extend the base configuration.
code_symbols: [SYM#2a6650db2a04b7eb03cfe02be64ee94b0e0e0e18, SYM#7dcc662114beb69932d02b031db76440ec4fc17c]
section_id: e5498863-c05c-4254-858b-9a1913d85402_configuration_architecture
doc_type: guide
section_type: guide
---
The configuration system in this codebase is built on a hierarchical structure using Python's `dataclasses`. This approach ensures type safety and provides a clear inheritance path for environment-specific overrides.

## The Configuration Hierarchy

The architecture centers around the `BaseConfig` class located in `app/config.py`. This class defines the shared defaults and the structure that all environment-specific configurations must follow.

### Base Configuration

`BaseConfig` serves as the blueprint for the application's settings. It utilizes `dataclasses.field` and `default_factory` to handle dynamic defaults, such as reading from environment variables.

```python
@dataclass
class BaseConfig:
    """Base configuration shared across all environments."""

    SECRET_KEY: str = field(default_factory=lambda: os.environ.get("SECRET_KEY", "change-me"))
    DEBUG: bool = False
    TESTING: bool = False
    PAGE_SIZE: int = DEFAULT_PAGE_SIZE

    def get_cache_config(self) -> Dict[str, Any]:
        """Return cache settings for this environment."""
        return _build_cache_config()
```

Key attributes include:
- **SECRET_KEY**: Defaults to "change-me" unless provided via the `SECRET_KEY` environment variable.
- **PAGE_SIZE**: Controlled by the `DEFAULT_PAGE_SIZE` constant (set to 25).
- **Validation**: The internal `_validate()` method ensures that the `SECRET_KEY` is present and that `PAGE_SIZE` does not exceed the `MAX_PAGE_SIZE` limit of 100.

## Environment Specializations

Specific environments extend `BaseConfig` to override defaults or enforce stricter requirements.

### Development Configuration
`DevelopmentConfig` is optimized for local iteration. It enables `DEBUG` mode and reduces the `PAGE_SIZE` to 10 to make manual testing of pagination easier. It also overrides `get_cache_config` to provide a shorter TTL (30 seconds) and a smaller cache size (128 entries).

### Production Configuration
`ProductionConfig` enforces security and performance standards:
- **Strict Secret Key**: Unlike the base class, it uses `os.environ["SECRET_KEY"]` without a default. This causes the application to fail fast with a `KeyError` if the secret key is missing in production.
- **Optimized Cache**: It increases the cache TTL to 600 seconds and the maximum size to 4096 entries via `get_cache_config`.

### Testing Configuration
`TestingConfig` is designed for the test suite. It sets the `TESTING` flag to `True` and minimizes the `PAGE_SIZE` to 5 to speed up test execution and verify pagination logic with minimal data.

## Application Integration

The configuration classes are integrated into the Flask application via the application factory in `app/__init__.py`. The `create_app` function accepts a `config_class` argument, which defaults to `DevelopmentConfig`.

```python
def create_app(config_class=DevelopmentConfig) -> Flask:
    """Application factory."""
    app = Flask(__name__)
    app.config.from_object(config_class)
    
    # ... registration of blueprints ...
    
    return app
```

By using `app.config.from_object(config_class)`, Flask inspects the dataclass and loads all uppercase attributes into its own configuration dictionary.

## Cache Configuration Architecture

The system uses an internal helper function, `_build_cache_config`, to standardize the structure of cache settings across different environments.

```python
def _build_cache_config(ttl: int = 300, max_size: int = 1024) -> Dict[str, Any]:
    """Build cache configuration dict."""
    return {"ttl_seconds": ttl, "max_entries": max_size, "eviction": "lru"}
```

Each configuration class can customize these settings by overriding the `get_cache_config()` method. This allows the application to adapt its caching strategy (e.g., shorter TTLs in development vs. longer TTLs in production) while maintaining a consistent return format. Although defined in the configuration layer, these settings are intended to be consumed by services requiring caching logic.