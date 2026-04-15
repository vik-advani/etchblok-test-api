---
section_type: guide
---
# Getting started

Pagemark API is a bookmark management REST API built with Flask. It allows you to save, organize, and search bookmarks with tagging and collections using an in-memory data store.

## Prerequisites

- **Python 3.8+**
- **pip** (Python package manager)

## Install

1. **Clone the repository** (or navigate to the project root).
2. **Create and activate a virtual environment**:
   ```bash
   python -m venv venv
   source venv/bin/activate  # Windows: venv\Scripts\activate
   ```
3. **Install dependencies**:
   ```bash
   pip install -r requirements.txt
   ```

## Hello world

To start the API server, run the provided entry point:

```bash
python run.py
```

The server will start on `http://localhost:5000`. You can create your first bookmark using `curl`:

```bash
curl -X POST http://localhost:5000/api/bookmarks/ \
     -H "Content-Type: application/json" \
     -d '{"url": "https://github.com", "title": "GitHub"}'
```

## Configure

The application uses `python-dotenv` to manage configuration. You can create a `.env` file in the root directory to override defaults:

| Variable | Default | Description |
|----------|---------|-------------|
| `SECRET_KEY` | `change-me` | Used for session security (required for production) |
| `FLASK_ENV` | `development` | Set to `production` for production settings |

## Verify

You can verify the service is running by hitting the internal health check endpoint:

```bash
curl http://localhost:5000/_internal/health
```

Alternatively, list all bookmarks to ensure the repository is active:

```bash
curl http://localhost:5000/api/bookmarks/
```

## Next steps

- Explore the full list of [API Reference](/api_ref/app)
- Learn how to organize bookmarks in the [Organizing Bookmarks with Tags and Collections](/guides/bookmark-management-service/organizing-bookmarks-with-tags-and-collections)
- Configure production settings in the [Application Configuration](/guides/application-configuration)

## Troubleshooting

### Port 5000 already in use
If you receive an "Address already in use" error, another process is using port 5000 (common on macOS). You can change the port in `run.py` or stop the conflicting service.

### Missing Secret Key
In production mode, the application will fail to start if `SECRET_KEY` is not set in your environment variables. Ensure you have a secure key defined in your `.env` file.