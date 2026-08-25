# 📬 Scalable Notification Service

A prototype **multi-channel notification service** designed to demonstrate how a system can receive, prioritise, process, retry, and track notifications across **Email, SMS, and Push** channels.

The project is implemented as a Python-based system design prototype and demonstrates key concepts required for building a notification platform that could evolve to support millions of users and high notification volumes.

## ✨ Features

* 📧 Email notifications
* 📱 SMS notifications
* 🔔 Push notifications
* 🚦 Transactional and bulk notification prioritisation
* 🔁 Retry handling with exponential backoff
* 🛡️ Idempotency and de-duplication
* 👤 User notification preferences
* 💤 Quiet-hour configuration support
* 📊 Notification status tracking
* 📝 Delivery attempt history
* 🔌 Extensible notification channel architecture
* ⚙️ Background worker processing
* 🌐 REST API built with Flask
* 🖥️ HTML, CSS, and JavaScript dashboard
* 📈 Production scaling architecture and trade-off analysis

---

# 🎯 Project Overview

Modern applications need to send large volumes of notifications for events such as:

* Password resets
* Payment confirmations
* Security alerts
* Order updates
* Marketing campaigns
* Product announcements

These notifications have different levels of importance. A password reset or fraud alert should not be delayed behind millions of marketing messages.

This project demonstrates a notification service that separates **transactional traffic** from **bulk traffic**, allowing high-priority notifications to be processed first.

---

# 🏗️ Architecture

```text
                         ┌─────────────────┐
                         │     Clients     │
                         │ Internal Apps   │
                         └────────┬────────┘
                                  │
                                  ▼
                      ┌───────────────────────┐
                      │   Notification API    │
                      │       Flask API       │
                      └───────────┬───────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    ▼                           ▼
         ┌────────────────────┐      ┌────────────────────┐
         │ Transactional Queue │      │     Bulk Queue     │
         │    High Priority    │      │    Low Priority    │
         └──────────┬─────────┘      └──────────┬─────────┘
                    │                           │
                    └─────────────┬─────────────┘
                                  ▼
                      ┌───────────────────────┐
                      │ Notification Workers  │
                      │                       │
                      │ • Retry Logic         │
                      │ • Exponential Backoff │
                      │ • Status Updates      │
                      └───────────┬───────────┘
                                  │
                 ┌────────────────┼────────────────┐
                 ▼                ▼                ▼
          ┌────────────┐   ┌────────────┐   ┌────────────┐
          │   Email    │   │    SMS     │   │    Push    │
          │  Channel   │   │  Channel   │   │  Channel   │
          └────────────┘   └────────────┘   └────────────┘
                 │                │                │
                 └────────────────┼────────────────┘
                                  ▼
                      ┌───────────────────────┐
                      │ External Providers    │
                      └───────────────────────┘
```

---

# 🧩 Core Components

## 1. Notification API

The Flask REST API receives notification requests and validates the required fields.

### Endpoint

```http
POST /notifications
```

### Example Request

```json
{
  "user_id": "user_001",
  "channel": "push",
  "message": "Your payment was successful.",
  "notification_type": "transactional",
  "idempotency_key": "payment-001"
}
```

The API validates:

* `user_id`
* `channel`
* `message`
* `notification_type`
* `idempotency_key`

Supported channels are:

```text
email
sms
push
```

Supported notification types are:

```text
transactional
bulk
```

---

## 2. User Preferences

Before a notification is queued, the service checks whether the selected communication channel is enabled for the user.

The system stores preferences for:

* Email notifications
* SMS notifications
* Push notifications
* Quiet-hour settings

Example:

```python
create_user_preferences(
    user_id="user_001",
    email_enabled=True,
    sms_enabled=False,
    push_enabled=True
)
```

If a user has disabled a channel, the notification request is rejected before entering the processing queue.

---

# 🚦 3. Priority Queues

The service separates notifications into two queues.

| Notification Type | Priority | Purpose                                    |
| ----------------- | -------: | ------------------------------------------ |
| Transactional     |      `1` | Password resets, payments, security alerts |
| Bulk              |     `10` | Marketing campaigns and promotions         |

Lower priority values represent higher importance.

```python
TRANSACTIONAL_PRIORITY = 1
BULK_PRIORITY = 10
```

The worker always checks the transactional queue first.

```text
Transactional Queue
        │
        ▼
  Process First
        │
        ▼
    Bulk Queue
```

This prevents large marketing campaigns from delaying important notifications.

---

# 🔁 4. Retry Logic

External notification providers can fail temporarily.

The system handles temporary failures using retries and **exponential backoff**.

```text
QUEUED
   │
   ▼
Attempt 1
   │
   ├── Success ──► SENT
   │
   └── Failure
         │
         ▼
       Retry
         │
         ▼
Attempt 2
         │
         ▼
       Retry
         │
         ▼
Attempt 3
         │
         ├── Success ──► SENT
         │
         └── Failure ──► FAILED
```

The prototype uses:

```python
MAX_RETRIES = 3
```

Retry delays increase exponentially:

```python
delay = 2 ** attempt
```

Example:

```text
Attempt 1 → Retry after 2 seconds
Attempt 2 → Retry after 4 seconds
Attempt 3 → Mark as FAILED
```

---

# 🛡️ 5. Idempotency and De-duplication

Each notification request includes an `idempotency_key`.

If the same request is received more than once with the same key, the system does not create another notification.

Example:

```python
idempotency_key="payment-001"
```

The service checks whether the key already exists.

```text
Request 1
    │
    ▼
Create Notification
    │
    ▼
QUEUED

Duplicate Request
    │
    ▼
Existing Idempotency Key Found
    │
    ▼
Return Existing Notification
```

This is important because an at-least-once processing system may receive repeated requests.

---

# 📬 6. Notification Channels

All notification channels implement the same interface:

```python
class NotificationChannel:
    def send(self, notification):
        raise NotImplementedError
```

Current implementations include:

```text
EmailChannel
SMSChannel
PushChannel
```

This design makes the system extensible.

For example, a future WhatsApp channel could be added without changing the worker logic:

```python
class WhatsAppChannel(NotificationChannel):
    def send(self, notification):
        # Provider-specific logic
        pass
```

The worker only interacts with the common `send()` interface.

---

# 🗄️ 7. Data Storage

The prototype uses SQLite and stores information about:

## Notifications

```text
Notification ID
User ID
Channel
Message
Notification Type
Priority
Idempotency Key
Status
Retry Count
Created At
Updated At
```

## User Preferences

```text
User ID
Email Enabled
SMS Enabled
Push Enabled
Quiet Start
Quiet End
```

## Delivery Attempts

Each delivery attempt records:

```text
Attempt ID
Notification ID
Attempt Number
Status
Error Message
Created At
```

This makes it possible to track the complete delivery lifecycle of a notification.

---

# 🔄 Notification Status Flow

A notification can move through the following states:

```text
QUEUED
   │
   ▼
PROCESSING
   │
   ├── Success ─────────► SENT
   │
   └── Failure
         │
         ▼
       RETRY
         │
         ├── Success ───► SENT
         │
         └── Max Retries Reached
                    │
                    ▼
                  FAILED
```

The prototype records both successful and failed delivery attempts.

---

# ⚙️ 8. Worker System

The API is responsible for receiving and queueing requests.

The worker is responsible for:

* Reading notifications from queues
* Prioritising transactional traffic
* Selecting the correct notification channel
* Sending notifications
* Retrying temporary failures
* Recording delivery attempts
* Updating notification status

```python
def notification_worker():
    while True:
        try:
            if not transactional_queue.empty():
                _, notification = transactional_queue.get_nowait()
            else:
                _, notification = bulk_queue.get_nowait()

            process_notification(notification)

        except Empty:
            break
```

This separation prevents the API from waiting for external providers before responding.

---

# 🌐 9. REST API

## Create a Notification

```http
POST /notifications
```

### Example

```json
{
  "user_id": "user_001",
  "channel": "email",
  "message": "Reset your password.",
  "notification_type": "transactional",
  "idempotency_key": "password-reset-001"
}
```

### Example Response

```json
{
  "duplicate": false,
  "notification_id": "generated-notification-id",
  "status": "QUEUED"
}
```

---

## Retrieve Notifications

```http
GET /notifications
```

This endpoint returns notifications and their delivery information, including:

* Channel
* Message
* Notification type
* Status
* Retry count
* Creation time

---

# 🖥️ 10. Notification Dashboard

The project includes a simple frontend built with:

* HTML
* CSS
* JavaScript

The dashboard allows users to:

1. Enter a user ID.
2. Select Email, SMS, or Push.
3. Select Transactional or Bulk.
4. Enter a notification message.
5. Provide an idempotency key.
6. Submit the notification.
7. View notification statuses.
8. Refresh delivery information.

---

# 🧪 End-to-End Example

The prototype demonstrates both high-priority and low-priority traffic.

```python
transactional = create_notification(
    user_id="user_002",
    channel="email",
    message="Your password was changed.",
    notification_type="transactional",
    idempotency_key="security-event-001"
)

bulk = create_notification(
    user_id="user_002",
    channel="push",
    message="Check out our latest offers.",
    notification_type="bulk",
    idempotency_key="campaign-001"
)
```

When workers run:

```text
1. Transactional notification is processed first.
2. Bulk notification is processed afterwards.
3. Delivery attempts are recorded.
4. Notification status is updated.
```

---

# 📈 Production Architecture

The notebook prototype uses SQLite, in-memory queues, and local threads for simplicity.

For a production environment handling approximately **10 million notifications per day**, the architecture could evolve to:

```text
                        Load Balancer
                              │
                              ▼
                ┌──────────────────────────┐
                │ Notification API Cluster │
                └────────────┬─────────────┘
                             │
                             ▼
                     Durable Database
                             │
                             ▼
                  Kafka / RabbitMQ / SQS
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
    Transactional Workers              Bulk Workers
              │                             │
              └──────────────┬──────────────┘
                             ▼
                    Provider Adapters
                    /       |       \
                   ▼        ▼        ▼
                Email      SMS      Push
```

Worker groups can scale independently based on:

* Queue depth
* Notification volume
* Provider capacity
* Channel-specific traffic

---

# ⚖️ Key Design Trade-offs

## At-Least-Once Delivery

The system retries notifications instead of silently dropping them.

**Benefit:** Improved reliability.

**Trade-off:** A notification may be processed more than once.

This is why idempotency is important.

---

## Idempotency

A stable idempotency key prevents duplicate notification records when clients retry requests.

**Benefit:** Protects against duplicate requests.

**Trade-off:** Idempotency keys must be stored and managed.

---

## Separate Priority Queues

Transactional and bulk traffic use separate queues.

**Benefit:** Important notifications are protected from marketing traffic.

**Trade-off:** The system requires separate queue management and worker scaling strategies.

---

## Exponential Backoff

Retries use increasing delays.

**Benefit:** Reduces pressure on temporarily unavailable providers.

**Trade-off:** Failed notifications may take longer to complete.

---

## SQLite and In-Memory Queues

The prototype uses lightweight local infrastructure.

**Benefit:** Easy to demonstrate and understand.

**Trade-off:** The system is not durable or distributed.

For production, these components should be replaced with durable, replicated infrastructure.

---

# 🚀 Future Improvements

* [ ] Replace SQLite with PostgreSQL.
* [ ] Replace in-memory queues with Kafka, RabbitMQ, or SQS.
* [ ] Add real Email provider integration.
* [ ] Add real SMS provider integration.
* [ ] Add real Push notification provider integration.
* [ ] Implement provider-specific rate limiting.
* [ ] Add circuit breakers.
* [ ] Add dead-letter queues.
* [ ] Implement complete quiet-hour evaluation.
* [ ] Add delivery webhooks.
* [ ] Add API authentication and authorization.
* [ ] Add metrics and monitoring.
* [ ] Add structured logging.
* [ ] Add distributed tracing.
* [ ] Add alerting.
* [ ] Add horizontal worker scaling.
* [ ] Add automated integration tests.
* [ ] Add load and performance testing.

---

# 🛠️ Technology Stack

| Technology      | Purpose                      |
| --------------- | ---------------------------- |
| Python          | Core service implementation  |
| Flask           | REST API                     |
| SQLite          | Prototype data storage       |
| `PriorityQueue` | Notification prioritisation  |
| Threading       | Background worker processing |
| HTML            | Dashboard structure          |
| CSS             | Dashboard styling            |
| JavaScript      | Frontend API integration     |

---

# 📂 Project Structure

A suggested repository structure is:

```text
notification-service/
│
├── Notification_Service_System_Design.ipynb
├── index.html
├── README.md
│
└── requirements.txt
```

The Jupyter/Colab notebook contains the complete prototype implementation, including:

```text
1. Database setup
2. User preferences
3. Priority queues
4. Idempotency and de-duplication
5. Notification channel abstractions
6. Retry and delivery logic
7. Background workers
8. Flask REST API
9. API testing
10. Frontend dashboard
11. End-to-end demonstration
12. Production scaling design
13. Architecture trade-offs
```

---

# ▶️ Getting Started

Clone the repository:

```bash
git clone <your-repository-url>
cd notification-service
```

Install the required dependencies:

```bash
pip install flask flask-cors
```

Run the Jupyter notebook or open it in Google Colab:

```text
Notification_Service_System_Design.ipynb
```

Run the cells in order to:

1. Create the database.
2. Configure user preferences.
3. Create notification queues.
4. Send test notifications.
5. Process notifications with the worker.
6. Test the Flask API.
7. View the frontend dashboard.

---

# 💡 Conclusion

This project demonstrates the core architecture of a **scalable, multi-channel notification service**.

It supports **Email, SMS, and Push notifications**, checks **user preferences**, prevents duplicate requests through **idempotency**, records **delivery attempts**, retries temporary provider failures with **exponential backoff**, and separates **transactional and bulk traffic** to ensure important notifications receive priority.

While the prototype uses SQLite, in-memory queues, and local threads, the architecture is designed to demonstrate how these components could be replaced with **durable queues, distributed databases, provider adapters, and horizontally scalable workers** in a production environment.

## 👩‍💻 Author

**Kimberly Ruzivo Munyoro**

Aspiring **AI Engineer & Full-Stack Developer** with interests in scalable systems, software engineering, artificial intelligence, and data-driven applications.
::: 
