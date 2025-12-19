
# Spring-Mongodb-Document-Service

## Overview

This project is a **multi-microservice demonstration** of **MongoDB** with **Spring Boot 3.x** and **Spring Data MongoDB**, focusing on **document-oriented modeling** and the advantages of **flexible, schemaless design**.

It showcases how NoSQL document stores excel at handling evolving data structures, embedded sub-documents, and complex hierarchical relationships — patterns commonly used in modern applications.

## Real-World Scenario (Simulated)

In social media platforms like **Instagram**:
- User posts contain rich, variable content: images, captions, tags, locations, multiple media types.
- New features (e.g., reels, stories, collaborative posts) are added frequently → schema changes would be painful in relational DBs.
- Comments, likes, and tags are naturally hierarchical and grow unbounded.
- Queries often fetch a post with all its nested data in one go.

We simulate a simplified social feed where **posts** are stored as flexible documents with embedded/ referenced comments, tags, and media — demonstrating schema evolution without migrations.

## Microservices Involved

| Service                | Responsibility                                                                 | Port  |
|------------------------|--------------------------------------------------------------------------------|-------|
| **mongodb**            | MongoDB server + Mongo Express UI                                              | 27017 / 8081 (Express) |
| **eureka-server**      | Service discovery (Netflix Eureka)                                             | 8761  |
| **user-service**       | Manages user profiles (stored as documents)                                    | 8082  |
| **post-service**       | Core service: CRUD for posts with embedded/referenced comments and media       | 8083  |
| **comment-service**    | Optional separate service for comment operations (hybrid embedded + referenced)| 8084  |

## Tech Stack

- Spring Boot 3.x
- Spring Data MongoDB (Reactive + Imperative repositories)
- MongoDB 7.x
- Spring Cloud Netflix Eureka
- Spring Web / WebFlux (imperative style)
- Lombok
- Maven (multi-module)
- Docker & Docker Compose
- Mongo Express (web UI for DB inspection)

## Docker Containers

```yaml
services:
  mongodb:
    image: mongo:7.0
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
    volumes:
      - mongodb_data:/data/db

  mongo-express:
    image: mongo-express:latest
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: admin
      ME_CONFIG_MONGODB_ADMINPASSWORD: password
      ME_CONFIG_MONGODB_URL: mongodb://admin:password@mongodb:27017/

  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"

  user-service:
    build: ./user-service
    depends_on:
      - mongodb
      - eureka-server
    ports:
      - "8082:8082"

  post-service:
    build: ./post-service
    depends_on:
      - mongodb
      - eureka-server
    ports:
      - "8083:8083"

  comment-service:
    build: ./comment-service
    depends_on:
      - mongodb
      - eureka-server
    ports:
      - "8084:8084"

volumes:
  mongodb_data:
```

Run with: `docker-compose up --build`

## Document Modeling Strategy

| Entity          | Modeling Choice                          | Rationale                                                                 |
|-----------------|------------------------------------------|---------------------------------------------------------------------------|
| **User**        | Single document                          | Profile + preferences evolve frequently                                   |
| **Post**        | Root document with embedded media array  | Always fetched together; supports multiple images/videos                  |
| **Comments**    | Hybrid: First N embedded, rest referenced| Balances read performance vs document size growth                        |
| **Tags/Hashtags**| Embedded array of strings                | Simple, frequently queried together                                       |
| **Likes**       | Embedded array of userIds (or counter)   | Fast like checks; can evolve to separate collection if needed             |

**Schema Evolution Demo**:
- Start with basic post (caption + image)
- Add new fields (location, music, collaboration) → no migration needed

## Key Features

- Flexible document structure — add fields anytime
- Embedded documents for performance (post + media)
- Hybrid comment modeling (embedded + referenced)
- Indexes on frequently queried fields (authorId, tags, createdAt)
- Aggregation queries for feed (recent posts with comment count)
- TTL index demo for temporary stories
- Versioned documents (optional field for schema versioning)
- Mongo Express UI to inspect actual documents

## Expected Endpoints (Post Service - `http://localhost:8083`)

| Method | Endpoint                        | Description                                      |
|--------|---------------------------------|--------------------------------------------------|
| POST   | `/api/posts`                    | Create post (flexible JSON — extra fields allowed)|
| GET    | `/api/posts/{id}`               | Get full post with embedded comments/media       |
| GET    | `/api/posts/feed`               | Recent posts (pagination + sorting)              |
| POST   | `/api/posts/{id}/comments`      | Add comment (embedded if < threshold)            |
| GET    | `/api/posts/search`             | Search by tags or text                           |

### Sample Post Document
```json
{
  "_id": "post123",
  "authorId": "user456",
  "caption": "Beautiful sunset!",
  "media": [
    { "type": "image", "url": "...", "width": 1080 },
    { "type": "video", "url": "...", "duration": 15 }
  ],
  "tags": ["sunset", "travel"],
  "location": { "type": "Point", "coordinates": [ -122.0, 37.0 ] },
  "comments": [
    { "authorId": "user789", "text": "Amazing!", "createdAt": "..." }
  ],
  "likesCount": 42,
  "createdAt": "2025-12-19T10:00:00Z"
}
```

## Architecture Overview

```
Client
   ↓
Post Service → MongoDB Repository
   ↓
Document Operations
   ├── Embedded: media, initial comments
   └── Referenced: overflow comments (via comment-service)
User Service → separate user documents
   ↓
MongoDB (single instance, multiple collections)
```

**Feed Query Example** (Aggregation Pipeline):
- $match (recent)
- $lookup (author details)
- $sort + $limit

## How to Run

1. Clone repository
2. Start Docker: `docker-compose up --build`
3. Access Mongo Express: `http://localhost:8081` (admin/password)
4. Create posts via POST `/api/posts`
5. View raw documents in Mongo Express → see flexible structure
6. Add new fields to request → accepted without code changes
7. Query feed → observe embedded data returned

## Testing Flexibility

1. Create post with only caption + image
2. Create another with location, multiple media, tags → same endpoint
3. Add comment → embedded
4. Add many comments → automatically overflows to referenced
5. Search by tag → uses index
6. Change post structure (add "music") → works immediately

## Skills Demonstrated

- Document modeling best practices
- Embedded vs referenced data decisions
- Schema flexibility and evolution
- Spring Data MongoDB repositories and aggregations
- Indexing strategies for performance
- Hybrid modeling for real-world scalability
- Geo-spatial queries (location support)
- TTL indexes for expiring content

## Future Extensions

- Reactive version with MongoDB Reactive Streams
- Change streams for real-time updates
- Sharded cluster demo
- Full-text search with MongoDB Atlas
- GridFS for large media files
- Multi-document transactions (when needed)
