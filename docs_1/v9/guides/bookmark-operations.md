---
title: Bookmark Operations
description: The primary service facade for creating, updating, archiving, and managing the lifecycle of bookmarks and their associations.
code_symbols: [SYM#dd5fd545693e00d6c0b38892bbd18ed4afa136a1]
section_id: 7099b7ab-047c-42ce-a459-80a770cab065_bookmark_operations
doc_type: guide
section_type: guide
---
Bookmark operations are managed through a centralized service facade that orchestrates domain logic, persistence, and cross-entity consistency. The system uses a state-based lifecycle for bookmarks, allowing them to transition between active, archived, and trashed states without immediate hard deletion.

## The Bookmark Service Facade

The `BookmarkService` (found in `app/services/bookmark_service.py`) acts as the primary entry point for all bookmark-related actions. It is implemented as a **Singleton**, ensuring that state—such as the in-memory repository and search index—is shared across different Flask blueprints.

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

The service coordinates several internal components:
- **`BookmarkRepository`**: Handles in-memory storage and retrieval via `_repo`.
- **`SearchIndex`**: Provides full-text search capabilities via `_search`.
- **`LRUCache`**: Optimizes frequent lookups via `_cache`.

## Bookmark Lifecycle and States

The `Bookmark` entity (defined in `app/models/bookmark.py`) is a dataclass that tracks its current visibility via the `BookmarkStatus` enum.

### Status Transitions
Bookmarks move through three primary states:
1.  **ACTIVE**: The default state for new bookmarks.
2.  **ARCHIVED**: For bookmarks that are saved but no longer in active use.
3.  **TRASHED**: A "soft-delete" state.

The `Bookmark` class provides internal methods for these transitions, which are exposed via the `BookmarkService`:

```python
# app/models/bookmark.py

def archive(self) -> None:
    """Move the bookmark to the archive."""
    self.status = BookmarkStatus.ARCHIVED
    self._touch()

def trash(self) -> None:
    """Soft-delete the bookmark by moving it to the trash."""
    self.status = BookmarkStatus.TRASHED
    self._touch()

def restore(self) -> None:
    """Restore a trashed or archived bookmark to active status."""
    self.status = BookmarkStatus.ACTIVE
    self._touch()
```

### Soft Deletion vs. Hard Deletion
In this implementation, `delete_bookmark` in the service layer performs a **soft-delete** by calling `bookmark.trash()`. This allows users to restore bookmarks later. In contrast, operations like `delete_tag` are destructive; when a tag is deleted via `delete_tag(tag_id)`, the service iterates through all associated bookmarks and removes the tag ID from their internal lists.

## Creating and Updating Bookmarks

The service layer enforces validation before persisting data. When creating a bookmark, the service validates the URL and title using internal validators (`_validate_url`, `_validate_title`).

```python
# app/services/bookmark_service.py

def create_bookmark(self, data: Dict[str, Any]) -> Tuple[Optional[Bookmark], Optional[str]]:
    error = _validate_url(data.get("url", "")) or _validate_title(data.get("title", ""))
    if error:
        return None, error

    bookmark = Bookmark.from_dict(data)
    self._repo.save_bookmark(bookmark)
    self._search.index_bookmark(bookmark)
    self._cache.invalidate(bookmark.id)
    return bookmark, None
```

Updates are handled via `update_bookmark`, which performs partial updates on fields like `title`, `description`, and `url`. Every modification triggers a call to `bookmark._touch()`, which updates the `updated_at` timestamp.

## Managing Associations

Bookmarks can be associated with **Tags** and **Collections**. These relationships are managed by the service to ensure referential integrity.

### Tag Management
Tags are stored as a list of IDs on the `Bookmark` object. While tags can be initialized during bookmark creation, the `BookmarkService` also manages the global lifecycle of tags. If a tag is deleted via `delete_tag`, the service ensures it is stripped from all bookmarks:

```python
# app/services/bookmark_service.py

def delete_tag(self, tag_id: str) -> bool:
    # ...
    for bookmark in self._repo.get_bookmarks_with_tag(tag_id):
        bookmark.remove_tag(tag_id)
        self._repo.save_bookmark(bookmark)
        self._cache.invalidate(bookmark.id)
    self._repo.delete_tag(tag_id)
    return True
```

### Collection Management
Collections (defined in `app/models/collection.py`) are separate entities that group bookmarks. The `BookmarkService` provides methods to manage these groupings:

- `add_to_collection(collection_id, bookmark_id)`: Adds a bookmark reference to a collection.
- `remove_from_collection(collection_id, bookmark_id)`: Removes the reference.

## Search and Retrieval

The service provides two primary ways to retrieve bookmarks: paginated listing and full-text search.

### Paginated Listing
The `list_bookmarks` method allows filtering by `BookmarkStatus` and supports pagination. It delegates the heavy lifting to the `BookmarkRepository`.

```python
# app/services/bookmark_service.py
def list_bookmarks(self, page: int = 1, per_page: int = 25, status: Optional[str] = None):
    return self._repo.list_bookmarks(page=page, per_page=per_page, status=status)
```

### Full-Text Search
The `search` method queries the internal `SearchIndex`. This index is updated whenever a bookmark is created or updated, ensuring search results remain consistent with the repository.

```python
# app/services/bookmark_service.py
def search(self, query: str, limit: int = 20) -> List[Bookmark]:
    """Full-text search across bookmarks."""
    return self._search.search(query, limit=limit)
```