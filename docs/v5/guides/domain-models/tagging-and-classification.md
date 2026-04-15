---
title: Tagging and Classification
description: Learn how to create tags, assign colors, and attach them to bookmarks to improve searchability.
code_symbols: [SYM#e15c91e9e7eae95d052f50388a60d0bc30b3fe67, SYM#a903c58b7b9829413e7dde33ff94fc7516b965f1, SYM#8d687603253ce34b667ab707a05b20e9357dd6af]
section_id: c93b8f22-33ff-4e90-b530-2999ad88666f_tagging_and_classification
doc_type: tutorial
section_type: guide
---
In this tutorial, you will learn how to use the tagging system to organize your bookmarks. You will create a new tag with a custom color, attach it to a bookmark, and manage the tag's lifecycle using the service layer.

### Prerequisites

To follow this tutorial, you should have the `BookmarkService` initialized. This service acts as the primary interface for managing tags and bookmarks.

```python
from app.services.bookmark_service import BookmarkService

# BookmarkService is a singleton
service = BookmarkService()
```

### Step 1: Create a Tag with a Custom Color

Tags are represented by the `Tag` class and can be assigned a visual color using the `TagColor` enum. To create a tag, use the `create_tag` method of the `BookmarkService`.

```python
from app.models.tag import TagColor

# Define tag data
tag_data = {
    "name": "Research",
    "color": TagColor.BLUE.value,
    "description": "Academic papers and technical articles"
}

# Create the tag via the service
tag, error = service.create_tag(tag_data)

if error:
    print(f"Failed to create tag: {error}")
else:
    print(f"Created tag: {tag.name} (ID: {tag.id}) with color {tag.color.value}")
```

The `create_tag` method validates the name and returns a `Tag` instance. If you provide an invalid name (like a reserved word or a name longer than 50 characters), it returns an error message instead.

### Step 2: Attach the Tag to a Bookmark

Once you have a tag, you can attach it to a bookmark by adding the tag's unique ID to the bookmark's `tags` list.

```python
# 1. Create a bookmark first
bookmark_data = {
    "url": "https://arxiv.org",
    "title": "arXiv.org e-Print archive"
}
bookmark, _ = service.create_bookmark(bookmark_data)

# 2. Attach the tag ID to the bookmark
# The Bookmark model provides an add_tag method
if bookmark.add_tag(tag.id):
    # 3. Persist the change via the service
    service.update_bookmark(bookmark.id, {"tags": bookmark.tags})
    print(f"Tagged '{bookmark.title}' with '{tag.name}'")
```

The `Bookmark.add_tag` method ensures that duplicate tags are not added. After modifying the bookmark object, you must call `service.update_bookmark` to persist the changes to the repository and update the search index.

### Step 3: Update Tag Metadata

You can change a tag's name or color at any time. The `BookmarkService.update_tag` method handles validation and persistence.

```python
# Update the tag to a different color and name
update_data = {
    "name": "Science",
    "color": TagColor.PURPLE.value
}

updated_tag, error = service.update_tag(tag.id, update_data)

if updated_tag:
    print(f"Tag updated to: {updated_tag.name} ({updated_tag.color.value})")
```

When you rename a tag, the `Tag.rename` method is called internally, which performs basic sanitization like stripping whitespace and checking length limits.

### Step 4: Delete a Tag and Clean Up

Deleting a tag in this system is a "cascading" operation. When you delete a tag through the service, it is automatically removed from every bookmark that was using it.

```python
# Delete the tag
success = service.delete_tag(tag.id)

if success:
    print("Tag deleted and removed from all bookmarks.")
    
    # Verify the bookmark no longer has the tag
    refreshed_bookmark = service.get_bookmark(bookmark.id)
    print(f"Bookmark tags: {refreshed_bookmark.tags}") # Should be an empty list []
```

The `service.delete_tag` method iterates through all bookmarks associated with the tag ID, calls `bookmark.remove_tag(tag_id)`, and invalidates the cache for each affected bookmark to ensure data consistency.

### Validation Rules and Reserved Names

When creating or renaming tags, keep the following constraints in mind:

1.  **Length**: Tag names must be between 1 and 50 characters.
2.  **Reserved Names**: You cannot use the following names as they are reserved for system filters:
    *   `all`
    *   `untagged`
    *   `archived`
    *   `trash`
3.  **Uniqueness**: While the `Tag` model itself doesn't enforce global uniqueness, the `BookmarkService` and underlying repository typically ensure tag names are unique per user context.

If you attempt to use a reserved name, the service will return an error:

```python
_, error = service.create_tag({"name": "trash"})
print(error) # Output: 'trash' is a reserved tag name
```