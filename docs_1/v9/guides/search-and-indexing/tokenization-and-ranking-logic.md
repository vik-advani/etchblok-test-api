---
title: Tokenization and Ranking Logic
description: A technical explanation of how text is processed into tokens and how results are scored based on keyword frequency.
code_symbols: [SYM#b0cc1ddc5c9e6b6675ff73174df52949062c8da5]
section_id: 17dd74ad-a4cd-42b7-9fa8-94369bf81a35_tokenization_and_ranking_logic
doc_type: explanation
section_type: guide
---
The `SearchIndex` class in `app.services.search_service` provides an in-memory full-text search capability optimized for small-to-medium datasets. It implements a classic inverted index pattern, mapping individual words (tokens) to the bookmarks that contain them, allowing for efficient retrieval without scanning the entire database for every query.

## Tokenization Pipeline

The search process begins with tokenization, which transforms raw text into a normalized list of searchable terms. This logic is encapsulated in the `_tokenize` method and follows a three-step process:

1.  **Normalization**: The input text is converted to lowercase to ensure search is case-insensitive.
2.  **Regex Splitting**: The system uses a module-level regular expression `_TOKEN_RE = re.compile(r"[a-z0-9]+")` to extract alphanumeric sequences. This effectively strips punctuation and treats only words and numbers as valid tokens.
3.  **Stop Word Filtering**: Common English words that carry little semantic weight (e.g., "the", "and", "is") are filtered out using a predefined `_STOP_WORDS` set.

```python
def _tokenize(self, text: str) -> List[str]:
    """Split text into lowercase tokens, removing stop words."""
    tokens = _TOKEN_RE.findall(text.lower())
    return [t for t in tokens if t not in _STOP_WORDS]
```

This pipeline is applied both when indexing a bookmark (using its title and description) and when processing a user's search query.

## Inverted Index Structure

The core of the search engine is the `_index` attribute, a `defaultdict(set)` where keys are tokens and values are sets of bookmark IDs. 

When `index_bookmark` is called, the bookmark's title and description are concatenated and tokenized. Each resulting token is then used as a key in the index to store the bookmark's ID. To handle updates, the system first removes any existing entries for that bookmark ID via `_remove_bookmark_from_index` before adding the new tokens.

```python
def index_bookmark(self, bookmark: Bookmark) -> None:
    self._remove_bookmark_from_index(bookmark.id)
    tokens = self._tokenize(f"{bookmark.title} {bookmark.description}")
    for token in tokens:
        self._index[token].add(bookmark.id)
```

## Search and Intersection Logic

The `search` method implements a strict **AND** logic. For a bookmark to be considered a candidate, it must contain *all* tokens present in the search query. 

The implementation achieves this by retrieving the set of IDs for the first token and then performing a set intersection (`&=`) with the ID sets of all subsequent tokens. This approach is highly efficient as it leverages Python's optimized set operations to narrow down results quickly.

```python
candidate_ids: Set[str] = self._index.get(tokens[0], set()).copy()
for token in tokens[1:]:
    candidate_ids &= self._index.get(token, set())
```

If a query contains a token that does not exist in the index, the intersection immediately results in an empty set, and the search returns no results.

## Ranking and Scoring

Once the candidate bookmarks are identified, they are ranked by relevance using the `_rank_results` method. The scoring mechanism is based on keyword frequency: it counts how many times the query tokens appear in the combined title and description of the bookmark.

```python
@staticmethod
def _rank_results(bookmarks: List[Bookmark], tokens: List[str]) -> List[Bookmark]:
    """Rank results by number of token occurrences in title + description."""
    def score(b: Bookmark) -> int:
        text = f"{b.title} {b.description}".lower()
        return sum(text.count(t) for t in tokens)

    return sorted(bookmarks, key=score, reverse=True)
```

This simple frequency-based ranking ensures that bookmarks where the search terms appear multiple times (or where multiple different search terms appear) are surfaced at the top of the results.

## Design Trade-offs and Constraints

The implementation of `SearchIndex` reflects several specific design choices:

*   **In-Memory Volatility**: The index is entirely in-memory. While this provides extremely fast lookups, it must be rebuilt from the `BookmarkRepository` every time the application starts (via the `_rebuild` method).
*   **Strict AND Logic**: By requiring all tokens to match, the search is precise but may feel "unforgiving" to users compared to OR-based or fuzzy search engines. If a user searches for "Python Tutorial" and a bookmark only contains "Python", it will not be returned.
*   **Status Agnosticism**: The index itself does not track bookmark status (e.g., active vs. trashed). Because `BookmarkService.delete_bookmark` performs a soft-delete (moving the bookmark to the "trash" status without removing it from the repository), trashed bookmarks will still appear in search results as long as they remain in the underlying repository.
*   **Simple Scoring**: The `text.count(t)` method used in ranking is a "raw" string count. It does not account for word boundaries during the scoring phase, though the initial filtering of candidates via the tokenized index mitigates this for the most part.