---
section_type: guide
---
Pagemark API is a lightweight, layered REST service built with Flask for managing personal bookmarks. It provides a structured way to save URLs, organize them with tags and collections, and find them instantly using a built-in full-text search engine.

## The "Digital Attic" Problem
Most people save links in browser bookmarks, chat apps, or "read later" services, but these often become "black holes" where information is hard to retrieve or organize programmatically. Pagemark API provides a clean, developer-friendly backend to build your own bookmarking tools, browser extensions, or personal knowledge bases.

## Core Concepts

*   **Bookmarks**: The primary entity. Each bookmark contains a URL, title, description, and a set of tags. They can be in `active`, `archived`, or `trashed` states.
*   **Tags**: Flat labels used for cross-cutting organization (e.g., "python", "recipes", "work").
*   **Collections**: Named groups of bookmarks. They can be **Manual** (you add bookmarks explicitly) or **Smart** (defined by a filter rule, though currently primarily a data model concept).
*   **Search Index**: An in-memory inverted index that tokenizes titles and descriptions for fast, relevance-ranked retrieval.
*   **Layered Architecture**: The project strictly separates concerns into Routes (HTTP), Services (Business Logic), and Repositories (Data Access).

## How it Works
The application follows a classic three-tier service architecture:

1.  **API Layer**: Flask Blueprints in `app.routes` handle request parsing and JSON serialization.
2.  **Service Layer**: The `BookmarkService` (a singleton) orchestrates all operations, including validation, cache management, and search indexing.
3.  **Data Layer**: The `BookmarkRepository` abstracts the storage. In the current version, this is an **in-memory** store using Python dictionaries.
4.  **Caching & Search**: An `LRUCache` speeds up frequent lookups, while the `SearchIndex` maintains a map of keywords to bookmark IDs.

## Use Cases

### Creating a Bookmark
You can create a bookmark by sending a POST request. The service validates the URL and title before indexing it for search.

```python
# Example using the internal service directly
from app.services.bookmark_service import BookmarkService

service = BookmarkService()
bookmark, error = service.create_bookmark({
    "url": "https://flask.palletsprojects.com/",
    "title": "Flask Documentation",
    "description": "The official docs for the Flask web framework."
})

if bookmark:
    print(f"Created bookmark: {bookmark.id}")
```

### Searching Bookmarks
The search engine performs an "AND" search across tokens and ranks results by frequency.

```python
# Search for bookmarks containing 'flask' and 'docs'
results = service.search("flask docs", limit=5)
for b in results:
    print(f"Found: {b.title} ({b.url})")
```

### Organizing with Tags
Tags are managed independently and can be attached to multiple bookmarks.

```python
# Create a tag and attach it
tag, _ = service.create_tag({"name": "Development", "color": "blue"})
service.update_bookmark(bookmark.id, {"tags": [tag.id]})
```

## When to Use Pagemark API

*   **Use it for**: Building personal tools, prototyping bookmarking browser extensions, or learning how to implement a layered architecture in Flask.
*   **Don't use it for**: Production environments requiring data persistence (it's in-memory), multi-user applications (it lacks authentication), or storing millions of bookmarks (it's not optimized for large-scale horizontal scaling).

## Stack Compatibility
*   **Framework**: Flask 3.0+
*   **Language**: Python 3.10+ (uses modern type hinting and dataclasses)
*   **Dependencies**: `python-dotenv` for configuration.
*   **Storage**: In-memory (volatile).

## Getting Started Pointers
*   [[LINK: API Reference]] for a full list of endpoints and JSON schemas.
*   [[LINK: Architecture Deep-dive]] to understand the singleton service pattern used.
*   [[LINK: Configuration Guide]] for setting up environment variables.

## Limitations & Assumptions
*   **Volatile Storage**: All data is lost when the server restarts. The `BookmarkRepository` is designed to be swapped with a database implementation (like SQLAlchemy) in the future.
*   **No Authentication**: The API is currently open and assumes it is running in a trusted environment (like localhost).
*   **Smart Collections**: While the `Collection` model supports `filter_rule`, the automatic population of these collections is not yet fully integrated into the service layer.

## FAQ

**How do I persist my bookmarks?**
Currently, the API uses an in-memory repository. To persist data, you would need to implement a new version of `BookmarkRepository` that interfaces with a database like SQLite or PostgreSQL.

**Does it support full-text search?**
Yes. It uses a custom `SearchIndex` class that builds an inverted index of all bookmark titles and descriptions.

**Can I use this as a library?**
While designed as a REST API, the `BookmarkService` and `app.models` are decoupled from Flask and can be imported into other Python projects.

**How does the caching work?**
The `BookmarkService` uses an `LRUCache` (Least Recently Used) to store the most frequently accessed bookmark objects, reducing the need to query the repository.

**Is there a frontend?**
This project is a headless API. You can build a frontend using React, Vue, or even a simple CLI tool that consumes these endpoints.