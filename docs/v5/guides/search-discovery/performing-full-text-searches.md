---
title: Performing Full-Text Searches
description: Learn how to query the search service to find bookmarks, including details on the AND-based token matching and result limiting.
code_symbols: [SYM#b0cc1ddc5c9e6b6675ff73174df52949062c8da5]
section_id: e216dfa2-3a67-44f7-8dd6-ccf06ded9045_performing_full-text_searches
doc_type: how_to
section_type: guide
---
To perform full-text searches across bookmark titles and descriptions, use the `BookmarkService.search` method. This service leverages an in-memory `SearchIndex` that provides token-based matching and relevance ranking.

### Searching via the Service
The most common way to perform a search is through the `BookmarkService` singleton.

```python
from app.services.bookmark_service import BookmarkService

# Get the service instance
service = BookmarkService()

# Search for bookmarks containing "python" and "tutorial"
# Returns a list of Bookmark objects
results = service.search(query="python tutorial", limit=10)

for bookmark in results:
    print(f"Found: {bookmark.title} ({bookmark.url})")
```

### Searching via the REST API
The search functionality is exposed via the `/api/bookmarks/search` endpoint.

```bash
# Search for bookmarks with the query "flask"
curl "http://localhost:5000/api/bookmarks/search?q=flask&limit=5"
```

The API returns a JSON object containing the matching bookmarks and the total count:
```json
{
  "count": 1,
  "results": [
    {
      "id": "...",
      "title": "Flask Documentation",
      "description": "The official documentation for the Flask web framework.",
      "url": "https://flask.palletsprojects.com/",
      "status": "active",
      "tags": [],
      "created_at": "...",
      "updated_at": "..."
    }
  ]
}
```

### How Search Works
The `SearchIndex` class in `app.services.search_service` implements the search logic using the following principles:

1.  **Tokenization**: The query is split into lowercase tokens using the regex `[a-z0-9]+`.
2.  **Stop Word Removal**: Common words defined in `_STOP_WORDS` (e.g., "the", "and", "is") are filtered out of both the index and the search query.
3.  **AND-based Matching**: All tokens in the query must be present in either the title or the description for a bookmark to be included in the results.
4.  **Relevance Ranking**: Results are sorted by the number of times the query tokens appear in the bookmark's title and description combined.

### Ranking Logic
The `_rank_results` method calculates a score for each matching bookmark based on token frequency:

```python
# From app/services/search_service.py
@staticmethod
def _rank_results(bookmarks: List[Bookmark], tokens: List[str]) -> List[Bookmark]:
    """Rank results by number of token occurrences in title + description."""
    def score(b: Bookmark) -> int:
        text = f"{b.title} {b.description}".lower()
        return sum(text.count(t) for t in tokens)

    return sorted(bookmarks, key=score, reverse=True)
```

### Troubleshooting and Gotchas

#### Stop Words are Ignored
If your search query consists only of stop words (e.g., searching for "the and"), the `search` method will return an empty list because no valid tokens remain after filtering.

#### Search Includes Trashed Bookmarks
The `SearchIndex` is updated when bookmarks are created or modified, but the `BookmarkService.delete_bookmark` method performs a soft-delete (changing status to `trashed`) without removing the bookmark from the search index. Consequently, search results may include bookmarks that have been archived or moved to the trash.

#### Case Insensitivity
All searches are case-insensitive. Both the indexed content and the search queries are converted to lowercase during tokenization.

#### Index Rebuilds
The search index is entirely in-memory. It is rebuilt from the repository whenever the `BookmarkService` is initialized (e.g., on application startup). For large datasets, this may cause a slight delay during the initial bootstrap.