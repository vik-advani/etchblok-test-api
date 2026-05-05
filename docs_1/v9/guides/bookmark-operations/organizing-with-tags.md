---
title: Organizing with Tags
description: A guide to creating tags and the automated cleanup process that strips tags from bookmarks upon deletion.
code_symbols: [SYM#dd5fd545693e00d6c0b38892bbd18ed4afa136a1, SYM#fdcfaed9972e477ae4585fd1b279ac107fa534cd]
section_id: 791b8fe5-8a46-4b10-92e3-eb0cf3e49e40_organizing_with_tags
doc_type: how_to
section_type: guide
---
To organize bookmarks in this system, you use the `BookmarkService` to manage `Tag` entities. The service handles validation, persistence, and the automated cleanup process that ensures no orphaned tag references remain on bookmarks when a tag is deleted.

## Creating and Deleting Tags

You manage the lifecycle of tags through the `BookmarkService` singleton. When a tag is deleted, the service automatically iterates through all bookmarks containing that tag to remove the reference.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()

# 1. Create a new tag
tag_data = {
    "name": "Research",
    "color": "blue",
    "description": "Academic and technical papers"
}
tag, error = service.create_tag(tag_data)

if error:
    print(f"Failed to create tag: {error}")
else:
    print(f"Created tag with ID: {tag.id}")

# 2. Delete a tag (triggers automated cleanup)
# This will strip the tag from all bookmarks before deleting the tag itself
success = service.delete_tag(tag.id)
if success:
    print("Tag deleted and removed from all bookmarks.")
```

### Automated Cleanup Process
The `delete_tag` method in `BookmarkService` performs a cascading cleanup to maintain data integrity. It follows these steps:
1. Retrieves the tag from the repository.
2. Queries the repository for all bookmarks associated with that `tag_id` using `_repo.get_bookmarks_with_tag(tag_id)`.
3. Calls `bookmark.remove_tag(tag_id)` for each found bookmark.
4. Saves the updated bookmarks and invalidates their entries in the `LRUCache`.
5. Finally, deletes the tag entity from the repository.

## Managing Tag Metadata

You can update a tag's name or visual color using the `update_tag` method.

```python
# Update an existing tag
update_data = {
    "name": "Deep Learning",
    "color": "purple"
}
updated_tag, error = service.update_tag(tag_id="tag123", data=update_data)

if not updated_tag and not error:
    print("Tag not found")
elif error:
    print(f"Validation error: {error}")
```

### Available Colors
The `TagColor` enum (found in `app.models.tag`) supports the following values:
*   `red`
*   `blue`
*   `green`
*   `yellow`
*   `purple`
*   `gray` (Default)

## Assigning Tags to Bookmarks

Tags are assigned to bookmarks during creation by providing a list of tag IDs in the `tags` field of the data dictionary.

```python
bookmark_data = {
    "url": "https://arxiv.org",
    "title": "ArXiv",
    "tags": ["research_id", "science_id"]
}
bookmark, error = service.create_bookmark(bookmark_data)
```

## Troubleshooting and Constraints

### Reserved Names
The system prevents the creation of tags with names that conflict with internal filters. The `_validate_tag_name` helper (in `app.models._validators`) will return an error if you attempt to use:
*   `all`
*   `untagged`
*   `archived`
*   `trash`

### Validation Limits
*   **Name Length**: Tag names are limited to **50 characters**. Attempting to rename a tag to a longer string via `Tag.rename` or `BookmarkService.update_tag` will result in a `ValueError` or a validation error message.
*   **Empty Names**: Tag names cannot be empty or consist only of whitespace.

### Performance Considerations
Because `delete_tag` performs a linear scan and update of all bookmarks associated with the tag, deleting a very popular tag may be performance-intensive. The service invalidates the cache for every affected bookmark to ensure consistency across the application.