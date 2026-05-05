---
title: Understanding Collection Types
description: Explaining the conceptual difference between manual bookmark grouping and rule-based smart collections, including design trade-offs.
code_symbols: [SYM#97d8a6cbf0c47108aa2beb39fafa695229654067, SYM#67d634120bee630945450829e511171fd936cf36, SYM#664bcdae74f24d832fff86d384111517f11be0db]
section_id: d1bb999c-1ae8-4537-aa7c-a6f75b0fa3a2_understanding_collection_types
doc_type: explanation
section_type: guide
---
Collections in this API provide a flexible way to group bookmarks, catering to both explicit user curation and automated organization. This is achieved through the `Collection` class and the `CollectionType` enumeration, which distinguish between **Manual** and **Smart** collections.

## The Dual Nature of Collections

The `CollectionType` enum (found in `app/models/collection.py`) defines the two fundamental modes of operation:

```python
class CollectionType(Enum):
    """The kind of collection."""
    MANUAL = "manual"
    SMART = "smart"
```

This design choice allows the system to reuse the same `Collection` data structure while enforcing different behavioral rules based on the type.

### Manual Collections: Explicit Curation
Manual collections are designed for user-driven organization, such as a "Reading List" or "Project Research." In this mode, the `bookmark_ids` list acts as a stable, ordered container. 

The implementation enforces explicit control through the `add_bookmark` method:

```python
def add_bookmark(self, bookmark_id: str) -> bool:
    if self.is_smart or bookmark_id in self.bookmark_ids:
        return False
    self.bookmark_ids.append(bookmark_id)
    return True
```

Because manual collections are curated, they support custom ordering via the `reorder` method. This method ensures data integrity by requiring that the new list of IDs exactly matches the existing set, preventing accidental additions or removals during a simple reordering operation.

### Smart Collections: Rule-Based Automation
Smart collections are intended to be dynamic views rather than static containers. They rely on a `filter_rule`—a string used to automatically identify relevant bookmarks. 

The core logic for this automation resides in the `_apply_filter` method:

```python
def _apply_filter(self, bookmarks: list) -> List[str]:
    if not self.filter_rule:
        return []
    keyword = self.filter_rule.lower()
    return [b.id for b in bookmarks if keyword in b.title.lower() or keyword in b.description.lower()]
```

Currently, the implementation uses a simple case-insensitive keyword match against the bookmark's `title` and `description`. This provides a low-friction way for users to create "auto-folders" for specific topics (e.g., a collection with the rule "python" would capture all bookmarks mentioning Python in their metadata).

## Design Tradeoffs and Constraints

The implementation of collections reveals several deliberate design decisions and some current limitations:

### Exclusivity of Type
The system strictly separates manual and smart behaviors. If `is_smart` is true, `add_bookmark` will always return `False`. This prevents "polluting" a rule-based collection with manual entries, ensuring that the collection's contents always reflect its defining rule.

### Static Storage vs. Dynamic Evaluation
A notable tradeoff in the current architecture is that `Collection` objects store a physical list of `bookmark_ids` regardless of their type. While `_apply_filter` provides the logic for smart population, the `BookmarkService` (in `app/services/bookmark_service.py`) does not yet automatically invoke this filter during bookmark creation or updates.

This means that smart collections in the current version of the API are "static snapshots." They are created with a rule, but their `bookmark_ids` list must be populated by an external caller or a future service-layer integration. This design favors performance on read operations (since the IDs are already stored) over real-time accuracy.

### Simple Filtering Logic
The `filter_rule` is currently limited to basic string matching. While this is easy to understand and implement, it lacks the complexity of a full query language (e.g., boolean operators like `AND`/`OR`). However, the internal nature of `_apply_filter` allows this logic to be swapped out for a more robust search engine or regex-based filter without changing the public API of the `Collection` class.

## UI and Organization
Both collection types support a "pinning" mechanism via the `is_pinned` attribute and the `pin()`/`unpin()` methods. This is a UI-centric design choice, allowing users to prioritize specific collections (whether manual or smart) at the top of their sidebar, regardless of creation date or size.

```python
def pin(self) -> None:
    """Pin the collection to the top of the sidebar."""
    self.is_pinned = True
```

This attribute, combined with the `size` property, provides the necessary metadata for client applications to render a rich, organized navigation interface.