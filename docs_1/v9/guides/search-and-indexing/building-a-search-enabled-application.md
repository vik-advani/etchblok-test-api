---
title: Building a Search-Enabled Application
description: A step-by-step tutorial on initializing the SearchIndex service and integrating it into a bookmarking workflow.
code_symbols: [SYM#0f269a750bc62c4d874086090a88d14329456024, SYM#b0cc1ddc5c9e6b6675ff73174df52949062c8da5]
section_id: 243110ad-7c6e-4810-83a5-e8c62ecd1d20_building_a_search-enabled_application
doc_type: tutorial
section_type: guide
---
This tutorial guides you through building a search-enabled bookmarking application using the `SearchIndex` service. You will learn how to initialize the index, populate it with data, and perform ranked searches.

## Prerequisites

To follow this tutorial, you need the following components from the codebase:
- `Bookmark`: The core data model (`app.models.bookmark`).
- `BookmarkRepository`: The storage layer (`app.db.repository`).
- `SearchIndex`: The in-memory search engine (`app.services.search_service`).

## Step 1: Initializing the Search Index

The `SearchIndex` requires a `BookmarkRepository` instance during initialization. When you create a `SearchIndex`, it automatically performs a full rebuild by scanning all existing bookmarks in the repository.

```python
from app.db.repository import BookmarkRepository
from app.services.search_service import SearchIndex

# 1. Initialize the repository (storage)
repository = BookmarkRepository()

# 2. Initialize the search index (linked to the repository)
search_index = SearchIndex(repository)
```

The `SearchIndex` constructor calls `_rebuild()`, which uses `repository.list_bookmarks` to populate its internal inverted index.

## Step 2: Indexing Bookmarks

To make a bookmark searchable, you must add it to the index. The `SearchIndex` tokenizes the `title` and `description` fields, removing common stop words and converting text to lowercase.

```python
from app.models.bookmark import Bookmark

# Create a new bookmark
bookmark = Bookmark(
    url="https://python.org",
    title="Python Programming Language",
    description="The official website for the Python programming language."
)

# Save to repository first
repository.save_bookmark(bookmark)

# Index the bookmark for search
search_index.index_bookmark(bookmark)
```

When `index_bookmark` is called, the service:
1. Removes any existing entries for that bookmark ID (to handle updates).
2. Tokenizes the combined title and description.
3. Maps each token to the bookmark's ID in an internal dictionary.

## Step 3: Performing a Search

You can search the index using free-text queries. The search logic uses an **AND** approach: every token in your query must be present in the bookmark for it to appear in the results.

```python
# Search for bookmarks matching "python website"
results = search_index.search("python website", limit=10)

for bm in results:
    print(f"Found: {bm.title} ({bm.url})")
```

The `search` method performs the following:
- **Tokenization**: Splits your query into tokens (e.g., "python", "website").
- **Intersection**: Finds bookmark IDs that contain *all* query tokens.
- **Ranking**: Orders results by relevance using `_rank_results`, which counts how many times the query tokens appear in the bookmark's title and description.

## Step 4: Integrating into a Workflow

In a real application, you should automate indexing so that every creation or update is reflected in the search results. The `BookmarkService` class demonstrates this integration pattern.

```python
from app.services.bookmark_service import BookmarkService

# BookmarkService is a singleton that manages both Repo and SearchIndex
service = BookmarkService()

# Creating a bookmark via the service automatically indexes it
data = {
    "url": "https://github.com",
    "title": "GitHub: Where the world builds software",
    "description": "Development platform inspired by the way you work."
}
bookmark, error = service.create_bookmark(data)

# Searching via the service facade
search_results = service.search("development platform")
```

In `app/services/bookmark_service.py`, the `create_bookmark` and `update_bookmark` methods ensure that `self._search.index_bookmark(bookmark)` is called immediately after the repository save operation.

## Complete Result

By combining these components, you have a functional, in-memory search system. Here is how the final integration looks:

```python
from app.db.repository import BookmarkRepository
from app.services.search_service import SearchIndex
from app.models.bookmark import Bookmark

# Setup
repo = BookmarkRepository()
search = SearchIndex(repo)

# Populate
b1 = Bookmark(url="https://docs.pytest.org", title="pytest: unit testing", description="Simple powerful testing with Python")
repo.save_bookmark(b1)
search.index_bookmark(b1)

# Search
matches = search.search("python testing")
# matches[0] will be b1 because 'python' and 'testing' are in the description/title
```

### Next Steps
- Explore `app/routes/bookmarks.py` to see how the `/api/bookmarks/search` endpoint exposes this functionality over HTTP.
- Review `app/services/search_service.py` to see the list of `_STOP_WORDS` that are ignored during indexing.