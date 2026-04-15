---
title: Data Serialization and Persistence
description: How to use the built-in serialization methods to convert domain models into JSON-safe dictionaries for API responses.
code_symbols: [SYM#6d5b77b3f04288620db996a9bc18b13d3bd1ad3b, SYM#a903c58b7b9829413e7dde33ff94fc7516b965f1, SYM#664bcdae74f24d832fff86d384111517f11be0db]
section_id: efbab318-d97e-4ec9-96de-eebbd1283871_data_serialization_and_persistence
doc_type: how_to
section_type: guide
---
To convert domain models into JSON-safe dictionaries for API responses or to instantiate models from request payloads, use the built-in `to_dict()` and `from_dict()` methods.

## Serializing Models for API Responses

Use the `to_dict()` method on any model instance to generate a dictionary compatible with Flask's `jsonify()`. This method handles the conversion of internal types like Enums (to strings) and Datetimes (to ISO 8601 strings).

```python
from flask import jsonify
from app.models.bookmark import Bookmark

def get_bookmark(bookmark_id: str):
    # Retrieve a model instance (e.g., from a service or repo)
    bookmark = _service.get_bookmark(bookmark_id)
    
    # Convert to dict and return as JSON
    return jsonify(bookmark.to_dict())
```

### Serialization Output Examples

Each model includes specific fields in its `to_dict()` output:

*   **Bookmark**: Includes `id`, `url`, `title`, `description`, `tags` (list of IDs), `status` (string), `created_at`, `updated_at`, and the `metadata` dictionary.
*   **Tag**: Includes `id`, `name`, `color` (string), `description`, and `usage_count`.
*   **Collection**: Includes `id`, `name`, `type` (string), `bookmark_ids`, `filter_rule`, `is_pinned`, `created_at`, and a computed `size` property.

## Deserializing Data into Models

Use the `@classmethod from_dict()` to create a new model instance from a dictionary, typically sourced from `request.get_json()`.

```python
from app.models.bookmark import Bookmark

def create_bookmark(data: dict):
    # Instantiate the model from a dictionary
    # Note: This should happen after validation
    bookmark = Bookmark.from_dict(data)
    
    # The model is now ready for persistence
    self._repo.save_bookmark(bookmark)
    return bookmark
```

### Required Fields for Deserialization

When using `from_dict()`, ensure the input dictionary contains the following required keys:

| Model | Required Keys | Optional Keys |
| :--- | :--- | :--- |
| **Bookmark** | `url`, `title` | `description`, `tags` |
| **Tag** | `name` | `color`, `description` |
| **Collection** | `name` | `type`, `filter_rule` |

## Handling Complex Types

The serialization layer automatically manages the conversion between Python objects and JSON-safe primitives.

### Enums and Constants
Models like `Bookmark` and `Tag` use Enums for status and color. `to_dict()` extracts the `.value` of these Enums, while `from_dict()` converts string values back into Enum members.

```python
# In app/models/tag.py
def to_dict(self) -> Dict[str, Any]:
    return {
        "id": self.id,
        "name": self.name,
        "color": self.color.value,  # Converts TagColor Enum to string
        # ...
    }

@classmethod
def from_dict(cls, data: Dict[str, Any]) -> "Tag":
    # Converts string back to TagColor Enum, defaulting to GRAY
    color = TagColor(data["color"]) if "color" in data else TagColor.GRAY
    return cls(name=data["name"], color=color, description=data.get("description", ""))
```

### Timestamps
`Bookmark` and `Collection` models include `created_at` and `updated_at` fields. These are serialized using `.isoformat()`.

```python
# Example Bookmark.to_dict() timestamp handling
"created_at": self.created_at.isoformat(),
"updated_at": self.updated_at.isoformat(),
```

## Computed Properties and Metadata

Some models include extra information in their serialized form that isn't stored as a simple attribute:

1.  **Collection Size**: The `Collection.to_dict()` method includes a `size` key, which is a computed property returning the length of `bookmark_ids`.
2.  **Bookmark Metadata**: The `Bookmark` model includes a `metadata` attribute (a `Dict[str, Any]`) that is passed through directly to the dictionary, allowing for arbitrary key-value pairs to be stored and retrieved.

## Troubleshooting

### Missing Required Fields
`Bookmark.from_dict()` will raise a `KeyError` if the `url` or `title` keys are missing. Always validate the presence of these fields in the service layer before calling `from_dict()`.

```python
# Recommended pattern in app/services/bookmark_service.py
def create_bookmark(self, data: Dict[str, Any]):
    # 1. Validate
    if not data.get("url") or not data.get("title"):
        return None, "Missing required fields"
    
    # 2. Deserialize
    bookmark = Bookmark.from_dict(data)
    return bookmark, None
```

### ID Generation
Do not provide an `id` in the dictionary passed to `from_dict()`. All models (`Bookmark`, `Tag`, `Collection`) generate their own unique IDs using `uuid` during instantiation if an ID is not explicitly provided to the constructor. `from_dict()` does not map an `id` field from the input dictionary.