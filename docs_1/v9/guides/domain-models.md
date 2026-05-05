---
title: Domain Models
description: Core data structures representing Bookmarks, Tags, and Collections, including their metadata and serialization logic.
code_symbols: [SYM#d731bc2f45cac29b2961ab00083bab5345db0a5e, SYM#e15c91e9e7eae95d052f50388a60d0bc30b3fe67, SYM#97d8a6cbf0c47108aa2beb39fafa695229654067]
section_id: d2591072-fe94-4e4c-911d-9ec9ddbedf0b_domain_models
doc_type: explanation
section_type: guide
---
Domain models in this project are implemented as Python `dataclasses`, serving as the primary data structures for the application. They encapsulate both state and core business logic, such as state transitions for bookmarks and filtering logic for collections.

## Bookmark Lifecycle and State

The `Bookmark` class (found in `app/models/bookmark.py`) is the central entity. It represents a saved URL and its associated metadata. A key design choice is the use of an explicit `BookmarkStatus` enum to manage the visibility and lifecycle of a bookmark.

### State Transitions
Rather than relying on external services to modify status, the `Bookmark` model provides internal methods to handle state transitions. This ensures that the `updated_at` timestamp is consistently updated via the private `_touch()` helper.

```python
def archive(self) -> None:
    """Move the bookmark to the archive."""
    self.status = BookmarkStatus.ARCHIVED
    self._touch()

def trash(self) -> None:
    """Soft-delete the bookmark by moving it to the trash."""
    self.status = BookmarkStatus.TRASHED
    self._touch()
```

### Tag Association
Bookmarks maintain a list of tag IDs. The model includes methods like `add_tag` and `remove_tag` which return boolean values to indicate if the operation resulted in a state change, preventing duplicate tags or unnecessary updates.

## Tag Management and Constraints

The `Tag` model (`app/models/tag.py`) provides organizational labels. Beyond simple naming, it includes a `usage_count` to track how many bookmarks are associated with it, which is manually incremented or decremented by the service layer during bookmark updates.

### Reserved Keywords and Validation
The system enforces strict naming rules for tags. While the `Tag` model itself handles basic length validation in its `rename` method, broader validation logic is centralized in `app/models/_validators.py`. This includes a set of reserved names that cannot be used for user-defined tags:

```python
_RESERVED_TAG_NAMES = frozenset({"all", "untagged", "archived", "trash"})
```

These reserved names are used by the UI and API to represent virtual collections or system states, so preventing users from creating tags with these names avoids naming collisions in the routing layer.

## Collections: Manual vs. Smart

The `Collection` model (`app/models/collection.py`) supports two distinct behaviors defined by the `CollectionType` enum:

1.  **MANUAL**: A static list of `bookmark_ids` explicitly managed by the user.
2.  **SMART**: A dynamic collection that populates itself based on a `filter_rule`.

### Smart Filtering Logic
Smart collections do not store a list of IDs permanently. Instead, they provide an internal `_apply_filter` method that the service layer uses to evaluate which bookmarks match the collection's criteria. The current implementation uses a naive case-insensitive string match against the bookmark's title and description:

```python
def _apply_filter(self, bookmarks: list) -> List[str]:
    if not self.filter_rule:
        return []
    keyword = self.filter_rule.lower()
    return [b.id for b in bookmarks if keyword in b.title.lower() or keyword in b.description.lower()]
```

### Manual Ordering
For manual collections, the `reorder` method allows users to define a custom sequence for bookmarks. It includes a safety check to ensure the new list of IDs exactly matches the existing set, preventing accidental data loss during a reorder operation.

## Serialization and Identity

The project follows a consistent pattern for object identity and serialization across all domain models.

### ID Strategy
Each model generates its own unique identifier using `uuid.uuid4().hex`, but truncates them to different lengths to distinguish between entity types:
*   **Bookmarks**: 12 characters (`uuid.uuid4().hex[:12]`)
*   **Collections**: 10 characters (`uuid.uuid4().hex[:10]`)
*   **Tags**: 8 characters (`uuid.uuid4().hex[:8]`)

### Serialization Pattern
Every model implements `to_dict()` and `from_dict()` methods. 
*   `to_dict()`: Prepares the model for JSON serialization, converting enums to strings and datetimes to ISO format.
*   `from_dict()`: Acts as a factory method, typically used when creating new instances from API request bodies. Note that `from_dict` often only maps a subset of fields (like `url` and `title`), allowing the class defaults (like `id` and `created_at`) to handle the rest.

```python
@classmethod
def from_dict(cls, data: Dict[str, Any]) -> "Bookmark":
    return cls(
        url=data["url"],
        title=data["title"],
        description=data.get("description", ""),
        tags=data.get("tags", []),
    )
```