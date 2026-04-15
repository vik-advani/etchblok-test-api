---
title: Smart Collection Filtering Logic
description: A technical deep-dive into how filter rules are evaluated against bookmark metadata to populate smart collections.
code_symbols: [SYM#97d8a6cbf0c47108aa2beb39fafa695229654067, SYM#664bcdae74f24d832fff86d384111517f11be0db, SYM#6d5b77b3f04288620db996a9bc18b13d3bd1ad3b]
section_id: 7369eb80-4979-47d8-94c5-f5265bd2c961_smart_collection_filtering_logic
doc_type: guide
section_type: guide
---
The **vik-advani-etchblok-test-api-7ee56a2** codebase distinguishes between two types of bookmark groupings via the `CollectionType` enum in `app/models/collection.py`. While **Manual** collections require explicit user action to add or remove bookmarks, **Smart** collections use a keyword-based filtering mechanism to dynamically identify relevant bookmarks.

## Smart Collection Definition

A collection is designated as "smart" by setting its `collection_type` to `CollectionType.SMART`. This is typically handled during creation in the `BookmarkService.create_collection` method, which consumes a `filter_rule` from the input data.

```python
# app/models/collection.py

class CollectionType(Enum):
    MANUAL = "manual"
    SMART = "smart"

@dataclass
class Collection:
    name: str
    collection_type: CollectionType = CollectionType.MANUAL
    bookmark_ids: List[str] = field(default_factory=list)
    filter_rule: str = ""
    # ...
```

The `filter_rule` is a simple string used as a search term against bookmark metadata.

## The Filtering Mechanism

The core logic for evaluating which bookmarks belong in a smart collection resides in the `Collection._apply_filter` method. This method performs a case-insensitive keyword search across two specific fields of the `Bookmark` model: `title` and `description`.

```python
# app/models/collection.py

def _apply_filter(self, bookmarks: list) -> List[str]:
    """Evaluate the filter_rule against a list of bookmarks.

    Internal method used by the service layer to populate smart collections.
    """
    if not self.filter_rule:
        return []
    keyword = self.filter_rule.lower()
    return [
        b.id for b in bookmarks 
        if keyword in b.title.lower() or keyword in b.description.lower()
    ]
```

### Evaluation Logic
1.  **Normalization**: The `filter_rule` is converted to lowercase.
2.  **Field Matching**: For every bookmark provided to the method, it checks if the keyword exists as a substring in:
    *   `bookmark.title` (lowercased)
    *   `bookmark.description` (lowercased)
3.  **Result**: It returns a list of matching `bookmark.id` strings.

## Constraints and Enforcement

The system enforces a strict separation between manual and smart collection management. If a collection is marked as `SMART`, the `add_bookmark` method will explicitly reject manual attempts to insert a bookmark ID.

```python
# app/models/collection.py

def add_bookmark(self, bookmark_id: str) -> bool:
    """Add a bookmark to a manual collection."""
    if self.is_smart or bookmark_id in self.bookmark_ids:
        return False
    self.bookmark_ids.append(bookmark_id)
    return True
```

This ensures that the `bookmark_ids` list of a smart collection remains purely a product of its `filter_rule` rather than manual user intervention.

## Integration and Architectural State

Currently, the `BookmarkService` in `app/services/bookmark_service.py` facilitates the creation of smart collections and persists them via the `BookmarkRepository`. 

```python
# app/services/bookmark_service.py

def create_collection(self, data: Dict[str, Any]) -> Tuple[Optional[Collection], Optional[str]]:
    """Create a new collection."""
    name = data.get("name", "").strip()
    if not name:
        return None, "Collection name is required"
    collection = Collection.from_dict(data)
    self._repo.save_collection(collection)
    return collection, None
```

### Current Limitations
*   **Dynamic Population**: While `_apply_filter` exists in the model, the `BookmarkService` does not currently trigger this logic during standard retrieval (`get_collection` or `list_collections`). The `bookmark_ids` list must be populated by an external caller or a future service-layer update that passes the full list of bookmarks through the filter.
*   **Rule Complexity**: The filtering is limited to a single keyword substring match. It does not support boolean operators (AND/OR), tag-based filtering, or metadata-specific queries.
*   **Scope**: The filter only inspects `title` and `description`; it ignores the `url`, `tags`, and the `metadata` dictionary on the `Bookmark` object.