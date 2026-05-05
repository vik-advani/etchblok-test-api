---
title: Managing Bookmark Records
description: A practical guide to performing CRUD operations on bookmarks, including saving, retrieving, and hard-deleting entries.
code_symbols: [SYM#adb8232356346a5957ff3a1a1b7ff70581f37649]
section_id: 061fad4a-2551-40b2-a666-52f1d75316f5_managing_bookmark_records
doc_type: how_to
section_type: guide
---
Managing bookmark records in this application involves interacting with the `BookmarkService`, which acts as a facade over the in-memory `BookmarkRepository`. While the repository handles raw storage, the service layer manages validation, caching, and search indexing.

### Creating a Bookmark

To create a new bookmark, use the `create_bookmark` method of the `BookmarkService`. This method validates the URL and title before persisting the record and indexing it for search.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()

data = {
    "url": "https://example.com",
    "title": "Example Domain",
    "description": "A useful example site"
}

bookmark, error = service.create_bookmark(data)

if error:
    print(f"Failed to create bookmark: {error}")
else:
    print(f"Created bookmark with ID: {bookmark.id}")
```

### Retrieving Bookmarks

You can retrieve a single bookmark by its ID or fetch a paginated list. The service layer uses an `LRUCache` to speed up single-item lookups.

#### Fetching a Single Record
```python
# Uses cache if available, otherwise hits the repository
bookmark = service.get_bookmark("some-id-123")

if bookmark:
    print(f"Found: {bookmark.title}")
```

#### Listing and Filtering
The `list_bookmarks` method supports pagination and filtering by status (`active`, `archived`, or `trashed`).

```python
# Get the first page of archived bookmarks
bookmarks, total = service.list_bookmarks(
    page=1, 
    per_page=10, 
    status="archived"
)

print(f"Total archived items: {total}")
for b in bookmarks:
    print(f"- {b.title}")
```

### Updating a Bookmark

The `update_bookmark` method performs a partial update. It validates any new URL or title provided in the data dictionary before saving.

```python
update_data = {
    "title": "Updated Example Title",
    "description": "New description for this bookmark"
}

updated_bookmark, error = service.update_bookmark("some-id-123", update_data)

if error:
    print(f"Update failed: {error}")
```

### Soft-Deleting vs. Hard-Deleting

This codebase distinguishes between "trashing" a bookmark (soft-delete) and "deleting" it from storage (hard-delete).

#### Soft-Delete (Trash)
Use the `BookmarkService.delete_bookmark` method to move a bookmark to the `trashed` status. This keeps the record in the repository but excludes it from default listings.

```python
# Soft-delete via the service
success = service.delete_bookmark("some-id-123")
```

#### Hard-Delete (Purge)
To permanently remove a bookmark from memory, you must call the `delete_bookmark` method directly on the `BookmarkRepository`. Note that this bypasses the service's cache invalidation and search index removal.

```python
from app.db.repository import BookmarkRepository

repo = BookmarkRepository()
# Hard-delete via the repository
existed = repo.delete_bookmark("some-id-123")
```

### Archiving and Restoring

Bookmarks can be moved between `active`, `archived`, and `trashed` states using dedicated lifecycle methods in the `BookmarkService`.

```python
# Archive a bookmark
archived = service.archive_bookmark("some-id-123")

# Restore a trashed or archived bookmark to 'active' status
restored = service.restore_bookmark("some-id-123")
```

### Troubleshooting

*   **Data Persistence**: The `BookmarkRepository` is currently in-memory. All bookmark records are lost when the application process restarts.
*   **Invalid Status Filters**: If an invalid status string is passed to `list_bookmarks`, the repository silently ignores the filter and returns bookmarks of all statuses.
*   **Cache Inconsistency**: If you modify the repository directly (e.g., using `repo.delete_bookmark`), the `BookmarkService` cache may still return the old object until it expires or is manually invalidated via `service._cache.invalidate(id)`.