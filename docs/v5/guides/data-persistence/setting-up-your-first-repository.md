---
title: Setting Up Your First Repository
description: A beginner-friendly tutorial on initializing the BookmarkRepository and performing basic data operations to get started.
code_symbols: [SYM#adb8232356346a5957ff3a1a1b7ff70581f37649]
section_id: 6b2b5d41-1aad-4578-ba6d-729a3ea3c3c5_setting_up_your_first_repository
doc_type: tutorial
section_type: guide
---
The `BookmarkRepository` is the central data hub for the application. It provides an in-memory storage layer for bookmarks, tags, and collections, serving as a clean abstraction for data persistence during the application's runtime.

In this tutorial, you will learn how to initialize the repository and perform basic CRUD (Create, Read, Update, Delete) operations to manage your data.

### Prerequisites

To follow this tutorial, you need to have the core models and the repository class available in your environment:

```python
from app.db.repository import BookmarkRepository
from app.models.bookmark import Bookmark, BookmarkStatus
from app.models.tag import Tag, TagColor
from app.models.collection import Collection, CollectionType
```

### Step 1: Initializing the Repository

The `BookmarkRepository` is designed to be instantiated once and used throughout your application or service. Since it is in-memory, it starts empty every time you create a new instance.

```python
# Initialize the repository
repo = BookmarkRepository()

# Verify it's empty
counts = repo._count_all()
print(f"Initial state: {counts}") 
# Output: {'bookmarks': 0, 'tags': 0, 'collections': 0}
```

The `_count_all()` method is a helpful internal tool for diagnostics to verify the number of entities currently stored.

### Step 2: Creating and Saving Bookmarks

To store a bookmark, you first create a `Bookmark` instance and then pass it to the repository's `save_bookmark` method.

```python
# Create a new bookmark
new_bookmark = Bookmark(
    url="https://github.com/etchblok",
    title="Etchblok GitHub",
    description="The official repository for Etchblok."
)

# Persist it to the repository
repo.save_bookmark(new_bookmark)

# Retrieve it by ID
retrieved = repo.get_bookmark(new_bookmark.id)
print(f"Retrieved: {retrieved.title} ({retrieved.url})")
# Output: Retrieved: Etchblok GitHub (https://github.com/etchblok)
```

The `save_bookmark` method handles both inserts and updates. If you modify the `retrieved` object and call `save_bookmark` again with the same ID, it will overwrite the existing entry.

### Step 3: Organizing with Tags

Tags allow you to categorize bookmarks. You must save the `Tag` to the repository before or after associating its ID with a bookmark.

```python
# 1. Create and save a tag
dev_tag = Tag(name="Development", color=TagColor.BLUE)
repo.save_tag(dev_tag)

# 2. Associate the tag with our bookmark
bookmark = repo.get_bookmark(new_bookmark.id)
if bookmark:
    bookmark.add_tag(dev_tag.id)
    repo.save_bookmark(bookmark)

# 3. Query bookmarks by tag
dev_bookmarks = repo.get_bookmarks_with_tag(dev_tag.id)
print(f"Bookmarks with '{dev_tag.name}': {len(dev_bookmarks)}")
# Output: Bookmarks with 'Development': 1
```

Note that `BookmarkRepository` does not automatically update the `usage_count` on the `Tag` model; that logic is typically handled by the higher-level `BookmarkService`.

### Step 4: Grouping with Collections

Collections provide a way to group bookmarks. They can be **Manual** (you add IDs yourself) or **Smart** (logic-based).

```python
# Create a manual collection
reading_list = Collection(name="Reading List", collection_type=CollectionType.MANUAL)
reading_list.add_bookmark(new_bookmark.id)
repo.save_collection(reading_list)

# Verify the collection
my_collection = repo.get_collection(reading_list.id)
print(f"Collection '{my_collection.name}' has {my_collection.size} items.")
# Output: Collection 'Reading List' has 1 items.
```

### Step 5: Listing and Pagination

The repository provides a powerful `list_bookmarks` method that supports 1-based pagination and status filtering.

```python
# Add a second bookmark and archive it
archived_bm = Bookmark(url="https://example.com", title="Old Reference")
archived_bm.archive()
repo.save_bookmark(archived_bm)

# List only active bookmarks
active_items, total = repo.list_bookmarks(page=1, per_page=10, status="active")
print(f"Active bookmarks: {len(active_items)} (Total matching: {total})")

# List all bookmarks (ignoring status)
all_items, total = repo.list_bookmarks(page=1, per_page=25)
print(f"Total bookmarks in repo: {total}")
```

### Summary and Next Steps

You have successfully:
1.  Initialized a `BookmarkRepository`.
2.  Created and retrieved `Bookmark` entities.
3.  Categorized data using `Tag` and `Collection`.
4.  Used pagination and filtering to query your data.

While the `BookmarkRepository` handles the "how" of storage, the `BookmarkService` (found in `app.services.bookmark_service`) handles the "why"—including validation, search indexing, and complex business rules. Your next step is to explore how the Service layer wraps this Repository to provide a robust API.