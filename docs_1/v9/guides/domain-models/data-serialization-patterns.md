---
title: Data Serialization Patterns
description: Instructions on using the to_dict and from_dict methods to handle JSON data exchange for API integration.
code_symbols: [SYM#6d5b77b3f04288620db996a9bc18b13d3bd1ad3b, SYM#a903c58b7b9829413e7dde33ff94fc7516b965f1, SYM#664bcdae74f24d832fff86d384111517f11be0db]
section_id: 2e6c783b-b338-458d-b9c0-220d4a29907c_data_serialization_patterns
doc_type: how_to
section_type: guide
---
To handle JSON data exchange for API integration in this project, use the `to_dict` and `from_dict` methods provided by the domain models. These methods serve as the bridge between internal Python objects and the external JSON-safe dictionaries used by Flask's `jsonify` and `request.get_json()`.

## Basic Integration Pattern

The standard pattern involves receiving JSON in a route, passing it to a service that uses `from_dict` to create an instance, and then returning the result of `to_dict` back to the client.

```python
from flask import request, jsonify
from app.services.bookmark_service import BookmarkService

_service = BookmarkService()

@bookmarks_bp.route("/", methods=["POST"])
def create_bookmark():
    # 1. Get raw JSON from request
    data = request.get_json(force=True)
    
    # 2. Service uses Model.from_dict(data) internally
    bookmark, error = _service.create_bookmark(data)
    
    if error:
        return jsonify({"error": error}), 400
        
    # 3. Convert model back to JSON-safe dict for response
    return jsonify(bookmark.to_dict()), 201
```

## Serializing Models to JSON (`to_dict`)

The `to_dict` method converts complex Python types (like Enums and Datetimes) into JSON-compatible primitives.

### Bookmark Serialization
The `Bookmark.to_dict` method handles ISO-8601 formatting for timestamps and extracts the string value from the `BookmarkStatus` enum.

```python
# app/models/bookmark.py

def to_dict(self) -> Dict[str, Any]:
    return {
        "id": self.id,
        "url": self.url,
        "title": self.title,
        "description": self.description,
        "tags": self.tags,
        "status": self.status.value,           # Enum to string
        "created_at": self.created_at.isoformat(), # Datetime to ISO string
        "updated_at": self.updated_at.isoformat(),
        "metadata": self.metadata,
    }
```

### Collection Serialization with Calculated Fields
The `Collection.to_dict` method includes a calculated `size` property that is not a stored attribute but is useful for the API consumer.

```python
# app/models/collection.py

def to_dict(self) -> Dict[str, Any]:
    return {
        "id": self.id,
        "name": self.name,
        "type": self.collection_type.value,
        "bookmark_ids": self.bookmark_ids,
        "filter_rule": self.filter_rule,
        "is_pinned": self.is_pinned,
        "size": self.size,                     # Calculated property
        "created_at": self.created_at.isoformat(),
    }
```

## Deserializing JSON to Models (`from_dict`)

The `from_dict` class method instantiates a model from a dictionary. It typically only maps user-provided fields, allowing the class constructor to generate system fields like `id` and `created_at`.

### Creating a Bookmark
When creating a `Bookmark`, only the core content fields are extracted from the input.

```python
# app/models/bookmark.py

@classmethod
def from_dict(cls, data: Dict[str, Any]) -> "Bookmark":
    return cls(
        url=data["url"],
        title=data["title"],
        description=data.get("description", ""),
        tags=data.get("tags", []),
    )
```

### Creating a Tag with Enum Mapping
The `Tag.from_dict` method includes logic to map a string color name back to the `TagColor` enum.

```python
# app/models/tag.py

@classmethod
def from_dict(cls, data: Dict[str, Any]) -> "Tag":
    # Map string to Enum, defaulting to GRAY if missing
    color = TagColor(data["color"]) if "color" in data else TagColor.GRAY
    return cls(
        name=data["name"], 
        color=color, 
        description=data.get("description", "")
    )
```

## Troubleshooting and Gotchas

### System Fields are Ignored in `from_dict`
The `from_dict` methods in this codebase are designed for **creation**, not full state restoration. Fields like `id`, `status`, `created_at`, and `updated_at` are ignored even if present in the input dictionary. These are generated automatically by the `dataclass` field factories:

```python
# These fields are NOT populated by from_dict
id: str = field(default_factory=lambda: uuid.uuid4().hex[:12])
created_at: datetime = field(default_factory=datetime.utcnow)
```

### Enum Validation
The `from_dict` methods for `Tag` and `Collection` pass raw strings directly into Enum constructors (e.g., `TagColor(data["color"])`). If the input string does not match a valid Enum member, a `ValueError` will be raised. Ensure data is validated before calling `from_dict` or handle the exception in the service layer.

### Missing Required Fields
`Bookmark.from_dict` accesses `data["url"]` and `data["title"]` directly. If these keys are missing from the API payload, it will raise a `KeyError`. The `BookmarkService` typically validates these fields before calling the model method:

```python
# app/services/bookmark_service.py

def create_bookmark(self, data: Dict[str, Any]):
    # Validation happens BEFORE from_dict
    error = _validate_url(data.get("url", "")) or _validate_title(data.get("title", ""))
    if error:
        return None, error

    bookmark = Bookmark.from_dict(data)
    # ...
```