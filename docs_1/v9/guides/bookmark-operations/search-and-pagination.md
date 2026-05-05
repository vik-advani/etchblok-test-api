---
title: Search and Pagination
description: How to perform full-text searches and retrieve paginated lists of bookmarks with status filters.
code_symbols: [SYM#dd5fd545693e00d6c0b38892bbd18ed4afa136a1, SYM#fdcfaed9972e477ae4585fd1b279ac107fa534cd]
section_id: 2027e134-0d67-4308-b97c-4ebee21fd3ea_search_and_pagination
doc_type: how_to
section_type: guide
---
To retrieve bookmarks in this system, you can either fetch a paginated list with optional status filters or perform a full-text search across titles and descriptions. Both operations are managed by the `BookmarkService`.

## Paginated Listing with Filters

To retrieve a specific page of bookmarks, use the `list_bookmarks` method. This method returns a tuple containing the list of bookmarks for the requested page and the total count of matching items.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()

# Retrieve the first page of active bookmarks
bookmarks, total = service.list_bookmarks(
    page=1, 
    per_page=25, 
    status="active"
)

for b in bookmarks:
    print(f"{b.title}: {b.url}")
print(f"Total matching bookmarks: {total}")
```

### Key Parameters
- **`page`**: The 1-based page number (default is `1`).
- **`per_page`**: The number of items to return per page (default is `25`).
- **`status`**: An optional filter string. Valid values are defined in `BookmarkStatus`: `"active"`, `"archived"`, or `"trashed"`.

### Implementation in Routes
In the API layer (`app/routes/bookmarks.py`), this is typically exposed via query parameters:

```python
@bookmarks_bp.route("/", methods=["GET"])
def list_bookmarks():
    page = request.args.get("page", 1, type=int)
    per_page = request.args.get("per_page", 25, type=int)
    status = request.args.get("status", None)
    
    bookmarks, total = _service.list_bookmarks(page=page, per_page=per_page, status=status)
    
    return jsonify({
        "bookmarks": [b.to_dict() for b in bookmarks], 
        "total": total
    })
```

## Full-Text Search

The `search` method provides a full-text search capability across bookmark titles and descriptions. It uses an internal `SearchIndex` that ranks results based on relevance.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()

# Search for bookmarks containing "python" and "tutorial"
results = service.search(query="python tutorial", limit=10)

for bookmark in results:
    print(f"Found: {bookmark.title}")
```

### Search Logic
The underlying `SearchIndex` (found in `app/services/search_service.py`) implements the following logic:
1.  **Tokenization**: The query is split into lowercase tokens, and common stop words (e.g., "the", "and", "for") are removed.
2.  **AND-based Matching**: A bookmark must contain **all** tokens from the query to be included in the results.
3.  **Ranking**: Results are ranked by the frequency of the query tokens appearing in the bookmark's title and description.

### Implementation in Routes
The search endpoint in `app/routes/bookmarks.py` demonstrates how to handle search requests:

```python
@bookmarks_bp.route("/search", methods=["GET"])
def search_bookmarks():
    query = request.args.get("q", "")
    limit = request.args.get("limit", 20, type=int)
    
    results = _service.search(query, limit=limit)
    
    return jsonify({
        "results": [b.to_dict() for b in results], 
        "count": len(results)
    })
```

## Troubleshooting and Constraints

### Search Index Persistence
The `SearchIndex` is an in-memory inverted index. It is rebuilt from the `BookmarkRepository` every time the `BookmarkService` (a singleton) is initialized. While it updates incrementally when you call `create_bookmark` or `update_bookmark`, any data not persisted in the repository will be lost on restart.

### Status Filter Fail-soft
If an invalid status string is passed to `list_bookmarks`, the `BookmarkRepository` catches the `ValueError` and returns all bookmarks regardless of status, rather than raising an error.

### Search Limits
The `SearchIndex` has a hard-coded `MAX_SEARCH_RESULTS` of 100 (defined in `app/services/search_service.py`). Even if you request a higher `limit` in the `search()` method, the internal index will not return more than this maximum.

### Pagination Indexing
Pagination is **1-based**. Passing `page=0` will result in an empty list or incorrect slicing because the internal calculation used in `BookmarkRepository.list_bookmarks` is `start = (page - 1) * per_page`.