---
title: Data Persistence
description: In-memory repository implementation for storing and retrieving bookmarks, tags, and collections.
code_symbols: [SYM#09a56e7acb86a9afef18a62134c27802cd473050]
section_id: 9d6c77c6-c27e-4730-9fdd-fff9db31035e_data_persistence
doc_type: guide
section_type: guide
---
Data persistence in this system is handled by a repository pattern that abstracts the underlying storage mechanism. The primary implementation is an in-memory store, designed to decouple business logic from data access while providing a blueprint for future database integration.

## Bookmark Repository

The `BookmarkRepository` class in `app/db/repository.py` serves as the central data access point for the application. It manages three primary entities: `Bookmark`, `Tag`, and `Collection`.

### In-Memory Storage Implementation

The repository uses standard Python dictionaries to store entities in memory. This approach ensures high performance for development and testing but means that data is volatile and will be lost when the application process restarts.

```python
class BookmarkRepository:
    def __init__(self) -> None:
        self._bookmarks: Dict[str, Bookmark] = {}
        self._tags: Dict[str, Tag] = {}
        self._collections: Dict[str, Collection] = {}
```

### Data Access Operations

The repository provides standard CRUD operations for all entities. Mutation methods like `save_bookmark` persist changes immediately to the internal dictionaries.

*   **Bookmarks**: Supports saving, retrieving by ID, hard-deletion, and listing with pagination and status filtering.
*   **Tags**: Supports saving, retrieving by ID, hard-deletion, and listing all tags.
*   **Collections**: Supports saving, retrieving by ID, hard-deletion, and listing all collections.

#### Pagination and Filtering

The `list_bookmarks` method implements in-memory pagination and filtering by `BookmarkStatus`. It sorts results by `created_at` in descending order before slicing the list for pagination.

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

## Service Integration

The repository is not used directly by the API controllers. Instead, it is managed by the service layer, primarily `BookmarkService` and `SearchIndex`.

### BookmarkService Usage

The `BookmarkService` (in `app/services/bookmark_service.py`) initializes the repository as a private member during its bootstrap process. It uses the repository to persist validated bookmark data and retrieve entities for business operations.

```python
def _init_services(self) -> None:
    """Bootstrap repository, cache, and search index."""
    self._repo = BookmarkRepository()
    # ... other initializations ...
```

### SearchIndex Usage

The `SearchIndex` (in `app/services/search_service.py`) depends on the repository to fetch full `Bookmark` objects after identifying matching IDs through its inverted index.

```python
def search(self, query: str, limit: int = 20) -> List[Bookmark]:
    # ... tokenization and index lookup ...
    results = []
    for bid in candidate_ids:
        bookmark = self._repo.get_bookmark(bid)
        if bookmark:
            results.append(bookmark)
    return self._rank_results(results, tokens)[:limit]
```

## Internal Connection Management

While the current implementation is in-memory, the codebase includes internal utilities in `app/db/_connection.py` that demonstrate how the system is prepared for real database integration.

### Connection Pooling

The `_ConnectionPool` class provides a thread-safe mechanism for managing a pool of `_Connection` objects. It supports:
*   **Min/Max Pool Sizes**: Configurable limits on the number of active connections.
*   **Health Checks**: Verifying that pooled connections are still alive.
*   **Thread Safety**: Using `threading.Lock` to manage concurrent access to the pool.

### Transaction Support

The `_Connection` class includes stubs for transaction management, including support for nested transactions via a `_transaction_depth` counter.

```python
class _Connection:
    def begin_transaction(self) -> None:
        """Start a new transaction (supports nesting via savepoints)."""
        self._transaction_depth += 1

    def commit(self) -> None:
        """Commit the current transaction."""
        if self._transaction_depth > 0:
            self._transaction_depth -= 1
```

These classes are prefixed with underscores to indicate they are internal implementation details and are not intended for use outside the database package. They serve as a blueprint for transitioning from the current in-memory `BookmarkRepository` to a persistent database-backed implementation.