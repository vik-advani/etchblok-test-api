---
title: Domain Model Architecture
description: An overview of the core entities within the system, explaining the relationships between bookmarks, tags, and collections.
code_symbols: [SYM#d731bc2f45cac29b2961ab00083bab5345db0a5e, SYM#e15c91e9e7eae95d052f50388a60d0bc30b3fe67, SYM#97d8a6cbf0c47108aa2beb39fafa695229654067]
section_id: b5275deb-2eaf-4da2-af58-2f6f0d6be87c_domain_model_architecture
doc_type: guide
section_type: guide
---
The domain model architecture of this system is built around three primary entities: **Bookmarks**, **Tags**, and **Collections**. These entities are implemented as Python dataclasses in the `app/models/` directory, providing a clean separation between data structure and business logic.

## The Bookmark Entity

The `Bookmark` class (found in `app/models/bookmark.py`) is the central entity of the system. It represents a saved URL along with its associated metadata.

### State and Lifecycle
Bookmarks use the `BookmarkStatus` enum to manage their visibility and lifecycle:
- `ACTIVE`: The default state for new bookmarks.
- `ARCHIVED`: For bookmarks that are saved but no longer in active use.
- `TRASHED`: A soft-delete state.

The entity tracks its own modification state using a `_touch()` helper, which updates the `updated_at` timestamp whenever a state change occurs (e.g., archiving, trashing, or modifying tags).

```python
# app/models/bookmark.py

def archive(self) -> None:
    """Move the bookmark to the archive."""
    self.status = BookmarkStatus.ARCHIVED
    self._touch()

def _touch(self) -> None:
    """Update the modification timestamp."""
    self.updated_at = datetime.utcnow()
```

### Identification
Bookmarks are identified by a 12-character hex UUID, generated automatically upon instantiation:
`id: str = field(default_factory=lambda: uuid.uuid4().hex[:12])`

## Tags and Categorization

Tags provide a flexible, many-to-many categorization system. The relationship is managed by storing a list of `Tag` IDs directly on the `Bookmark` entity.

### The Tag Entity
The `Tag` class (`app/models/tag.py`) includes:
- **Metadata**: A name, a `TagColor` enum (e.g., `RED`, `BLUE`, `GREEN`), and an optional description.
- **Usage Tracking**: A `usage_count` integer that is incremented or decremented as the tag is attached to or removed from bookmarks.
- **Identification**: An 8-character hex UUID.

### Relationship Management
While the `Bookmark` entity provides methods like `add_tag(tag_id)` and `remove_tag(tag_id)`, the `BookmarkService` orchestrates the integrity of these relationships. For example, when a tag is deleted, the service ensures it is stripped from all bookmarks before the tag itself is removed from the repository.

```python
# app/services/bookmark_service.py

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

## Collections and Grouping

Collections (`app/models/collection.py`) allow users to group bookmarks. They are identified by a 10-character hex UUID. There are two distinct types of collections defined by the `CollectionType` enum:

### Manual Collections
In a `MANUAL` collection, bookmarks are explicitly added or removed by the user. The collection maintains an ordered list of `bookmark_ids`.

### Smart Collections
A `SMART` collection automatically populates itself based on a `filter_rule`. The `_apply_filter` method evaluates this rule against bookmark metadata (currently checking if a keyword exists in the title or description).

```python
# app/models/collection.py

def _apply_filter(self, bookmarks: list) -> List[str]:
    """Evaluate the filter_rule against a list of bookmarks."""
    if not self.filter_rule:
        return []
    keyword = self.filter_rule.lower()
    return [b.id for b in bookmarks if keyword in b.title.lower() or keyword in b.description.lower()]
```

## Service Layer Orchestration

The `BookmarkService` (`app/services/bookmark_service.py`) acts as a facade over the domain models and the repository. It is responsible for:
1.  **Instantiation**: Using `from_dict` methods to create entities from raw request data.
2.  **Validation**: Ensuring URLs and titles meet system requirements before persistence.
3.  **Cross-Entity Logic**: Managing the side effects of operations, such as updating the `SearchIndex` or invalidating the `LRUCache` when a bookmark is modified.

This architecture ensures that the domain models remain focused on data structure and internal state transitions, while the service layer handles the complexities of system-wide data integrity.