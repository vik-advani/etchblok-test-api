---
title: Executing Search Queries
description: How to perform free-text searches across bookmark titles and descriptions using the search API.
code_symbols: [SYM#b0cc1ddc5c9e6b6675ff73174df52949062c8da5]
section_id: a81131f9-fec8-447d-927b-c405b928734a_executing_search_queries
doc_type: how_to
section_type: guide
---
To perform free-text searches across bookmark titles and descriptions, you can use the search API or the underlying service layer. The system uses an in-memory inverted index that tokenizes text and ranks results based on keyword frequency.

### Search via the API

The most common way to execute a search is via the `GET /api/bookmarks/search` endpoint. This endpoint accepts a query string and an optional limit.

```bash
# Search for bookmarks containing "python" and "tutorial"
curl "http://localhost:5000/api/bookmarks/search?q=python+tutorial&limit=10"
```

The response returns a list of matching bookmarks and the total count:

```json
{
  "count": 1,
  "results": [
    {
      "id": "b123",
      "title": "Advanced Python Tutorial",
      "description": "A deep dive into Python internals.",
      "url": "https://example.com/python",
      "status": "active",
      "tags": [],
      "created_at": "2023-10-27T10:00:00Z",
      "updated_at": "2023-10-27T10:00:00Z"
    }
  ]
}
```

### Search via the Service Layer

If you are working within the application code, use the `BookmarkService.search` method. This method acts as a facade for the `SearchIndex` class.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()

# Execute a search
results = service.search("python tutorial", limit=5)

for bookmark in results:
    print(f"Found: {bookmark.title} ({bookmark.url})")
```

### How Search Works

The `SearchIndex` class in `app/services/search_service.py` manages the search logic:

1.  **Tokenization**: The query is split into tokens using the regex `[a-z0-9]+`. All tokens are converted to lowercase.
2.  **Stop Word Removal**: Common words defined in `_STOP_WORDS` (e.g., "the", "and", "is") are filtered out.
3.  **AND Logic**: For multi-word queries, the index performs an intersection. A bookmark must contain **all** non-stop-word tokens from the query to be included in the results.
4.  **Ranking**: Results are ranked by the `_rank_results` method, which calculates a score based on the total number of times the query tokens appear in the bookmark's title and description.

```python
# Internal ranking logic from app/services/search_service.py
def _rank_results(bookmarks: List[Bookmark], tokens: List[str]) -> List[Bookmark]:
    """Rank results by number of token occurrences in title + description."""
    def score(b: Bookmark) -> int:
        text = f"{b.title} {b.description}".lower()
        return sum(text.count(t) for t in tokens)

    return sorted(bookmarks, key=score, reverse=True)
```

### Troubleshooting and Limitations

#### Stop Words Filtering
If your search query consists entirely of stop words (e.g., searching for "the and"), the `_tokenize` method will return an empty list, and the search will return no results.

#### Soft-Deleted Bookmarks
The `SearchIndex` is updated when bookmarks are created or updated via `BookmarkService`. However, when a bookmark is "deleted" (moved to trash), it is not automatically removed from the `SearchIndex`. Because `SearchIndex.search` retrieves bookmarks directly from the repository by ID, trashed or archived bookmarks will still appear in search results.

#### In-Memory Nature
The search index is entirely in-memory and is rebuilt from the repository every time the application starts:

```python
# From app/services/search_service.py
def _rebuild(self) -> None:
    """Rebuild the entire index from the repository."""
    self._index.clear()
    all_bookmarks, _ = self._repo.list_bookmarks(page=1, per_page=10000)
    for bookmark in all_bookmarks:
        self.index_bookmark(bookmark)
```

For very large datasets, this initialization process and memory usage will increase significantly.