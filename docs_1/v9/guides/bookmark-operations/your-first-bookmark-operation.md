---
title: Your First Bookmark Operation
description: A step-by-step tutorial on initializing the BookmarkService and creating your first searchable bookmark.
code_symbols: [SYM#dd5fd545693e00d6c0b38892bbd18ed4afa136a1, SYM#fdcfaed9972e477ae4585fd1b279ac107fa534cd]
section_id: facc45df-a927-45fb-a8c5-79607eee19d9_your_first_bookmark_operation
doc_type: tutorial
section_type: guide
---
In this tutorial, you will learn how to use the central business logic layer of the application to manage your bookmarks. You will initialize the primary service, create a new bookmark with validation, and perform a full-text search to retrieve it.

### Prerequisites

To follow this tutorial, ensure you have the application environment configured. The `BookmarkService` automatically initializes its own dependencies, including the `BookmarkRepository` and `SearchIndex`, so no manual database setup is required for this walkthrough.

### Step 1: Initialize the Bookmark Service

The `BookmarkService` is designed as a singleton. This ensures that the search index, cache, and repository state are shared across your entire application, whether you are calling it from a Flask route or a background task.

```python
from app.services.bookmark_service import BookmarkService

# Initialize the service
service = BookmarkService()

# Because it is a singleton, subsequent calls return the same instance
another_reference = BookmarkService()
assert service is another_reference
```

When you call `BookmarkService()`, the class runs `_init_services()` internally to bootstrap the `BookmarkRepository`, an `LRUCache` for performance, and the `SearchIndex` for querying.

### Step 2: Create a Searchable Bookmark

To create a bookmark, you pass a dictionary of data to the `create_bookmark` method. This method performs validation on the URL and title before persisting the data.

The method returns a tuple: `(bookmark, error)`. If validation fails, `bookmark` will be `None` and `error` will contain a descriptive message.

```python
bookmark_data = {
    "url": "https://www.python.org",
    "title": "Python Programming Language",
    "description": "The official home of the Python Programming Language"
}

bookmark, error = service.create_bookmark(bookmark_data)

if error:
    print(f"Failed to create bookmark: {error}")
else:
    print(f"Created bookmark with ID: {bookmark.id}")
```

The service uses internal validators from `app.models._validators` to ensure:
- The **URL** follows a valid `http://` or `https://` format.
- The **Title** is present and does not exceed 256 characters.

Once validated, the service saves the bookmark to the repository, adds it to the search index, and invalidates any existing cache entries for that ID.

### Step 3: Search for Your Bookmark

Now that the bookmark is indexed, you can retrieve it using the `search` method. This performs a full-text search across the titles and descriptions of all active bookmarks.

```python
# Search for the bookmark we just created
results = service.search("Python Programming")

for result in results:
    print(f"Found: {result.title} ({result.url})")
```

The `search` method returns a list of `Bookmark` objects. By default, it limits results to 20 items, but you can adjust this using the `limit` parameter.

### Complete Example

Here is the complete script combining these steps:

```python
from app.services.bookmark_service import BookmarkService

def main():
    # 1. Initialize
    service = BookmarkService()

    # 2. Create
    data = {
        "url": "https://github.com/features/actions",
        "title": "GitHub Actions",
        "description": "Automate your workflow from GitHub"
    }
    
    bookmark, error = service.create_bookmark(data)
    
    if error:
        print(f"Error: {error}")
        return

    print(f"Successfully created: {bookmark.title}")

    # 3. Search
    query = "workflow"
    results = service.search(query)
    
    print(f"\nSearch results for '{query}':")
    for b in results:
        print(f"- {b.title}: {b.url}")

if __name__ == "__main__":
    main()
```

### Next Steps
Now that you can create and search bookmarks, consider exploring:
- **Tagging**: Use `service.create_tag()` and `bookmark.add_tag()` to organize your data.
- **Collections**: Group bookmarks together using `service.create_collection()` and `service.add_to_collection()`.
- **Status Management**: Use `service.archive_bookmark()` or `service.delete_bookmark()` to manage the lifecycle of your saved links.