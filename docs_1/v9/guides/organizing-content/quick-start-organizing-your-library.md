---
title: 'Quick Start: Organizing Your Library'
description: A beginner-friendly tutorial on setting up your first tags and collections to organize a set of bookmarks from scratch.
code_symbols: [SYM#e15c91e9e7eae95d052f50388a60d0bc30b3fe67, SYM#97d8a6cbf0c47108aa2beb39fafa695229654067, SYM#dd5fd545693e00d6c0b38892bbd18ed4afa136a1, SYM#fdcfaed9972e477ae4585fd1b279ac107fa534cd]
section_id: 31779a0a-3a16-4c7b-b584-f0e328f9d5b2_quick_start-_organizing_your_library
doc_type: tutorial
section_type: guide
---
In this tutorial, you will learn how to use the `BookmarkService` to build a structured library from scratch. You will create tags for categorization, save bookmarks with those tags, and group them into collections for better organization.

### Prerequisites

Before starting, ensure your environment is set up to import the application modules. The `BookmarkService` is a singleton that manages its own internal repository and search index, so you can initialize it directly in your scripts or routes.

### Step 1: Initialize the Bookmark Service

The `BookmarkService` acts as a facade for all operations in the system. Because it implements the singleton pattern, calling the constructor always returns the same instance, ensuring your data remains consistent across different parts of your application.

```python
from app.services.bookmark_service import BookmarkService

# Initialize the service singleton
service = BookmarkService()
```

### Step 2: Create Your First Tags

Tags are the primary way to categorize content. Each tag has a name and an optional color from the `TagColor` enum (e.g., "red", "blue", "green").

```python
# Define tag data
tag_data = {
    "name": "Research",
    "color": "blue",
    "description": "Academic and technical papers"
}

# Create the tag
tag, error = service.create_tag(tag_data)

if error:
    print(f"Failed to create tag: {error}")
else:
    print(f"Created tag: {tag.name} (ID: {tag.id})")
```

The `create_tag` method returns a tuple containing the created `Tag` object and an error message. Always check for errors, as tag names must pass validation (e.g., they cannot be empty).

### Step 3: Save a Bookmark with Tags

When creating a bookmark, you can immediately associate it with tags by providing a list of tag IDs. The service validates the URL and title before persisting the bookmark to the repository and indexing it for search.

```python
# Define bookmark data with the tag ID from the previous step
bookmark_data = {
    "url": "https://arxiv.org/",
    "title": "arXiv.org e-Print archive",
    "description": "Open access to 2 million scholarly articles",
    "tags": [tag.id]
}

# Create the bookmark
bookmark, error = service.create_bookmark(bookmark_data)

if error:
    print(f"Failed to create bookmark: {error}")
else:
    print(f"Saved bookmark: {bookmark.title} (ID: {bookmark.id})")
```

### Step 4: Organize Bookmarks into Collections

While tags are great for broad categorization, **Collections** allow you to create curated lists. By default, collections are "manual," meaning you explicitly add or remove bookmarks.

```python
# 1. Create a new collection
collection_data = {"name": "Thesis Project"}
collection, error = service.create_collection(collection_data)

if collection:
    # 2. Add the bookmark to the collection
    success = service.add_to_collection(collection.id, bookmark.id)
    
    if success:
        print(f"Added '{bookmark.title}' to collection '{collection.name}'")
    else:
        print("Failed to add bookmark. It might already be in the collection.")
```

Note that `add_to_collection` will return `False` if the collection is a "Smart Collection" (which populates automatically based on rules) or if the bookmark is already present.

### Step 5: Verify Your Library via Search

Now that you have organized your content, you can use the built-in search functionality to find it. The `BookmarkService` uses an internal `SearchIndex` to perform full-text searches across titles and descriptions.

```python
# Search for bookmarks related to 'scholarly'
results = service.search("scholarly")

for result in results:
    print(f"Found: {result.title} | Tags: {result.tags}")
```

### Summary

You have successfully:
1.  Initialized the **BookmarkService** singleton.
2.  Created a **Tag** with a specific color.
3.  Saved a **Bookmark** and linked it to your tag.
4.  Grouped your bookmark into a **Collection**.
5.  Verified the entry using the **Search** index.

### Next Steps
*   **Smart Collections**: Try creating a collection with `type="smart"` and a `filter_rule` (e.g., "Python") to see bookmarks automatically group themselves.
*   **Archiving**: Use `service.archive_bookmark(bookmark_id)` to hide completed research from your main list without deleting it.
*   **Tag Management**: Use `service.delete_tag(tag_id)` and notice how the service automatically strips that tag from all associated bookmarks.