---
title: Tagging Fundamentals
description: Overview of the Tag model, including color presets, usage tracking, and alphabetical sorting logic.
code_symbols: [SYM#e15c91e9e7eae95d052f50388a60d0bc30b3fe67, SYM#a903c58b7b9829413e7dde33ff94fc7516b965f1, SYM#8d687603253ce34b667ab707a05b20e9357dd6af]
section_id: 8358221d-320e-410b-90b0-5fcc2189ac14_tagging_fundamentals
doc_type: guide
section_type: guide
---
Tags in this system serve as the primary organizational tool for bookmarks. Implemented as a lightweight label, a `Tag` can be associated with multiple bookmarks to facilitate filtering and categorization.

## The Tag Model

The `Tag` class (found in `app/models/tag.py`) is a data structure that tracks metadata and usage statistics for a specific label.

### Core Attributes
- **id**: A unique 8-character hexadecimal string generated using `uuid.uuid4().hex[:8]`.
- **name**: The display name of the tag. This is the primary identifier for users and must be unique.
- **color**: A visual preset defined by the `TagColor` enum.
- **usage_count**: An integer tracking how many bookmarks are currently associated with this tag.

```python
class Tag:
    name: str
    color: TagColor = TagColor.GRAY
    description: str = ""
    id: str = field(default_factory=lambda: uuid.uuid4().hex[:8])
    usage_count: int = 0
```

## Visual Categorization

The `TagColor` enum provides a set of preset colors for UI rendering. When a tag is created without a specified color, it defaults to `TagColor.GRAY`.

Available colors in `app/models/tag.py`:
- `RED`, `BLUE`, `GREEN`, `YELLOW`, `PURPLE`, `GRAY`

These colors are serialized to their string values (e.g., `"red"`) when the tag is converted to a dictionary via `to_dict()`.

## Lifecycle and Validation

Tag creation and modification are governed by strict validation rules defined in `app/models/_validators.py`. These rules ensure consistency and prevent conflicts with system-reserved keywords.

### Validation Rules
The `_validate_tag_name` function enforces the following:
1.  **Non-empty**: Tag names cannot be empty or consist only of whitespace.
2.  **Length**: Names must be 50 characters or fewer.
3.  **Reserved Names**: The system prohibits names that conflict with built-in filters: `all`, `untagged`, `archived`, and `trash`.

### Renaming Logic
The `Tag.rename()` method handles the update of a tag's display name while performing basic sanitization:

```python
def rename(self, new_name: str) -> None:
    new_name = new_name.strip()
    if not new_name:
        raise ValueError("Tag name cannot be empty")
    if len(new_name) > 50:
        raise ValueError("Tag name cannot exceed 50 characters")
    self.name = new_name
```

## Sorting and Organization

Tags are designed to be displayed in alphabetical order. The `Tag` class implements the `__lt__` dunder method to provide case-insensitive sorting based on the tag's name.

```python
def __lt__(self, other: "Tag") -> bool:
    """Allow sorting tags alphabetically by name."""
    return self.name.lower() < other.name.lower()
```

This logic is utilized in the API layer (e.g., `app/routes/tags.py`) to ensure that the list of tags returned to the client is always ordered:

```python
@tags_bp.route("/", methods=["GET"])
def list_tags():
    tags = _service.list_tags()
    return jsonify({"tags": [t.to_dict() for t in sorted(tags)]})
```

## Usage Tracking

The `Tag` model includes methods for tracking how often it is used across the system: `increment_usage()` and `decrement_usage()`. 

While these methods exist to update the `usage_count` attribute, the current implementation in `BookmarkService` and `Bookmark` focuses on the relationship management. For instance, when a tag is deleted via `BookmarkService.delete_tag`, the service iterates through all associated bookmarks and removes the tag ID from their internal lists before deleting the tag from the repository:

```python
def delete_tag(self, tag_id: str) -> bool:
    tag = self._repo.get_tag(tag_id)
    if not tag:
        return False
    for bookmark in self._repo.get_bookmarks_with_tag(tag_id):
        bookmark.remove_tag(tag_id)
        self._repo.save_bookmark(bookmark)
    self._repo.delete_tag(tag_id)
    return True
```

## Integration with Bookmarks

Bookmarks do not store full `Tag` objects; instead, they store a list of tag IDs in the `Bookmark.tags` attribute. This decoupling allows tags to be renamed or modified globally without requiring updates to every individual bookmark record. The `Bookmark` model (in `app/models/bookmark.py`) provides `add_tag(tag_id)` and `remove_tag(tag_id)` methods to manage these associations.