---
title: Collection Management
description: Procedures for grouping bookmarks into collections and managing membership within those groups.
code_symbols: [SYM#dd5fd545693e00d6c0b38892bbd18ed4afa136a1, SYM#fdcfaed9972e477ae4585fd1b279ac107fa534cd]
section_id: 7cade144-fdb2-4f5c-b2b1-dddf3d2ed147_collection_management
doc_type: how_to
section_type: guide
---
To group bookmarks into collections, use the `BookmarkService` to create collections and manage their membership. The system supports both **Manual** collections (where you explicitly add bookmarks) and **Smart** collections (which automatically include bookmarks based on a filter rule).

### Creating a Manual Collection

To create a standard collection where you manually manage the bookmarks, provide a name and set the type to `manual`.

```python
from app.services.bookmark_service import BookmarkService

service = BookmarkService()

# Create a manual collection
data = {
    "name": "Research Project",
    "type": "manual"
}
collection, error = service.create_collection(data)

if error:
    print(f"Failed to create collection: {error}")
else:
    print(f"Created collection: {collection.id}")
```

### Managing Collection Membership

For manual collections, you can add or remove bookmarks using their unique IDs.

```python
collection_id = "abc123def4"
bookmark_id = "xyz789"

# Add a bookmark to the collection
success = service.add_to_collection(collection_id, bookmark_id)

if success:
    print("Bookmark added successfully")
else:
    # This can fail if the collection is SMART or the bookmark is already present
    print("Failed to add bookmark")

# Remove a bookmark from the collection
removed = service.remove_from_collection(collection_id, bookmark_id)
```

### Creating a Smart Collection

Smart collections use a `filter_rule` to automatically identify relevant bookmarks. The system matches the `filter_rule` string against the title and description of bookmarks.

```python
# Create a smart collection for Python-related bookmarks
data = {
    "name": "Python Resources",
    "type": "smart",
    "filter_rule": "python"
}
collection, error = service.create_collection(data)
```

> [!NOTE]
> While the `Collection` model includes a `_apply_filter` method for evaluating these rules, the `BookmarkService` currently persists the `filter_rule` but does not automatically populate the `bookmark_ids` list for smart collections during retrieval.

### Listing and Retrieving Collections

You can retrieve all collections or a specific one by its ID.

```python
# List all collections
all_collections = service.list_collections()
for col in all_collections:
    print(f"{col.name} ({col.size} bookmarks)")

# Get a specific collection
collection = service.get_collection("abc123def4")
if collection:
    print(f"Retrieved: {collection.name}")
```

### Advanced Collection State

The `Collection` model (found in `app/models/collection.py`) provides utility methods for managing the state of a collection instance, such as pinning it to the top of a UI sidebar or reordering its contents.

```python
collection = service.get_collection("abc123def4")

if collection:
    # Pin the collection
    collection.pin()
    
    # Reorder bookmarks (must provide the exact same set of IDs)
    new_order = ["id_2", "id_1", "id_3"]
    collection.reorder(new_order)
    
    # Note: You must manually save the collection back to the repository 
    # if you modify it directly via model methods.
```

### Troubleshooting

*   **Smart Collection Additions**: If `add_to_collection` returns `False`, check if the collection is a `SMART` type. The `Collection.add_bookmark` method explicitly prevents manual additions to smart collections.
*   **Name Validation**: The `BookmarkService.create_collection` method requires a non-empty name. It will return an error message `"Collection name is required"` if the name is missing or only contains whitespace.
*   **Reordering Errors**: The `reorder` method will raise a `ValueError` if the list of IDs provided does not exactly match the set of IDs already in the collection.