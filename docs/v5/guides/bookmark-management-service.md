---
title: Bookmark Management Service
description: The primary business logic layer for creating, updating, archiving, and organizing bookmarks and tags.
code_symbols: [SYM#dd5fd545693e00d6c0b38892bbd18ed4afa136a1]
section_id: b5b0878f-df21-4ec0-8a58-56b2c4f97c91_bookmark_management_service
doc_type: guide
section_type: guide
---
The **Bookmark Management Service** is the central business logic layer of the application. It is implemented primarily by the `BookmarkService` class in `app/services/bookmark_service.py`, which acts as a facade over the data repository, search index, and caching layers.

## Service Architecture and State Management

The `BookmarkService` is implemented as a **Singleton** to ensure that state (the in-memory repository, cache, and search index) is shared across all Flask blueprint modules. This is achieved via the `__new__` method:

```python
class BookmarkService:
    _instance: Optional["BookmarkService"] = None

    def __new__(cls) -> "BookmarkService":
        """Singleton — share state across blueprint modules."""
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._init_services()
        return cls._instance
```

When initialized, the service bootstraps three core internal components:
- **`BookmarkRepository`**: An in-memory store for bookmarks, tags, and collections.
- **`LRUCache`**: A fixed-capacity (256 items) least-recently-used cache for fast bookmark retrieval.
- **`SearchIndex`**: An inverted index for full-text search over bookmark titles and descriptions.

## Bookmark Lifecycle and Operations

The service manages the full lifecycle of a `Bookmark` (defined in `app/models/bookmark.py`). Each bookmark transitions through different states defined by the `BookmarkStatus` enum: `ACTIVE`, `ARCHIVED`, and `TRASHED`.

### Creation and Validation
When creating a bookmark via `create_bookmark(data)`, the service performs several steps:
1.  **Validation**: It uses internal helpers from `app/models/_validators.py` to check the URL format and title length (max 256 characters).
2.  **Persistence**: The bookmark is saved to the `BookmarkRepository`.
3.  **Indexing**: The `SearchIndex` is updated with the new bookmark's content.
4.  **Cache Management**: Any existing cache entry for the ID is invalidated.

### State Transitions
The service provides explicit methods for managing bookmark status:
- `archive_bookmark(id)`: Moves a bookmark to the `ARCHIVED` status.
- `delete_bookmark(id)`: Performs a **soft-delete** by moving the bookmark to the `TRASHED` status using the `trash()` method on the `Bookmark` model.
- `restore_bookmark(id)`: Returns a trashed or archived bookmark to the `ACTIVE` status.

## Tag Management and Cascading Cleanup

Tags are managed via the `Tag` model in `app/models/tag.py`. The `BookmarkService` ensures data integrity between tags and bookmarks, particularly during deletion.

### Cascading Deletion
A critical feature of the `BookmarkService` is the `delete_tag(tag_id)` method. When a tag is deleted, the service automatically:
1.  Identifies all bookmarks associated with that tag using `_repo.get_bookmarks_with_tag(tag_id)`.
2.  Removes the tag reference from each bookmark.
3.  Saves the updated bookmarks back to the repository.
4.  Invalidates the cache for every affected bookmark.

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

## Collections: Manual and Smart

The service supports organizing bookmarks into `Collection` entities (defined in `app/models/collection.py`). There are two types of collections defined by the `CollectionType` enum:

1.  **Manual Collections**: Users explicitly add or remove bookmark IDs using `add_to_collection` and `remove_from_collection`.
2.  **Smart Collections**: These use a `filter_rule` (a query string) to dynamically determine which bookmarks belong to the collection. While the `Collection` model includes an internal `_apply_filter` method, the `BookmarkService` currently focuses on managing the collection metadata and manual assignments.

## Performance and Search Indexing

To maintain performance with an in-memory dataset, the service utilizes two specialized components:

### LRU Cache
The `LRUCache` in `app/services/_cache.py` reduces repository lookups. The `get_bookmark` method implements a standard "read-through" pattern:

```python
def get_bookmark(self, bookmark_id: str) -> Optional[Bookmark]:
    cached = self._cache.get(bookmark_id)
    if cached is not None:
        return cached
    bookmark = self._repo.get_bookmark(bookmark_id)
    if bookmark:
        self._cache.put(bookmark.id, bookmark)
    return bookmark
```

### Full-Text Search
The `SearchIndex` in `app/services/search_service.py` provides an inverted index. It tokenizes the `title` and `description` of bookmarks, removing common stop words (e.g., "the", "and", "or"). 

- **Incremental Updates**: Every time a bookmark is created or updated via the service, `self._search.index_bookmark(bookmark)` is called to keep the index fresh.
- **Ranking**: Search results are ranked by relevance based on the frequency of query tokens within the bookmark's metadata.