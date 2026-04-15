---
title: Core Domain Overview
description: 'An introduction to the primary entities within the application: Bookmarks, Tags, and Collections, and how they relate to one another.'
code_symbols: [SYM#d731bc2f45cac29b2961ab00083bab5345db0a5e, SYM#e15c91e9e7eae95d052f50388a60d0bc30b3fe67, SYM#97d8a6cbf0c47108aa2beb39fafa695229654067]
section_id: 1c156bc8-e51b-483a-8de1-0cefbcb22110_core_domain_overview
doc_type: guide
section_type: guide
---
The core domain of the application is built around three primary entities: **Bookmarks**, **Tags**, and **Collections**. These entities are managed by the `BookmarkService` (found in `app/services/bookmark_service.py`), which acts as a facade over the in-memory `BookmarkRepository` and handles business logic, validation, and cross-entity operations.

## Bookmarks and Lifecycle

The `Bookmark` class (defined in `app/models/bookmark.py`) is the central entity representing a saved URL. It includes metadata such as a title, description, and a list of associated tag IDs.

### State Transitions
Bookmarks follow a specific lifecycle managed by the `BookmarkStatus` enum:
- **ACTIVE**: The default state for new bookmarks.
- **ARCHIVED**: For bookmarks that are no longer needed but should be kept.
- **TRASHED**: A soft-delete state for bookmarks intended for removal.

The `Bookmark` model provides methods to transition between these states, each of which updates the `updated_at` timestamp via the internal `_touch()` helper:

```python
# From app/models/bookmark.py
def archive(self) -> None:
    """Move the bookmark to the archive."""
    self.status = BookmarkStatus.ARCHIVED
    self._touch()

def trash(self) -> None:
    """Soft-delete the bookmark by moving it to the trash."""
    self.status = BookmarkStatus.TRASHED
    self._touch()

def restore(self) -> None:
    """Restore a trashed or archived bookmark to active status."""
    self.status = BookmarkStatus.ACTIVE
    self._touch()
```

The `BookmarkService` orchestrates these transitions and ensures they are persisted to the repository and reflected in the search index and cache.

## Tags and Metadata

Tags (defined in `app/models/tag.py`) provide a many-to-many metadata layer for organizing bookmarks. Each `Tag` has a unique name, a description, and a visual color defined by the `TagColor` enum (e.g., `RED`, `BLUE`, `GREEN`).

### Relationship Management
Bookmarks maintain a list of tag IDs in their `tags` attribute. While the `Bookmark` model has methods like `add_tag` and `remove_tag`, the `BookmarkService` handles the broader implications of tag management. For example, deleting a tag requires a cross-entity operation to strip that tag from all bookmarks that reference it:

```python
# From app/services/bookmark_service.py
def delete_tag(self, tag_id: str) -> bool:
    """Delete a tag and strip it from all bookmarks."""
    tag = self._repo.get_tag(tag_id)
    if not tag:
        return False
    # Clean up all bookmarks referencing this tag
    for bookmark in self._repo.get_bookmarks_with_tag(tag_id):
        bookmark.remove_tag(tag_id)
        self._repo.save_bookmark(bookmark)
        self._cache.invalidate(bookmark.id)
    self._repo.delete_tag(tag_id)
    return True
```

## Collections and Organization

Collections (defined in `app/models/collection.py`) allow users to group bookmarks into named sets. The application supports two distinct types of collections via the `CollectionType` enum.

### Manual vs. Smart Collections
- **Manual Collections**: Users explicitly add or remove bookmarks. The order of bookmarks is preserved in the `bookmark_ids` list and can be updated via the `reorder` method.
- **Smart Collections**: These are dynamic groupings based on a `filter_rule`. They do not allow manual addition of bookmarks; instead, they evaluate their rule against the bookmark library.

The `Collection` model implements this logic in its `add_bookmark` method:

```python
# From app/models/collection.py
def add_bookmark(self, bookmark_id: str) -> bool:
    """Add a bookmark to a manual collection."""
    if self.is_smart or bookmark_id in self.bookmark_ids:
        return False
    self.bookmark_ids.append(bookmark_id)
    return True
```

Smart collections use a simple keyword-based filtering mechanism that checks the bookmark's title and description:

```python
# From app/models/collection.py
def _apply_filter(self, bookmarks: list) -> List[str]:
    """Evaluate the filter_rule against a list of bookmarks."""
    if not self.filter_rule:
        return []
    keyword = self.filter_rule.lower()
    return [b.id for b in bookmarks if keyword in b.title.lower() or keyword in b.description.lower()]
```

### Pinning
Both manual and smart collections can be "pinned," which is a boolean flag (`is_pinned`) intended to signal to the UI that the collection should be prioritized in navigation.

## Service Orchestration

The `BookmarkService` is implemented as a singleton to ensure consistent state across the application's various API blueprints. It coordinates the interaction between models and the `BookmarkRepository` (found in `app/db/repository.py`).

When a bookmark is created or updated, the service handles:
1. **Validation**: Using helpers from `app.models._validators`.
2. **Persistence**: Saving the entity to the repository.
3. **Search Indexing**: Updating the `SearchIndex` for full-text search.
4. **Cache Management**: Invalidating the `LRUCache` to ensure subsequent reads fetch the latest data.

This centralized logic ensures that domain rules—such as a bookmark's status or a collection's type—are enforced consistently regardless of which API endpoint triggers the change.