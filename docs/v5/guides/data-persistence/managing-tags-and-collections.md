---
title: Managing Tags and Collections
description: How to persist and manage organizational entities like tags and collections within the repository system.
code_symbols: [SYM#adb8232356346a5957ff3a1a1b7ff70581f37649]
section_id: 1d147722-b30a-4612-92e8-87791e7fbb7d_managing_tags_and_collections
doc_type: how_to
section_type: guide
---
To manage organizational entities like tags and collections, use the `BookmarkService` which orchestrates operations between the `BookmarkRepository` and the underlying models.

### Managing Tags

Tags are created and updated through the `BookmarkService`, which ensures that tag names are validated and that deletions are propagated to all associated bookmarks.

```python
from app.services.bookmark_service import BookmarkService
from app.models.tag import TagColor

service = BookmarkService()

# Create a new tag
tag_data = {
    "name": "Research",
    "color": "blue",
    "description": "Academic and professional research"
}
tag, error = service.create_tag(tag_data)

# Update an existing tag
update_data = {
    "name": "Deep Research",
    "color": "purple"
}
updated_tag, error = service.update_tag(tag.id, update_data)

# Delete a tag (strips it from all bookmarks automatically)
success = service.delete_tag(tag.id)
```

When `delete_tag` is called, the service performs the following steps to maintain referential integrity:
1. Retrieves the tag from `BookmarkRepository`.
2. Finds all bookmarks containing that tag using `repo.get_bookmarks_with_tag(tag_id)`.
3. Calls `bookmark.remove_tag(tag_id)` on each bookmark.
4. Saves the updated bookmarks back to the repository and invalidates their cache entries.
5. Finally, removes the tag from the repository.

### Managing Collections

Collections group bookmarks either manually or automatically via "smart" filters. You manage these by interacting with the `Collection` model and the service layer.

#### Manual Collections
Manual collections require you to explicitly add or remove bookmark IDs.

```python
# Create a manual collection
collection_data = {
    "name": "Reading List",
    "type": "manual"
}
collection, error = service.create_collection(collection_data)

# Add a bookmark to the collection
# This updates the collection's internal bookmark_ids list and persists it
success = service.add_to_collection(collection.id, "bookmark_123")

# Remove a bookmark
success = service.remove_from_collection(collection.id, "bookmark_123")
```

#### Smart Collections
Smart collections use a `filter_rule` to automatically include bookmarks based on their content.

```python
# Create a smart collection for Python-related bookmarks
smart_data = {
    "name": "Python Stuff",
    "type": "smart",
    "filter_rule": "python"
}
smart_collection, error = service.create_collection(smart_data)
```

The `Collection` model implements `_apply_filter(bookmarks)`, which performs a case-insensitive substring match against the `title` and `description` of bookmarks to determine membership.

### Persistence Layer
The `BookmarkRepository` (found in `app/db/repository.py`) serves as the in-memory store for these entities. You can access the raw storage if needed, though using the service is recommended for consistency.

```python
from app.db.repository import BookmarkRepository

repo = BookmarkRepository()

# Direct repository access
all_tags = repo.list_tags()
all_collections = repo.list_collections()
specific_tag = repo.get_tag("tag_id")
```

### Troubleshooting and Performance
*   **In-Memory Storage**: The `BookmarkRepository` uses Python dictionaries (`self._tags`, `self._collections`). Data is not persisted to disk and will be lost when the application restarts.
*   **Tag Deletion Overhead**: Deleting a tag triggers a linear scan of all bookmarks in the repository to remove the tag reference. In large datasets, this operation may become a performance bottleneck.
*   **Smart Collection Logic**: The `filter_rule` in `Collection` is limited to simple keyword matching in the title and description. It does not support complex boolean logic or regex.
*   **Pagination**: While `list_bookmarks` in the repository supports 1-based pagination, `list_tags` and `list_collections` return the entire list of entities at once.