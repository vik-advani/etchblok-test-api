---
title: The Repository Architecture
description: An overview of the repository pattern used to abstract data persistence, focusing on the centralized management of bookmarks, tags, and collections.
code_symbols: [SYM#09a56e7acb86a9afef18a62134c27802cd473050, SYM#adb8232356346a5957ff3a1a1b7ff70581f37649]
section_id: 27c0bac4-2341-4fcc-9ca7-2d8ef0e922f2_the_repository_architecture
doc_type: guide
section_type: guide
---
The `BookmarkRepository` class in `app/db/repository.py` serves as the centralized data access layer for the application. It implements the Repository pattern to decouple the business logic in the service layer from the underlying persistence mechanism. While currently implemented as an in-memory store, its interface is designed to provide a clean abstraction that could be swapped for a persistent database.

## Core Data Structures

The repository maintains three primary entity types in memory using Python dictionaries. These dictionaries use unique identifiers (UUIDs) as keys and domain objects as values:

*   `_bookmarks`: Stores `Bookmark` objects.
*   `_tags`: Stores `Tag` objects.
*   `_collections`: Stores `Collection` objects.

```python
class BookmarkRepository:
    """In-memory storage for bookmarks, tags, and collections."""

    def __init__(self) -> None:
        self._bookmarks: Dict[str, Bookmark] = {}
        self._tags: Dict[str, Tag] = {}
        self._collections: Dict[str, Collection] = {}
```

## Entity Management and CRUD

The repository provides a standard CRUD (Create, Read, Update, Delete) interface for each entity type. Mutation methods perform immediate updates to the internal dictionaries.

### Bookmarks
The repository handles bookmark persistence and retrieval, including specialized queries for filtering and pagination. The `list_bookmarks` method supports filtering by `BookmarkStatus` (active, archived, trashed) and returns a paginated slice of results sorted by creation date.

```python
def list_bookmarks(
    self,
    page: int = 1,
    per_page: int = 25,
    status: Optional[str] = None,
) -> Tuple[List[Bookmark], int]:
    """Return a paginated slice of bookmarks."""
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

### Tags and Collections
Tags and collections are managed through similar CRUD methods such as `save_tag`, `list_tags`, `save_collection`, and `list_collections`. The repository also facilitates cross-entity lookups with methods like `get_bookmarks_with_tag`.

```python
def get_bookmarks_with_tag(self, tag_id: str) -> List[Bookmark]:
    """Return all bookmarks that have a specific tag attached."""
    return [b for b in self._bookmarks.values() if tag_id in b.tags]
```

## Architectural Integration

The `BookmarkRepository` is a foundational component used by higher-level services to maintain state across the application.

### Service Layer Orchestration
The `BookmarkService` (in `app/services/bookmark_service.py`) acts as a facade over the repository. It handles validation and orchestrates complex operations that involve multiple repository calls. Because the repository does not enforce referential integrity, the service layer must manage cascading updates. For example, when a tag is deleted, the service manually strips that tag from all associated bookmarks:

```python
# From app/services/bookmark_service.py
def delete_tag(self, tag_id: str) -> bool:
    """Delete a tag and strip it from all bookmarks."""
    tag = self._repo.get_tag(tag_id)
    if not tag:
        return False
    for bookmark in self._repo.get_bookmarks_with_tag(tag_id):
        bookmark.remove_tag(tag_id)
        self._repo.save_bookmark(bookmark)
        self._cache.invalidate(bookmark.id)
    self._repo.delete_tag(tag_id)
    return True
```

### Search Indexing
The `SearchIndex` (in `app/services/search_service.py`) depends on the repository to build and maintain its inverted index. During initialization, it performs a full scan of the repository to populate the index, and it retrieves full bookmark objects from the repository when returning search results.

```python
# From app/services/search_service.py
def _rebuild(self) -> None:
    """Rebuild the entire index from the repository."""
    self._index.clear()
    # Scans the repository to index all existing bookmarks
    all_bookmarks, _ = self._repo.list_bookmarks(page=1, per_page=10000)
    for bookmark in all_bookmarks:
        self.index_bookmark(bookmark)
```

## Design Considerations

*   **In-Memory Nature**: The current implementation is volatile; data is lost when the application restarts. This is suitable for the current test-api scope but would require a persistent backend for production.
*   **Immediate Persistence**: Mutation methods like `save_bookmark` update the internal state immediately. However, there is no built-in transaction support for multi-entity updates, which are instead managed by the `BookmarkService`.
*   **Diagnostic Methods**: The repository includes internal helpers `_count_all()` for health checks and `_clear_all()` for resetting state during test execution.

```python
def _count_all(self) -> Dict[str, int]:
    """Return entity counts. Used for diagnostics."""
    return {
        "bookmarks": len(self._bookmarks),
        "tags": len(self._tags),
        "collections": len(self._collections),
    }
```