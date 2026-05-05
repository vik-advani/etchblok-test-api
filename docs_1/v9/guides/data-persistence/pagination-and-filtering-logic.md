---
title: Pagination and Filtering Logic
description: How to use the repository to fetch paginated slices of bookmarks and apply status-based filters.
code_symbols: [SYM#adb8232356346a5957ff3a1a1b7ff70581f37649]
section_id: 4885f1b0-e753-495b-a4ab-b5c0f46a29d5_pagination_and_filtering_logic
doc_type: guide
section_type: guide
---
The `BookmarkRepository` in `app/db/repository.py` provides a centralized mechanism for retrieving bookmarks with support for pagination and status-based filtering. Because the repository is an in-memory store, these operations are performed on a list of `Bookmark` objects stored in a dictionary.

## The list_bookmarks Method

The primary entry point for fetching multiple bookmarks is the `list_bookmarks` method. It is designed to handle large sets of data by returning only a specific "slice" of the total bookmarks based on the requested page and page size.

```python
def list_bookmarks(
    self,
    page: int = 1,
    per_page: int = 25,
    status: Optional[str] = None,
) -> Tuple[List[Bookmark], int]:
    """Return a paginated slice of bookmarks."""
    items = list(self._bookmarks.values())
    
    # 1. Filtering
    if status:
        try:
            target = BookmarkStatus(status)
            items = [b for b in items if b.status == target]
        except ValueError:
            # Invalid status strings are ignored, returning all items
            pass
            
    # 2. Sorting
    items.sort(key=lambda b: b.created_at, reverse=True)
    
    # 3. Pagination
    total = len(items)
    start = (page - 1) * per_page
    return items[start : start + per_page], total
```

### Status Filtering

Filtering is applied using the `BookmarkStatus` enum defined in `app/models/bookmark.py`. The repository attempts to convert the provided `status` string into a `BookmarkStatus` member.

Supported status values include:
- `active`: The default state for new bookmarks.
- `archived`: Bookmarks moved to the archive via `Bookmark.archive()`.
- `trashed`: Bookmarks soft-deleted via `Bookmark.trash()`.

If an invalid status string is provided (e.g., `?status=unknown`), the repository catches the `ValueError` and proceeds without applying any filter, effectively returning bookmarks of all statuses.

### Pagination Mechanics

The repository uses **1-based indexing** for the `page` parameter, which is common for consumer-facing APIs. 

- **Page Calculation**: The starting index for the slice is calculated as `(page - 1) * per_page`.
- **Total Count**: The method returns a tuple containing the list of items for the current page and the *total* count of items matching the filter. This total count is essential for front-end components to render pagination controls.
- **Empty Results**: If the `page` requested is beyond the available data (e.g., page 10 when only 2 pages exist), the repository returns an empty list and the correct total count, following standard Python slicing behavior.

### Sorting and Ordering

The repository enforces a strict "Newest First" ordering. Before slicing the data for pagination, it sorts the filtered list by the `created_at` attribute in reverse chronological order:

```python
items.sort(key=lambda b: b.created_at, reverse=True)
```

## Integration Flow

The pagination and filtering logic flows from the HTTP request through the service layer to the repository.

1.  **Route Layer (`app/routes/bookmarks.py`)**: Extracts query parameters from the request.
    ```python
    page = request.args.get("page", 1, type=int)
    per_page = request.args.get("per_page", 25, type=int)
    status = request.args.get("status", None)
    bookmarks, total = _service.list_bookmarks(page=page, per_page=per_page, status=status)
    ```
2.  **Service Layer (`app/services/bookmark_service.py`)**: The `BookmarkService.list_bookmarks` method acts as a pass-through to the repository.
3.  **Repository Layer (`app/db/repository.py`)**: Performs the actual filtering, sorting, and slicing as described above.

## Related Query Methods

While `list_bookmarks` is the primary method for general listing, the repository also provides `get_bookmarks_with_tag(tag_id: str)`. Unlike `list_bookmarks`, this method returns a simple list of all matching bookmarks without pagination or status filtering, intended for internal relational lookups (such as when deleting a tag).