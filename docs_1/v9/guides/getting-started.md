---
section_type: guide
---
Pagemark API is a bookmark management REST API built with Flask. It provides a layered architecture for saving, organizing, and searching bookmarks with tagging and collections.

## Prerequisites

- **Python 3.8+**
- **pip** (Python package manager)

## Installation

1. **Clone the repository** (if you haven't already):
   ```bash
   git clone <repository-url>
   cd pagemark-api
   ```

2. **Create a virtual environment** (recommended):
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies**:
   ```bash
   pip install -r requirements.txt
   ```

## Hello World / Quick Start

### 1. Run the Server
Start the Flask application using the provided entry point:

```bash
python run.py
```
The API will be available at `http://localhost:5000`.

### 2. Create Your First Bookmark
Use `curl` to save a new bookmark:

```bash
curl -X POST http://localhost:5000/api/bookmarks/ \
     -H "Content-Type: application/json" \
     -d '{"url": "https://flask.palletsprojects.com", "title": "Flask Documentation"}'
```

### 3. List Bookmarks
Retrieve all saved bookmarks:

```bash
curl http://localhost:5000/api/bookmarks/
```

## Configuration

The application uses environment variables for configuration. You can set these in a `.env` file in the root directory.

| Variable | Description | Default |
|----------|-------------|---------|
| `SECRET_KEY` | Used for session security and signing. | `change-me` |
| `FLASK_ENV` | Set to `development` or `production`. | `development` |

### Production Setup
For production, ensure you set a secure `SECRET_KEY`:

```bash
export SECRET_KEY=$(python -c 'import os; print(os.urandom(24).hex())')
```

## Verify Installation

You can verify that the API is running and healthy by hitting the internal health endpoint:

```bash
curl http://localhost:5000/_internal/health
```

Expected response:
```json
{"status": "ok"}
```

To check if the core services (Repository, Search Index) are ready:
```bash
curl http://localhost:5000/_internal/ready
```

## Next Steps

- **Explore the API**: See the full list of endpoints in the [Overview](/guides/overview) for tagging and collections.
- **Search**: Try the search endpoint at `/api/bookmarks/search?q=flask`.
- **Architecture**: Learn about the internal structure in the `app/` directory, including `app.services.bookmark_service.BookmarkService` and `app.db.repository.BookmarkRepository`.
