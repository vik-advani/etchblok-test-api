---
title: Getting Started with BookmarkService
description: Learn how to initialize the BookmarkService singleton and create your first bookmark with automatic validation and indexing.
code_symbols: [SYM#dd5fd545693e00d6c0b38892bbd18ed4afa136a1, SYM#fdcfaed9972e477ae4585fd1b279ac107fa534cd]
section_id: 9eba2822-dfe8-4c43-826d-8dc73f6d1298_getting_started_with_bookmarkservice
doc_type: tutorial
section_type: guide
---
You will learn how to use the `BookmarkService` to manage your bookmarks. This service acts as a central facade that coordinates validation, database persistence, search indexing, and caching.

By the end of this tutorial, you will be able to initialize the service, create a validated bookmark, and perform a full-text search.

### Prerequisites

To follow this tutorial, ensure your environment is set up with the core models and services:
- `BookmarkService` (app.services.bookmark_service)
- `Bookmark` (app.models.bookmark)
- A configured environment where `app.db.repository.BookmarkRepository` can initialize.

### Step 1: Initialize the BookmarkService Singleton

The `BookmarkService` is implemented as a singleton. This ensures that the in-memory search index and the LRU cache are shared across your entire application (e.g., between different Flask blueprints).

```python
from app.services.bookmark_service import BookmarkService

# Initialize the service
# Calling this multiple times always returns the same instance
service = BookmarkService()
```

When you initialize the service, it automatically bootstraps its internal dependencies: a `BookmarkRepository` for storage, an `LRUCache` (limited to 256 items), and a `SearchIndex`.

### Step 2: Create a Bookmark with Automatic Validation

To create a bookmark, you pass a dictionary of data to `create_bookmark`. The service uses a specific return pattern: it returns a tuple containing the created object and an error message. If the error message is not `None`, the operation failed.

```python
# Prepare bookmark data
new_bookmark_data = {
    "url": "https://github.com/vik-advani/etchblok",
    "title": "Etchblok Repository",
    "description": "A test API project for bookmark management."
}

# Attempt to create the bookmark
bookmark, error = service.create_bookmark(new_bookmark_data)

if error:
    print(f"Validation failed: {error}")
else:
    print(f"Successfully created bookmark with ID: {bookmark.id}")
```

The service performs several actions behind the scenes:
1. **Validation**: It runs `_validate_url` and `_validate_title`. If the URL is empty or the title is missing, it returns an error.
2. **Persistence**: It saves the bookmark to the repository.
3. **Indexing**: It adds the bookmark to the `SearchIndex` for full-text search.
4. **Cache Management**: It invalidates any existing cache entry for that ID to ensure data consistency.

### Step 3: Retrieve and Search Bookmarks

Once bookmarks are created, you can retrieve them by ID or search through them using the full-text index.

```python
# Retrieve a single bookmark (uses LRU cache)
bookmark_id = bookmark.id
retrieved = service.get_bookmark(bookmark_id)

# Perform a full-text search
# This searches across titles and descriptions
search_results = service.search("etchblok", limit=10)

for result in search_results:
    print(f"Found: {result.title} ({result.url})")
```

The `get_bookmark` method first checks the `LRUCache`. If the bookmark isn't cached, it fetches it from the repository and populates the cache for future requests.

### Step 4: Perform a Soft-Delete

In this project, deleting a bookmark does not immediately remove it from the database. Instead, it "trashes" the bookmark.

```python
# Soft-delete the bookmark
success = service.delete_bookmark(bookmark_id)

if success:
    print("Bookmark moved to trash.")
else:
    print("Bookmark not found.")
```

The `delete_bookmark` method updates the bookmark status to "trash", saves the change, and invalidates the cache entry.

### Complete Example

Here is the complete code for a script that initializes the service and performs the basic lifecycle of a bookmark.

```python
from app.services.bookmark_service import BookmarkService

def main():
    # 1. Access the singleton service
    service = BookmarkService()

    # 2. Create a bookmark
    data = {
        "url": "https://python.org",
        "title": "Python Programming Language"
    }
    bookmark, error = service.create_bookmark(data)
    
    if error:
        print(f"Error: {error}")
        return

    # 3. Search for the bookmark
    # The search index is updated immediately after creation
    results = service.search("Python")
    print(f"Search found {len(results)} results.")

    # 4. Cleanup (Soft-delete)
    service.delete_bookmark(bookmark.id)
    print("Lifecycle complete.")

if __name__ == "__main__":
    main()
```

### Next Steps
- Explore **Tag Operations** using `service.create_tag()` and `service.add_to_collection()`.
- Learn how `delete_tag()` automatically cleans up bookmark associations across the system.
- Check the `app/routes/bookmarks.py` file to see how this service is integrated into Flask endpoints.