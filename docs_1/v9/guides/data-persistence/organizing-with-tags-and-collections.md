---
title: Organizing with Tags and Collections
description: Instructions for persisting metadata entities like tags and collections to maintain organizational structure.
code_symbols: [SYM#adb8232356346a5957ff3a1a1b7ff70581f37649]
section_id: c19c39c0-5cf2-41ff-b938-549b3629bb56_organizing_with_tags_and_collections
doc_type: how_to
section_type: guide
---
To organize bookmarks in this system, you use **Tags** for labeling and **Collections** for grouping. These entities are managed through the `BookmarkService`, which coordinates updates between the `BookmarkRepository` and the application's search and cache layers.

## Managing Tags

Tags are labels that can be attached to multiple bookmarks. You create and manage them using the `BookmarkService`, which ensures that tag names are validated and that deletions are propagated correctly.

### Creating and Assigning Tags

To create a tag, provide a dictionary containing at least a `name`. You can also specify a `color` using the `TagColor` enum.

```python
from app.services.bookmark_service import BookmarkService
from app.models.tag import TagColor

service = BookmarkService()

# Create a new tag
tag_data = {
    "name": "Research",
    "color": "blue",
    "description": "Academic papers and articles"
}
tag, error = service.create_tag(tag_data)

if tag:
    print(f"Created tag: {tag.id}")
```

To assign a tag to a bookmark, you update the bookmark's `tags` list. Note that the `Bookmark` model stores tag IDs, not the tag objects themselves.

```python
# Assigning a tag to a bookmark via update
bookmark_id = "abc123"
service.update_bookmark(bookmark_id, {"tags": [tag.id]})
```

### Updating and Deleting Tags

When you update a tag's name or color, the changes are persisted in the `BookmarkRepository`. Deleting a tag is a "heavy" operation because the service must strip the tag ID from every bookmark that uses it.

```python
# Update a tag
service.update_tag(tag.id, {"color": "green", "name": "Deep Research"})

# Delete a tag (automatically removes it from all bookmarks)
success = service.delete_tag(tag.id)
```

The `BookmarkService.delete_tag` method performs the following cleanup:
1. Retrieves all bookmarks containing the tag using `BookmarkRepository.get_bookmarks_with_tag`.
2. Calls `bookmark.remove_tag(tag_id)` on each.
3. Saves the updated bookmarks and invalidates their cache entries.
4. Finally, removes the tag from the repository.

## Grouping with Collections

Collections allow you to group bookmarks either manually or automatically via "Smart" rules.

### Manual Collections

Manual collections require you to explicitly add or remove bookmarks.

```python
# Create a manual collection
collection_data = {
    "name": "Project Alpha",
    "type": "manual"
}
collection, error = service.create_collection(collection_data)

# Add a bookmark to the collection
service.add_to_collection(collection.id, "bookmark_id_123")

# Remove a bookmark
service.remove_from_collection(collection.id, "bookmark_id_123")
```

### Smart Collections

Smart collections use a `filter_rule` to automatically include bookmarks based on their title or description. You cannot manually add bookmarks to a smart collection.

```python
# Create a smart collection for Python-related bookmarks
smart_data = {
    "name": "Python Stuff",
    "type": "smart",
    "filter_rule": "python"
}
smart_collection, error = service.create_collection(smart_data)

# Check if it's smart
if smart_collection.is_smart:
    print(f"Filter rule: {smart_collection.filter_rule}")
```

The `Collection._apply_filter` method is used internally to evaluate which bookmarks match the rule:
```python
# Internal logic in app/models/collection.py
def _apply_filter(self, bookmarks: list) -> List[str]:
    if not self.filter_rule:
        return []
    keyword = self.filter_rule.lower()
    return [b.id for b in bookmarks if keyword in b.title.lower() or keyword in b.description.lower()]
```

## Persistence and Constraints

The `BookmarkRepository` acts as the storage layer for these entities. Because it is an in-memory implementation, all tags and collections are lost when the application restarts.

### Validation Rules
The system enforces several constraints defined in `app/models/_validators.py` and the model classes:
- **Tag Names**: Must be between 1 and 50 characters.
- **Collection Names**: Required and cannot be empty.
- **Tag Colors**: Must be one of the values in `TagColor` (`red`, `blue`, `green`, `yellow`, `purple`, `gray`).

### Troubleshooting
- **Data Loss**: Remember that `BookmarkRepository` uses Python dictionaries (`self._tags`, `self._collections`). Data does not persist across sessions.
- **Smart Collection Errors**: If you attempt to call `add_to_collection` on a collection where `collection_type` is `CollectionType.SMART`, the operation will return `False` and fail.
- **Performance**: Deleting a tag that is used by thousands of bookmarks may be slow, as `BookmarkService.delete_tag` iterates through all matching bookmarks to update them individually.