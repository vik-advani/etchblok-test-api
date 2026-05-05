---
title: Search System Architecture
description: An overview of the inverted index implementation and how it maintains a mapping between text tokens and bookmark identifiers.
code_symbols: [SYM#0f269a750bc62c4d874086090a88d14329456024, SYM#b0cc1ddc5c9e6b6675ff73174df52949062c8da5]
section_id: 855da2de-33e3-4cec-ae7a-24269a0cb145_search_system_architecture
doc_type: guide
section_type: guide
---
The search system in this codebase is implemented as an in-memory inverted index managed by the `SearchIndex` class in `app.services.search_service`. It provides full-text search capabilities across bookmark titles and descriptions without requiring an external search engine like Elasticsearch or Typesense.

## The Inverted Index Structure

The core of the search system is an inverted index stored in the `_index` attribute of the `SearchIndex` class. It uses a `defaultdict(set)` where:
- **Keys** are individual text tokens (words).
- **Values** are sets of bookmark IDs that contain that token.

```python
# app/services/search_service.py

class SearchIndex:
    def __init__(self, repository: "BookmarkRepository") -> None:
        self._repo = repository
        self._index: Dict[str, Set[str]] = defaultdict(set)
        self._rebuild()
```

This structure allows for extremely fast lookups: to find all bookmarks containing the word "python", the system simply retrieves the set associated with the "python" key.

## The Indexing Pipeline

When a bookmark is indexed via `index_bookmark(bookmark)`, the system processes the text through a pipeline to ensure consistent and relevant results.

### 1. Tokenization and Normalization
The `_tokenize` method converts the combined title and description to lowercase and uses a regular expression (`_TOKEN_RE = re.compile(r"[a-z0-9]+")`) to extract alphanumeric tokens.

### 2. Stop Word Removal
To prevent the index from being bloated by common words that don't add semantic value, the system filters out tokens found in the `_STOP_WORDS` set.

```python
# app/services/search_service.py

_STOP_WORDS: Set[str] = {"the", "a", "an", "and", "or", "but", "in", "on", "at", "to", "for", "is", "it"}

def _tokenize(self, text: str) -> List[str]:
    """Split text into lowercase tokens, removing stop words."""
    tokens = _TOKEN_RE.findall(text.lower())
    return [t for t in tokens if t not in _STOP_WORDS]
```

### 3. Incremental Updates
The `index_bookmark` method handles both new bookmarks and updates to existing ones. It first removes any existing references to the bookmark ID to prevent stale tokens from persisting, then adds the new tokens.

```python
def index_bookmark(self, bookmark: Bookmark) -> None:
    self._remove_bookmark_from_index(bookmark.id)
    tokens = self._tokenize(f"{bookmark.title} {bookmark.description}")
    for token in tokens:
        self._index[token].add(bookmark.id)
```

## Search Logic and Ranking

The `search` method implements a strict **AND** logic for multi-word queries and a basic frequency-based ranking system.

### Intersection-Based Matching
When a user searches for multiple words (e.g., "python tutorial"), the system tokenizes the query and finds the intersection of the ID sets for every token. This means a bookmark must contain *all* query tokens to be returned.

```python
# Logic inside SearchIndex.search
candidate_ids: Set[str] = self._index.get(tokens[0], set()).copy()
for token in tokens[1:]:
    candidate_ids &= self._index.get(token, set())
```

### Relevance Ranking
Once matching bookmarks are retrieved from the `BookmarkRepository`, they are ranked by the `_rank_results` method. The score is calculated by counting the total number of times the query tokens appear in the bookmark's title and description.

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

The `SearchIndex` is managed by the `BookmarkService`, which ensures the index stays synchronized with the underlying data.

### Initialization
When the application starts, `BookmarkService` initializes the `SearchIndex`. The index performs a full rebuild by fetching all bookmarks from the `BookmarkRepository` via the `_rebuild` method.

### Synchronization Points
The `BookmarkService` triggers indexing during the following operations:
- **Creation**: `create_bookmark` calls `self._search.index_bookmark(bookmark)`.
- **Updates**: `update_bookmark` calls `self._search.index_bookmark(bookmark)`.

### Limitations and Performance
- **Memory Usage**: Since the index is entirely in-memory, it is limited by the available RAM. It is designed for small to medium datasets.
- **Deletion Complexity**: The `_remove_bookmark_from_index` method iterates over every token in the index to remove a bookmark ID. This is an $O(N)$ operation where $N$ is the number of unique tokens in the entire system.
- **Status Filtering**: The current implementation of `delete_bookmark` in `BookmarkService` performs a "soft delete" (changing status to `trashed`) but does not explicitly remove the bookmark from the `SearchIndex`. Consequently, trashed bookmarks may still appear in search results until the next full rebuild.