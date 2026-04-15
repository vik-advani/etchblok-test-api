---
{title: BaseConfig, description: API Reference for app.config.BaseConfig, section_id: app_config_baseconfig, section_type: class_ref}
---
# BaseConfig

Base configuration shared across all environments.

## Attributes

| Attribute | Type | Description |
|----------|------|-------------|
| **SECRET_KEY** | `string` = change-me | Cryptographic key used for signing session cookies and securing sensitive data, defaulting to an environment variable or a fallback string. |
| **DEBUG** | `boolean` = false | Flag that enables or disables the application's debug mode for troubleshooting during development. |
| **TESTING** | `boolean` = false | Boolean toggle that activates the testing mode to modify behavior for automated test suites. |
| **PAGE_SIZE** | `integer` = DEFAULT_PAGE_SIZE | Integer defining the number of records returned per page in paginated responses, constrained by the system's maximum page size. |

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
| `Dict[str, Any]` | A dictionary containing the environment-specific cache configuration settings. |

---