---
title: Working with Manual Collections
description: Step-by-step guide to adding, removing, and reordering bookmarks within a manual collection using the service layer.
code_symbols: [SYM#97d8a6cbf0c47108aa2beb39fafa695229654067, SYM#664bcdae74f24d832fff86d384111517f11be0db, SYM#dd5fd545693e00d6c0b38892bbd18ed4afa136a1, SYM#fdcfaed9972e477ae4585fd1b279ac107fa534cd]
section_id: bc4b2d6a-f5d8-4c5d-90bf-fa40ba2c1077_working_with_manual_collections
doc_type: how_to
section_type: guide
---
To manage bookmarks within a manual collection, use the `BookmarkService` to handle persistence and the `Collection` model to manage the internal ordering of bookmark IDs.

### Basic Operations

The `BookmarkService` provides high-level methods for creating collections and managing their contents.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()

# 1. Create a manual collection
collection, error = service.create_collection({"name": "Reading List"})
collection_id = collection.id

# 2. Add a bookmark to the collection
# Returns True if successful, False if bookmark already exists or collection is smart
success = service.add_to_collection(collection_id, "bookmark_123")

# 3. Remove a bookmark from the collection
# Returns True if successful, False if bookmark or collection not found
success = service.remove_from_collection(collection_id, "bookmark_123")
```

### Adding and Removing Bookmarks

The `BookmarkService` methods `add_to_collection` and `remove_from_collection` encapsulate the logic of fetching the collection from the repository, updating its internal list of IDs, and saving it back.

*   **`add_to_collection(collection_id, bookmark_id)`**: This method calls `collection.add_bookmark(bookmark_id)`. It will fail (return `False`) if the collection is a **Smart Collection** or if the bookmark is already present.
*   **`remove_from_collection(collection_id, bookmark_id)`**: This method calls `collection.remove_bookmark(bookmark_id)`. It returns `False` if the bookmark ID is not found in the collection.

### Reordering Bookmarks

The `Collection` model maintains an ordered list of bookmark IDs in its `bookmark_ids` attribute. To change the order, you must use the `reorder` method on the `Collection` instance.

Note that the `BookmarkService` does not currently have a dedicated `reorder` method, so you must interact with the model directly and then ensure the state is persisted.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()
collection = service.get_collection("coll_abc")

if collection:
    # The new list must contain EXACTLY the same IDs as the current list
    current_ids = collection.bookmark_ids
    new_order = list(reversed(current_ids))
    
    try:
        collection.reorder(new_order)
        # In the current in-memory implementation, changes to the object 
        # are reflected immediately, but for future-proofing, 
        # you would typically trigger a save here.
    except ValueError as e:
        print(f"Reorder failed: {e}")
```

### Troubleshooting and Constraints

#### Smart Collections
Manual operations like `add_bookmark` and `reorder` are restricted for smart collections. If `collection.is_smart` is `True`, `add_bookmark` will return `False` and manual modifications to `bookmark_ids` should be avoided as they are intended to be populated via `filter_rule`.

#### Reorder Validation
The `reorder` method in `app/models/collection.py` performs a set comparison to ensure no bookmarks are lost or added during the reordering process:

```python
def reorder(self, bookmark_ids: List[str]) -> None:
    if set(bookmark_ids) != set(self.bookmark_ids):
        raise ValueError("Reorder list must contain exactly the same bookmark IDs")
    self.bookmark_ids = bookmark_ids
```

If you need to add or remove bookmarks while reordering, perform the `add_bookmark` or `remove_bookmark` operations first before calling `reorder`.

#### Persistence
The current `BookmarkRepository` is in-memory. While `BookmarkService` methods like `add_to_collection` explicitly call `self._repo.save_collection(collection)`, modifying a collection object returned by `get_collection` will update the instance held in memory. However, always use the service layer methods when available to ensure consistency with search indexes and caches.