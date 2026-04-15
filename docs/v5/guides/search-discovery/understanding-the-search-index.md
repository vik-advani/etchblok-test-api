---
title: Understanding the Search Index
description: An overview of the inverted index architecture used for bookmark discovery, explaining how tokens are mapped to IDs and the role of the SearchIndex class.
code_symbols: [SYM#0f269a750bc62c4d874086090a88d14329456024, SYM#b0cc1ddc5c9e6b6675ff73174df52949062c8da5]
section_id: df7673fd-e5de-435c-b731-163b262c8123_understanding_the_search_index
doc_type: guide
section_type: guide
---
The `SearchIndex` class in `app/services/search_service.py` provides the discovery mechanism for bookmarks. It uses an in-memory inverted index to map text tokens to bookmark IDs, enabling fast full-text search across titles and descriptions.

## Inverted Index Architecture

The core of the search system is the `_index` attribute, which is a `defaultdict(set)` mapping strings (tokens) to sets of strings (bookmark IDs).

```python
# app/services/search_service.py

class SearchIndex:
    def __init__(self, repository: "BookmarkRepository") -> None:
        self._repo = repository
        self._index: Dict[str, Set[str]] = defaultdict(set)
        self._rebuild()
```

When a bookmark is indexed via `index_bookmark()`, its title and description are combined and broken down into tokens. Each token then becomes a key in the `_index` dictionary, with the bookmark's ID added to the associated set. This structure allows the system to instantly identify all bookmarks containing a specific word without scanning the entire database.

## Tokenization and Preprocessing

The `_tokenize` method transforms raw text into a searchable format. It performs three main operations:

1.  **Normalization**: Converts all text to lowercase.
2.  **Filtering**: Uses a regular expression (`_TOKEN_RE = re.compile(r"[a-z0-9]+")`) to extract alphanumeric sequences, effectively stripping punctuation.
3.  **Stop Word Removal**: Filters out common words defined in `_STOP_WORDS` (e.g., "the", "and", "is") that do not contribute significantly to search relevance.

```python
def _tokenize(self, text: str) -> List[str]:
    """Split text into lowercase tokens, removing stop words."""
    tokens = _TOKEN_RE.findall(text.lower())
    return [t for t in tokens if t not in _STOP_WORDS]
```

## Search and Ranking Logic

The `search()` method implements a strict "AND" logic. For a bookmark to be considered a candidate, it must contain **all** tokens present in the search query.

1.  **Candidate Selection**: The method retrieves the set of IDs for the first token and then performs a set intersection (`&=`) with the sets for all subsequent tokens.
2.  **Hydration**: It fetches the full `Bookmark` objects from the `BookmarkRepository` using the resulting IDs.
3.  **Ranking**: The results are passed to `_rank_results()`, which scores each bookmark based on the total number of times the query tokens appear in its title and description.

```python
@staticmethod
def _rank_results(bookmarks: List[Bookmark], tokens: List[str]) -> List[Bookmark]:
    """Rank results by number of token occurrences in title + description."""
    def score(b: Bookmark) -> int:
        text = f"{b.title} {b.description}".lower()
        return sum(text.count(t) for t in tokens)

    return sorted(bookmarks, key=score, reverse=True)
```

## Lifecycle and Synchronization

The `SearchIndex` is managed by the `BookmarkService` and is kept in sync with the underlying data store through several mechanisms:

*   **Initial Rebuild**: On startup, `SearchIndex.__init__` calls `_rebuild()`, which fetches all bookmarks from the repository to populate the index.
*   **Incremental Updates**: Whenever a bookmark is created or updated via `BookmarkService.create_bookmark` or `update_bookmark`, the service calls `index_bookmark()`. This method first removes any existing entries for that ID to prevent stale data before re-indexing the new content.
*   **Deletion**: When a bookmark is deleted, `remove_bookmark()` is called to purge its ID from all token sets in the index.

## Performance Considerations

Because the index is entirely in-memory, search operations are extremely fast. However, there are trade-offs:

*   **Memory Usage**: The index grows with the number of unique tokens and bookmarks.
*   **Removal Complexity**: The `_remove_bookmark_from_index` method iterates over every token in the index to discard the bookmark ID. This is an $O(N)$ operation relative to the number of unique tokens, which may become a bottleneck as the dataset grows.
*   **Volatility**: Since the index is not persisted to disk, it must be rebuilt from the `BookmarkRepository` every time the application starts.