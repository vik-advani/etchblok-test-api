---
title: Setting Up Your Development Environment
description: A step-by-step tutorial for newcomers to configure their local machine for development using the built-in defaults.
code_symbols: [SYM#de19a9e9116dcd6a24f5962096a9ebb6f40d5857]
section_id: b1dd0ece-08ea-4c5c-87b5-5c97b2b44f93_setting_up_your_development_environment
doc_type: tutorial
section_type: guide
---
This tutorial guides you through setting up your local environment to run the Pagemark API. You will use the built-in `DevelopmentConfig` to ensure the application runs with appropriate settings for debugging and local testing.

### Prerequisites

Before starting, ensure you have the following:
- Python 3.8 or higher installed.
- A virtual environment tool (like `venv` or `virtualenv`).

### Step 1: Install Dependencies

First, clone the repository and install the required packages. The application relies on Flask and dataclasses for its configuration management.

```bash
# Create and activate a virtual environment
python -m venv venv
source venv/bin/activate  # On Windows use `venv\Scripts\activate`

# Install dependencies
pip install -r requirements.txt
```

### Step 2: Understand the Development Defaults

The project uses a specialized configuration class for local work found in `app/config.py`. When you start the app without specifying a configuration, it defaults to `DevelopmentConfig`.

This class provides several helpful defaults for developers:
- **`DEBUG`**: Set to `True` to enable the Flask debugger and auto-reloading.
- **`PAGE_SIZE`**: Set to `10` (instead of the default 25) to make testing pagination logic easier with fewer records.
- **Cache Settings**: Uses a shorter TTL (30 seconds) and a smaller maximum size (128 entries) to ensure you see changes quickly during development.

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

### Step 3: Configure Environment Variables

While `DevelopmentConfig` works out of the box, it relies on `BaseConfig` for certain security settings. The most important is the `SECRET_KEY`.

In `app/config.py`, the `SECRET_KEY` is retrieved from your environment:

```python
# app/config.py snippet
SECRET_KEY: str = field(default_factory=lambda: os.environ.get("SECRET_KEY", "change-me"))
```

For local development, it defaults to `"change-me"`. If you need to simulate a more secure environment, you can export this variable in your terminal:

```bash
export SECRET_KEY='your-custom-dev-key'
```

### Step 4: Launch the Application

The project includes a `run.py` script in the root directory. This script uses the `create_app` factory from `app/__init__.py`, which automatically applies the `DevelopmentConfig` by default.

```python
# run.py
from app import create_app

app = create_app()

if __name__ == "__main__":
    app.run(debug=True, port=5000)
```

Run the server with the following command:

```bash
python run.py
```

### Step 5: Verify the Setup

Once the server is running, you should see output indicating that the debugger is active. You can verify the API is responding correctly by hitting the internal health endpoint:

```bash
curl http://127.0.0.1:5000/_internal/health
```

If successful, you will receive a JSON response indicating the service is healthy. You are now ready to begin developing against the Pagemark API.

### Next Steps

- Explore the `app/routes/` directory to see available endpoints.
- Review `app/config.py` if you need to adjust the `MAX_PAGE_SIZE` or other global constants.
- Use the `TestingConfig` class in `app/config.py` when writing unit tests to ensure `TESTING` mode is active.