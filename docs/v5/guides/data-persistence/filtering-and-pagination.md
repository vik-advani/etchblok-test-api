---
title: Filtering and Pagination
description: A guide to using the repository's built-in pagination and status-based filtering for efficient data retrieval.
code_symbols: [SYM#adb8232356346a5957ff3a1a1b7ff70581f37649]
section_id: edeffd7a-34e6-47c2-bd79-95398750674d_filtering_and_pagination
doc_type: guide
section_type: guide
---
The repository layer in this application provides a unified mechanism for retrieving bookmarks with support for pagination, status-based filtering, and chronological sorting. This functionality is primarily implemented in the `BookmarkRepository` and exposed through the `BookmarkService`.

## Status-Based Filtering

Filtering is driven by the `BookmarkStatus` enumeration defined in `app/models/bookmark.py`. Every bookmark exists in one of three states:

```python
class BookmarkStatus(Enum):
    """Visibility status of a bookmark."""
    ACTIVE = "active"
    ARCHIVED = "archived"
    TRASHED = "trashed"
```

When listing bookmarks, the repository can filter the results to match one of these specific statuses. If no status is provided, the repository returns all bookmarks regardless of their state.

### Implementation Details
In `app/db/repository.py`, the `list_bookmarks` method handles the filtering logic. It attempts to convert the provided status string into a `BookmarkStatus` enum member. If the string does not match a valid status (e.g., "deleted" instead of "trashed"), the filter is silently ignored, and the full list is processed.

```python
# app/db/repository.py

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
            # Invalid status strings result in no filtering
            pass
    # ... sorting and pagination follow
```

## Pagination and Sorting

The repository implements a 1-based pagination system. It returns a tuple containing the requested slice of items and the total count of items matching the filter criteria (before slicing).

### Sorting Order
Before pagination is applied, the repository always sorts the items by their `created_at` timestamp in descending order. This ensures that the most recently created bookmarks appear first in the results.

```python
items.sort(key=lambda b: b.created_at, reverse=True)
```

### Slicing Logic
The pagination uses the `page` and `per_page` arguments to calculate the start index. Note that because the system uses 1-based indexing for pages, the start index is calculated as `(page - 1) * per_page`.

```python
total = len(items)
start = (page - 1) * per_page
return items[start : start + per_page], total
```

**Warning:** Providing a `page` value of 0 or less will result in negative indexing during the slice operation, which may lead to unexpected results.

## Service Layer Integration

The `BookmarkService` in `app/services/bookmark_service.py` acts as a facade for the repository. It provides a clean interface for the rest of the application to access these features without interacting with the data store directly.

```python
# app/services/bookmark_service.py

def list_bookmarks(
    self, page: int = 1, per_page: int = 25, status: Optional[str] = None
) -> Tuple[List[Bookmark], int]:
    """Return a paginated list of bookmarks."""
    return self._repo.list_bookmarks(page=page, per_page=per_page, status=status)
```

## API Usage

The filtering and pagination features are exposed via the `/bookmarks/` GET endpoint. The route handler in `app/routes/bookmarks.py` extracts these parameters from the request's query string.

| Parameter | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `page` | `int` | `1` | The 1-based page index to retrieve. |
| `per_page` | `int` | `25` | The number of items to return per page. |
| `status` | `str` | `None` | Filter by `active`, `archived`, or `trashed`. |

### Example Request
To retrieve the second page of archived bookmarks with 10 items per page:

`GET /bookmarks/?page=2&per_page=10&status=archived`

The response includes the total count to allow clients to calculate the total number of available pages:

```json
{
  "bookmarks": [...],
  "total": 42
}
```