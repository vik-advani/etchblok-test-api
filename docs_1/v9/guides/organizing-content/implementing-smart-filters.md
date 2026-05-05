---
title: Implementing Smart Filters
description: How to define and apply filter rules to automatically populate smart collections based on bookmark titles and descriptions.
code_symbols: [SYM#97d8a6cbf0c47108aa2beb39fafa695229654067, SYM#664bcdae74f24d832fff86d384111517f11be0db]
section_id: 51b7088a-72ee-4022-a248-70a8d5d1b779_implementing_smart_filters
doc_type: guide
section_type: guide
---
In this codebase, collections are used to group bookmarks. While most collections are managed manually, the system provides a foundation for **Smart Collections**—groups that automatically include bookmarks based on specific criteria.

### Collection Types

The `Collection` model (found in `app/models/collection.py`) distinguishes between two types of collections via the `CollectionType` enum:

*   **Manual (`CollectionType.MANUAL`)**: The default type. Bookmarks are added and removed explicitly by the user.
*   **Smart (`CollectionType.SMART`)**: Bookmarks are intended to be populated automatically based on a `filter_rule`.

### Defining a Smart Filter

A smart collection is defined by its `filter_rule`, which is a simple string keyword. When creating a collection through the `BookmarkService` or the REST API, you can specify the type and the rule:

```python
# Example of creating a smart collection via the model
from app.models.collection import Collection, CollectionType

smart_collection = Collection(
    name="Python Resources",
    collection_type=CollectionType.SMART,
    filter_rule="python"
)
```

In the REST API (`app/routes/collections.py`), this is handled by passing the `type` and `filter_rule` in the POST request body:

```python
@collections_bp.route("/", methods=["POST"])
def create_collection():
    # Expects JSON with "name", "type": "smart", and "filter_rule"
    data = request.get_json(force=True)
    collection, error = _service.create_collection(data)
    # ...
```

### Filtering Logic

The logic for evaluating which bookmarks belong in a smart collection is encapsulated in the `_apply_filter` method within `app/models/collection.py`. 

The filter performs a **case-insensitive keyword match** against two fields of the `Bookmark` model:
1.  `title`
2.  `description`

```python
def _apply_filter(self, bookmarks: list) -> List[str]:
    """Evaluate the filter_rule against a list of bookmarks."""
    if not self.filter_rule:
        return []
    keyword = self.filter_rule.lower()
    return [
        b.id for b in bookmarks 
        if keyword in b.title.lower() or keyword in b.description.lower()
    ]
```

### Constraints on Smart Collections

To maintain the integrity of the automated grouping, smart collections explicitly prevent manual bookmark management. The `add_bookmark` method in the `Collection` class checks the `is_smart` property and rejects any manual additions:

```python
def add_bookmark(self, bookmark_id: str) -> bool:
    """Add a bookmark to a manual collection."""
    if self.is_smart or bookmark_id in self.bookmark_ids:
        return False
    self.bookmark_ids.append(bookmark_id)
    return True
```

If a user attempts to add a bookmark to a smart collection via the API, the `BookmarkService.add_to_collection` method will return `False`, resulting in a `400 Bad Request` error from the route handler.

### Current Implementation Status

While the `Collection` model provides the `_apply_filter` method and the necessary attributes, the current implementation of `BookmarkService` and `BookmarkRepository` does not automatically invoke this filtering logic during bookmark creation or retrieval. 

The `_apply_filter` method is designed as an internal utility for the service layer to eventually synchronize the `bookmark_ids` list with the set of bookmarks matching the `filter_rule`. Currently, smart collections created via the API will remain empty until this integration is completed in the service layer.