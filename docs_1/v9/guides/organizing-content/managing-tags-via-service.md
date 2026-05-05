---
title: Managing Tags via Service
description: How to create, update, and delete tags using the BookmarkService, ensuring data consistency and automatic cleanup across bookmarks.
code_symbols: [SYM#dd5fd545693e00d6c0b38892bbd18ed4afa136a1, SYM#fdcfaed9972e477ae4585fd1b279ac107fa534cd, SYM#e15c91e9e7eae95d052f50388a60d0bc30b3fe67, SYM#a903c58b7b9829413e7dde33ff94fc7516b965f1]
section_id: dc54e235-c641-47f5-9ab2-2a03cebae3c0_managing_tags_via_service
doc_type: how_to
section_type: guide
---
To manage tags in this application, use the `BookmarkService` singleton. This service ensures that tag operations maintain data consistency, such as automatically removing a deleted tag from all associated bookmarks and invalidating relevant caches.

### Create a Tag

To create a new tag, pass a dictionary containing the tag details to `BookmarkService.create_tag()`. The service validates the name and returns a tuple of `(Tag, error)`.

```python
from app.services.bookmark_service import BookmarkService
from app.models.tag import TagColor

service = BookmarkService()

tag_data = {
    "name": "Research",
    "color": TagColor.BLUE.value,
    "description": "Academic and technical papers"
}

tag, error = service.create_tag(tag_data)

if error:
    print(f"Failed to create tag: {error}")
else:
    print(f"Created tag: {tag.id} ({tag.name})")
```

### Update a Tag

Use `BookmarkService.update_tag()` to modify an existing tag's name or color. The service handles validation and updates the underlying repository.

```python
from app.services.bookmark_service import BookmarkService
from app.models.tag import TagColor

service = BookmarkService()

# Update tag name and color
update_data = {
    "name": "Deep Learning",
    "color": TagColor.PURPLE.value
}

tag, error = service.update_tag("tag_id_123", update_data)

if tag:
    print(f"Updated tag to: {tag.name} with color {tag.color}")
elif error:
    print(f"Validation error: {error}")
else:
    print("Tag not found")
```

### Delete a Tag and Cleanup

When you delete a tag via `BookmarkService.delete_tag()`, the service performs a "strip" operation. It iterates through every bookmark associated with that tag, removes the tag reference, saves the bookmark, and invalidates the cache for each affected bookmark before finally deleting the tag itself.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()

# This will remove the tag from all bookmarks and then delete the tag
success = service.delete_tag("tag_id_123")

if success:
    print("Tag deleted and removed from all bookmarks.")
else:
    print("Tag not found.")
```

### Listing All Tags

You can retrieve all tags currently stored in the repository using `list_tags()`.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()
tags = service.list_tags()

for tag in tags:
    print(f"{tag.name} (Used by {tag.usage_count} bookmarks)")
```

### Validation Rules and Constraints

The `BookmarkService` enforces several rules defined in `app.models._validators` and the `Tag` model:

*   **Reserved Names**: You cannot create tags named `all`, `untagged`, `archived`, or `trash`.
*   **Length Limits**: Tag names must be between 1 and 50 characters.
*   **Unique Colors**: Use the `TagColor` enum values: `RED`, `BLUE`, `GREEN`, `YELLOW`, `PURPLE`, or `GRAY`.

### Troubleshooting

*   **Performance on Delete**: Because `delete_tag` iterates through all bookmarks containing that tag to update them, deleting a very popular tag (used by thousands of bookmarks) may be a slow operation as it triggers multiple repository saves and cache invalidations.
*   **Case Sensitivity**: Tag names are normalized (stripped and lowercased) during validation to prevent duplicates that differ only by case.
*   **Usage Counts**: While the `Tag` model includes a `usage_count` attribute and methods like `increment_usage()`, these are primarily managed by the repository layer when bookmarks are saved or loaded. Avoid manually modifying `usage_count` unless you are implementing custom repository logic.