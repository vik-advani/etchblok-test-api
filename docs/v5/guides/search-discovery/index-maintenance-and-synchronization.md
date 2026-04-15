---
title: Index Maintenance and Synchronization
description: Instructions for keeping the search index synchronized with the bookmark repository during create, update, and delete operations.
code_symbols: [SYM#0f269a750bc62c4d874086090a88d14329456024, SYM#b0cc1ddc5c9e6b6675ff73174df52949062c8da5]
section_id: e5d8369f-1df9-4a62-9422-93b01ef21b5e_index_maintenance_and_synchronization
doc_type: how_to
section_type: guide
---
To keep the search index synchronized with the bookmark repository, you must trigger incremental updates during write operations. The `SearchIndex` class provides an in-memory inverted index that maps tokens from bookmark titles and descriptions to their respective IDs.

### Synchronizing on Creation and Updates

When a bookmark is created or modified, you must call `index_bookmark` to ensure the search results reflect the latest content. The `SearchIndex` automatically handles removing old tokens before adding new ones, making it safe to call for both new and existing bookmarks.

In `app.services.bookmark_service.BookmarkService`, this is implemented as follows:

```python
def create_bookmark(self, data: Dict[str, Any]) -> Tuple[Optional[Bookmark], Optional[str]]:
    # ... validation and persistence ...
    bookmark = Bookmark.from_dict(data)
    self._repo.save_bookmark(bookmark)
    
    # Update the search index
    self._search.index_bookmark(bookmark)
    
    self._cache.invalidate(bookmark.id)
    return bookmark, None

def update_bookmark(self, bookmark_id: str, data: Dict[str, Any]) -> Tuple[Optional[Bookmark], Optional[str]]:
    bookmark = self._repo.get_bookmark(bookmark_id)
    # ... update fields ...
    
    self._repo.save_bookmark(bookmark)
    
    # Re-index the bookmark with updated title/description
    self._search.index_bookmark(bookmark)
    
    self._cache.invalidate(bookmark.id)
    return bookmark, None
```

### Initial Bootstrapping

The `SearchIndex` is an in-memory service and does not persist its state to disk. It automatically synchronizes with the repository upon initialization by performing a full rebuild. This occurs when the `BookmarkService` singleton is first instantiated.

The `SearchIndex.__init__` method triggers this process:

```python
# app/services/search_service.py

class SearchIndex:
    def __init__(self, repository: "BookmarkRepository") -> None:
        self._repo = repository
        self._index: Dict[str, Set[str]] = defaultdict(set)
        self._rebuild()

    def _rebuild(self) -> None:
        """Rebuild the entire index from the repository."""
        self._index.clear()
        # Fetches all bookmarks to populate the in-memory index
        all_bookmarks, _ = self._repo.list_bookmarks(page=1, per_page=10000)
        for bookmark in all_bookmarks:
            self.index_bookmark(bookmark)
```

### Handling Deletions and Status Changes

In this codebase, `delete_bookmark` performs a "soft delete" by moving the bookmark to the trash. Currently, the `SearchIndex` **does not** remove bookmarks that are trashed or archived.

If you need to manually remove a bookmark from the index (e.g., for a hard delete), use the `remove_bookmark` method:

```python
def hard_delete_bookmark(self, bookmark_id: str) -> bool:
    # Example of manual removal from index
    self._search.remove_bookmark(bookmark_id)
    return self._repo.delete_bookmark(bookmark_id)
```

### Troubleshooting and Limitations

*   **Soft-Deleted Bookmarks**: The `search` method in `SearchIndex` does not filter by status. If a bookmark is in the trash but matches the query tokens, it will still be returned in search results.
*   **AND Logic**: The search implementation uses strict `AND` logic. All tokens in a query must be present in either the title or description for a bookmark to match.
*   **Indexed Fields**: Only the `title` and `description` fields are indexed. Changes to `url` or `tags` do not require a search index update as they are not searchable via `SearchIndex.search`.
*   **Stop Words**: The indexer filters out common stop words (e.g., "the", "is", "and"). If a search query consists only of stop words, `SearchIndex.search` will return an empty list.
*   **Memory Usage**: Since the index is stored entirely in memory as a `defaultdict(set)`, memory usage scales with the number of unique tokens and total bookmarks. The `_rebuild` method is capped at indexing the first 10,000 bookmarks.