---
title: Tagging and Metadata
description: Understanding how tags are used to categorize content and how usage counts are tracked across the system.
code_symbols: [SYM#e15c91e9e7eae95d052f50388a60d0bc30b3fe67, SYM#a903c58b7b9829413e7dde33ff94fc7516b965f1, SYM#8d687603253ce34b667ab707a05b20e9357dd6af]
section_id: 3d6cb1f0-fe19-4928-adad-2e75ed0fb59a_tagging_and_metadata
doc_type: guide
section_type: guide
---
Tagging in this system provides a flexible way to categorize bookmarks using user-defined labels. The implementation centers around the `Tag` model and is managed through the `BookmarkService`, which ensures data integrity and handles the relationship between tags and the bookmarks that use them.

## The Tag Model

The core of the tagging system is the `Tag` class located in `app/models/tag.py`. Each tag is a lightweight object designed for both organization and UI rendering.

### Attributes and Visuals
A `Tag` consists of:
- **id**: A unique 8-character hex string generated via `uuid4`.
- **name**: The display name (e.g., "Research", "Recipes").
- **color**: A `TagColor` enum value used for UI categorization.
- **usage_count**: An integer tracking how many bookmarks are currently associated with this tag.

The `TagColor` enum provides a set of preset visual styles:
```python
class TagColor(Enum):
    """Preset colours available for tags."""
    RED = "red"
    BLUE = "blue"
    GREEN = "green"
    YELLOW = "yellow"
    PURPLE = "purple"
    GRAY = "gray"
```

## Validation and Constraints

The system enforces strict rules on tag names to prevent collisions with system-level filters and to ensure UI consistency. These rules are defined in `app/models/_validators.py` and applied within the `BookmarkService`.

### Reserved Names
Certain names are reserved for system views and cannot be used for custom tags:
- `all`
- `untagged`
- `archived`
- `trash`

### Length and Format
Tag names must be:
1.  **Non-empty**: Stripped names cannot be empty strings.
2.  **Length-limited**: Names must be 50 characters or fewer.
3.  **Unique**: While the model doesn't enforce uniqueness internally, the `BookmarkRepository` typically manages tags in a way that prevents duplicate names for the same user context.

The `_validate_tag_name` function in `app/models/_validators.py` handles these checks:
```python
def _validate_tag_name(name: str) -> Optional[str]:
    """Return an error if the tag name is reserved or too short."""
    normalized = name.strip().lower()
    if not normalized:
        return "Tag name is required"
    if normalized in _RESERVED_TAG_NAMES:
        return f"'{name}' is a reserved tag name"
    if len(normalized) > 50:
        return "Tag name must be 50 characters or fewer"
    return None
```

## Tag Lifecycle and Management

The `BookmarkService` in `app/services/bookmark_service.py` acts as the orchestrator for tag operations, ensuring that changes to tags are reflected across the system.

### Creation and Updates
When a tag is created or renamed, the service validates the input before persisting it to the `BookmarkRepository`.
```python
def update_tag(self, tag_id: str, data: Dict[str, Any]) -> Tuple[Optional[Tag], Optional[str]]:
    """Update a tag's name or colour."""
    tag = self._repo.get_tag(tag_id)
    if not tag:
        return None, None
    if "name" in data:
        err = _validate_tag_name(data["name"])
        if err:
            return None, err
        tag.rename(data["name"])
    if "color" in data:
        tag.color = TagColor(data["color"])
    self._repo.save_tag(tag)
    return tag, None
```

### Cascade Deletion
Deleting a tag is not a simple removal of the `Tag` object. The `BookmarkService.delete_tag` method performs a "cascade" operation where it:
1.  Identifies every bookmark using that tag.
2.  Removes the tag ID from each bookmark's internal list.
3.  Invalidates the cache for those bookmarks.
4.  Finally, removes the tag from the repository.

```python
def delete_tag(self, tag_id: str) -> bool:
    """Delete a tag and strip it from all bookmarks."""
    tag = self._repo.get_tag(tag_id)
    if not tag:
        return False
    for bookmark in self._repo.get_bookmarks_with_tag(tag_id):
        bookmark.remove_tag(tag_id)
        self._repo.save_bookmark(bookmark)
        self._cache.invalidate(bookmark.id)
    self._repo.delete_tag(tag_id)
    return True
```

## Relationship with Bookmarks

Bookmarks do not store full `Tag` objects. Instead, they store a list of tag IDs (`List[str]`). This decoupling allows tags to be renamed or recolored without needing to update every associated bookmark.

When a user views a bookmark, the system resolves these IDs into `Tag` objects to display the correct names and colors.

## Usage Tracking

The `Tag` model includes a `usage_count` attribute and helper methods `increment_usage()` and `decrement_usage()`. 

```python
def increment_usage(self) -> int:
    """Record that a bookmark now uses this tag. Returns new count."""
    self.usage_count += 1
    return self.usage_count
```

> [!NOTE]
> In the current implementation of `BookmarkService`, these usage methods are defined in the model but are not yet automatically called during bookmark creation or update. Developers manually managing tags should ensure `usage_count` is updated if they require accurate metadata for UI sorting or filtering.