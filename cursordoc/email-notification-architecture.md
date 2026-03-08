# Email Notification System Architecture Flow

## Overview

This document describes the architecture and flow of the email notification system for Policy Revision and Policy Deviation approval workflows.

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Approval Workflow                            │
│  (Policy Revision / Policy Deviation)                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ Approval Event Triggered
                         ▼
         ┌───────────────────────────────────────┐
         │   Approval Adapter                     │
         │   (PolicyApprovalAdapter /            │
         │    DeviationApprovalAdapter)          │
         └───────────────┬───────────────────────┘
                         │
                         │ Create/Update Approval Instance
                         ▼
         ┌───────────────────────────────────────┐
         │   Notification Service                 │
         │   - Determine Recipients              │
         │   - Generate Email Content             │
         │   - Enqueue Email                      │
         └───────────────┬───────────────────────┘
                         │
                         │ Enqueue Email
                         ▼
         ┌───────────────────────────────────────┐
         │   Email Queue Service (Redis)          │
         │   - Priority Queue                     │
         │   - Pending Queue                      │
         │   - Processing Queue (Sorted Set)      │
         │   - Failed Queue (Dead Letter)         │
         └───────────────┬───────────────────────┘
                         │
                         │ Background Worker Dequeues
                         ▼
         ┌───────────────────────────────────────┐
         │   Email Queue Processor               │
         │   (Background Task)                  │
         └───────────────┬───────────────────────┘
                         │
                         │ Send Email
                         ▼
         ┌───────────────────────────────────────┐
         │   Email Service (SMTP)                 │
         │   - Connect to SMTP Server             │
         │   - Send HTML Email                    │
         │   - Handle Errors                      │
         └───────────────┬───────────────────────┘
                         │
                         │ Success / Failure
                         ▼
         ┌───────────────────────────────────────┐
         │   Result Handling                      │
         │   - Success: Mark as Sent              │
         │   - Retryable Error: Schedule Retry     │
         │   - Permanent Error: Move to Failed     │
         └───────────────────────────────────────┘
```

## Detailed Flow Diagrams

### 1. Approval Instance Creation Flow

```
Policy Revision/Deviation Created
         │
         ▼
Create Approval Instance
         │
         ▼
Get First Approver ID
         │
         ▼
┌────────────────────────┐
│ Notification Service   │
│ notify_*_created()      │
└────────┬───────────────┘
         │
         ▼
┌────────────────────────┐
│ Email Queue Service    │
│ enqueue_email()        │
│ - priority=True        │
└────────┬───────────────┘
         │
         ▼
┌────────────────────────┐
│ Redis Priority Queue   │
│ email:queue:priority   │
└────────────────────────┘
```

### 2. Approval Decision Flow

```
Approver Makes Decision (Approve/Reject)
         │
         ▼
Process Approval Decision
         │
         ▼
Check Workflow Status
         │
         ├─── Workflow Completed ───┐
         │                           │
         │                           ▼
         │                  ┌─────────────────────┐
         │                  │ Notify Creator       │
         │                  │ (Approved/Rejected)  │
         │                  └─────────────────────┘
         │
         └─── Workflow In Progress ──┐
                                     │
                                     ▼
                          ┌─────────────────────┐
                          │ Notify Next Approver │
                          │ (Forwarded)          │
                          └─────────────────────┘
```

### 3. Email Queue Processing Flow

```
Background Task (Every 5 seconds)
         │
         ▼
Cleanup Stale Processing Emails
         │
         ▼
Dequeue Email (Priority First)
         │
         ├─── Queue Empty ────► Wait for Next Cycle
         │
         └─── Email Found ────► Move to Processing Queue
                     │
                     ▼
         ┌──────────────────────┐
         │ Email Queue Processor│
         │ process_batch()      │
         └──────────┬───────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │ Email Service         │
         │ send_email()         │
         └──────────┬───────────┘
                    │
         ┌──────────┴──────────┐
         │                     │
    Success              Error
         │                     │
         ▼                     ▼
    Mark Success      Classify Error
         │                     │
         │              ┌──────┴──────┐
         │              │             │
         │         Retryable    Permanent
         │              │             │
         │              ▼             ▼
         │        Schedule Retry   Move to Failed
         │              │             │
         │              ▼             │
         │    ┌─────────────────┐    │
         │    │ Retry Queue     │    │
         │    │ (Priority)      │    │
         │    └─────────────────┘    │
         │                            │
         └────────────────────────────┘
```

### 4. Retry Mechanism Flow

```
Email Failed (Retryable Error)
         │
         ▼
Check Retry Count
         │
         ├─── Retry Count < Max (6) ───┐
         │                               │
         │                               ▼
         │                      Calculate Delay
         │                      (Exponential Backoff)
         │                               │
         │                      ┌────────┴────────┐
         │                      │ Retry Delays:    │
         │                      │ 1min → 5min →   │
         │                      │ 15min → 1h →    │
         │                      │ 6h → 24h        │
         │                      └────────┬────────┘
         │                               │
         │                               ▼
         │                      Store Retry Info
         │                      (Redis Hash)
         │                               │
         │                               ▼
         │                      Re-enqueue to
         │                      Priority Queue
         │                               │
         │                               ▼
         │                      Wait for Retry Time
         │                               │
         │                               ▼
         │                      Process Again
         │
         └─── Retry Count >= Max ────► Move to Failed Queue
                                         (Dead Letter Queue)
```

## Component Interactions

### Component Responsibilities

1. **Approval Adapters** (`policy_approval_adapter.py`, `deviation_approval_adapter.py`)
   - Trigger notifications after approval events
   - Check Redis availability before sending

2. **Notification Service** (`notification_service.py`)
   - Business logic for determining recipients
   - Generate email content using templates
   - Enqueue emails with appropriate priority

3. **Email Queue Service** (`email_queue_service.py`)
   - Manage Redis queues (pending, priority, processing, failed)
   - Schedule retries with exponential backoff
   - Track statistics

4. **Email Queue Processor** (`email_queue_processor.py`)
   - Background worker that processes queue
   - Batch processing (configurable batch size)
   - Handle retries and failures

5. **Email Service** (`email_service.py`)
   - SMTP connection and sending
   - Error classification (retryable vs permanent)
   - HTML email support

6. **Email Templates** (`email_templates/__init__.py`)
   - HTML templates for different notification types
   - Dynamic content generation

## Data Flow

### Email Data Structure

```json
{
  "id": "uuid",
  "to_email": "user@example.com",
  "subject": "[Policy Management] New Approval Request: ...",
  "html_content": "<html>...</html>",
  "metadata": {
    "resource_type": "policy_revision",
    "resource_id": 123,
    "notification_type": "approval_pending"
  },
  "created_at": "2025-01-21T10:00:00Z",
  "retry_count": 0,
  "priority": true
}
```

### Redis Queue Structure

```
email:queue:pending          → List (normal priority emails)
email:queue:priority         → List (high priority emails, retries)
email:queue:processing       → Sorted Set (emails being processed, with timeout)
email:queue:failed           → List (dead letter queue)
email:retry:{email_id}       → Hash (retry information)
email:queue:stats            → Hash (statistics)
```

## Notification Scenarios

### Scenario 1: New Approval Request

```
1. Policy Revision/Deviation Created
2. Approval Instance Created
3. First Approver Identified
4. Notification Service → Enqueue Email (Priority)
5. Email Queue Processor → Send Email
6. Approver Receives Email
```

### Scenario 2: Approval Forwarded

```
1. Approver Approves Request
2. Next Approver Identified
3. Notification Service → Enqueue Email (Priority)
4. Email Queue Processor → Send Email
5. Next Approver Receives Email
```

### Scenario 3: Approval Completed

```
1. Last Approver Makes Decision
2. Workflow Completed
3. Notification Service → Enqueue Email (Normal Priority)
4. Email Queue Processor → Send Email
5. Creator Receives Result Email
```

### Scenario 4: Email Send Failure with Retry

```
1. Email Send Attempt Fails (Connection Error)
2. Error Classified as Retryable
3. Retry Count < Max → Schedule Retry
4. Calculate Delay (Exponential Backoff)
5. Store Retry Info in Redis
6. Re-enqueue to Priority Queue
7. Wait for Retry Time
8. Retry Send → Success
```

### Scenario 5: Email Send Failure (Permanent)

```
1. Email Send Attempt Fails (Invalid Recipient)
2. Error Classified as Permanent
3. Move to Failed Queue (Dead Letter)
4. Log Error
5. No Retry Attempted
```

## Background Task Scheduling

```
Application Startup
         │
         ▼
BackgroundTaskService.start()
         │
         ├─── Check Redis Connection
         │         │
         │         ├─── Connected ────► Initialize EmailQueueProcessor
         │         │                           │
         │         │                           ▼
         │         │                  Schedule Task (Every 5 seconds)
         │         │                           │
         │         │                           ▼
         │         │                  Process Email Queue Batch
         │         │
         │         └─── Not Connected ────► Skip Email Processing
         │
         └─── Schedule File Status Sync Task
```

## Error Handling Strategy

### Error Classification

**Retryable Errors** (will retry):
- SMTP connection errors
- Network timeouts
- Server disconnections
- 5xx server errors

**Permanent Errors** (no retry):
- Authentication failures
- Invalid recipient (4xx)
- Invalid sender
- SMTP not configured

### Retry Strategy

- **Exponential Backoff**: Delays increase exponentially
- **Max Retries**: 6 attempts
- **Retry Delays**: 1min, 5min, 15min, 1h, 6h, 24h
- **Dead Letter Queue**: Emails exceeding max retries

## Performance Considerations

1. **Asynchronous Processing**: Emails processed in background, not blocking API
2. **Batch Processing**: Process multiple emails per cycle (configurable)
3. **Priority Queue**: Urgent notifications processed first
4. **Graceful Degradation**: System continues if Redis unavailable
5. **Non-blocking**: Email failures don't affect approval workflow

## Monitoring and Statistics

The system tracks:
- Emails enqueued
- Emails sent successfully
- Emails failed
- Emails retried
- Queue lengths (pending, priority, processing, failed)

Statistics stored in Redis Hash: `email:queue:stats`

## Configuration

Key configuration parameters in `config.py`:

- `EMAIL_QUEUE_BATCH_SIZE`: Number of emails processed per batch (default: 10)
- `EMAIL_QUEUE_PROCESS_INTERVAL`: Seconds between processing cycles (default: 5.0)
- `EMAIL_MAX_RETRY_COUNT`: Maximum retry attempts (default: 6)

## Sequence Diagram

```
User Action → Approval Adapter → Notification Service → Email Queue Service
                                                              │
                                                              ▼
                                                         Redis Queue
                                                              │
                                                              ▼
Background Task ← Email Queue Processor ← Dequeue Email
      │
      ▼
Email Service → SMTP Server
      │
      ├─── Success ────► Mark Success
      │
      └─── Failure ────► Classify Error ────► Retry or Failed Queue
```

