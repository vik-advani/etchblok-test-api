---
title: Managing Bookmark Lifecycles
description: A practical guide to performing CRUD operations, archiving, and trashing bookmarks through the service layer.
code_symbols: [SYM#fdcfaed9972e477ae4585fd1b279ac107fa534cd]
section_id: 195b5b92-05be-446f-a554-5ba0df71ded5_managing_bookmark_lifecycles
doc_type: how_to
section_type: guide
---
To manage the lifecycle of bookmarks in this application, you use the `BookmarkService` class from `app.services.bookmark_service`. This service acts as a singleton facade that coordinates data persistence, search indexing, and caching.

## Creating a Bookmark

To create a bookmark, pass a dictionary containing at least a `url` and `title` to `create_bookmark`. The service performs validation and returns a tuple containing the new bookmark object and an error message (if any).

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()

data = {
    "url": "https://github.com/vik-advani",
    "title": "GitHub Profile",
    "description": "Personal GitHub account"
}

bookmark, error = service.create_bookmark(data)

if error:
    # Handle validation errors (e.g., invalid URL or missing title)
    print(f"Failed to create bookmark: {error}")
else:
    print(f"Created bookmark with ID: {bookmark.id}")
```

## Retrieving and Listing Bookmarks

The service provides methods for both single-item retrieval and paginated listing. Single-item retrieval via `get_bookmark` automatically utilizes an internal `LRUCache`.

### Fetching a Single Bookmark
```python
bookmark = service.get_bookmark("bookmark-uuid-123")

if bookmark:
    print(f"Found: {bookmark.title}")
else:
    print("Bookmark not found")
```

### Listing with Pagination and Filters
Use `list_bookmarks` to retrieve a subset of bookmarks. You can filter by status (e.g., "active", "archived", "trashed").

```python
# Get the first 10 active bookmarks
bookmarks, total_count = service.list_bookmarks(
    page=1, 
    per_page=10, 
    status="active"
)

for b in bookmarks:
    print(f"- {b.title} ({b.url})")
```

## Updating Bookmark Metadata

The `update_bookmark` method allows for partial updates. Only the fields provided in the data dictionary will be modified.

```python
update_data = {
    "title": "Updated GitHub Profile",
    "description": "Updated description"
}

bookmark, error = service.update_bookmark("bookmark-uuid-123", update_data)

if error:
    print(f"Update failed: {error}")
elif not bookmark:
    print("Bookmark not found")
```

## Managing the Bookmark Lifecycle

Bookmarks transition through different states: `active`, `archived`, and `trashed`. The service provides specific methods to handle these transitions, ensuring the search index and cache remain synchronized.

### Archiving a Bookmark
Archiving moves a bookmark out of the primary list without deleting it.

```python
bookmark = service.archive_bookmark("bookmark-uuid-123")
if bookmark:
    print(f"Status is now: {bookmark.status}") # 'archived'
```

### Trashing (Soft-Delete)
The `delete_bookmark` method performs a **soft-delete**. It changes the bookmark's status to `trashed` rather than removing it from the database.

```python
success = service.delete_bookmark("bookmark-uuid-123")
if success:
    print("Bookmark moved to trash.")
```

### Restoring a Bookmark
You can restore a bookmark from either the `archived` or `trashed` state back to `active`.

```python
bookmark = service.restore_bookmark("bookmark-uuid-123")
if bookmark:
    print(f"Bookmark restored. Status: {bookmark.status}")
```

## Troubleshooting and Gotchas

### Singleton Instance
The `BookmarkService` is implemented as a singleton. Calling `BookmarkService()` multiple times returns the same instance to ensure consistent state across different modules (like `app.routes.bookmarks` and `app.routes.tags`).

### Soft-Delete vs. Hard-Delete
*   **Bookmarks**: `delete_bookmark()` is a soft-delete (status change).
*   **Tags**: `delete_tag()` is a **hard-delete**. When a tag is deleted, the service automatically iterates through all bookmarks containing that tag to remove the reference and invalidate their cache entries.

### Validation Logic
Validation for `url` and `title` is enforced at the service layer. If `create_bookmark` or `update_bookmark` receives invalid data, the first element of the returned tuple will be `None`, and the second will contain the validation error message.