# 🔔 Notification System — System Design

**Subject:** System Design  
**Author:** Saksham Joshi  
**System Type:** Multi-channel notification delivery platform  
**Core Components:** Event Producers, Notification Ingestion Gateway, Rate Limiter, Deduplication Filter, User Preferences & Quiet Hours Filter, Priority Message Queues, Template Rendering Engine, Channel Workers, Retry Engine, Dead Letter Queue, Storage & Audit Tier

A notification system receives events from different services, applies validation and delivery policies, routes messages according to priority, renders notification templates, and dispatches notifications through push, email, and SMS channels. It also includes retry handling, dead-letter processing, operational alerting, and persistent storage for user preferences and notification history.

---

## 📌 1. Requirements Engineering

### Functional Requirements

- **FR1 — Event Ingestion:** Accept notification events from the Order Management Service, Auth Service / OTP Generator, and Marketing & Campaign Engine.
- **FR2 — API Communication:** Support gRPC / REST communication from event producers to the Notification Ingestion Gateway.
- **FR3 — Request Validation and Authorization:** Validate incoming notification requests and perform authorization at the ingestion gateway.
- **FR4 — Rate Limiting:** Apply a Token Bucket rate limiter to control the rate of incoming notification requests.
- **FR5 — Deduplication:** Identify duplicate notification requests using a deduplication filter with a Redis TTL window.
- **FR6 — User Preferences:** Apply user notification preferences before dispatching notifications.
- **FR7 — Quiet Hours:** Filter notifications according to user-configured quiet hours.
- **FR8 — Priority Classification:** Route notifications into high, medium, or low priority queues.
- **FR9 — Template Rendering:** Render notification templates with localization and variable substitution.
- **FR10 — Multi-Channel Delivery:** Dispatch notifications through push notification workers, SMS dispatch workers, and email dispatch workers.
- **FR11 — Delivery Status Storage:** Write notification status information to persistent storage.
- **FR12 — Retry Handling:** Retry failed notification deliveries using exponential backoff and jitter.
- **FR13 — Dead Letter Handling:** Move unrecoverable or poison messages to a Dead Letter Queue (DLQ) after the maximum retry limit is exceeded.
- **FR14 — Operational Alerting:** Send alerts for dead-lettered messages through the Ops Alerting / PagerDuty Integration.
- **FR15 — Audit and History:** Maintain notification history and audit information.

### Non-Functional Requirements

- **Scalability:** Separate ingestion, policy processing, priority queues, and channel workers so that different parts of the notification pipeline can scale independently.
- **Reliability:** Use retry handling and a Dead Letter Queue to manage transient failures and unrecoverable messages.
- **Traffic Control:** Apply rate limiting to protect the ingestion pipeline from excessive request volume.
- **Duplicate Prevention:** Use a deduplication filter to reduce repeated processing of duplicate notification requests within the configured Redis TTL window.
- **Priority-Aware Processing:** Keep transactional OTPs and critical alerts separate from order updates and promotional notifications.
- **Auditability:** Persist notification history and delivery status for operational review.
- **User Preference Compliance:** Apply user preferences and quiet-hours rules before notifications are placed into delivery channels.

> The supplied diagram does not specify numerical targets for throughput, latency, availability, retry counts, or retention periods.

---

## 🧩 2. Component Deep Dive

| Component | Responsibility | Technology / Supporting Component |
|---|---|---|
| Order Management Service | Produces order-related notification events. | gRPC / REST |
| Auth Service / OTP Generator | Produces authentication and OTP-related notification events. | gRPC / REST |
| Marketing & Campaign Engine | Produces marketing and campaign notification events. | gRPC / REST |
| Notification Ingestion Gateway | Receives notification requests and performs validation and authorization. | gRPC / REST API |
| Token Bucket Rate Limiter | Controls the rate of incoming notification requests. | Token Bucket |
| Deduplication Filter | Filters duplicate notification requests within a time window. | Redis TTL Window |
| Redis | Supports deduplication state and TTL-based tracking. | Redis |
| User Preferences & Quiet Hours Filter | Applies user notification preferences and quiet-hours rules. | User Notification Preferences DB |
| Priority Message Queues | Separates notifications by delivery priority. | High, Medium, Low queues |
| Template Rendering Engine | Renders notification content using localization and variable substitution. | Template engine |
| Push Notification Workers | Dispatches push notifications. | Push channel |
| SMS Dispatch Workers | Dispatches SMS notifications. | SMS channel |
| Email Dispatch Workers | Dispatches email notifications. | Email channel |
| Retry Engine | Retries failed deliveries using exponential backoff and jitter. | Retry mechanism |
| Dead Letter Queue (DLQ) | Stores unrecoverable or poison messages after retry exhaustion. | DLQ |
| Ops Alerting | Raises operational alerts for dead-lettered messages. | PagerDuty Integration |
| User Notification Preferences DB | Stores user notification preferences. | PostgreSQL |
| Notification History & Audit Log | Stores notification history and audit information. | Cassandra / DynamoDB |

### Priority Message Queues

| Priority | Message Category | Purpose |
|---|---|---|
| **High Priority** | Transactional OTPs & Critical Alerts | Handles time-sensitive and critical notifications. |
| **Medium Priority** | Order Tracking & Account Updates | Handles order progress and account-related updates. |
| **Low Priority** | Promotions & Newsletters | Handles promotional and marketing notifications. |

### Template & Channel Workers

**Template Rendering Engine**
- Localization
- Variable substitution

**Channel workers**
- Push Notification Workers
- SMS Dispatch Workers
- Email Dispatch Workers

Each worker is responsible for dispatching notifications through its corresponding channel. The diagram shows delivery-status writes from the channel-worker area to the storage tier.

---

## 🗄️ 3. Data Models & Storage Design

The supplied architecture identifies Redis for deduplication, PostgreSQL for user notification preferences, and Cassandra / DynamoDB for notification history and audit logs.

### 1. Redis — Deduplication Filter

Redis is used by the Deduplication Filter with a TTL window to identify duplicate notification requests.

| Data | Purpose |
|---|---|
| Deduplication key | Identifies a notification request for duplicate checking |
| TTL window | Limits how long the deduplication record remains active |

**Illustrative key format**

```text
notification:dedupe:<deduplication_key>
```

The diagram specifies Redis TTL-based deduplication but does not define the exact key format, TTL duration, or stored value.

### 2. User Notification Preferences DB — PostgreSQL

Stores user notification preferences used by the User Preferences & Quiet Hours Filter.

| Example Data | Purpose |
|---|---|
| User identifier | Identifies the user whose preferences are stored |
| Channel preferences | Represents the user's notification channel choices |
| Quiet-hours settings | Represents periods during which notifications should be filtered |

The fields above describe likely data represented by the component's responsibility; the supplied diagram does not define a database schema or exact column names.

### 3. Notification History & Audit Log — Cassandra / DynamoDB

Stores notification history and audit information, including delivery-status information written by channel workers.

| Example Data | Purpose |
|---|---|
| Notification identifier | Identifies a notification |
| User identifier | Associates the notification with a user |
| Channel | Identifies push, SMS, or email delivery |
| Delivery status | Records the notification's processing or delivery status |
| Event / creation time | Supports notification history and audit records |

These are illustrative fields for the history and audit responsibility. The diagram does not prescribe a table design, partition key, sort key, or exact event structure.

---

## 🔄 4. Key System Workflows

### A. Event Ingestion and Policy Processing

1. An event is generated by the Order Management Service, Auth Service / OTP Generator, or Marketing & Campaign Engine.
2. The producer sends the notification request to the Notification Ingestion Gateway using gRPC / REST.
3. The gateway performs request validation and authorization.
4. The request passes through the Token Bucket Rate Limiter.
5. The Deduplication Filter checks for duplicate requests using a Redis TTL window.
6. The User Preferences & Quiet Hours Filter applies the user's notification preferences and quiet-hours rules.
7. The notification is routed to the appropriate priority message queue.

### B. Priority-Based Notification Routing

1. The policy pipeline determines the notification's priority.
2. **High Priority:** Transactional OTPs and critical alerts are routed to the high-priority queue.
3. **Medium Priority:** Order tracking and account updates are routed to the medium-priority queue.
4. **Low Priority:** Promotions and newsletters are routed to the low-priority queue.
5. Messages from the priority queues are passed to the Template Rendering Engine.

### C. Template Rendering and Channel Dispatch

1. A queued notification reaches the Template Rendering Engine.
2. The engine performs localization and variable substitution.
3. The rendered notification is passed to the relevant channel worker.
4. Push Notification Workers handle push notifications.
5. SMS Dispatch Workers handle SMS notifications.
6. Email Dispatch Workers handle email notifications.
7. Channel workers write notification status information to the Notification History & Audit Log.

### D. Failure, Retry, and Dead Letter Flow

1. A channel worker encounters a delivery failure, represented in the diagram by a 5xx failure.
2. The failed message is directed to the Retry Engine.
3. The Retry Engine applies exponential backoff and jitter before retrying.
4. If the message is successfully processed on a retry, the delivery workflow can continue.
5. If the maximum retry limit is exceeded, the message is moved to the Dead Letter Queue (DLQ).
6. The DLQ holds unrecoverable or poison messages.
7. The DLQ triggers an alert through Ops Alerting / PagerDuty Integration.

---

## 💡 5. Architectural Decisions

1. **Separate Event Producers from Notification Processing**
   - Order Management, Authentication, and Marketing systems produce events independently.
   - The Notification Ingestion Gateway provides a common entry point for notification requests.

2. **Validation and Authorization at Ingestion**
   - The gateway validates incoming requests and checks authorization before further processing.

3. **Token Bucket Rate Limiting**
   - The Token Bucket Rate Limiter controls incoming request rates.
   - This helps protect the notification pipeline from excessive traffic.

4. **Redis-Based Deduplication**
   - The Deduplication Filter uses Redis TTL windows.
   - This helps prevent duplicate notification requests from being processed repeatedly within the configured window.

5. **User Preferences and Quiet Hours**
   - A dedicated filter applies user preferences and quiet-hours rules before priority routing.
   - User preference data is stored in PostgreSQL.

6. **Priority-Based Queues**
   - High, medium, and low priority queues separate critical transactional messages from regular updates and promotional messages.
   - This gives time-sensitive notifications a distinct processing path.

7. **Dedicated Template and Channel Workers**
   - Template rendering is separated from channel dispatch.
   - Push, SMS, and email workers handle their respective delivery channels.

8. **Retry with Exponential Backoff and Jitter**
   - Failed deliveries are routed to the Retry Engine.
   - Exponential backoff and jitter are used to space retry attempts.

9. **Dead Letter Queue for Unrecoverable Messages**
   - Messages that exceed the maximum retry limit are moved to the DLQ.
   - Ops Alerting integrates with PagerDuty to raise an alert.

10. **Separate Preference and Audit Storage**
    - PostgreSQL stores user notification preferences.
    - Cassandra / DynamoDB stores notification history and audit logs.

---

## ⚠️ 6. Scope and Design Notes

- The supplied diagram identifies gRPC / REST between event producers and the Notification Ingestion Gateway.
- Redis is explicitly shown for the deduplication filter's TTL window.
- The diagram does not name a specific message-broker technology for the priority message queues.
- The diagram does not specify the exact queue implementation, queue names, partitioning, or message-ordering guarantees.
- PostgreSQL is specified for the User Notification Preferences DB.
- Cassandra / DynamoDB is specified for the Notification History & Audit Log.
- Push, SMS, and email are represented as delivery channels, but no external provider or vendor is specified.
- The diagram shows 5xx failures being handled by the Retry Engine, but does not define retryable status codes or error classification rules.
- Exponential backoff and jitter are specified, but the maximum retry count and delay values are not.
- The diagram shows delivery-status writes to the storage tier, but does not specify the exact status lifecycle or schema.
- The diagram does not specify how quiet-hours exceptions, urgent-message overrides, or deferred delivery scheduling are handled.

---

## ✅ 7. Conclusion

The Notification System uses an event-driven pipeline to receive notification requests from order management, authentication, and marketing services. The Notification Ingestion Gateway validates and authorizes requests, while rate limiting and Redis-based deduplication help control incoming traffic. User preferences and quiet-hours rules are applied before notifications are routed into high, medium, or low priority queues.

The Template Rendering Engine prepares localized notification content, and dedicated push, SMS, and email workers handle channel delivery. Failed deliveries are processed through a retry mechanism using exponential backoff and jitter. Messages that exceed the maximum retry limit are moved to a Dead Letter Queue, with operational alerts sent through the PagerDuty integration. PostgreSQL stores user notification preferences, while Cassandra / DynamoDB stores notification history and audit logs.

This design provides a structured foundation for a multi-channel notification platform. Further low-level design can define queue semantics, message contracts, retry policies, delivery-status transitions, preference schemas, and channel-provider integrations.
