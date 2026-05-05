---
title: In-Memory Storage Design Decisions
description: An explanation of the current in-memory persistence strategy, its immediate mutation behavior, and considerations for future database migrations.
code_symbols: [SYM#09a56e7acb86a9afef18a62134c27802cd473050, SYM#adb8232356346a5957ff3a1a1b7ff70581f37649]
section_id: c4b9f700-70fb-43fb-a9aa-40611a3a4334_in-memory_storage_design_decisions
doc_type: explanation
section_type: guide
---
The `BookmarkRepository` in `app/db/repository.py` serves as the primary data access layer for the application. While the current implementation is entirely in-memory, the design follows the Repository Pattern to decouple the business logic in the service layer from the underlying storage mechanism.

## In-Memory Persistence Strategy
The repository maintains state using standard Python dictionaries for its three core entities: bookmarks, tags, and collections. This choice prioritizes development speed and simplicity for testing.

```python
class BookmarkRepository:
    def __init__(self) -> None:
        self._bookmarks: Dict[str, Bookmark] = {}
        self._tags: Dict[str, Tag] = {}
        self._collections: Dict[str, Collection] = {}
```

However, this introduces a significant constraint: **data volatility**. All stored information is lost when the application process terminates. To mitigate this during the development phase, the `BookmarkService` initializes the repository as a singleton-like component within its `_init_services` method, ensuring a consistent state throughout the application's lifecycle.

## Immediate Mutation Behavior
Unlike traditional database repositories that might use a "Unit of Work" pattern or explicit transaction commits, `BookmarkRepository` performs immediate mutations. Every `save_*` method directly updates the internal dictionary.

```python
def save_bookmark(self, bookmark: Bookmark) -> None:
    """Insert or update a bookmark."""
    self._bookmarks[bookmark.id] = bookmark
```

This design simplifies the implementation but places the responsibility for cross-entity consistency on the caller. For example, in `BookmarkService.delete_tag`, the service must manually iterate through bookmarks to remove the tag reference before deleting the tag itself from the repository:

```python
# app/services/bookmark_service.py
def delete_tag(self, tag_id: str) -> bool:
    tag = self._repo.get_tag(tag_id)
    if not tag:
        return False
    for bookmark in self._repo.get_bookmarks_with_tag(tag_id):
        bookmark.remove_tag(tag_id)
        self._repo.save_bookmark(bookmark) # Immediate update
        self._cache.invalidate(bookmark.id)
    self._repo.delete_tag(tag_id) # Immediate update
    return True
```

## Pagination and Performance Tradeoffs
Because the data is stored in-memory, the repository must handle operations like sorting and pagination manually. In `list_bookmarks`, the repository converts the entire dictionary of bookmarks into a list, sorts them by creation date, and then slices the list based on the requested page.

```python
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
            pass
    items.sort(key=lambda b: b.created_at, reverse=True)
    total = len(items)
    start = (page - 1) * per_page
    return items[start : start + per_page], total
```

While efficient for small datasets, this approach has an $O(N \log N)$ complexity for every request due to the sort operation. In a persistent database implementation, this would typically be offloaded to the database engine using `ORDER BY` and `LIMIT/OFFSET` clauses.

## Future Migration Path
The current implementation is explicitly designed as a placeholder for a persistent database. The class docstring notes: *"For a real database you'd add transaction support here."*

The transition to a database like SQLite or PostgreSQL is facilitated by several design choices:

*   **Interface Stability**: The `BookmarkService` interacts only with the public methods of `BookmarkRepository`. As long as a new SQL-based repository implements these methods, the service layer remains unchanged.
*   **Internal Helpers**: Methods like `_clear_all` and `_count_all` are provided for diagnostics and testing, allowing the test suite to remain agnostic of the storage backend.
*   **Search Integration**: The `SearchIndex` in `app/services/search_service.py` depends on the repository to rebuild its index. By maintaining the `list_bookmarks` interface, the search engine can continue to function regardless of whether the data comes from memory or a disk-based table.

## Hard vs. Soft Deletion
A notable design distinction exists between the repository and the service layer regarding data removal. The `BookmarkRepository` performs **hard deletes**—actually removing the object from the dictionary:

```python
def delete_bookmark(self, bookmark_id: str) -> bool:
    """Hard-delete a bookmark. Returns True if it existed."""
    return self._bookmarks.pop(bookmark_id, None) is not None
```

In contrast, the `BookmarkService` implements **soft deletion** (moving to "trash") by updating the bookmark's status and saving it back to the repository. The repository's `delete_bookmark` method is typically reserved for permanent removal or cleanup operations.