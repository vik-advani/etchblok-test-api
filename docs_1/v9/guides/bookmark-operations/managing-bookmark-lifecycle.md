---
title: Managing Bookmark Lifecycle
description: An overview of bookmark states, including active, archived, and trashed statuses, and how to transition between them.
code_symbols: [SYM#dd5fd545693e00d6c0b38892bbd18ed4afa136a1, SYM#fdcfaed9972e477ae4585fd1b279ac107fa534cd]
section_id: a15b8c7f-981e-4984-8a53-6eff2ea2bfc9_managing_bookmark_lifecycle
doc_type: guide
section_type: guide
---
The lifecycle of a bookmark in this system is managed through a set of defined states and transitions orchestrated by the `BookmarkService`. This ensures that operations like archiving or trashing a bookmark are handled consistently across the application, including cache invalidation and search index updates.

## Bookmark States

The status of a bookmark is defined by the `BookmarkStatus` enum in `app/models/bookmark.py`. Every bookmark exists in exactly one of these three states:

*   **`ACTIVE`**: The default state for new bookmarks. They are visible in general listings and search results.
*   **`ARCHIVED`**: Bookmarks that are saved for long-term storage but removed from the primary active view.
*   **`TRASHED`**: Bookmarks that have been "soft-deleted." They are kept in the system but are typically hidden from the user unless they specifically view the trash.

## State Transitions

The `BookmarkService` (located in `app/services/bookmark_service.py`) provides the primary interface for moving bookmarks between these states.

### Creation and Initial State
When a bookmark is created using `BookmarkService.create_bookmark`, it is initialized with the `ACTIVE` status by default.

```python
# app/services/bookmark_service.py

def create_bookmark(self, data: Dict[str, Any]) -> Tuple[Optional[Bookmark], Optional[str]]:
    # ... validation logic ...
    bookmark = Bookmark.from_dict(data) # Defaults to BookmarkStatus.ACTIVE
    self._repo.save_bookmark(bookmark)
    self._search.index_bookmark(bookmark)
    self._cache.invalidate(bookmark.id)
    return bookmark, None
```

### Archiving
Archiving a bookmark moves it to the `ARCHIVED` state. This is useful for decluttering the main list without permanently deleting the data.

```python
# app/services/bookmark_service.py

def archive_bookmark(self, bookmark_id: str) -> Optional[Bookmark]:
    bookmark = self._repo.get_bookmark(bookmark_id)
    if not bookmark:
        return None
    bookmark.archive() # Sets status to BookmarkStatus.ARCHIVED
    self._repo.save_bookmark(bookmark)
    self._cache.invalidate(bookmark_id)
    return bookmark
```

### Soft-Deletion (Trashing)
The `BookmarkService.delete_bookmark` method implements a **soft-delete** pattern. Instead of removing the record from the database, it transitions the bookmark to the `TRASHED` state.

```python
# app/services/bookmark_service.py

def delete_bookmark(self, bookmark_id: str) -> bool:
    """Soft-delete by trashing the bookmark."""
    bookmark = self._repo.get_bookmark(bookmark_id)
    if not bookmark:
        return False
    bookmark.trash() # Sets status to BookmarkStatus.TRASHED
    self._repo.save_bookmark(bookmark)
    self._cache.invalidate(bookmark_id)
    return True
```

### Restoration
The `restore_bookmark` method is the inverse of both archiving and trashing. It returns a bookmark to the `ACTIVE` state regardless of whether it was previously archived or trashed.

```python
# app/services/bookmark_service.py

def restore_bookmark(self, bookmark_id: str) -> Optional[Bookmark]:
    bookmark = self._repo.get_bookmark(bookmark_id)
    if not bookmark:
        return None
    bookmark.restore() # Sets status to BookmarkStatus.ACTIVE
    self._repo.save_bookmark(bookmark)
    self._cache.invalidate(bookmark_id)
    return bookmark
```

## Filtering by Status

The `list_bookmarks` method in `BookmarkService` allows for retrieving bookmarks based on their current lifecycle state. This is passed down to the `BookmarkRepository`, which filters the results.

```python
# app/db/repository.py

def list_bookmarks(
    self,
    page: int = 1,
    per_page: int = 25,
    status: Optional[str] = None,
) -> Tuple[List[Bookmark], int]:
    items = list(self._bookmarks.values())
    if status:
        try:
            target = BookmarkStatus(status)
            items = [b for b in items if b.status == target]
        except ValueError:
            # Silently ignore invalid status strings
            pass
    # ... pagination and sorting ...
```

## Internal Mechanics

### Timestamp Management
Every state transition (archive, trash, restore) triggers the `_touch()` method on the `Bookmark` model. This updates the `updated_at` timestamp, ensuring that the system tracks when the lifecycle state last changed.

### Cache and Search Synchronization
The `BookmarkService` ensures that whenever a bookmark's state changes:
1.  The **Repository** persists the change.
2.  The **Cache** (`LRUCache`) is invalidated for that specific ID to prevent stale data from being served.
3.  The **Search Index** is updated (in the case of creation or updates) to reflect the current metadata.

### Soft vs. Hard Delete
It is important to distinguish between the service layer and the repository layer:
*   `BookmarkService.delete_bookmark`: Performs a **soft-delete** (status change).
*   `BookmarkRepository.delete_bookmark`: Performs a **hard-delete** (complete removal from storage).

In standard application flow, the service layer's soft-delete is preferred to allow for restoration. Hard-deletion is typically reserved for administrative cleanup or automated data retention policies.