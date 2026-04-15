---
title: Search & Discovery
description: Full-text search indexing and ranking capabilities to find bookmarks based on titles and descriptions.
code_symbols: [SYM#0f269a750bc62c4d874086090a88d14329456024]
section_id: 1a99dd5b-97ce-4ebc-aa33-03192867fe66_search___discovery
doc_type: explanation
section_type: guide
---
The search and discovery system in this project is built around a custom, in-memory inverted index. It provides full-text search capabilities across bookmark titles and descriptions, prioritizing simplicity and speed for small-to-medium datasets.

## The Inverted Index

The core of the search functionality resides in the `SearchIndex` class within `app/services/search_service.py`. This class maintains a mapping of tokens (words) to sets of bookmark IDs.

### Tokenization and Normalization
Before indexing or searching, text is processed through a private `_tokenize` method. This method ensures consistency by:
1.  Converting all text to lowercase.
2.  Extracting alphanumeric tokens using the regex `[a-z0-9]+`.
3.  Filtering out common stop words (e.g., "the", "and", "is") defined in the `_STOP_WORDS` set.

```python
# app/services/search_service.py

_STOP_WORDS: Set[str] = {"the", "a", "an", "and", "or", "but", "in", "on", "at", "to", "for", "is", "it"}
_TOKEN_RE = re.compile(r"[a-z0-9]+")

def _tokenize(self, text: str) -> List[str]:
    """Split text into lowercase tokens, removing stop words."""
    tokens = _TOKEN_RE.findall(text.lower())
    return [t for t in tokens if t not in _STOP_WORDS]
```

### Indexing Strategy
The index is built by concatenating the bookmark's title and description. When a bookmark is indexed via `index_bookmark`, any existing entries for that bookmark ID are first purged to prevent stale data during updates.

```python
def index_bookmark(self, bookmark: Bookmark) -> None:
    """Add or update a bookmark in the index."""
    self._remove_bookmark_from_index(bookmark.id)
    tokens = self._tokenize(f"{bookmark.title} {bookmark.description}")
    for token in tokens:
        self._index[token].add(bookmark.id)
```

## Search Execution and Ranking

The search process follows a strict "AND" matching strategy. For a bookmark to be returned as a result, it must contain **all** tokens present in the search query.

### Query Processing
The `search` method tokenizes the input query and performs a set intersection across the matching bookmark IDs for each token. This ensures that multi-word queries narrow down results effectively.

```python
# app/services/search_service.py

def search(self, query: str, limit: int = 20) -> List[Bookmark]:
    tokens = self._tokenize(query)
    if not tokens:
        return []

    # Start with the set of IDs for the first token
    candidate_ids: Set[str] = self._index.get(tokens[0], set()).copy()
    
    # Intersect with sets for subsequent tokens (AND logic)
    for token in tokens[1:]:
        candidate_ids &= self._index.get(token, set())

    results = []
    for bid in candidate_ids:
        bookmark = self._repo.get_bookmark(bid)
        if bookmark:
            results.append(bookmark)

    return self._rank_results(results, tokens)[:limit]
```

### Relevance Ranking
Once matching bookmarks are identified, they are ranked using a simple frequency-based algorithm in `_rank_results`. The score for a bookmark is the total number of times the search tokens appear in its combined title and description.

```python
@staticmethod
def _rank_results(bookmarks: List[Bookmark], tokens: List[str]) -> List[Bookmark]:
    """Rank results by number of token occurrences in title + description."""
    def score(b: Bookmark) -> int:
        text = f"{b.title} {b.description}".lower()
        return sum(text.count(t) for t in tokens)

    return sorted(bookmarks, key=score, reverse=True)
```

## Service Integration

The `BookmarkService` acts as a facade, orchestrating the `SearchIndex` alongside the `BookmarkRepository`. This integration ensures that the search index remains synchronized with the underlying data store.

### Incremental Updates
The index is updated incrementally during standard CRUD operations. In `app/services/bookmark_service.py`, both `create_bookmark` and `update_bookmark` trigger a re-indexing of the affected bookmark:

```python
# app/services/bookmark_service.py

def create_bookmark(self, data: Dict[str, Any]) -> Tuple[Optional[Bookmark], Optional[str]]:
    # ... validation and saving ...
    self._search.index_bookmark(bookmark)
    return bookmark, None

def update_bookmark(self, bookmark_id: str, data: Dict[str, Any]) -> Tuple[Optional[Bookmark], Optional[str]]:
    # ... validation and updating ...
    self._search.index_bookmark(bookmark)
    return bookmark, None
```

### Initialization and Persistence
Because the index is entirely in-memory, it is rebuilt from the repository every time the application starts. The `SearchIndex.__init__` method calls `_rebuild`, which iterates through all existing bookmarks in the repository to populate the index.

## Design Tradeoffs and Constraints

The current implementation makes several specific design choices that impact its behavior:

1.  **In-Memory Volatility**: The index does not persist to disk. While this makes search operations extremely fast, it introduces a startup delay as the index is rebuilt from the repository.
2.  **Strict AND Matching**: The "AND" strategy means that a query for "Python tutorial" will not return a bookmark that only contains "Python". This provides high precision but may result in zero hits for longer, more specific queries.
3.  **Stop Word Limitations**: If a user searches for a phrase consisting entirely of stop words (e.g., "the and"), the `_tokenize` method returns an empty list, and the search returns no results.
4.  **Soft-Delete Synchronization**: When a bookmark is trashed via `delete_bookmark`, it is not explicitly removed from the `SearchIndex`. However, because `SearchIndex.search` retrieves the actual bookmark object from the repository before returning it, the system relies on the repository's state to determine if a result is still valid.
5.  **Scaling**: The `_rebuild` method fetches up to 10,000 bookmarks at once. For datasets exceeding this size, the initialization logic would need to be paginated or moved to a dedicated search engine like Elasticsearch or Typesense.