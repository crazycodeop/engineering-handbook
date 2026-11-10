# Amazon SQS & Asynchronous Messaging with Boto3

> This guide explains Amazon Simple Queue Service (SQS) using Boto3: Standard vs FIFO queues, the visibility timeout lifecycle, long polling, dead-letter queues, and comparing SQS with Celery/Redis.

---

# 1. What is Amazon SQS?

## Interview Question

**What is Amazon SQS, and how does it compare to running your own Celery + Redis cluster?**

---

### Answer

Amazon SQS (Simple Queue Service) is a **fully managed, highly scalable distributed message queue**.

```
Producer (Django Web App) ──► Amazon SQS Queue ──► Consumer (Background Worker)
 (Pushes JSON message)                              (Pulls and processes job)
```

### SQS vs Celery + Redis: The Architectural Trade-off

| Dimension | Celery + Redis | Amazon SQS |
|---|---|---|
| **Infrastructure Management** | You manage Redis server, memory, backups, persistence | **Zero server management** (Fully serverless AWS managed service) |
| **Scalability** | Limited by Redis RAM size (can run out of memory) | **Virtually infinite** queue depth (billions of messages) |
| **Latency** | Sub-millisecond (Extremely fast) | ~15–50ms (HTTP API network roundtrip) |
| **Operational Overhead** | High (Upgrades, memory tuning, Sentinel/Cluster) | None (Pay only per API call) |
| **Best For** | Millisecond tasks, complex workflows (chords, chains)| Cloud-native microservices, decoupling serverless pipelines |

---

# 2. Standard Queues vs FIFO Queues

## Interview Question

**What is the difference between a Standard SQS Queue and a FIFO SQS Queue? When must you use FIFO?**

---

### Comparison Matrix

| Feature | Standard Queue | FIFO Queue (First-In-First-Out) |
|---|---|---|
| **Queue Name Suffix** | `my-queue` | Must end in `.fifo` (e.g. `orders.fifo`) |
| **Message Ordering** | **Best-effort**: Messages may arrive slightly out of order | **Strictly preserved order**: First-in is first-out |
| **Delivery Guarantee** | **At-least-once**: A message may be delivered > once! | **Exactly-once processing** (Deduplication ID) |
| **Throughput** | Unlimited transactions per second | 300 to 3,000 transactions per second (batching) |
| **Cost** | Cheaper | Slightly higher cost per million requests |
| **Best For** | Sending emails, transcribing audio, resizing photos | **Financial transactions, stock trades, order processing** |

---

# 3. Message Lifecycle & The Visibility Timeout

## Interview Question

**What is the SQS Visibility Timeout? What happens if your worker takes longer than the visibility timeout to finish processing a message?**

---

### Answer

When a worker retrieves a message from SQS, SQS does **not** delete the message.

Instead, SQS hides the message from all other workers for a configurable duration called the **Visibility Timeout** (default: 30 seconds):

```
Time: 0s  ──► Worker 1 receives Message A. SQS sets Message A to INVISIBLE.
Time: 10s ──► Worker 2 calls receive_message(). SQS hides Message A!
Time: 20s ──► Worker 1 finishes job and calls delete_message(ReceiptHandle).
              (Message A is permanently removed from the queue!)
```

---

### The Disaster Scenario: Worker Timeout Exceeded

Suppose your task takes **45 seconds** to execute, but your visibility timeout is configured to **30 seconds**:

```
Time: 0s  ──► Worker 1 receives Message A (Timeout = 30s).
Time: 30s ──► Worker 1 is STILL running!
              Visibility timeout expires. SQS assumes Worker 1 crashed.
              SQS makes Message A VISIBLE again!
Time: 31s ──► Worker 2 pulls Message A and starts executing the exact same job!
Time: 45s ──► Worker 1 finishes and tries to delete Message A.
```

**Result**: Both workers processed the message concurrently, leading to duplicate database records or double-charging customers!

### How to Fix:
1. Set the queue's default visibility timeout higher than the maximum expected processing time.
2. In your worker loop, if processing takes longer, call `change_message_visibility` to extend the lease dynamically.

---

# 4. Long Polling vs Short Polling

## Interview Question

**Why should you always enable Long Polling on SQS queues? How does it save money?**

---

### Answer

- **Short Polling (`WaitTimeSeconds = 0`)**: SQS samples a subset of its distributed servers and returns immediately, even if no messages are found. If your queue is empty, your worker spins in a tight while-loop making 50 requests per second, costing you thousands of dollars in empty API calls!
- **Long Polling (`WaitTimeSeconds = 20`)**: SQS keeps the HTTP connection open for up to 20 seconds waiting for a message to arrive. If a message is published at second 4, SQS returns it instantly.

**Result**: Long polling reduces empty responses by 95% and drops AWS API costs significantly.

---

# 5. Complete Worker Implementation in Python

```python
import boto3
import json
import time

sqs = boto3.client("sqs", region_name="us-east-1")
QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/123456789012/my-orders-queue"

def send_order_message(order_id, user_id):
    """Producer: Sends message to SQS"""
    response = sqs.send_message(
        QueueUrl=QUEUE_URL,
        MessageBody=json.dumps({"order_id": order_id, "user_id": user_id}),
        MessageAttributes={
            "MessageType": {
                "DataType": "String",
                "StringValue": "OrderCreated"
            }
        }
    )
    return response["MessageId"]

def worker_consume_loop():
    """Consumer: Continuously polls and processes messages"""
    print("Worker listening for SQS messages...")
    while True:
        # Long polling with WaitTimeSeconds=20
        response = sqs.receive_message(
            QueueUrl=QUEUE_URL,
            MaxNumberOfMessages=5,
            WaitTimeSeconds=20,
            VisibilityTimeout=60,  # 60s lease
        )

        messages = response.get("Messages", [])
        if not messages:
            continue

        for msg in messages:
            receipt_handle = msg["ReceiptHandle"]
            payload = json.loads(msg["Body"])

            try:
                print(f"Processing Order #{payload['order_id']}...")
                time.sleep(2)  # Simulating business logic

                # MANDATORY: Delete message so it doesn't reappear!
                sqs.delete_message(
                    QueueUrl=QUEUE_URL,
                    ReceiptHandle=receipt_handle
                )
                print(f"Successfully processed and deleted message.")

            except Exception as e:
                print(f"Failed to process message: {e}")
                # Do NOT delete message. It will reappear after VisibilityTimeout expires!
```
