---
{title: BaseConfig, description: API Reference for app.config.BaseConfig, section_id: app_config_baseconfig, section_type: class_ref}
---
# BaseConfig

Base configuration shared across all environments.

## Attributes

| Attribute | Type | Description |
|----------|------|-------------|
| **SECRET_KEY** | `str` = change-me | Cryptographic key used for securing session cookies and signing sensitive data, defaulting to an environment variable or a fallback string. |
| **DEBUG** | `bool` = false | Flag that enables or disables detailed error messages and development-specific features within the application. |
| **TESTING** | `bool` = false | Boolean toggle that activates testing mode to modify behavior for automated test suites, such as suppressing email sending. |
| **PAGE_SIZE** | `int` = DEFAULT_PAGE_SIZE | Integer defining the number of records returned per request in paginated responses, constrained by a system-wide maximum. |

---

## Constructor

### Signature

```python
def BaseConfig() - > null
```

---



## Methods

---

#### `get_cache_config()`

```python
@classmethod
def get_cache_config() - > Dict[str, Any]
```

Return cache settings for this environment.

#### Returns

| Type | Description |
|------|-------------|
| `Dict[str, Any]` | A dictionary containing the specific cache configuration parameters for the current environment |

---