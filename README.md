System Design Projects
A collection of three High-Level Design (HLD) projects covering common backend system architecture patterns.
Projects
1. Food Delivery System
A food delivery platform design covering customer, delivery partner, and restaurant workflows. It includes core microservices for users, restaurant menus, cart and pricing, orders, payments, kitchen operations, delivery, and tracking.

Key components: PostgreSQL, MongoDB, Redis, OpenSearch, RabbitMQ, SSE / WebSocket.

2. URL Shortener
A URL shortening service that validates long URLs, generates unique short codes, stores URL mappings, redirects users to original URLs, and tracks link clicks.

Key components: URL Creation Service, Redirect Service, Redis cache, URL Mapping Database, Click Tracking / Analytics, Analytics Store.

3. Notification System
A multi-channel notification platform that ingests events, applies validation and delivery policies, routes messages by priority, and dispatches push, SMS, and email notifications. It also covers retries, dead-letter handling, and audit storage.

Key components: Notification Ingestion Gateway, Redis, priority queues, template rendering, channel workers, Retry Engine, DLQ, PostgreSQL, Cassandra / DynamoDB, PagerDuty integration.
Scope
Each design document includes requirements, component responsibilities, data/storage design, key workflows, architectural decisions, scope notes, and a conclusion. The documentation describes the components and technologies represented in each respective architecture.
