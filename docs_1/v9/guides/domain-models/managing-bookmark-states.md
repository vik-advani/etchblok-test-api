---
title: Managing Bookmark States
description: How to transition bookmarks between active, archived, and trashed states using the built-in lifecycle methods.
code_symbols: [SYM#d731bc2f45cac29b2961ab00083bab5345db0a5e, SYM#d570461c1ff2b0eb81e078e185a46de87938f933, SYM#6d5b77b3f04288620db996a9bc18b13d3bd1ad3b]
section_id: ea3ba441-4fac-425f-9f7b-cdb10b659685_managing_bookmark_states
doc_type: how_to
section_type: guide
---
To manage the lifecycle of a bookmark in this system, you use the built-in state transition methods on the `Bookmark` model. These methods update the visibility status and ensure the modification timestamp is refreshed.

### Transitioning Bookmark States

You can transition a bookmark between `ACTIVE`, `ARCHIVED`, and `TRASHED` states using the `archive()`, `trash()`, and `restore()` methods.

```python
from app.models.bookmark import Bookmark, BookmarkStatus

# Create a new active bookmark
bookmark = Bookmark(url="https://example.com", title="Example")
print(bookmark.status) # BookmarkStatus.ACTIVE

# Move to archive
bookmark.archive()
print(bookmark.status) # BookmarkStatus.ARCHIVED

# Move to trash (soft-delete)
bookmark.trash()
print(bookmark.status) # BookmarkStatus.TRASHED

# Restore to active
bookmark.restore()
print(bookmark.status) # BookmarkStatus.ACTIVE
```

Each of these methods automatically calls the internal `_touch()` helper, which updates the `updated_at` attribute to the current UTC time.

### Using the Service Layer

In a production environment, you should use the `BookmarkService` to manage states. The service handles the underlying repository persistence and cache invalidation.

#### Archiving a Bookmark
Use `archive_bookmark` to move a bookmark to the archive state.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()
bookmark = service.archive_bookmark("bookmark_id_123")

if bookmark:
    print(f"Bookmark {bookmark.id} is now {bookmark.status.value}")
```

#### Trashing (Soft-Deleting) a Bookmark
The `delete_bookmark` method in the service layer performs a **soft-delete** by moving the bookmark to the trash rather than removing it from the database.

```python
# This calls bookmark.trash() internally
success = service.delete_bookmark("bookmark_id_123")
```

#### Restoring a Bookmark
Use `restore_bookmark` to return a bookmark to the `ACTIVE` state from either `ARCHIVED` or `TRASHED`.

```python
bookmark = service.restore_bookmark("bookmark_id_123")
```

### Filtering by Status

When listing bookmarks, you can filter the results by providing a status string to the `list_bookmarks` method. The `BookmarkRepository` uses these strings to filter the in-memory store.

```python
# Get only archived bookmarks
archived_bookmarks, total = service.list_bookmarks(status="archived")

# Get only trashed bookmarks
trashed_bookmarks, total = service.list_bookmarks(status="trashed")
```

The valid status strings correspond to the values in the `BookmarkStatus` enum:
- `"active"`
- `"archived"`
- `"trashed"`

### Troubleshooting: Soft-delete vs Hard-delete

A common point of confusion is the difference between the Service and Repository delete methods:

1.  **`BookmarkService.delete_bookmark(id)`**: Performs a **soft-delete**. It calls `bookmark.trash()`, saves the change, and invalidates the cache. The bookmark remains in the system with a `TRASHED` status.
2.  **`BookmarkRepository.delete_bookmark(id)`**: Performs a **hard-delete**. It physically removes the bookmark object from the internal storage dictionary.

**Recommendation:** Always use `BookmarkService.delete_bookmark()` unless you specifically intend to permanently purge the data from the system.

```python
# RECOMMENDED: Soft-delete via Service
service.delete_bookmark("id_123") 

# CAUTION: Hard-delete via Repository (bypasses lifecycle logic)
repo = BookmarkRepository()
repo.delete_bookmark("id_123")
```