---
title: Organizing Bookmarks with Tags and Collections
description: How to categorize bookmarks using tags and group them into collections, including handling cross-entity updates.
code_symbols: [SYM#fdcfaed9972e477ae4585fd1b279ac107fa534cd]
section_id: 8a8b482c-a0cd-48ff-ab6a-1ca3259e02cd_organizing_bookmarks_with_tags_and_collections
doc_type: how_to
section_type: guide
---
To organize bookmarks in this project, you use the `BookmarkService` to create tags and group bookmarks into collections. The service ensures that when you delete a tag, it is automatically removed from all associated bookmarks, maintaining data integrity across the system.

## Categorizing with Tags

Tags are created independently and then associated with bookmarks by their ID.

### Creating and Applying Tags

To create a new tag, use the `create_tag` method. You can then include the tag's ID when creating a bookmark.

```python
from app.services.bookmark_service import BookmarkService
from app.models.tag import TagColor

service = BookmarkService()

# 1. Create a tag
tag_data = {
    "name": "Research",
    "color": TagColor.BLUE.value,
    "description": "Academic and technical papers"
}
tag, error = service.create_tag(tag_data)

# 2. Create a bookmark with the tag
bookmark_data = {
    "url": "https://arxiv.org",
    "title": "arXiv.org e-Print archive",
    "tags": [tag.id]
}
bookmark, error = service.create_bookmark(bookmark_data)
```

### Automatic Tag Cleanup

When a tag is deleted via the `BookmarkService`, the service automatically iterates through all bookmarks containing that tag, removes the reference, and invalidates the cache for each affected bookmark.

```python
# Deleting a tag automatically strips it from all bookmarks
# This is an O(N) operation where N is the number of bookmarks using the tag
success = service.delete_tag("tag_id_123")
```

## Grouping with Collections

Collections allow you to group bookmarks either manually or dynamically using "Smart" filters.

### Creating a Manual Collection

Manual collections require you to explicitly add or remove bookmarks using their IDs.

```python
# 1. Create the collection
collection_data = {
    "name": "Project Alpha",
    "type": "manual"
}
collection, error = service.create_collection(collection_data)

# 2. Add a bookmark to it
success = service.add_to_collection(collection.id, "bookmark_id_456")
```

### Creating a Smart Collection

Smart collections use a `filter_rule` to automatically include bookmarks based on their title or description. Note that you cannot manually add bookmarks to a smart collection.

```python
# Create a smart collection for all "Python" related bookmarks
smart_data = {
    "name": "Python Resources",
    "type": "smart",
    "filter_rule": "python"
}
smart_collection, error = service.create_collection(smart_data)

# Attempting to add manually will return False
# success = service.add_to_collection(smart_collection.id, "any_id") # Returns False
```

## Cross-Entity Updates

The `BookmarkService` acts as a facade to ensure that updates to one entity (like a Tag) are reflected in others (like Bookmarks).

| Operation | Cross-Entity Effect |
| :--- | :--- |
| `delete_tag(tag_id)` | Removes `tag_id` from the `tags` list of every `Bookmark` in the repository. |
| `update_tag(tag_id, data)` | Updates the tag metadata; bookmarks remain linked via the immutable `tag_id`. |
| `add_to_collection(col_id, b_id)` | Appends the `bookmark_id` to the collection's internal list if the collection is `MANUAL`. |

## Troubleshooting

### Tag Deletion Performance
Deleting a tag that is used by thousands of bookmarks can be slow. The `delete_tag` method in `BookmarkService` performs a save operation and a cache invalidation for every single bookmark that uses the tag:

```python
# From app/services/bookmark_service.py
for bookmark in self._repo.get_bookmarks_with_tag(tag_id):
    bookmark.remove_tag(tag_id)
    self._repo.save_bookmark(bookmark)
    self._cache.invalidate(bookmark.id)
```

### Adding to Smart Collections
If `add_to_collection` returns `False`, verify the collection type. The `Collection.add_bookmark` method explicitly prevents manual additions to smart collections:

```python
# From app/models/collection.py
def add_bookmark(self, bookmark_id: str) -> bool:
    if self.is_smart or bookmark_id in self.bookmark_ids:
        return False
    self.bookmark_ids.append(bookmark_id)
    return True
```