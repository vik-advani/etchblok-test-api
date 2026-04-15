---
title: Repository Architecture
description: An overview of the repository pattern used for data persistence and the in-memory implementation details of the storage layer.
code_symbols: [SYM#09a56e7acb86a9afef18a62134c27802cd473050, SYM#adb8232356346a5957ff3a1a1b7ff70581f37649]
section_id: fdb870f8-f38f-4ecc-9b31-fbfccf62d7af_repository_architecture
doc_type: explanation
section_type: guide
---
The repository architecture in this project centers on the `BookmarkRepository` class, located in `app/db/repository.py`. It serves as a centralized, in-memory data store that abstracts the persistence logic for bookmarks, tags, and collections. By decoupling the data access layer from the business logic in the service layer, the project maintains a clean separation of concerns, allowing the rest of the application to interact with data through a consistent API regardless of the underlying storage mechanism.

## In-Memory Storage Implementation

The `BookmarkRepository` implements persistence using standard Python dictionaries. This choice reflects a design optimized for speed and simplicity in a development or testing environment, though it introduces the constraint that all data is volatile and lost upon application restart.

The repository maintains three internal dictionaries to store different entity types:

```python
def __init__(self) -> None:
    self._bookmarks: Dict[str, Bookmark] = {}
    self._tags: Dict[str, Tag] = {}
    self._collections: Dict[str, Collection] = {}
```

Each entity is indexed by its unique ID string, allowing for $O(1)$ retrieval. This structure is mirrored across the CRUD methods for all three entities (Bookmarks, Tags, and Collections), providing a uniform interface for the `BookmarkService`.

## Data Access and Querying

The repository provides standard CRUD operations (`save_*`, `get_*`, `delete_*`, `list_*`), but also includes specialized querying logic to support the application's requirements for filtering and pagination.

### Pagination and Filtering
The `list_bookmarks` method implements server-side pagination and status filtering. It sorts bookmarks by their creation date in descending order before slicing the list based on the requested page.

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

This implementation highlights a specific tradeoff: because the data is stored in a simple dictionary, every call to `list_bookmarks` involves converting the dictionary values to a list and sorting them. While efficient for small datasets, this would become a performance bottleneck as the number of bookmarks grows.

### Relationship Management
The repository also handles basic relationship queries, such as `get_bookmarks_with_tag(tag_id: str)`. Since the data model uses a flat structure where bookmarks store tag IDs in a list, the repository performs a linear scan to find matches:

```python
def get_bookmarks_with_tag(self, tag_id: str) -> List[Bookmark]:
    """Return all bookmarks that have a specific tag attached."""
    return [b for b in self._bookmarks.values() if tag_id in b.tags]
```

## Integration and Lifecycle

The `BookmarkRepository` is designed to be a long-lived component, typically instantiated as a singleton within the `BookmarkService`.

### Service Layer Dependency
In `app/services/bookmark_service.py`, the repository is initialized during the service's bootstrap phase. The service relies on the repository for all state changes, ensuring that validation and business rules are applied before data reaches the persistence layer.

```python
def _init_services(self) -> None:
    """Bootstrap repository, cache, and search index."""
    self._repo = BookmarkRepository()
    # ... other initializations ...
```

### Search Index Synchronization
The `SearchIndex` class in `app/services/search_service.py` also depends directly on the repository. Upon initialization, the search index performs a full rebuild by querying the repository for all existing bookmarks:

```python
def _rebuild(self) -> None:
    """Rebuild the entire index from the repository."""
    self._index.clear()
    all_bookmarks, _ = self._repo.list_bookmarks(page=1, per_page=10000)
    for bookmark in all_bookmarks:
        self.index_bookmark(bookmark)
```

## Design Tradeoffs and Constraints

The current repository architecture makes several deliberate design choices:

1.  **Immediate Persistence**: Unlike many ORMs that use a "Unit of Work" pattern with explicit commits, the `BookmarkRepository` persists changes immediately to its internal dictionaries. This simplifies the API but means there is no support for transactions or rollbacks.
2.  **No Internal Validation**: The repository assumes that any model passed to its `save` methods is already valid. It does not perform schema checks or constraint validation, leaving those responsibilities to the `Bookmark` model's `from_dict` method and the `BookmarkService`.
3.  **Hard Deletes**: The `delete_bookmark` and `delete_tag` methods perform hard deletes (removing the entry from the dictionary). Soft deletes (e.g., moving a bookmark to a "trash" status) are handled at the service layer by updating the bookmark's status rather than calling the repository's delete method.
4.  **Testability**: The inclusion of `_clear_all()` and `_count_all()` methods specifically supports automated testing, allowing tests to reset the global state between runs without re-instantiating the entire service stack.