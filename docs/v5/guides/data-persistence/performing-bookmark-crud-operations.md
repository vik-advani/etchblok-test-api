---
title: Performing Bookmark CRUD Operations
description: Step-by-step instructions on how to create, retrieve, update, and delete bookmarks using the repository methods.
code_symbols: [SYM#adb8232356346a5957ff3a1a1b7ff70581f37649]
section_id: 9e953509-1cf1-4330-9c22-cecf031fa28a_performing_bookmark_crud_operations
doc_type: how_to
section_type: guide
---
To perform CRUD (Create, Read, Update, Delete) operations on bookmarks in this project, you interact with the `BookmarkRepository` using `Bookmark` model instances.

### Creating and Saving a Bookmark

To create a new bookmark, instantiate a `Bookmark` object and pass it to the `save_bookmark` method. The repository handles both insertion and updates through this single method.

```python
from app.db.repository import BookmarkRepository
from app.models.bookmark import Bookmark

repo = BookmarkRepository()

# Create a new bookmark instance
new_bookmark = Bookmark(
    url="https://example.com",
    title="Example Domain",
    description="A site for examples"
)

# Persist to the in-memory store
repo.save_bookmark(new_bookmark)
```

### Retrieving Bookmarks

You can retrieve a single bookmark by its ID or fetch a paginated list with optional status filtering.

```python
# Retrieve a single bookmark by ID
bookmark = repo.get_bookmark("some-id-123")
if bookmark:
    print(f"Found: {bookmark.title}")

# List bookmarks with pagination and filtering
# Returns a Tuple[List[Bookmark], int] (items, total_count)
items, total = repo.list_bookmarks(page=1, per_page=10, status="active")

for b in items:
    print(f"{b.id}: {b.url}")
```

### Updating a Bookmark

Updates follow a fetch-modify-save pattern. Since the repository stores objects in memory, you retrieve the instance, modify its attributes, and call `save_bookmark` to ensure any internal state (like the `updated_at` timestamp) is correctly handled if you use model helpers.

```python
bookmark = repo.get_bookmark("some-id-123")

if bookmark:
    # Update fields
    bookmark.title = "Updated Title"
    bookmark.description = "New description"
    
    # Use model helper to update the timestamp
    bookmark._touch()
    
    # Persist changes
    repo.save_bookmark(bookmark)
```

### Deleting a Bookmark

The repository provides a `delete_bookmark` method for hard-deleting records. However, the application logic in `BookmarkService` typically prefers a "soft delete" by moving bookmarks to the trash.

#### Hard Delete (Repository Level)
This permanently removes the bookmark from the in-memory storage.

```python
# Returns True if the bookmark existed and was removed
success = repo.delete_bookmark("some-id-123")
```

#### Soft Delete (Service Pattern)
To follow the project's standard deletion pattern, update the bookmark's status instead of removing it.

```python
bookmark = repo.get_bookmark("some-id-123")
if bookmark:
    # Uses the Bookmark.trash() helper to set status to BookmarkStatus.TRASHED
    bookmark.trash() 
    repo.save_bookmark(bookmark)
```

### Filtering by Tags

The repository includes a specialized retrieval method for finding bookmarks associated with a specific tag.

```python
# Returns a list of Bookmark objects containing the tag ID
tagged_bookmarks = repo.get_bookmarks_with_tag("python-tag-id")
```

### Troubleshooting and Gotchas

*   **In-Memory Storage**: The `BookmarkRepository` uses in-memory dictionaries (`self._bookmarks`). All data is lost when the application process restarts.
*   **No Automatic Validation**: The repository does not validate URLs or titles. It assumes the `Bookmark` object passed to `save_bookmark` is already valid. Use `Bookmark.from_dict()` or manual validation before saving.
*   **Hardcoded Sorting**: The `list_bookmarks` method always sorts results by `created_at` in descending order.
*   **Manual Timestamps**: While the `Bookmark` model has a `_touch()` method to update `updated_at`, the repository's `save_bookmark` does not call this automatically. You must call `bookmark._touch()` before saving if you want to track the modification time.