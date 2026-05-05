---
title: Organizing Content
description: How to group bookmarks into manual or smart collections and apply categorical tags for better discoverability.
code_symbols: [SYM#e15c91e9e7eae95d052f50388a60d0bc30b3fe67, SYM#97d8a6cbf0c47108aa2beb39fafa695229654067, SYM#dd5fd545693e00d6c0b38892bbd18ed4afa136a1]
section_id: 06c4cbf2-6ac9-461c-97d3-5251456988ca_organizing_content
doc_type: tutorial
section_type: guide
---
This tutorial walks you through organizing your bookmarks using tags and collections. You will learn how to apply categorical labels, group bookmarks manually, and define smart collections based on keyword filters.

## Prerequisites

To follow this tutorial, you need to have the `BookmarkService` initialized. This service acts as the central facade for all organization logic.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()
```

## Categorizing with Tags

Tags are categorical labels with associated colors. They are useful for broad classification across your entire library.

### 1. Create a Tag
Use the `create_tag` method to define a new label. You can specify a color using the `TagColor` enum.

```python
from app.models.tag import TagColor

tag_data = {
    "name": "Research",
    "color": "blue", # Corresponds to TagColor.BLUE
    "description": "Academic and technical papers"
}

tag, error = service.create_tag(tag_data)
if tag:
    print(f"Created tag: {tag.name} with ID: {tag.id}")
```

The `TagColor` enum supports `RED`, `BLUE`, `GREEN`, `YELLOW`, `PURPLE`, and `GRAY`.

### 2. Apply a Tag to a Bookmark
Once a tag exists, you can add its ID to a bookmark's `tags` list.

```python
bookmark_id = "your-bookmark-id"
bookmark = service.get_bookmark(bookmark_id)

if bookmark:
    bookmark.add_tag(tag.id)
    # Persist the change via the repository (internal to service logic)
    service.update_bookmark(bookmark_id, {"tags": bookmark.tags})
```

When you delete a tag using `service.delete_tag(tag_id)`, the service automatically strips that tag ID from all bookmarks that were using it, ensuring data integrity.

## Grouping with Collections

Collections allow you to group bookmarks into specific folders or lists. They come in two types: **Manual** and **Smart**.

### 1. Create a Manual Collection
Manual collections require you to explicitly add or remove bookmarks.

```python
collection_data = {
    "name": "Project Alpha",
    "type": "manual"
}

collection, error = service.create_collection(collection_data)
```

### 2. Add Bookmarks to a Collection
Use the service layer to link a bookmark to your collection.

```python
success = service.add_to_collection(collection.id, bookmark_id)
if success:
    print("Bookmark added to Project Alpha")
```

Manual collections also support pinning to the top of the sidebar and reordering bookmarks:

```python
collection.pin()
collection.reorder([id2, id1, id3]) # Must contain the exact same IDs
```

### 3. Define a Smart Collection
Smart collections use a `filter_rule` to automatically identify relevant bookmarks based on keywords in their title or description.

```python
smart_data = {
    "name": "Python Resources",
    "type": "smart",
    "filter_rule": "python"
}

smart_collection, error = service.create_collection(smart_data)
```

> **Note**: In the current implementation, smart collections do not persist their results in the `bookmark_ids` list. The `_apply_filter` method in `app/models/collection.py` is used internally to evaluate which bookmarks match the rule at runtime.

## Complete Organization Example

The following script demonstrates creating a bookmark, tagging it, and placing it in a pinned collection.

```python
from app.services.bookmark_service import BookmarkService
from app.models.tag import TagColor

service = BookmarkService()

# 1. Create the bookmark
bookmark, _ = service.create_bookmark({
    "url": "https://docs.python.org",
    "title": "Python Documentation"
})

# 2. Create and apply a tag
tag, _ = service.create_tag({"name": "Programming", "color": "green"})
bookmark.add_tag(tag.id)
service.update_bookmark(bookmark.id, {"tags": bookmark.tags})

# 3. Create a collection and add the bookmark
collection, _ = service.create_collection({"name": "Learning", "type": "manual"})
service.add_to_collection(collection.id, bookmark.id)

# 4. Pin the collection for quick access
collection.pin()

print(f"Organized '{bookmark.title}' into '{collection.name}' with tag '{tag.name}'")
```

By using the `BookmarkService`, you ensure that all validations (like tag name length limits) and side effects (like cache invalidation) are handled correctly.