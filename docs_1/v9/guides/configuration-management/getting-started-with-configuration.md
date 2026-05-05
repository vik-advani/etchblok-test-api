---
title: Getting Started with Configuration
description: A step-by-step introduction to the class-based configuration system and setting up your first environment.
code_symbols: [SYM#2a6650db2a04b7eb03cfe02be64ee94b0e0e0e18, SYM#7dcc662114beb69932d02b031db76440ec4fc17c]
section_id: 69727874-6deb-4871-971c-a2bef4e2e7ba_getting_started_with_configuration
doc_type: tutorial
section_type: guide
---
This tutorial guides you through setting up and managing the environment-specific configurations for the Pagemark API. You will learn how to use the class-based configuration system to switch between development, testing, and production environments.

### Prerequisites

To follow this tutorial, you need the following dependencies installed (as used in this project):
- **Flask**: The core web framework.
- **Python 3.7+**: Required for the `dataclasses` module used in the configuration system.

### Step 1: Understand the Base Configuration

The foundation of the configuration system is the `BaseConfig` class located in `app/config.py`. This class defines the default settings shared across all environments.

```python
from dataclasses import dataclass, field
import os
from app.config import DEFAULT_PAGE_SIZE

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

By default, `BaseConfig` attempts to pull a `SECRET_KEY` from your environment variables. If one isn't found, it defaults to `"change-me"`. It also sets a standard `PAGE_SIZE` (defaulting to 25) and disables debug mode.

### Step 2: Configure for Local Development

When working locally, you want features like detailed error messages and smaller data sets for easier debugging. The `DevelopmentConfig` class inherits from `BaseConfig` and overrides these settings.

```python
from app.config import BaseConfig, _build_cache_config

@dataclass
class DevelopmentConfig(BaseConfig):
    """Configuration for local development."""

    DEBUG: bool = True
    PAGE_SIZE: int = 10

    def get_cache_config(self) -> Dict[str, Any]:
        return _build_cache_config(ttl=30, max_size=128)
```

In this environment, `DEBUG` is enabled, and the `PAGE_SIZE` is reduced to 10. The cache is also configured with a shorter TTL (30 seconds) and a smaller capacity (128 entries) to ensure you see changes quickly during development.

### Step 3: Prepare for Production

The `ProductionConfig` class is stricter than the others. Most importantly, it **requires** a `SECRET_KEY` to be set in the environment; it will raise a `KeyError` if it is missing.

```python
@dataclass
class ProductionConfig(BaseConfig):
    """Configuration for production deployments."""

    SECRET_KEY: str = field(default_factory=lambda: os.environ["SECRET_KEY"])
    PAGE_SIZE: int = DEFAULT_PAGE_SIZE

    def get_cache_config(self) -> Dict[str, Any]:
        return _build_cache_config(ttl=600, max_size=4096)
```

To run the application in production, you must export the secret key in your terminal:

```bash
export SECRET_KEY='your-very-secure-random-string'
```

### Step 4: Apply Configuration to the Application

The application factory in `app/__init__.py` is responsible for applying these configuration classes to the Flask instance using `app.config.from_object()`.

```python
from flask import Flask
from app.config import DevelopmentConfig, ProductionConfig

def create_app(config_class=DevelopmentConfig) -> Flask:
    """Application factory."""
    app = Flask(__name__)
    
    # Load settings from the configuration class
    app.config.from_object(config_class)

    # ... blueprint registrations ...
    return app
```

By default, `create_app()` uses `DevelopmentConfig`. To use a different environment, pass the class when calling the factory.

### Step 5: Launch the Server

In your entry point file (`run.py`), you can choose which configuration to load. For a standard local run, you can simply call `create_app()` without arguments:

```python
from app import create_app

# Defaults to DevelopmentConfig
app = create_app()

if __name__ == "__main__":
    app.run(debug=True, port=5000)
```

If you were deploying to production, you would modify your entry point or WSGI server to pass `ProductionConfig`:

```python
from app import create_app
from app.config import ProductionConfig

app = create_app(config_class=ProductionConfig)
```

### Verifying the Configuration

You can verify that your configuration is active by checking the `app.config` values. For example, if you are using `DevelopmentConfig`, the following check should pass:

```python
assert app.config['DEBUG'] is True
assert app.config['PAGE_SIZE'] == 10
```

If you attempt to start the app with `ProductionConfig` without setting the `SECRET_KEY` environment variable, the application will fail to start with a `KeyError`, ensuring your production environment is never accidentally insecure.