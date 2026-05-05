---
title: Managing Index Consistency
description: Instructions for keeping the search index up-to-date as bookmarks are created, modified, or deleted in the repository.
code_symbols: [SYM#b0cc1ddc5c9e6b6675ff73174df52949062c8da5]
section_id: a67091ad-0310-47f9-84d6-9f14991db24f_managing_index_consistency
doc_type: how_to
section_type: guide
---
To keep the search index consistent with the bookmark repository, you must ensure that every creation or modification of a bookmark is followed by a call to the indexer. In this project, the `BookmarkService` acts as a facade that orchestrates these updates automatically.

### Updating the Index on Creation and Modification

When you create or update a bookmark via the `BookmarkService`, the service ensures the `SearchIndex` is updated immediately after the repository persists the change.

```python
# From app/services/bookmark_service.py

def create_bookmark(self, data: Dict[str, Any]) -> Tuple[Optional[Bookmark], Optional[str]]:
    # ... validation and model creation ...
    bookmark = Bookmark.from_dict(data)
    
    # 1. Persist to repository
    self._repo.save_bookmark(bookmark)
    
    # 2. Update the search index
    self._search.index_bookmark(bookmark)
    
    # 3. Invalidate cache
    self._cache.invalidate(bookmark.id)
    return bookmark, None
```

The `SearchIndex.index_bookmark` method handles both new bookmarks and updates to existing ones. It internally calls `_remove_bookmark_from_index(bookmark.id)` before processing the new tokens to ensure that old keywords associated with a modified bookmark are purged.

### Initializing the Index

The `SearchIndex` is an in-memory component. It is automatically populated when the `BookmarkService` (a singleton) is first initialized. 

```python
# From app/services/search_service.py

class SearchIndex:
    def __init__(self, repository: "BookmarkRepository") -> None:
        self._repo = repository
        self._index: Dict[str, Set[str]] = defaultdict(set)
        self._rebuild()

    def _rebuild(self) -> None:
        """Rebuild the entire index from the repository."""
        self._index.clear()
        # Fetches all bookmarks from the repo to populate the index
        all_bookmarks, _ = self._repo.list_bookmarks(page=1, per_page=10000)
        for bookmark in all_bookmarks:
            self.index_bookmark(bookmark)
```

If you need to force a full index refresh (e.g., after a bulk database operation), you can trigger a rebuild by re-initializing the service or calling the private `_rebuild()` method if you have access to the `SearchIndex` instance.

### Handling Deletions and Status Changes

In the current implementation, the `SearchIndex` does **not** automatically track status changes like trashing (soft-deletion) or archiving. 

#### Soft-Deletion Inconsistency
When `BookmarkService.delete_bookmark` is called, it marks the bookmark as trashed in the repository but does **not** remove it from the `SearchIndex`.

```python
# app/services/bookmark_service.py

def delete_bookmark(self, bookmark_id: str) -> bool:
    bookmark = self._repo.get_bookmark(bookmark_id)
    if not bookmark:
        return False
    bookmark.trash() # Status changed to 'trashed'
    self._repo.save_bookmark(bookmark)
    self._cache.invalidate(bookmark_id)
    # WARNING: self._search.remove_bookmark(bookmark_id) is NOT called here
    return True
```

#### Manual Removal
If you need to strictly exclude a bookmark from search results immediately after a status change, you must manually call `remove_bookmark`:

```python
# Example of manual consistency management
def hard_delete_and_unindex(service: BookmarkService, bookmark_id: str):
    # Accessing the internal search index from the service
    service._search.remove_bookmark(bookmark_id)
```

### Troubleshooting and Performance

*   **In-Memory Persistence:** Because the index is stored in `self._index` (a `defaultdict`), all search data is lost when the application restarts. It relies entirely on the `_rebuild()` process during startup to regain consistency with the `BookmarkRepository`.
*   **Removal Performance:** The `_remove_bookmark_from_index` method performs a full scan of the index dictionary to find and remove a bookmark ID. In a very large index, frequent updates or removals may cause performance degradation.
*   **Tokenization Logic:** The index only considers the `title` and `description` fields. Changes to other fields (like `url` or `tags`) do not require an index update as they are not currently indexed for full-text search.