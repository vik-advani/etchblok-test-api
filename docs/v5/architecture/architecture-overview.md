---
{title: Architecture Overview, description: Architecture diagrams and documentation for vik-advani/etchblok-test-api, displayed_sidebar: architectureSidebar, section_type: architecture}
---
# Architecture Overview

This section contains architecture diagrams and documentation for **vik-advani/etchblok-test-api**.

## Available Diagrams

### [System Context for Bookmark Management API](/architecture/architecture-overview/system-context-for-bookmark-management-api)

This system context diagram illustrates the Bookmark Management API and its interactions with users and external dependencies. The API serves as a central hub for managing bookmarks, tags, and collections.

- **End Users** interact with the system through a RESTful API, performing operations such as creating, updating, and searching for bookmarks.
- The **Bookmark Management API** (built with Flask) orchestrates business logic, validation, and coordination between storage and search components.
- The **Database** (currently stubbed for PostgreSQL) provides persistent storage for domain entities like bookmarks, tags, and collections.
- The **Search Service** (implemented as an in-memory inverted index) provides full-text search capabilities, designed to be replaced by external services like Elasticsearch or Typesense in production.
- The **Cache Service** (implemented as an in-memory LRU cache) improves performance by reducing redundant database lookups for frequently accessed bookmarks.

The architecture follows a layered pattern where the service layer ([BookmarkService](/api_ref/app/services/bookmark/service/bookmarkservice)) abstracts the underlying storage and search implementations from the API routes.

**Key Architectural Findings:**
- The system is a Flask-based REST API with a layered architecture (Routes -> Services -> Repositories).
- It features a dedicated search service using an inverted index for full-text search across bookmark titles and descriptions.
- An internal LRU cache is used to optimize bookmark retrieval performance.
- The database layer is abstracted via a repository pattern, with a connection pool stubbed for PostgreSQL (port 5432).
- Configuration is managed through environment-specific classes (Development, Production, Testing) using `os.environ`.

### [Component Architecture of the Pagemark Flask API](/architecture/architecture-overview/component-architecture-of-the-pagemark-flask-api)

This component architecture diagram illustrates the layered structure of the Pagemark Flask API. 

The application follows a classic layered architecture:
1.  **API Layer**: Composed of Flask Blueprints (`app.routes`) that handle HTTP requests and responses. These routes delegate all business logic to the `BookmarkService`.
2.  **Service Layer**: The core of the application. `BookmarkService` acts as a singleton facade that orchestrates operations across the repository, search index, and cache. It also handles data validation using internal helpers.
3.  **Data Layer**: Features an in-memory `BookmarkRepository` that provides a clean abstraction for data access, isolating the service layer from storage details.
4.  **Domain Layer**: Contains the core entities (`Bookmark`, `Tag`, `Collection`) and their associated business rules and validators.

The diagram highlights the central role of the [BookmarkService](/api_ref/app/services/bookmark/service/bookmarkservice) as the primary orchestrator, and the use of specialized internal services like [SearchIndex](/api_ref/app/services/search/service/searchindex) for full-text search and [Cache Configuration Strategy](/guides/application-configuration/cache-configuration-strategy) for performance optimization. All data is currently managed in-memory via the [BookmarkRepository](/api_ref/app/db/repository/bookmarkrepository).

**Key Architectural Findings:**
- The application uses a singleton `BookmarkService` as a central facade for all business logic.
- Routes are organized into Flask Blueprints (bookmarks, tags, collections, health) that all depend on the `BookmarkService`.
- Data persistence is abstracted through a `BookmarkRepository`, which currently implements an in-memory store.
- A custom `SearchIndex` provides full-text search capabilities by indexing bookmark titles and descriptions in-memory.
- An internal `LRUCache` is used by the `BookmarkService` to speed up bookmark retrieval by ID.
- The `app.config` module provides environment-specific settings (Development, Production, Testing) used during application factory initialization.