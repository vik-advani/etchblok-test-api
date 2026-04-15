---
title: Domain Models
description: Core data structures for bookmarks, tags, and collections, including their serialization and internal state logic.
code_symbols: [SYM#d731bc2f45cac29b2961ab00083bab5345db0a5e, SYM#e15c91e9e7eae95d052f50388a60d0bc30b3fe67, SYM#97d8a6cbf0c47108aa2beb39fafa695229654067]
section_id: b9d8bb33-9d94-4d4d-a295-e3d8a044aeb2_domain_models
doc_type: explanation
section_type: guide
---
The domain models in this project are implemented as Python dataclasses, providing a structured and type-safe representation of the core entities: bookmarks, tags, and collections. These models encapsulate not only data but also the business logic for state transitions, usage tracking, and internal validation.

## Bookmark Lifecycle and State

The `Bookmark` class (defined in `app/models/bookmark.py`) is the central entity of the system. It represents a saved URL and its associated metadata. A key feature of the bookmark model is its explicit state management through the `BookmarkStatus` enum.

### State Transitions
Bookmarks transition between three states: `ACTIVE`, `ARCHIVED`, and `TRASHED`. These transitions are handled by dedicated methods that ensure the `updated_at` timestamp is refreshed via an internal `_touch()` helper:

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

### Identity and Metadata
Each bookmark is assigned a unique 12-character hex ID generated from a UUID. Beyond standard fields like `url`, `title`, and `description`, the model includes a `metadata` dictionary for extensibility, allowing the storage of arbitrary key-value pairs without altering the schema.

## Tagging and Usage Tracking

Tags provide a flexible way to organize bookmarks. The `Tag` model in `app/models/tag.py` includes logic for visual representation and usage statistics.

### Visual Customization
Tags support a predefined set of colors via the `TagColor` enum (RED, BLUE, GREEN, YELLOW, PURPLE, GRAY). This is serialized as a string but enforced through the `Tag.from_dict` factory method.

### Usage Counting
To optimize UI rendering (e.g., showing how many bookmarks have a specific tag), the `Tag` model maintains a `usage_count`. This count is updated by the service layer using the model's increment/decrement methods:

```python
def increment_usage(self) -> int:
    """Record that a bookmark now uses this tag. Returns new count."""
    self.usage_count += 1
    return self.usage_count
```

Tags also implement the `__lt__` dunder method, which enables native Python sorting alphabetically by name, ignoring case.

## Collections: Manual vs. Smart

The `Collection` model (`app/models/collection.py`) supports two distinct organizational patterns defined by `CollectionType`.

### Manual Collections
Manual collections act as traditional folders. Users explicitly add or remove bookmark IDs. The model provides a `reorder` method that allows users to define a custom sequence for bookmarks, provided the new list contains exactly the same IDs as the current set.

### Smart Collections
Smart collections are dynamic and populate themselves based on a `filter_rule`. The `_apply_filter` method implements a simple keyword-based search against bookmark titles and descriptions:

```python
def _apply_filter(self, bookmarks: list) -> List[str]:
    if not self.filter_rule:
        return []
    keyword = self.filter_rule.lower()
    return [b.id for b in bookmarks if keyword in b.title.lower() or keyword in b.description.lower()]
```

Note that `add_bookmark` will return `False` if called on a smart collection, as membership is determined solely by the filter rule.

## Serialization and Validation Logic

The project follows a consistent pattern for data exchange using `to_dict` and `from_dict` methods. This decouples the internal dataclass representation from the JSON structure used in API responses.

### Internal Validation
Validation is centralized in `app/models/_validators.py`. This module is treated as private to the models and contains logic for:
- **URL Formatting**: Enforced via a complex regex (`_URL_PATTERN`).
- **Length Constraints**: Titles are limited to 256 characters, descriptions to 2048, and tag names to 50.
- **Reserved Names**: Tag names like `all`, `untagged`, `archived`, and `trash` are reserved for system use and will fail validation.

### Design Tradeoffs
The implementation reveals several specific design choices:
- **ID Inconsistency**: The system uses inconsistent ID lengths across entities (12 chars for Bookmarks, 8 for Tags, and 10 for Collections). While this visually distinguishes IDs, it requires developers to be aware of the specific entity type when handling raw ID strings.
- **Naive Filtering**: The smart collection filtering is case-insensitive but limited to basic keyword matching. It does not support complex boolean logic (AND/OR) or tag-based filtering within the model itself.
- **UTC Timestamps**: The models use `datetime.utcnow()` for all timestamps, ensuring consistency across timezones, though it relies on the application server's clock accuracy.