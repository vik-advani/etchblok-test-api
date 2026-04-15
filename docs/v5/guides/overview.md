---
section_type: guide
---
# Pagemark: A REST API for Bookmarks
Pagemark is a simple, self-contained REST API for managing bookmarks, tags, and collections. It's built with Flask and designed as a straightforward backend for personal knowledge management tools, "read-it-later" apps, or as a reference for layered application architecture in Python.
## Why Pagemark?
Pagemark is really cool. It's for things like this.
Building a backend for a bookmarking service involves a lot of repetitive work: creating CRUD endpoints, designing a data model, and implementing features like tagging and search. Instead of starting from scratch, Pagemark provides these core components out of the box.
It's a ready-to-run API that lets you focus on building your client application, whether it's a web UI, a mobile app, or a command-line tool. It also serves as a clear, practical example of a layered API design in Flask, separating concerns between routing, business logic, and data access.
## Your Digital Library: A Mental Model
Think of Pagemark as the engine for a personal digital library. The main concepts are the items you store and the ways you organize them.
- **Bookmarks are your books.** Each bookmark is a core item, representing a URL with a title, description, and other metadata. It's the "what" you are saving.
- **Tags are sticky notes.** A tag is a simple, reusable label (e.g., `python`, `career`, `cooking`) that you can attach to any bookmark. They provide a flexible, many-to-many way to categorize your content.
- **Collections are your shelves.** A collection is a curated group of bookmarks, like a folder or a playlist. You can add bookmarks to a collection manually, or create "smart" collections that automatically include bookmarks matching a search query.
## How It Works
Pagemark follows a classic layered architecture. When you send an API request, it flows through these layers:
1. **Routes:** An incoming HTTP request first hits a Flask route (e.g., `POST /api/bookmarks`), which is responsible for handling web-specific concerns.
1. **Services:** The route calls a service method, which contains the business logic. It validates input, orchestrates operations, and decides what to do (e.g., create a `Bookmark` object).
1. **Repository:** The service uses the repository to interact with the data store. This layer abstracts away the details of how data is saved, retrieved, or deleted.
1. **Models:** Plain Python classes (`Bookmark`, `Tag`, `Collection`) represent the data structures used throughout the application.
This separation makes the API easier to understand and modify. For example, you could swap the in-memory repository for a SQL database without changing the routes or services.
## Common Use Cases
Here are a few examples of how to interact with the Pagemark API using `curl`.
### Save a new bookmark
When to use: You want to save a new link.
Example:
```bash
curl -X POST http://localhost:5000/api/bookmarks \
  -H "Content-Type: application/json" \
  -d '{"url": "https://flask.palletsprojects.com/", "title": "Flask Documentation"}'
```
### List all your bookmarks
When to use: You need to display all saved links.
Example:
```bash
curl http://localhost:5000/api/bookmarks
```
### Search for a bookmark
When to use: You want to find bookmarks matching a specific keyword.
Example:
```bash
curl http://localhost:5000/api/bookmarks/search?q=flask
```
### Create a new tag
When to use: You want to create a new category to organize bookmarks with.
Example:
```bash
curl -X POST http://localhost:5000/api/tags \
  -H "Content-Type: application/json" \
  -d '{"name": "python-frameworks", "color": "blue"}'
```
## When to Use Pagemark
Pagemark is a great fit if you are:
- **Building a prototype** for a bookmarking or content-saving application.
- **Learning Flask** and want a reference project with a clean, layered architecture.
- **Creating a personal tool** where data persistence across restarts is not a primary concern.
- **Needing a simple backend** for a technical blog post or a workshop.
## When to Look Elsewhere
Pagemark is likely **not** the right tool if you need:
- **A production-ready, persistent database.** Pagemark uses an in-memory store, meaning all data is lost when the server stops.
- **Multi-user support or authentication.** The API has no concept of users, permissions, or security.
- **High-performance, advanced search.** The search is a basic, in-memory implementation suitable for small datasets.
- **A system that scales** to handle millions of bookmarks or high request volumes.
## Integrations
Pagemark is a standalone Flask service with minimal dependencies.
- **Language/Framework:** Python 3, Flask.
- **Dependencies:** `flask`, `python-dotenv`.
- **Client Compatibility:** Any client that can make HTTP requests (e.g., JavaScript frontends, mobile apps, shell scripts) can use this API.
- **Database:** None. The data store is in-memory and part of the application process.
## Getting Started
You can get the API running in two commands:
```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Run the server
python run.py
```
The API will be available at `http://localhost:5000`.
## Limitations & Assumptions
- **Data is not persistent.** The API uses an in-memory dictionary as its database. All bookmarks, tags, and collections will be lost when the application process terminates.
- **No user accounts.** The API is designed for a single-user context. There is no authentication, authorization, or data isolation.
- **Search is basic.** The full-text search is a simple implementation that works well for small datasets but lacks the power of a dedicated search engine like Elasticsearch or Meilisearch.
## Frequently Asked Questions
**1. Is the data saved to a database?**
No. Pagemark uses an in-memory data store, which means all data is cleared when the server is restarted. To make data persistent, you would need to replace the `BookmarkRepository` class with an implementation that connects to a real database.
**2. How do I handle user accounts and authentication?**
This functionality is not included. You would need to add an authentication layer yourself, for example by using a library like Flask-Login or by putting the service behind an authenticating proxy.
**3. Can I use a different database like PostgreSQL or SQLite?**
Yes. The layered architecture is designed to make this possible. You can write a new `BookmarkRepository` that uses the database of your choice and swap it in without changing the service or route layers.
**4. How does search work?**
It uses a simple in-memory inverted index that maps search tokens (words) to the bookmarks that contain them. It performs a full-text search on bookmark titles and descriptions. All search terms in a query must be present for a bookmark to match (AND logic).
**5. What are "smart" collections?**
A smart collection automatically populates itself with bookmarks that match a filter rule (e.g., a search query). This is in contrast to a "manual" collection, where you must add and remove bookmarks explicitly.
