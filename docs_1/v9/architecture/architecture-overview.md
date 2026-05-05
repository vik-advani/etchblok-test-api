---
{title: Architecture Overview, description: Architecture diagrams and documentation for etchblok-test-api, displayed_sidebar: architectureSidebar, section_type: architecture}
---
# Architecture Overview

This section contains architecture diagrams and documentation for **etchblok-test-api**.

## Available Diagrams

### [Bookmark Management System Context](/architecture/architecture-overview/bookmark-management-system-context)

The Bookmark Management System is a backend web service that provides a RESTful API for organizing web links. It allows users to create, update, and categorize bookmarks using tags and collections.

The system is built with the Flask framework and follows a layered architecture. The [Bookmark API Component Architecture](/architecture/architecture-overview/bookmark-api-component-architecture) acts as the central hub, orchestrating requests from users and delegating business logic to internal services.

Key external integrations include:
- **PostgreSQL Database**: Used for persistent storage of domain entities such as bookmarks, tags, and collections. The codebase includes a connection pool and repository layer designed for a relational database.
- **External Search Service**: Provides full-text search capabilities. While the current implementation uses an in-memory inverted index, the architecture is designed to integrate with external search engines like Typesense or Elasticsearch for production scalability.
- **End Users**: Interact with the system via standard HTTP methods (GET, POST, PUT, DELETE) to manage their bookmark library.

Internally, the system also employs an LRU (Least Recently Used) cache to improve performance for frequently accessed bookmarks, reducing the load on the underlying storage.

**Key Architectural Findings:**
- The system is a Flask-based REST API for managing bookmarks, tags, and collections.
- It uses a Repository pattern to abstract data access, with infrastructure in place for a PostgreSQL database.
- A dedicated Search Service provides full-text search, designed to be replaced by external engines like Elasticsearch in production.
- The API includes an internal LRU cache to optimize bookmark retrieval performance.
- The system is designed as a pure backend service, intended to be consumed by web, mobile, or browser extension clients.

### [Bookmark API Component Architecture](/architecture/architecture-overview/bookmark-api-component-architecture)

The Bookmark API follows a classic layered architecture for a Flask-based REST service. 

At the top level, the **Routes Layer** (blueprints) handles incoming HTTP requests and delegates all business logic to the [app.services.bookmark_service](/api_ref/app/bookmark_service). This service acts as a **Facade**, orchestrating interactions between the data storage, search indexing, and caching mechanisms.

The [app.services.bookmark_service](/api_ref/app/bookmark_service) is implemented as a singleton to ensure consistent state across different route modules. It manages:
- **Data Persistence**: Via the [app.db.repository](/api_ref/app/repository), which provides an in-memory abstraction for storing [Bookmark](/api_ref/app/models/bookmark/bookmark), tags, and collections.
- **Search**: Via the [app.services.search_service](/api_ref/app/search_service), which maintains an inverted index of bookmark content for fast full-text retrieval.
- **Performance**: Via an internal [Service Architecture and Caching](/guides/bookmark-operations/service-architecture-and-caching) (LRU implementation) that reduces repository lookups for frequently accessed bookmarks.

The **Model Layer** defines the core domain entities ([Bookmark](/api_ref/app/models/bookmark/bookmark), `Tag`, `Collection`) which are used across all layers of the application for data transfer and validation.

**Key Architectural Findings:**
- The application uses a Singleton pattern for the BookmarkService to maintain a shared state across Flask blueprints.
- The BookmarkRepository provides an in-memory storage abstraction, decoupling the service layer from the actual storage implementation.
- Search functionality is implemented as a separate SearchIndex component that rebuilds itself from the repository on startup.
- An internal LRU Cache is used within the service layer to optimize bookmark retrieval.
- The architecture strictly follows a layered approach where routes never touch the repository directly, always going through the service layer.

### [Bookmark Domain Data Model](/architecture/architecture-overview/bookmark-domain-data-model)

The data model for the Pagemark API is centered around three primary domain entities: [Bookmark](/api_ref/app/models/bookmark/bookmark), [Tag](/api_ref/app/models/tag/tag), and [Collection](/api_ref/app/models/collection/collection). 

- **Bookmark**: The core entity representing a saved URL. It contains metadata such as title, description, and status (Active, Archived, Trashed). It maintains a list of associated Tag IDs.
- **Tag**: A label used to organize bookmarks. It includes a name, color, and a usage count. While the code mentions that tag names should be unique per user, a formal `User` entity is not yet implemented in the current codebase.
- **Collection**: A grouping mechanism for bookmarks. Collections can be **manual** (where bookmarks are explicitly added) or **smart** (where bookmarks are automatically included based on a filter rule). It maintains an ordered list of Bookmark IDs.

The relationships are implemented using identifier lists (foreign keys) within the dataclasses, representing many-to-many associations between Bookmarks and Tags, and between Collections and Bookmarks.

**Key Architectural Findings:**
- The system uses Python dataclasses for domain models with in-memory storage.
- Relationships are managed through lists of IDs (e.g., `Bookmark.tags` and `Collection.bookmark_ids`) rather than direct object references.
- Enums are used for state management: `BookmarkStatus`, `TagColor`, and `CollectionType`.
- A 'User' entity is mentioned in comments and README but is not currently implemented as a class or database entity.
- Smart Collections use a `filter_rule` string to dynamically associate with bookmarks based on content matching.

### [Bookmark Lifecycle States](/architecture/architecture-overview/bookmark-lifecycle-states)

This state machine diagram illustrates the lifecycle of a [Bookmark](/api_ref/app/models/bookmark/bookmark) entity within the system. 

The lifecycle begins when a bookmark is created via the `create_bookmark` service method, which initializes it in the **Active** state. From there, the bookmark can transition between three primary states: **Active**, **Archived**, and **Trashed**.

Key transitions are triggered by specific service operations:
- **Archive**: Triggered by `archive_bookmark`, moving the bookmark to the **Archived** state from either **Active** or **Trashed**.
- **Trash**: Triggered by `delete_bookmark`, which performs a soft-delete by moving the bookmark to the **Trashed** state.
- **Restore**: Triggered by `restore_bookmark`, which returns a bookmark from **Archived** or **Trashed** back to the **Active** state.

The system uses an in-memory repository where these state changes are persisted. Notably, the current implementation allows transitions between any of these states without restrictive guard conditions, meaning a bookmark can be moved directly from **Trashed** to **Archived** or vice versa. There is currently no terminal "Deleted" state exposed via the public API, as the `delete` operation is implemented as a soft-delete (trashing).

**Key Architectural Findings:**
- Bookmarks are initialized in the 'active' status upon creation.
- The 'delete' operation is a soft-delete that transitions the bookmark to the 'trashed' status.
- The 'restore' operation transitions a bookmark from 'archived' or 'trashed' back to 'active'.
- Transitions are permissive; for example, a bookmark can move directly from 'trashed' to 'archived'.
- There is no hard-delete transition exposed in the service layer, although the repository supports it.