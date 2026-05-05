---
title: Data Persistence
description: The repository layer handling in-memory storage and CRUD operations for all system entities.
code_symbols: [SYM#09a56e7acb86a9afef18a62134c27802cd473050]
section_id: 85e111ca-191e-4f47-9537-5b88bb41fecb_data_persistence
doc_type: guide
section_type: guide
---
The data persistence layer in this project is designed around the Repository pattern, providing a clean abstraction over the underlying storage mechanism. While the current implementation is entirely in-memory, the architecture is structured to support future transitions to persistent databases like SQLite or PostgreSQL without modifying the business logic in the service layer.

## The Repository Layer

The `BookmarkRepository` class, located in `app/db/repository.py`, serves as the central data access object. It manages the lifecycle of three primary entities: **Bookmarks**, **Tags**, and **Collections**.

### In-Memory Storage
Data is stored within the repository instance using standard Python dictionaries, keyed by the entity's unique ID. This ensures O(1) lookup performance but means that all data is volatile and lost when the application process terminates.

```python
class BookmarkRepository:
    def __init__(self) -> None:
        self._bookmarks: Dict[str, Bookmark] = {}
        self._tags: Dict[str, Tag] = {}
        self._collections: Dict[str, Collection] = {}
```

### CRUD Operations
The repository provides standard CRUD methods for each entity type. These methods are designed to be "dumb" — they perform no validation or business logic, simply persisting or retrieving the objects as requested.

*   **Save**: `save_bookmark`, `save_tag`, `save_collection`
*   **Get**: `get_bookmark`, `get_tag`, `get_collection`
*   **Delete**: `delete_bookmark`, `delete_tag`, `delete_collection`
*   **List**: `list_bookmarks`, `list_tags`, `list_collections`

### Specialized Queries and Pagination
Beyond simple CRUD, the repository implements specialized filtering and pagination logic. For example, `list_bookmarks` supports filtering by `BookmarkStatus` and returns a paginated slice of the data.

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

The repository also handles relational queries, such as `get_bookmarks_with_tag(tag_id)`, which performs a linear scan of all bookmarks to find those containing the specified tag.

## Internal Connection Management

The module `app/db/_connection.py` contains a thread-safe connection pool stub. Although not currently utilized by the `BookmarkRepository`, it demonstrates the intended architecture for future database integration.

### Connection Pooling
The `_ConnectionPool` class manages a set of `_Connection` objects. It uses a `threading.Lock` to ensure thread safety when acquiring or releasing connections from the pool.

```python
class _ConnectionPool:
    def __init__(self, config: Optional[_ConnectionConfig] = None) -> None:
        self._config = config or _ConnectionConfig()
        self._available: List[_Connection] = []
        self._in_use: List[_Connection] = []
        self._lock = threading.Lock()
        self.__init_pool()
```

### Private Configuration
Configuration for the connection pool is managed via internal constants and the `_ConnectionConfig` dataclass. These are prefixed with underscores to indicate they are private to the database layer and should not be exposed to the rest of the application.

*   `_POOL_MIN_SIZE`: Default 2
*   `_POOL_MAX_SIZE`: Default 20
*   `_CONNECT_TIMEOUT_MS`: Default 5000ms

## Service Layer Integration

The `BookmarkService` (in `app/services/bookmark_service.py`) acts as the primary consumer of the repository. It instantiates the `BookmarkRepository` as a member variable during its own singleton initialization.

```python
def _init_services(self) -> None:
    """Bootstrap repository, cache, and search index."""
    self._repo = BookmarkRepository()
    # ... other initializations ...
```

### Orchestrating Persistence
The service layer is responsible for maintaining referential integrity across repository calls. For instance, when a tag is deleted via `delete_tag`, the service must first find all bookmarks associated with that tag, update them to remove the tag reference, and then finally delete the tag from the repository.

```python
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

This separation ensures that the `BookmarkRepository` remains a simple data store, while the `BookmarkService` handles the complex interactions between different entity types.