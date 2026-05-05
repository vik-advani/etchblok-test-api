---
title: Creating and Updating Bookmarks
description: Learn how to validate and persist new bookmarks or modify existing ones using the BookmarkService.
code_symbols: [SYM#dd5fd545693e00d6c0b38892bbd18ed4afa136a1, SYM#fdcfaed9972e477ae4585fd1b279ac107fa534cd]
section_id: 2cbc396e-a1ce-479e-b815-6158b04975d0_creating_and_updating_bookmarks
doc_type: how_to
section_type: guide
---
To create or modify bookmarks, use the `BookmarkService` class. This service acts as a facade that coordinates validation, persistence in the `BookmarkRepository`, full-text indexing in the `SearchIndex`, and cache management.

### Creating a New Bookmark

To create a bookmark, pass a dictionary containing at least a `url` and a `title` to the `create_bookmark` method.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()

data = {
    "url": "https://example.com",
    "title": "Example Domain",
    "description": "A site for examples"
}

bookmark, error = service.create_bookmark(data)

if error:
    # Handle validation error (e.g., "Invalid URL format" or "Title is required")
    print(f"Failed to create bookmark: {error}")
else:
    print(f"Created bookmark with ID: {bookmark.id}")
```

When you call `create_bookmark`, the service performs the following:
1.  **Validation**: It runs `_validate_url` and `_validate_title` from `app.models._validators`.
2.  **Persistence**: It saves the bookmark using `self._repo.save_bookmark`.
3.  **Indexing**: It adds the bookmark to the search index via `self._search.index_bookmark`.
4.  **Caching**: It invalidates any existing cache entry for that ID to ensure consistency.

### Updating an Existing Bookmark

The `update_bookmark` method performs a partial update. Only the fields provided in the data dictionary will be modified.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()
bookmark_id = "some-uuid-here"

# Only update the title and description
update_data = {
    "title": "Updated Title",
    "description": "New description content"
}

bookmark, error = service.update_bookmark(bookmark_id, update_data)

if error:
    print(f"Update failed: {error}")
elif not bookmark:
    print("Bookmark not found")
else:
    print(f"Successfully updated: {bookmark.title}")
```

Key behaviors of `update_bookmark`:
*   **Validation**: If `title` or `url` are present in the update dictionary, they are re-validated.
*   **Timestamping**: The service calls `bookmark._touch()` to update the `updated_at` timestamp.
*   **Cache Invalidation**: The cache is invalidated for the specific `bookmark_id` after the update.

### Managing Bookmark Lifecycle

The `BookmarkService` provides specific methods for changing a bookmark's status without requiring a full update dictionary.

#### Archiving and Restoring
Use `archive_bookmark` to move a bookmark to the archived state, and `restore_bookmark` to return it to active status.

```python
# Archive a bookmark
archived = service.archive_bookmark(bookmark_id)

# Restore a bookmark (from archive or trash)
restored = service.restore_bookmark(bookmark_id)
```

#### Trashing (Soft-Delete)
The `delete_bookmark` method performs a "soft-delete" by moving the bookmark to the trash.

```python
success = service.delete_bookmark(bookmark_id)
if not success:
    print("Bookmark not found")
```

### Troubleshooting and Gotchas

*   **Singleton Pattern**: `BookmarkService` is implemented as a singleton in `app/services/bookmark_service.py`. Calling `BookmarkService()` always returns the same instance, sharing the underlying repository, cache, and search index.
*   **Validation Errors**: Validation logic is centralized in `app/models/_validators.py`. Common errors include:
    *   `Invalid URL format`: The URL must match the `_URL_PATTERN` (requires `http://` or `https://`).
    *   `Title is required`: Titles cannot be empty or just whitespace.
    *   `Title must be 256 characters or fewer`: Titles exceeding `_MAX_TITLE_LENGTH` will fail validation.
*   **Search Index Sync**: The search index is updated synchronously during `create_bookmark` and `update_bookmark`. If you bypass the service and use the repository directly, the search index will become out of sync.