---
title: Search and Indexing
description: Full-text search capabilities using an inverted index to provide relevant results across bookmark titles and descriptions.
code_symbols: [SYM#0f269a750bc62c4d874086090a88d14329456024]
section_id: aaef9d06-afc8-45b3-9f5c-4a9a8431b6d6_search_and_indexing
doc_type: explanation
section_type: guide
---
The search functionality in this project is implemented as a custom, in-memory inverted index. This design provides fast, full-text search capabilities without the overhead of external dependencies like Elasticsearch or Typesense, making it ideal for the small-to-medium datasets typically managed by a personal bookmarking tool.

## The Inverted Index Implementation

The core of the search system is the `SearchIndex` class located in `app/services/search_service.py`. It maintains a mapping of tokens (words) to sets of bookmark IDs.

### Tokenization and Normalization
Before text is indexed or searched, it passes through the `_tokenize` method. This method ensures consistency by:
1.  Converting all text to lowercase.
2.  Using a regular expression (`[a-z0-9]+`) to extract alphanumeric tokens.
3.  Filtering out common "stop words" defined in the `_STOP_WORDS` set (e.g., "the", "and", "is").

```python
_STOP_WORDS: Set[str] = {"the", "a", "an", "and", "or", "but", "in", "on", "at", "to", "for", "is", "it"}
_TOKEN_RE = re.compile(r"[a-z0-9]+")

def _tokenize(self, text: str) -> List[str]:
    """Split text into lowercase tokens, removing stop words."""
    tokens = _TOKEN_RE.findall(text.lower())
    return [t for t in tokens if t not in _STOP_WORDS]
```

### Indexing Strategy
When a bookmark is indexed via `index_bookmark`, the system concatenates the `title` and `description` fields. This combined string is tokenized, and the bookmark's ID is added to the set of IDs associated with each token in the `self._index` dictionary.

```python
def index_bookmark(self, bookmark: Bookmark) -> None:
    self._remove_bookmark_from_index(bookmark.id)
    tokens = self._tokenize(f"{bookmark.title} {bookmark.description}")
    for token in tokens:
        self._index[token].add(bookmark.id)
```

## Search Logic and Ranking

The `search` method employs an **AND strategy**. For a bookmark to be returned as a result, it must contain *all* tokens present in the search query.

### Filtering
The search begins by retrieving the set of IDs for the first token and then performing an intersection (`&=`) with the sets of IDs for all subsequent tokens:

```python
candidate_ids: Set[str] = self._index.get(tokens[0], set()).copy()
for token in tokens[1:]:
    candidate_ids &= self._index.get(token, set())
```

### Relevance Ranking
Once the candidate bookmarks are retrieved from the `BookmarkRepository`, they are ranked using the `_rank_results` helper. The ranking is based on a simple frequency score: the total number of times the query tokens appear in the combined title and description of the bookmark.

```python
@staticmethod
def _rank_results(bookmarks: List[Bookmark], tokens: List[str]) -> List[Bookmark]:
    """Rank results by number of token occurrences in title + description."""
    def score(b: Bookmark) -> int:
        text = f"{b.title} {b.description}".lower()
        return sum(text.count(t) for t in tokens)

    return sorted(bookmarks, key=score, reverse=True)
```

## Integration and Lifecycle

The `SearchIndex` is managed by the `BookmarkService` in `app/services/bookmark_service.py`, which ensures the index stays synchronized with the underlying data.

### Initialization
The index is a singleton component within the `BookmarkService`. On application startup, the `_init_services` method instantiates the `SearchIndex`, which immediately triggers a full rebuild by fetching all bookmarks from the `BookmarkRepository`.

```python
def _rebuild(self) -> None:
    """Rebuild the entire index from the repository."""
    self._index.clear()
    all_bookmarks, _ = self._repo.list_bookmarks(page=1, per_page=10000)
    for bookmark in all_bookmarks:
        self.index_bookmark(bookmark)
```

### Incremental Updates
To maintain accuracy without full rebuilds, the `BookmarkService` calls `index_bookmark` during both `create_bookmark` and `update_bookmark` operations. This ensures that new content is searchable immediately and modified content has its index entries refreshed.

## Design Tradeoffs and Constraints

The choice of an in-memory inverted index involves several specific tradeoffs:

*   **Memory Usage**: Since the entire index is stored in RAM, memory consumption scales with the number of bookmarks and the length of their descriptions. This is acceptable for personal use but would require a persistent index (like SQLite FTS or Elasticsearch) for multi-tenant or extremely large datasets.
*   **Startup Latency**: The index must be rebuilt every time the application starts. While fast for thousands of bookmarks, this could become a bottleneck as the dataset grows into the hundreds of thousands.
*   **Soft-Delete Behavior**: In the current implementation of `BookmarkService.delete_bookmark`, bookmarks are "trashed" (a soft-delete) but are **not** explicitly removed from the `SearchIndex`. Because `SearchIndex.search` retrieves the bookmark object to verify its existence, trashed bookmarks will still appear in search results as long as they exist in the repository.
*   **Strict Matching**: The AND-based strategy means that a query for "python tutorial" will not return a bookmark that only contains "python". This prioritizes precision over recall.