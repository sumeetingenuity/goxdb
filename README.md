# Go DB Usage Guide

## Introduction

Welcome to Go DB! This guide provides practical instructions and examples on how to utilize Go DB's powerful features for building modern applications, particularly those leveraging AI.

Go DB is unique because it's a **unified platform**. It combines a high-performance vector database, a robust full-text search engine, a geospatial index, and a graph engine into a single, distributed system. This eliminates the need to stitch together multiple specialized databases, simplifying your architecture and development process. Its flexible document model allows you to store rich, interconnected data, including text, files, metadata, relationships, vectors, and operational details.

This guide focuses on using the RESTful HTTP API.

## Python SDK
https://github.com/sumeetingenuity/go_db_pythonsdk.git

## Getting Started

1.  **Prerequisites:** Ensure you have Go installed and the necessary dependencies for Go DB.
2.  **Configuration:**
    *   Copy the `.env.example` file to `.env`.
    *   Review and edit the variables in `.env`, especially:
        *   `JWT_SECRET`: **Change this to a strong, secret key!**
        *   `DATA_DIR`: Where database files will be stored.
        *   `HTTP_ADDR`, `GRPC_ADDR`, `RAFT_ADDR`: Network addresses if running a cluster or changing defaults.
        *   `TLS_ENABLED`, `TLS_CERT_FILE`, `TLS_KEY_FILE`: Configure if you need HTTPS/secure gRPC.
        *   Other performance/resource settings as needed.
3.  **Run the Server:** Navigate to the `go_db` directory and run the server:
    ```bash
    go run cmd/server/main.go
    ```
    You should see log output indicating the HTTP and gRPC servers are listening.

## Authentication

Most API endpoints require authentication.

1.  **Register:** Create a user and tenant (if they don't exist).
    ```bash
    curl -X POST http://localhost:8081/auth/register \
         -H "Content-Type: application/json" \
         -d '{
               "email": "test@example.com",
               "password": "password123",
               "tenant_id": "my_tenant"
             }'
    ```
    *(Expect a `201 Created` or `409 Conflict`)*

2.  **Login:** Obtain a JWT token.
    ```bash
    curl -X POST http://localhost:8081/auth/login \
         -H "Content-Type: application/json" \
         -d '{
               "email": "test@example.com",
               "password": "password123",
               "tenant_id": "my_tenant"
             }'
    ```
    *(Expect a `200 OK` with a JSON body like `{"token": "your_jwt_token"}`)*

3.  **Use Token:** Include the obtained token in the `Authorization` header for subsequent requests. Remember to also include your `tenant_id` in the `X-Tenant-ID` header and a user identifier (like the registered email or a user ID) in the `X-User-ID` header.
    ```bash
    export AUTH_TOKEN="your_jwt_token" # Store your token
    export TENANT_ID="my_tenant"
    export USER_ID="test@example.com" # Or a specific user ID

    curl -H "Authorization: Bearer $AUTH_TOKEN" \
         -H "X-Tenant-ID: $TENANT_ID" \
         -H "X-User-ID: $USER_ID" \
         http://localhost:8081/some/endpoint
    ```

## Core Concepts: The Flexible Document

Go DB revolves around a flexible document structure. You don't need predefined schemas. Each document has a unique `id` within its tenant and can contain various fields:

-   **`id` (string, required):** Unique identifier.
-   **`text_content` (string):** Primary text for indexing and embedding.
-   **`title` (string):** Document title.
-   **`metadata` (map[string]string):** Arbitrary key-value pairs for filtering and faceting (e.g., `{"category": "news", "year": "2024", "author": "..."}`).
-   **`location` (array[float64]):** `[latitude, longitude]` for geospatial queries.
-   **Relationships:** `parent_doc_id`, `child_doc_ids`, `is_parent`, `is_child`, `relation_type`, `hierarchy_path` allow modeling hierarchies or connections.
-   **Access Control:** `visibility` (`public`, `private`, `shared`), `access_list`, `acl` (fine-grained user/role permissions).
-   **Vectors:** Embeddings are usually generated automatically during indexing based on `text_content` or other sources. You can influence this via `embedding_options`.
-   **Other Fields:** `tags`, `categories`, `version`, `lang_code`, etc. (See API documentation for the full list).

## Basic Operations

*(Remember to include `Authorization`, `X-Tenant-ID`, and `X-User-ID` headers as shown previously)*

### Indexing / Updating a Document

Use `POST /documents`. If the `id` exists, it updates; otherwise, it creates.

```bash
curl -X POST http://localhost:8081/documents \
     -H "Authorization: Bearer $AUTH_TOKEN" \
     -H "X-Tenant-ID: $TENANT_ID" \
     -H "X-User-ID: $USER_ID" \
     -H "Content-Type: application/json" \
     -d '{
           "id": "doc-001",
           "title": "Introduction to Go DB",
           "text_content": "Go DB is a unified vector, text, geo, and graph database...",
           "metadata": {
             "category": "database",
             "published_year": "2024",
             "status": "draft"
           },
           "tags": ["go", "database", "vector", "ai"],
           "visibility": "shared",
           "acl": [{"user_id": "admin@example.com", "role": "owner"}]
         }'
```
*(Expect `201 Created`)*

### Indexing Related Documents

```bash
# Index Parent
curl -X POST http://localhost:8081/documents -H ... -d '{
      "id": "parent-proj-x",
      "title": "Project X Overview",
      "text_content": "Main project document.",
      "is_parent": true,
      "relation_type": "parent",
      "metadata": {"type": "project"}
    }'

# Index Child (linking to parent)
curl -X POST http://localhost:8081/documents -H ... -d '{
      "id": "child-task-1",
      "title": "Task 1: Setup",
      "text_content": "Setup development environment.",
      "parent_doc_id": "parent-proj-x",
      "is_child": true,
      "relation_type": "child",
      "metadata": {"type": "task", "status": "pending"}
    }'
```

### Retrieving a Document

```bash
curl http://localhost:8081/documents/doc-001 \
     -H "Authorization: Bearer $AUTH_TOKEN" \
     -H "X-Tenant-ID: $TENANT_ID" \
     -H "X-User-ID: $USER_ID"
```
*(Expect `200 OK` with the document JSON)*

### Updating Visibility

Make `doc-001` private (only owners can access). Requires owner/editor permission.

```bash
curl -X PUT http://localhost:8081/documents/doc-001/visibility \
     -H "Authorization: Bearer $AUTH_TOKEN" \
     -H "X-Tenant-ID: $TENANT_ID" \
     -H "X-User-ID: $USER_ID" \
     -H "Content-Type: application/json" \
     -d '{"visibility": "private"}'
```
*(Expect `200 OK`)*

### Updating ACLs

Grant `user2@example.com` read access to `doc-001`. Requires owner permission.

```bash
curl -X PUT http://localhost:8081/documents/doc-001/acl \
     -H "Authorization: Bearer $AUTH_TOKEN" \
     -H "X-Tenant-ID: $TENANT_ID" \
     -H "X-User-ID: $USER_ID" \
     -H "Content-Type: application/json" \
     -d '{
           "operation": "add",
           "acl": {
             "user_id": "user2@example.com",
             "role": "read"
           }
         }'
```
*(Expect `200 OK`)*

### Deleting a Document

```bash
curl -X DELETE http://localhost:8081/delete/documents/doc-001 \
     -H "Authorization: Bearer $AUTH_TOKEN" \
     -H "X-Tenant-ID: $TENANT_ID" \
     -H "X-User-ID: $USER_ID"
```
*(Expect `200 OK`)*

## Search Capabilities

Go DB offers multiple ways to search your data.

### Text Search

Find documents matching keywords or phrases.

```bash
curl -X POST http://localhost:8081/search/text \
     -H "Authorization: Bearer $AUTH_TOKEN" \
     -H "X-Tenant-ID: $TENANT_ID" \
     -H "X-User-ID: $USER_ID" \
     -H "Content-Type: application/json" \
     -d '{
           "query": "unified database",
           "limit": 5
         }'
```

### Vector Search (Semantic Search)

Find documents semantically similar to a text query. Go DB automatically converts your `query` text into a vector using the configured embedding model.

```bash
curl -X POST http://localhost:8081/search/vector \
     -H "Authorization: Bearer $AUTH_TOKEN" \
     -H "X-Tenant-ID: $TENANT_ID" \
     -H "X-User-ID: $USER_ID" \
     -H "Content-Type: application/json" \
     -d '{
           "query": "databases for artificial intelligence",
           "limit": 5,
           "threshold": 0.7 // Optional: Minimum similarity score
         }'
```

### Hybrid Search

Combine text and vector search for relevance across keywords and semantics. `alpha` controls the weighting (0=text only, 1=vector only, 0.5=equal weight).

```bash
curl -X POST http://localhost:8081/search/hybrid \
     -H "Authorization: Bearer $AUTH_TOKEN" \
     -H "X-Tenant-ID: $TENANT_ID" \
     -H "X-User-ID: $USER_ID" \
     -H "Content-Type: application/json" \
     -d '{
           "query": "distributed graph database",
           "limit": 10,
           "alpha": 0.6 # Slightly favor vector search
         }'
```

### Filtering

Most search endpoints accept an optional `filters` array to narrow results based on metadata.

```bash
# Find vector results for "AI database" published in 2024 or later
curl -X POST http://localhost:8081/search/vector \
     -H "Authorization: Bearer $AUTH_TOKEN" \
     -H "X-Tenant-ID: $TENANT_ID" \
     -H "X-User-ID: $USER_ID" \
     -H "Content-Type: application/json" \
     -d '{
           "query": "AI database",
           "limit": 5,
           "filters": [
             {
               "field": "published_year",
               "operator": "gte",
               "value": "2024" // Value type should match indexed data
             },
             {
                "field": "status",
                "operator": "eq",
                "value": "published"
             }
           ]
         }'
```
The `filters` array takes objects, each specifying a condition:
```json
{
  "field": "metadata_field_name", // The field in your document's metadata
  "operator": "operator_code",    // The comparison operator
  "value": "comparison_value"     // The value to compare against
}
```

**Common Filter Operators:**

*   `eq`: Equal to `value`.
*   `neq`: Not equal to `value`.
*   `gt`: Greater than `value` (for numbers/dates).
*   `gte`: Greater than or equal to `value`.
*   `lt`: Less than `value`.
*   `lte`: Less than or equal to `value`.
*   `in`: Field's value is one of the items in the provided `value` array (e.g., `"value": ["tag1", "tag2"]`).
*   `nin`: Field's value is *not* one of the items in the provided `value` array.
*   `exists`: Checks if the `field` exists (`value` must be boolean `true` or `false`).
*   `contains`: Text `field` contains the string `value` (case-insensitive).
*   `prefix`: Text `field` starts with the string `value` (case-insensitive).
*   `suffix`: Text `field` ends with the string `value` (case-insensitive).

For a complete list of operators, specific behaviors, and advanced usage notes (like type considerations for `value` and indexing requirements for optimal performance), please refer to the detailed API documentation.

### Geo Search

Find documents within a radius of a geographic point. Documents need a `location` field `[latitude, longitude]`.

```bash
curl -X POST http://localhost:8081/search/geo \
     -H "Authorization: Bearer $AUTH_TOKEN" \
     -H "X-Tenant-ID: $TENANT_ID" \
     -H "X-User-ID: $USER_ID" \
     -H "Content-Type: application/json" \
     -d '{
           "lat": 40.7128,
           "lon": -74.0060,
           "radius": 5000, // Meters
           "limit": 10
         }'
```

*(Other search types like Fuzzy, Faceted, Autocomplete are also available - see API documentation)*

## Advanced AI Use Cases

### Retrieval-Augmented Generation (RAG)

Use Go DB to find relevant context for Large Language Models (LLMs).

1.  **Index your knowledge base:** Add documents (textbooks, articles, FAQs, code snippets) to Go DB. Use meaningful metadata (e.g., `source`, `topic`).
2.  **User Query:** Get the user's question (e.g., "How does Raft consensus work?").
3.  **Retrieve Context:** Use Hybrid Search with filters to find the most relevant documents.
    ```bash
    curl -X POST http://localhost:8081/search/hybrid \
         -H ... \
         -d '{
               "query": "How does Raft consensus work?",
               "limit": 3, # Get top 3 contexts
               "alpha": 0.7, # Favor semantic similarity
               "filters": [
                 {"field": "topic", "operator": "eq", "value": "distributed_systems"}
               ]
             }'
    ```
4.  **Augment Prompt:** Combine the user query and the retrieved document content (`text_content` or relevant chunks) into a prompt for your LLM.
5.  **Generate Response:** Send the augmented prompt to the LLM to get a context-aware answer.

### Recommendation Systems

Generate recommendations using `/search/recommendations`.

```bash
# Recommend documents similar to "doc-001" (Content-based)
curl -X POST http://localhost:8081/search/recommendations \
     -H ... \
     -d '{
           "document_id": "doc-001",
           "limit": 5,
           "strategy": "content"
         }'

# Recommend items based on "user-abc"'s interactions (Collaborative - requires interaction data)
# (Requires interaction data to be indexed or accessible by the recommendation logic)
# curl -X POST http://localhost:8081/search/recommendations -H ... -d '{ "user_id": "user-abc", "limit": 5, "strategy": "collaborative" }'

# Recommend related projects based on graph links (Graph-based)
curl -X POST http://localhost:8081/search/recommendations \
     -H ... \
     -d '{
           "document_id": "parent-proj-x",
           "limit": 3,
           "strategy": "graph",
           "graph_depth": 1 # Look one step away in the graph
         }'
```

### Knowledge Graphs

Model and query relationships between documents.

1.  **Index Related Documents:** Use `parent_doc_id`, `child_doc_ids`, `relation_type` when indexing.
2.  **Query Relationships:** Use `/search/graph`.

    ```bash
    # Find tasks (children) for "parent-proj-x" (BFS depth 1)
    curl -X POST http://localhost:8081/search/graph \
         -H ... \
         -d '{
               "start_node_id": "parent-proj-x",
               "depth": 1,
               "query_type": "bfs",
               "limit": 10,
               "filters": [
                 {"field": "type", "operator": "eq", "value": "task"}
               ]
             }'

    # Find the shortest path between two documents
    curl -X POST http://localhost:8081/search/graph \
         -H ... \
         -d '{
               "start_node_id": "doc-a",
               "end_node_id": "doc-z",
               "query_type": "shortest_path"
             }'
    ```

## API Clients

While this guide uses `curl`, you can interact with the Go DB API using any HTTP client library in your preferred language (Python's `requests`, JavaScript's `fetch`, etc.). For gRPC interaction, you would use the generated client code from the `.proto` file.

## Conclusion

Go DB offers a unique combination of search, database, and graph capabilities. Its flexible data model and comprehensive API allow you to build sophisticated AI-powered applications efficiently on a single, scalable platform. Explore the API documentation for more details on specific endpoints and parameters.
