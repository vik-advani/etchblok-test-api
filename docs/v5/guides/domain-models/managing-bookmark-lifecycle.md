---
title: Managing Bookmark Lifecycle
description: A guide to the Bookmark model, covering state transitions between active, archived, and trashed statuses.
code_symbols: [SYM#d731bc2f45cac29b2961ab00083bab5345db0a5e, SYM#d570461c1ff2b0eb81e078e185a46de87938f933, SYM#6d5b77b3f04288620db996a9bc18b13d3bd1ad3b]
section_id: b30ecf5b-1abb-4e3a-b7c2-d35839bffd7d_managing_bookmark_lifecycle
doc_type: how_to
section_type: guide
---
To manage the lifecycle of a bookmark in this project, you use the `Bookmark` model's state transition methods or the `BookmarkService` facade. Bookmarks transition between three states defined in the `BookmarkStatus` enum: `ACTIVE`, `ARCHIVED`, and `TRASHED`.

## Transitioning Bookmark States

The most common way to manage a bookmark's lifecycle is through the `BookmarkService`, which handles persistence, cache invalidation, and search indexing automatically.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()
bookmark_id = "a1b2c3d4e5f6"

# Archive a bookmark
archived = service.archive_bookmark(bookmark_id)

# Restore a bookmark (from Archive or Trash)
restored = service.restore_bookmark(bookmark_id)

# Soft-delete a bookmark (moves it to Trash)
success = service.delete_bookmark(bookmark_id)
```

### Key Lifecycle Methods

The `Bookmark` class in `app/models/bookmark.py` implements the core logic for these transitions. Each method updates the `status` attribute and calls `_touch()` to refresh the `updated_at` timestamp.

*   **`archive()`**: Sets status to `BookmarkStatus.ARCHIVED`.
*   **`trash()`**: Sets status to `BookmarkStatus.TRASHED`.
*   **`restore()`**: Sets status to `BookmarkStatus.ACTIVE`.

```python
# Internal implementation in app/models/bookmark.py
def archive(self) -> None:
    """Move the bookmark to the archive."""
    self.status = BookmarkStatus.ARCHIVED
    self._touch()
```

## Filtering Bookmarks by Status

When retrieving bookmarks, you can filter the results based on their current lifecycle state. The `list_bookmarks` method in `BookmarkService` accepts a `status` string.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()

# List only archived bookmarks
archived_bookmarks, total = service.list_bookmarks(status="archived")

# List only trashed bookmarks
trashed_bookmarks, total = service.list_bookmarks(status="trashed")

# List active bookmarks (default behavior if status is not provided depends on repo implementation)
active_bookmarks, total = service.list_bookmarks(status="active")
```

This filtering is implemented in the `BookmarkRepository.list_bookmarks` method within `app/db/repository.py`, which iterates through the stored bookmarks and matches the `status.value`.

## API Integration

The lifecycle transitions are exposed via REST endpoints in `app/routes/bookmarks.py`.

| Action | Method | Endpoint | Service Call |
| :--- | :--- | :--- | :--- |
| **Archive** | `POST` | `/api/bookmarks/<id>/archive` | `service.archive_bookmark(id)` |
| **Restore** | `POST` | `/api/bookmarks/<id>/restore` | `service.restore_bookmark(id)` |
| **Trash** | `DELETE` | `/api/bookmarks/<id>` | `service.delete_bookmark(id)` |

Example of the archive route implementation:

```python
@bookmarks_bp.route("/<bookmark_id>/archive", methods=["POST"])
def archive_bookmark(bookmark_id: str):
    """Archive a bookmark."""
    bookmark = _service.archive_bookmark(bookmark_id)
    if not bookmark:
        return jsonify({"error": "Bookmark not found"}), 404
    return jsonify(bookmark.to_dict())
```

## Troubleshooting and Gotchas

### Soft-Delete vs. Hard-Delete
In this codebase, "deleting" a bookmark via the `BookmarkService` or the API is a **soft-delete**. It calls `bookmark.trash()`, which merely changes the status. 

If you need to permanently remove a bookmark from the underlying storage, you must use the `BookmarkRepository.delete_bookmark` method directly, which is not exposed by the `BookmarkService`.

```python
# Permanent removal (Hard-delete)
from app.db.repository import BookmarkRepository
repo = BookmarkRepository()
repo.delete_bookmark(bookmark_id)
```

### Serialization Limitations
The `Bookmark.from_dict` method is designed for creating new bookmarks from user input. It does **not** restore the `status` or `id` from the dictionary; it always initializes a new bookmark as `ACTIVE` with a new UUID.

```python
# app/models/bookmark.py
@classmethod
def from_dict(cls, data: Dict[str, Any]) -> "Bookmark":
    return cls(
        url=data["url"],
        title=data["title"],
        description=data.get("description", ""),
        tags=data.get("tags", []),
    )
```

If you are manually reconstructing a bookmark from a database export, you must set the `status` manually after instantiation.