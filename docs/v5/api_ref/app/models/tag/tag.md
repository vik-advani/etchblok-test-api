---
{title: Tag, description: API Reference for app.models.tag.Tag, section_id: app_models_tag_tag, section_type: class_ref}
---
# Tag

A label that can be attached to one or more bookmarks.

## Attributes

| Attribute | Type | Description |
|----------|------|-------------|
| **id** | `string` = uuid.uuid4().hex[:8] | Unique identifier. |
| **name** | `string` | Display name (must be unique per user). |
| **color** | `[TagColor](tagcolor.md?sid=app_models_tag_tagcolor)` = TagColor.GRAY | Visual colour for UI rendering. |
| **description** | `string` | Optional description of what this tag represents. |
| **usage_count** | `integer` = 0 | Number of bookmarks currently using this tag. |

---

## Constructor

### Signature

```python
def Tag(
    name: str,
    color: [TagColor](tagcolor.md?sid=app_models_tag_tagcolor) = TagColor.GRAY,
    description: str = "",
    id: str = uuid.uuid4().hex[:8],
    usage_count: int = 0
)
```

### Parameters

| Name | Type | Description |
|------|------|-------------|
| **name** | `str` | Display name of the tag. |
| **color** | `[TagColor](tagcolor.md?sid=app_models_tag_tagcolor)` = TagColor.GRAY | Visual colour for UI rendering. |
| **description** | `str` = "" | Optional description of what this tag represents. |
| **id** | `str` = uuid.uuid4().hex[:8] | Unique identifier for the tag. |
| **usage_count** | `int` = 0 | Number of bookmarks currently using this tag. |

---



## Methods

---

#### `rename()`

```python
@classmethod
def rename(
    new_name: str
)
```

Rename the tag.

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| **new_name** | `str` | The new display name for the tag. |

---

#### `increment_usage()`

```python
@classmethod
def increment_usage() - > int
```

Record that a bookmark now uses this tag. Returns new count.

#### Returns

| Type | Description |
|------|-------------|
| `int` | The updated number of bookmarks currently associated with this tag. |

---

#### `decrement_usage()`

```python
@classmethod
def decrement_usage() - > int
```

Record that a bookmark removed this tag. Returns new count.

#### Returns

| Type | Description |
|------|-------------|
| `int` | The updated number of bookmarks currently associated with this tag, ensuring it does not drop below zero. |

---

#### `to_dict()`

```python
@classmethod
def to_dict() - > Dict[str, Any]
```

Serialise to a JSON-safe dictionary.

#### Returns

| Type | Description |
|------|-------------|
| `Dict[str, Any]` | A dictionary representation of the tag containing its ID, name, color, description, and usage count. |

---

#### `from_dict()`

```python
@classmethod
def from_dict(
    data: Dict[str, Any]
) - > [Tag](tag.md?sid=app_models_tag_tag)
```

Construct a Tag from a dictionary.

#### Parameters

| Name | Type | Description |
|------|------|-------------|
| **data** | `Dict[str, Any]` | A dictionary containing tag attributes such as name, color, and description. |

#### Returns

| Type | Description |
|------|-------------|
| `[Tag](tag.md?sid=app_models_tag_tag)` | A new Tag instance populated with the provided dictionary data. |

---