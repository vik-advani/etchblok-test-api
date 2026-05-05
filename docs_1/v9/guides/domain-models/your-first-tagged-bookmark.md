---
title: Your First Tagged Bookmark
description: A step-by-step tutorial on creating a bookmark, defining a custom tag, and associating them together.
code_symbols: [SYM#d731bc2f45cac29b2961ab00083bab5345db0a5e, SYM#e15c91e9e7eae95d052f50388a60d0bc30b3fe67, SYM#6d5b77b3f04288620db996a9bc18b13d3bd1ad3b, SYM#a903c58b7b9829413e7dde33ff94fc7516b965f1]
section_id: 08684de1-6623-4cfc-9611-e4b1b9574f41_your_first_tagged_bookmark
doc_type: tutorial
section_type: guide
---
In this tutorial, you will learn how to use the core service and models to create a bookmark, define a custom tag with specific properties, and link them together. By the end, you will have a fully persisted, tagged bookmark ready for searching and organization.

### Prerequisites

To follow this tutorial, you need to have the application environment set up so you can import the following components:

*   `BookmarkService`: The primary facade for managing bookmarks and tags.
*   `TagColor`: An enumeration of allowed colors for tags.

### Step 1: Initialize the Bookmark Service

The `BookmarkService` is implemented as a singleton to ensure that the in-memory repository and search index are shared across your application.

```python
from app.services.bookmark_service import BookmarkService

# Initialize the singleton service
service = BookmarkService()
```

The service automatically bootstraps its own `BookmarkRepository` and `SearchIndex` upon initialization.

### Step 2: Create a Custom Tag

Before associating a tag with a bookmark, you must create the tag entity. Tags in this system have a name, a color, and an optional description.

```python
from app.models.tag import TagColor

# Define the tag properties
tag_data = {
    "name": "Programming",
    "color": TagColor.BLUE.value,
    "description": "Resources for software development"
}

# Persist the tag via the service
tag, error = service.create_tag(tag_data)

if error:
    print(f"Failed to create tag: {error}")
else:
    print(f"Created tag '{tag.name}' with ID: {tag.id}")
```

The `create_tag` method validates the name against a list of reserved words (like `all`, `untagged`, `archived`, and `trash`). It returns a tuple containing the created `Tag` object and an error message (if any).

### Step 3: Create a Bookmark with the Tag

Now that you have a tag ID, you can create a new bookmark and associate it with that tag immediately by including the ID in the `tags` list.

```python
# Define the bookmark properties
bookmark_data = {
    "url": "https://www.python.org",
    "title": "Python Programming Language",
    "description": "The official home of the Python Programming Language",
    "tags": [tag.id]  # Associate the tag created in Step 2
}

# Persist the bookmark via the service
bookmark, error = service.create_bookmark(bookmark_data)

if error:
    print(f"Failed to create bookmark: {error}")
else:
    print(f"Created bookmark '{bookmark.title}' (ID: {bookmark.id})")
    print(f"Tags: {bookmark.tags}")
```

When `create_bookmark` is called, the service validates the URL format and title length before saving the bookmark to the repository and indexing it for full-text search.

### Step 4: Add a Tag to an Existing Bookmark

If you need to add a tag to a bookmark that already exists, you can use the `add_tag` method on the `Bookmark` model and then persist the change using the service.

```python
# 1. Create a second tag
new_tag, _ = service.create_tag({"name": "Python", "color": TagColor.YELLOW.value})

# 2. Retrieve the existing bookmark
existing_bookmark = service.get_bookmark(bookmark.id)

# 3. Add the tag ID to the bookmark model
if existing_bookmark.add_tag(new_tag.id):
    # 4. Persist the change via the service's update method
    service.update_bookmark(existing_bookmark.id, {})
    print(f"Added tag '{new_tag.name}' to bookmark.")
```

The `add_tag` method on the `Bookmark` class handles the logic of preventing duplicate tags and automatically updates the `updated_at` timestamp via an internal `_touch()` call. Calling `update_bookmark` with an empty dictionary ensures the modified model is saved back to the repository and the search index is updated.

### Complete Result

You have now successfully created a tagged bookmark. You can verify the result by converting the bookmark to a dictionary, which is the format used for API responses:

```python
import json

# View the final state of the bookmark
print(json.dumps(bookmark.to_dict(), indent=2))
```

**Next Steps:**
*   Try archiving your bookmark using `service.archive_bookmark(bookmark.id)`.
*   Search for your bookmark by title or description using `service.search("Python")`.
*   Organize your bookmarks into groups using the `Collection` model.