# 🍔 Food Delivery System — High-Level Design

**Subject:** System Design  
**Author:** Saksham Joshi  
**Architecture Style:** Layered architecture with core microservices and event-driven communication  
**Primary Components:** API Gateway, RabbitMQ, Redis, OpenSearch, PostgreSQL, MongoDB, SSE / WebSocket

A high-level design for a food delivery platform connecting customers, delivery partners, and restaurant operators. The architecture is organized into client, API, core microservices, cache and search, persistent data, event and real-time, and live operations layers.

---

## 📌 1. Requirements Engineering

### Functional Requirements

- **FR1 — User and Profile Management:** Support customer, delivery partner, and restaurant partner access through the respective client applications.
- **FR2 — Restaurant and Menu Management:** Provide restaurant and menu information through the Restaurant & Menu Service.
- **FR3 — Cart and Pricing:** Manage cart information and pricing through the Cart & Pricing Service.
- **FR4 — Order Processing:** Handle order-related operations through the Order Service and make order updates available to downstream consumers.
- **FR5 — Payment Processing:** Handle payment and billing operations through the Payment & Billing Service and its persistent database.
- **FR6 — Kitchen Operations:** Support restaurant-side preparation workflows through the Kitchen & Prep Service.
- **FR7 — Delivery and Dispatch:** Coordinate delivery operations through the Delivery & Dispatch Service.
- **FR8 — Tracking and Notifications:** Provide tracking and notification capabilities, including a real-time order tracking feed and notification delivery.

### Non-Functional Requirements

- **Scalability:** Keep the major business capabilities separated into services so they can be developed and scaled independently.
- **Availability:** Use an API entry layer and separate service/data layers to isolate responsibilities.
- **Performance:** Use Redis for caching and OpenSearch for search indexing rather than relying only on persistent databases for every read.
- **Reliable Asynchronous Processing:** Use RabbitMQ as the message broker for event-driven communication.
- **Real-Time Updates:** Use an Order Update Gateway with SSE / WebSocket support for pushing order updates.
- **Observability:** Provide Analytics & Monitoring for orders, delays, and system health.

> These requirements describe the capabilities represented by the supplied architecture diagram. Exact service-level objectives, throughput targets, and retention policies are not specified in the diagram.

---

### Architecture Layers

1. **Clients Layer**
   - Customer App (Android / iOS / Web)
   - Delivery Partner App (Driver)
   - Restaurant Partner Portal (Owner / Kitchen)

2. **Global Edge and API Layer**
   - Global Load Balancer (DNS / Route 53)
   - API Gateway & Reverse Proxy
   - JWT Authentication, Rate Limiting, SSL Termination, and Request Routing

3. **Core Microservices**
   - Auth & User Service
   - Restaurant & Menu Service
   - Cart & Pricing Service
   - Order Service
   - Payment & Billing Service
   - Kitchen & Prep Service
   - Delivery & Dispatch Service
   - Tracking & Notification

4. **Cache & Search Layer**
   - Redis Cache
   - OpenSearch Search Index

5. **Data Layer — Persistent Storage**
   - User DB — PostgreSQL
   - Catalog DB — MongoDB
   - Order DB — PostgreSQL
   - Payment DB — PostgreSQL

6. **Event & Real-Time Layer**
   - RabbitMQ Message Broker
   - Order Update Gateway — SSE / WebSocket

7. **Live Operations & Consumers**
   - Real-Time Order Tracking Feed — order status and courier location
   - Notification Delivery — Push, SMS, and Email
   - Analytics & Monitoring — orders, delays, and system health

## 🧩 2. Component Deep Dive

| Component | Responsibility | Technology / Supporting Component |
|---|---|---|
| Global Load Balancer | Receives incoming client traffic and directs it toward the API layer. | DNS / Route 53 |
| API Gateway & Reverse Proxy | Central entry point for authentication, rate limiting, SSL termination, and request routing. | JWT, API Gateway |
| Auth & User Service | Handles user-facing authentication and profile-related operations. | User DB — PostgreSQL |
| Restaurant & Menu Service | Provides restaurant and menu information. | MongoDB, Redis, OpenSearch |
| Cart & Pricing Service | Manages cart and pricing operations. | Redis cache |
| Order Service | Coordinates order-related business operations and order persistence. | PostgreSQL, RabbitMQ |
| Payment & Billing Service | Handles payment and billing operations. | Payment DB — PostgreSQL, RabbitMQ |
| Kitchen & Prep Service | Supports restaurant and kitchen preparation operations. | Core microservice |
| Delivery & Dispatch Service | Coordinates delivery and dispatch operations. | Core microservice, RabbitMQ |
| Tracking & Notification | Supports tracking and notification-related operations. | RabbitMQ, real-time gateway |
| Redis Cache | Stores cache data to support faster reads. | Redis |
| OpenSearch Search Index | Supports search-oriented access to indexed restaurant/menu information. | OpenSearch |
| Order Update Gateway | Delivers order updates over persistent or server-pushed connections. | SSE / WebSocket |
| Notification Delivery | Delivers notifications through push, SMS, and email channels. | Push / SMS / Email |
| Analytics & Monitoring | Tracks orders, delays, and overall system health. | Analytics & Monitoring |

---

## 🗄️ 3. Data Layer & Storage Responsibilities

The architecture specifies four persistent databases and two supporting read-optimization components.

| Data Store | Type | Main Responsibility |
|---|---|---|
| User DB | PostgreSQL | Persistent user and profile data |
| Catalog DB | MongoDB | Restaurant and menu/catalog data |
| Order DB | PostgreSQL | Persistent order data |
| Payment DB | PostgreSQL | Persistent payment and billing data |
| Cache | Redis | Cached data used by application services |
| Search Index | OpenSearch | Search index for discovery/search operations |

**Storage design notes**

- PostgreSQL is used for user, order, and payment persistence in the supplied design.
- MongoDB is assigned to catalog data.
- Redis is shown as the cache layer.
- OpenSearch is shown as the search index.
- The diagram does not specify table structures, document fields, cache keys, index mappings, replication, or sharding strategies; those are implementation-level details to be designed separately.

---

## 🔄 4. Key System Workflows

The following flows explain the responsibilities and connections represented by the architecture. They are high-level flows; detailed API contracts and transaction semantics are not specified in the supplied diagram.

### A. Client Request Flow

1. A customer, delivery partner, or restaurant partner uses the relevant application or portal.
2. The request reaches the Global Load Balancer through the configured DNS entry.
3. The load balancer forwards the request to the API Gateway & Reverse Proxy.
4. The API Gateway applies the configured JWT authentication, rate limiting, SSL termination, and request routing responsibilities.
5. The request is routed to the relevant core microservice.
6. The service uses its assigned persistent store and, where applicable, the cache or search layer.

### B. Order and Payment Flow

1. The customer interacts with the platform through the Customer App.
2. Cart-related operations are handled by the Cart & Pricing Service.
3. Order-related operations are handled by the Order Service and persisted in the Order DB (PostgreSQL).
4. Payment-related operations are handled by the Payment & Billing Service and persisted in the Payment DB (PostgreSQL).
5. Relevant services publish messages through RabbitMQ for asynchronous downstream processing.
6. The Order Update Gateway and live consumers support order updates, notifications, and operational visibility.

### C. Kitchen and Delivery Flow

1. Restaurant-side operations enter through the Restaurant Partner Portal.
2. The Kitchen & Prep Service supports preparation-related operations.
3. The Delivery & Dispatch Service coordinates delivery operations.
4. Service events can be distributed through RabbitMQ.
5. Tracking & Notification supports the tracking and notification capabilities.
6. The Real-Time Order Tracking Feed presents order status and courier-location updates to the relevant consumer.

### D. Real-Time Update and Notification Flow

1. A relevant service produces an update.
2. RabbitMQ acts as the message broker for asynchronous event distribution.
3. The Order Update Gateway exposes order updates through SSE / WebSocket.
4. The Real-Time Order Tracking Feed consumes order status and courier-location updates.
5. Notification Delivery handles push, SMS, and email notifications.
6. Analytics & Monitoring observes orders, delays, and system health.

---

## 💡 5. Architectural Decisions

1. **Microservice Separation**
   - Business capabilities are divided into services such as authentication, catalog, cart, order, payment, kitchen, delivery, and tracking.
   - This separation gives each capability a clear responsibility.

2. **Centralized API Entry**
   - Client traffic enters through the Global Load Balancer and API Gateway.
   - Authentication, rate limiting, SSL termination, and routing are handled at the API boundary.

3. **Polyglot Persistence**
   - PostgreSQL is used for user, order, and payment data.
   - MongoDB is used for catalog data.
   - The data store is selected according to the responsibility shown in the architecture.

4. **Caching and Search Separation**
   - Redis provides a dedicated cache layer.
   - OpenSearch provides a dedicated search index.
   - This avoids making the persistent databases the only path for every read/search request.

5. **RabbitMQ for Asynchronous Communication**
   - RabbitMQ provides a message-broker layer between services and downstream consumers.
   - It supports decoupled event distribution for order updates, notifications, and monitoring workflows.

6. **SSE / WebSocket for Live Updates**
   - The Order Update Gateway supports server-pushed or bidirectional real-time communication.
   - The live tracking feed uses this layer to surface order status and courier-location updates.

7. **Operational Visibility**
   - Analytics & Monitoring is represented as a separate consumer for orders, delays, and system health.

---

## ⚠️ 6. Scope and Design Notes

- External payment providers and external notification providers are not drawn as separate architecture blocks in the supplied diagram.
- The diagram identifies the Payment & Billing Service and the Payment DB, but does not specify a payment gateway vendor or provider integration.
- The diagram identifies Push, SMS, and Email as notification channels, but does not specify the external delivery vendors.
- The exact RabbitMQ exchange, queue, retry, dead-letter, and message-ordering design is not specified.
- The exact division of responsibilities between the Tracking & Notification service and the Order Update Gateway can be refined during detailed design.
- No numerical availability, latency, throughput, or data-retention targets are specified in the supplied architecture.

---

## ✅ 7. Conclusion

The proposed Food Delivery System uses a layered architecture with dedicated microservices for customer identity, restaurant/menu discovery, cart and pricing, order processing, payment and billing, kitchen preparation, delivery dispatch, and tracking/notification. PostgreSQL and MongoDB provide persistent storage, Redis and OpenSearch support cache and search workloads, and RabbitMQ connects asynchronous workflows. The Order Update Gateway, using SSE / WebSocket, supports real-time order updates, while notification and analytics consumers provide communication and operational visibility.

The architecture provides a clear high-level foundation for a food delivery platform. Detailed API definitions, database schemas, event contracts, failure handling, and deployment topology can be specified in the low-level design.
