# 🔗 URL Shortener — High-Level Design

**Subject:** System Design  
**Author:** Saksham Joshi  
**System Type:** URL shortening and redirection platform  
**Core Components:** Load Balancer / API Gateway, URL Creation Service, Redirect Service, URL Mapping Database, Redis Cache, Click Tracking / Analytics, Analytics Store

A URL shortener converts a long URL into a compact short URL and redirects visitors from the short URL to the original destination. This document describes the responsibilities, data flow, storage components, and key design decisions represented in the supplied *URL Shortener* architecture diagram.

---

## 📌 1. Requirements Engineering

### Functional Requirements

- **FR1 — Submit Long URL:** Allow a user/client to submit a long URL for shortening.
- **FR2 — Validate URL:** The URL Creation Service accepts and validates the submitted long URL.
- **FR3 — Generate Short Code:** Generate a unique short code for the accepted long URL.
- **FR4 — Store URL Mapping:** Persist the relationship between the short code and original long URL, along with the creation date.
- **FR5 — Open Short URL:** Allow a user/client to open a generated short URL.
- **FR6 — Resolve Short Code:** The Redirect Service receives the short code and finds the corresponding original long URL.
- **FR7 — HTTP Redirection:** Return an HTTP 301 or 302 redirect to the original URL.
- **FR8 — Cache Frequently Used Links:** Use Redis to cache short-code-to-long-URL mappings for fast lookup.
- **FR9 — Track Clicks:** Increment the click count for each short URL when it is used.
- **FR10 — Store Analytics:** Persist click counts or click events in the Analytics Store.

### Non-Functional Requirements

- **Fast Redirection:** Frequently accessed short URLs should be resolved quickly using the Redis cache.
- **Unique Mapping:** Each generated short code must uniquely identify its stored original URL.
- **Persistent Storage:** URL mappings and analytics information must be stored persistently.
- **Scalability:** URL creation, redirection, caching, and analytics are separated into distinct components.
- **Reliable Redirection:** If a short-code mapping is not present in the cache, the Redirect Service should be able to read the mapping from the URL Mapping Database.
- **Analytics Availability:** Click-count information should be persisted in the Analytics Store.

> The supplied diagram does not specify numerical targets for latency, availability, throughput, or data retention.

---

## 🧩 2. Component Deep Dive

| Component | Responsibility | Data / Supporting Technology |
|---|---|---|
| User / Client | Submits long URLs and opens generated short URLs. | Web or application client |
| Load Balancer / API Gateway | Routes URL creation and redirection requests to the appropriate service. | Load Balancer / API Gateway |
| URL Creation Service | Accepts and validates a long URL, generates a unique short code, and saves the URL mapping. | URL Mapping Database |
| Redirect Service | Receives a short code, finds the original long URL, and returns an HTTP 301 / 302 redirect. | Redis, URL Mapping Database |
| URL Mapping Database | Stores the short code as a unique key, the original long URL, and the creation date. | Persistent database |
| Cache | Stores short-code-to-long-URL mappings to speed up frequent lookups. | Redis |
| Click Tracking / Analytics | Increments click counts for short URLs and stores click events or totals. | Analytics Store |
| Analytics Store | Persists click-count information associated with each short code. | Analytics storage |

### Component Responsibilities

**URL Creation Service**
1. Accept the long URL from the client.
2. Validate the submitted URL.
3. Generate a unique short code.
4. Save the short-code mapping in the URL Mapping Database.

**Redirect Service**
1. Receive the short code from an incoming short-URL request.
2. Look up the original long URL.
3. Use the cache or read the mapping from the database when required.
4. Return an HTTP 301 or 302 redirect to the original URL.

**Click Tracking / Analytics**
1. Receive or process a click-count increment for a short URL.
2. Associate the count with the corresponding short code.
3. Persist counts or click events in the Analytics Store.

---

## 🗄️ 3. Data Models & Storage Design

The supplied architecture identifies two persistent stores and one cache:

### 1. URL Mapping Database

The URL Mapping Database stores the relationship between a generated short code and its original long URL.

| Field | Description |
|---|---|
| `short_code` | Unique key used to identify the shortened URL |
| `original_long_url` | Original destination URL |
| `created_date` | Date on which the mapping was created |

**Illustrative SQL schema**

```sql
CREATE TABLE url_mappings (
    short_code VARCHAR(32) PRIMARY KEY,
    original_long_url TEXT NOT NULL,
    created_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

The schema is an illustrative relational representation of the fields shown in the supplied diagram; the diagram itself does not prescribe a database engine or exact SQL data types.

### 2. Redis Cache

Redis stores frequently used URL mappings for fast lookup.

| Key | Value |
|---|---|
| Short code | Original long URL |

**Illustrative key-value representation**

```text
Key:   url:<short_code>
Value: <original_long_url>
```

The diagram shows the cache being updated after URL mapping creation and being checked during redirection. It does not specify cache expiration (TTL), eviction policy, or persistence configuration.

### 3. Analytics Store

The Analytics Store persists click-count information.

| Field | Description |
|---|---|
| `short_code` | Short URL whose clicks are being counted |
| `click_count` | Total click count associated with that short code |

**Illustrative SQL schema**

```sql
CREATE TABLE url_analytics (
    short_code VARCHAR(32) PRIMARY KEY,
    click_count BIGINT NOT NULL DEFAULT 0
);
```

The diagram also mentions storing click events or totals. It does not specify whether individual events, aggregate totals, or both are stored, nor does it define event timestamps or visitor metadata.

---

## 🔄 4. Key System Workflows

### A. URL Creation Flow

1. The user/client submits a long URL.
2. The request reaches the Load Balancer / API Gateway.
3. The gateway routes the create request to the URL Creation Service.
4. The URL Creation Service accepts and validates the long URL.
5. The service generates a unique short code.
6. The short code, original long URL, and creation date are saved in the URL Mapping Database.
7. The cache is updated with the short-code-to-long-URL mapping.
8. The generated short URL can be returned to the client.

### B. Short URL Redirection Flow

1. The user/client opens a short URL.
2. The request reaches the Load Balancer / API Gateway.
3. The gateway routes the request to the Redirect Service.
4. The Redirect Service receives the short code.
5. The service looks up the short code in Redis.
6. If the mapping is present in Redis, the original long URL is obtained from the cache.
7. If the cache lookup misses, the Redirect Service reads the mapping from the URL Mapping Database.
8. The retrieved mapping can be used to update the cache.
9. The Redirect Service returns an HTTP 301 or 302 redirect to the original URL.

### C. Click Tracking and Analytics Flow

1. A short URL is opened and processed by the Redirect Service.
2. The click count for that short URL is incremented.
3. Click Tracking / Analytics handles the count update.
4. The updated count or click event is persisted in the Analytics Store.
5. The Analytics Store maintains the short-code-to-click-count relationship.

> The diagram shows click-count incrementing as part of the redirect path, but it does not specify whether analytics writes happen synchronously or asynchronously.

---

## 💡 5. Architectural Decisions

1. **Separate URL Creation and Redirection**
   - URL creation and URL redirection are handled by separate services.
   - This keeps the write-oriented creation workflow distinct from the frequently used redirect workflow.

2. **Centralized Request Routing**
   - The Load Balancer / API Gateway routes create and redirect requests.
   - This provides a common entry point for client requests.

3. **Unique Short-Code Mapping**
   - The short code is treated as the unique key in the URL Mapping Database.
   - The mapping connects the short code to the original long URL.

4. **Redis for Fast Lookup**
   - Redis stores short-code-to-long-URL mappings.
   - Frequently accessed links can be resolved through the cache instead of requiring a database read for every request.

5. **Database Fallback on Cache Miss**
   - When a mapping is not found in Redis, the Redirect Service reads the mapping from the URL Mapping Database.
   - The diagram also shows the database mapping being used to update the cache.

6. **Dedicated Analytics Storage**
   - Click tracking is separated from URL mapping persistence.
   - The Analytics Store maintains click counts associated with short codes.

7. **HTTP 301 / 302 Redirects**
   - The Redirect Service returns an HTTP redirect response to send the client to the original destination.
   - The supplied design leaves the selection between 301 and 302 to implementation requirements.

---

## ⚠️ 6. Scope and Design Notes

- The supplied diagram does not name a specific database engine for the URL Mapping Database or Analytics Store.
- Redis is explicitly identified as the cache technology.
- The diagram does not specify the short-code generation algorithm, code length, collision-handling strategy, or character set.
- The diagram does not define URL validation rules, blocked domains, or abuse-prevention policies.
- The diagram does not specify cache TTL, eviction policy, or Redis deployment topology.
- The diagram does not specify whether click tracking is synchronous or asynchronous.
- The diagram does not define analytics event fields, reporting queries, or retention policies.
- The diagram does not specify authentication, rate limits, custom aliases, link expiration, or user ownership of short URLs.

---

## ✅ 7. Conclusion

The URL Shortener design separates the main workflow into URL creation, redirection, caching, persistent URL mapping, and click analytics. The URL Creation Service validates long URLs, generates unique short codes, and stores mappings. The Redirect Service resolves short codes through Redis and falls back to the URL Mapping Database on a cache miss before returning an HTTP 301 / 302 redirect. Click Tracking / Analytics increments click counts and persists them in the Analytics Store.

This separation provides a clear foundation for a URL shortening platform. Further low-level design can define the short-code generation algorithm, database engine and indexes, cache policy, analytics write strategy, and operational safeguards.
