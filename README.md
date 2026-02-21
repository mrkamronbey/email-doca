Bobur:
# EMS API Documentation

Last Updated: February 2026

## Overview
This document contains the complete HTTP API reference for the Email Management System (EMS). All endpoints are under api_router (see `app/api/router.py`). The API manages email accounts, messages, threads, inboxes, attachments, and real-time SSE events with multi-company and multi-user support.

## Authentication & Authorization

### Bearer Token Authentication
Most endpoints require a JWT Bearer token in the Authorization header:

Authorization: Bearer <jwt_token>


The token decodes to CurrentUser:

{
  "user_id": 123,
  "role": "admin",
  "companies": [1, 2, 5]
}


Authorization Rules:
- Users can only access data from companies in their companies list.
- Account access is verified by checking if account.company_id is in user's companies.
- Similarly for messages, threads, and inbox entries (company-level filtering).

### SSE (Server-Sent Events) Authentication
GET /sse/events uses query parameter authentication:

GET /sse/events?token=<jwt_token>


### Webhook (No Auth)
- POST /webhooks/nylas verifies authenticity using the x-nylas-signature header and the verify_nylas_webhook() utility.
- Challenge request GET /webhooks/nylas?challenge=<str> has no auth.

---

## HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | Success; request completed |
| 201 | Resource created |
| 204 | No content; successful deletion |
| 400 | Bad request; invalid payload or data conflict |
| 401 | Unauthorized; missing/invalid token or webhook signature |
| 403 | Forbidden; user lacks access to the company/resource |
| 404 | Not found; resource does not exist |
| 500 | Server error |

---

## Response Format

Paginated responses follow this structure:

{
  "items": [ /* array of resource objects */ ],
  "total": 100,
  "page": 1,
  "size": 25
}


Non-paginated list responses are arrays:

[/* array of resource objects */]


Single resource responses are objects directly:

{ /* resource object */ }


---

## Endpoints Summary

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | /accounts/ | Create email account |
| GET | /accounts/ | List accounts |
| GET | /accounts/{account_id} | Get account details |
| PATCH | /accounts/{account_id}/ | Update account |
| DELETE | /accounts/{account_id} | Disconnect account |
| GET | /messages/ | List messages (paginated) |
| GET | /messages/{message_id} | Get message details |
| POST | /messages/send/ | Send email |
| POST | /messages/{message_id}/reply | Reply to message |
| GET | /inbox/ | List inbox items (paginated) |
| GET | /inbox/unread-counts | Get unread counts per account |
| PATCH | /inbox/{inbox_id}/read | Mark entry read |
| PATCH | /inbox/{inbox_id}/unread | Mark entry unread |
| PATCH | /inbox/{inbox_id}/star | Toggle star |
| PATCH | /inbox/{inbox_id}/archive | Archive entry |
| PATCH | /inbox/threads/{thread_id}/read | Mark thread read |
| GET | /threads/ | List threads (paginated) |
| GET | /threads/{thread_id} | Get thread with messages |
| PATCH | /threads/{thread_id} | Update thread |
| GET | /attachments/{attachment_id}/download | Get download URL |
| GET | /attachments/message/{message_id} | List attachments for message |
| GET | /webhooks/nylas?challenge=X | Webhook challenge (verify) |
| POST | /webhooks/nylas | Receive Nylas events |
| GET | /sse/events?token=X | SSE stream for real-time updates |

---

## Detailed Endpoint Reference

### Accounts
Email accounts represent individual Gmail/GoDaddy/etc. mailboxes connected to the system.

#### 1. Create Email Account
Endpoint: POST /accounts/  
Auth: Bearer JWT (required)  
Status Code: 201 Created

Request Body:

{
  "company_id": 123,
  "email_address": "support@acme.com",
  "display_name": "Support Team",
  "provider": "godaddy",
  "grant_id": "grant_abc123xyz"
}


Request Schema Details:
- company_id *(int, required)*: Company ID for access control.
- email_address *(string, email format, required)*: Unique email address. Will reject if email or grant_id already exists.
- display_name *(string, optional)*: Display name shown in sent emails.
- provider *(string, default: "godaddy")*: Email provider type (e.g., "godaddy", "gmail").
- grant_id *(string, optional)*: Nylas grant ID for the account.

Response Body:

{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "company_id": 123,
  "email_address": "support@acme.com",
  "display_name": "Support Team",
  "provider": "godaddy",
  "status": "syncing",
  "last_sync_at": null,
  "created_at": "2026-02-21T10:30:00Z",
  "company": {
    "id": 123,
    "name": "Acme Corporation"
  }
}


Response Schema Details:
- id *(UUID)*: Unique account identifier.
- status *(string)*: "active" | "syncing" | "disconnected".
- last_sync_at *(datetime, nullable)*: Timestamp of last sync from Nylas.
- created_at *(datetime)*: Creation timestamp (UTC).
- company *(object, nullable)*: Company info object.

Error Cases:
- 400: Email address or grant_id already exists.
- 403: User does not have access to the specified company_id.

---

#### 2. List Email Accounts
Endpoint: GET /accounts/  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Response Body: Array of EmailAccountResponse objects

Details:
- Returns all accounts for user's companies.
- No pagination on this endpoint (all accounts returned).

---

#### 3. Get Account Details
Endpoint: GET /accounts/{account_id}  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Path Parameters:
- account_id *(UUID)*: Account identifier.

Response Body: EmailAccountResponse object

Error Cases:
- 404: Account not found or user lacks access.

---

#### 4. Update Account
Endpoint: PATCH /accounts/{account_id}/  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Path Parameters:
- account_id *(UUID)*: Account identifier.

Request Body (partial update):

{
  "display_name": "Main Support",
  "status": "active"
}


Request Schema Details:
- display_name *(string, optional)*: Update display name.
- status *(string, optional)*: Update account status.

Response Body: Updated EmailAccountResponse

---

#### 5. Disconnect Account
Endpoint: DELETE /accounts/{account_id}  
Auth: Bearer JWT (required)  
Status Code: 204 No Content

Path Parameters:
- account_id *(UUID)*: Account identifier.

Details:
- Disconnects account from Nylas.
- Sets account status to "disconnected".
- All associated messages/threads remain in database.

---

### Messages
Messages represent individual emails (inbound, outbound, sent).

#### 1. List Messages (Paginated)
Endpoint: GET /messages/  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Query Parameters:
- page *(int, default: 1, min: 1)*: Page number.
- size *(int, default: 25)*: Items per page.
- account_id *(UUID, optional)*: Filter by email account.

Response Body:

{
  "items": [ /* array of MessageResponse */ ],
  "total": 542,
  "page": 1,
  "size": 25
}


Details:
- Ordered by sent_at descending (newest first).
- Implicit filter: Only messages from accounts in user's companies.
- If account_id omitted, shows messages from all user's accounts.

---

#### 2. Get Message Details
Endpoint: GET /messages/{message_id}  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Path Parameters:
- message_id *(UUID)*: Message identifier.

Response Body: MessageResponse object (with attachments loaded)

Error Cases:
- 404: Message not found or user lacks access.

---

#### 3. Send Email
Endpoint: POST /messages/send/  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Request Body:

{
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "to": [
    {
      "name": "John Customer",
      "email": "john@customer.com"
    }
  ],
  "cc": [
    {
      "name": "CC Person",
      "email": "cc@customer.com"
    }
  ],
  "bcc": null,
  "subject": "Re: Your Question",
  "body_html": "<p>Thanks for reaching out!</p>",
  "body_text": "Thanks for reaching out!",
  "reply_to_message_id": null
}


Request Schema Details:
- email_account_id *(UUID, required)*: Account to send from.
- to *(array[EmailRecipient], required)*: Recipients (each has optional name and required `email`).
- cc *(array[EmailRecipient], optional)*: CC recipients.
- bcc *(array[EmailRecipient], optional)*: BCC recipients.
- subject *(string, required)*: Email subject.
- body_html *(string, required)*: HTML email body.
- body_text *(string, optional)*: Plain text body.
- reply_to_message_id *(UUID, optional)*: Message ID being replied to.

Response Body: MessageResponse object (newly sent message)

Details:
- Sends via Nylas API using account's grant ID.
- Creates/updates thread.
- Message distributed to all company users' inboxes.
- Publishes to Redis stream for real-time SSE updates.

Error Cases:
- 403: User lacks access to the email account's company.

---

#### 4. Reply to Message
Endpoint: POST /messages/{message_id}/reply  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Path Parameters:
- message_id *(UUID)*: Original message ID.

Request Body: Same format as "Send Email"

Response Body: MessageResponse object (new reply message)

Details:
- Finds original message and associated thread.
- Creates new message in the same thread.

Error Cases:
- 404: Original message not found or user lacks access.

---

### Inbox
Inbox entries track per-user read, starred, and archived status.

#### 1. List Inbox Items (Paginated)
Endpoint: GET /inbox/  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Query Parameters:
- page *(int, default: 1, min: 1)*: Page number.
- size *(int, default: 25, max: 100)*: Items per page.
- is_read *(boolean, optional)*: Filter by read status. If omitted, returns both read and unread.
- account_id *(UUID, optional)*: Filter by email account.

Response Body:

{
  "items": [ /* array of InboxItemResponse */ ],
  "total": 18,
  "page": 1,
  "size": 25
}


Response Schema Details (InboxItemResponse):

{
  "id": "990e8400-e29b-41d4-a716-446655440000",
  "is_read": false,
  "is_starred": true,
  "is_archived": false,
  "read_at": null,
  "created_at": "2026-02-21T09:16:00Z",
  "message": {
    "id": "770e8400-e29b-41d4-a716-446655440000",
    "thread_id": "880e8400-e29b-41d4-a716-446655440000",
    "from_address": "customer@external.com",
    "from_name": "John Customer",
    "subject": "Question about pricing",
    "snippet": "Hi, I have a question...",
    "direction": "inbound",
    "has_attachments": false,
    "sent_at": "2026-02-21T09:15:00Z"
  },
  "thread_subject": "Question about pricing",
  "thread_message_count": 2,
  "account_email": "support@acme.com"
}


Details:
- Ordered by message sent_at descending (newest first).
- Excludes archived items by default (filter: `is_archived == False`).
- is_read=true shows only unread; is_read=false shows only read.

Filters:
- is_read: Unread/read status.
- account_id: Specific account.
- Implicit: Only user's inbox entries from their companies.

---

#### 2. Get Unread Counts Per Account
Endpoint: GET /inbox/unread-counts  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Query Parameters: None

Response Body:

[
  {
    "account_id": "550e8400-e29b-41d4-a716-446655440000",
    "email_address": "support@acme.com",
    "unread_count": 5
  },
  {
    "account_id": "660e8400-e29b-41d4-a716-446655440001",
    "email_address": "sales@acme.com",
    "unread_count": 12
  }
]


Details:
- Returns unread message counts grouped by account.
- Useful for badge notifications in UI.

---

#### 3. Mark Inbox Entry as Read
Endpoint: PATCH /inbox/{inbox_id}/read  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Path Parameters:
- inbox_id *(UUID)*: Inbox entry ID.

Request Body: Empty (`{}`)

Response Body: Updated InboxEntryResponse

Details:
- Sets is_read to true.
- Sets read_at to current UTC timestamp.

---

#### 4. Mark Inbox Entry as Unread
Endpoint: PATCH /inbox/{inbox_id}/unread  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Path Parameters:
- inbox_id *(UUID)*: Inbox entry ID.

Request Body: Empty (`{}`)

Response Body: Updated InboxEntryResponse with is_read = false, read_at = null

---

#### 5. Toggle Star on Inbox Entry
Endpoint: PATCH /inbox/{inbox_id}/star  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Path Parameters:
- inbox_id *(UUID)*: Inbox entry ID.

Request Body: Empty (`{}`)

Response Body: Updated InboxEntryResponse with is_starred toggled

---

#### 6. Archive Inbox Entry
Endpoint: PATCH /inbox/{inbox_id}/archive  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Path Parameters:
- inbox_id *(UUID)*: Inbox entry ID.

Request Body: Empty (`{}`)

Response Body: Updated InboxEntryResponse with is_archived = true

Details:
- Archived entries excluded from default inbox list view.

---

#### 7. Mark Entire Thread as Read
Endpoint: PATCH /inbox/threads/{thread_id}/read  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Path Parameters:
- thread_id *(UUID)*: Thread ID.

Request Body: Empty (`{}`)

Response Body:

{
  "updated": 3
}


Details:
- Marks all inbox entries for the user in the thread as read.
- Returns count of updated entries.

---

### Threads
Threads group related messages by conversation.

#### 1. List Threads (Paginated)
Endpoint: GET /threads/  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Query Parameters:
- account_id *(UUID, optional)*: Filter by email account.
- page *(int, default: 1, min: 1)*: Page number.
- size *(int, default: 25, max: 100)*: Items per page.

Response Body:

{
  "items": [
    {
      "id": "880e8400-e29b-41d4-a716-446655440000",
      "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
      "subject": "Question about pricing",
      "last_message_at": "2026-02-21T10:00:00Z",
      "message_count": 3,
      "do_not_delete": false,
      "created_at": "2026-02-21T08:00:00Z"
    }
  ],
  "total": 42,
  "page": 1,
  "size": 25
}


Details:
- Ordered by last_message_at descending (most recent first).
- Implicit filter: Only threads from accounts in user's companies.

---

#### 2. Get Thread with Messages
Endpoint: GET /threads/{thread_id}  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Path Parameters:
- thread_id *(UUID)*: Thread ID.

Response Body: ThreadDetailResponse (includes messages array)

Example:

{
  "id": "880e8400-e29b-41d4-a716-446655440000",
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "subject": "Question about pricing",
  "last_message_at": "2026-02-21T10:00:00Z",
  "message_count": 3,
  "do_not_delete": false,
  "created_at": "2026-02-21T08:00:00Z",
  "messages": [
    /* array of MessageResponse objects, sorted chronologically */
  ]
}


Details:
- Messages sorted chronologically (oldest first).
- Each message includes its attachments.

Error Cases:
- 404: Thread not found or user lacks access.

---

#### 3. Update Thread
Endpoint: PATCH /threads/{thread_id}  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Path Parameters:
- thread_id *(UUID)*: Thread ID.

Request Body (partial update):

{
  "do_not_delete": true
}


Request Schema Details:
- do_not_delete *(boolean, optional)*: Mark thread as important/protected.

Response Body: Updated ThreadDetailResponse

---

### Attachments
Attachments are files included in messages.

#### 1. Get Attachment Download URL
Endpoint: GET /attachments/{attachment_id}/download  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Path Parameters:
- attachment_id *(UUID)*: Attachment ID.

Response Body:

{
  "download_url": "https://s3.amazonaws.com/bucket/attachments/path/to/file.pdf?AWSAccessKeyId=...&Signature=...&Expires=..."
}


Details:
- Returns presigned AWS S3 URL (valid ~1 hour).
- URL includes authentication; can be shared for direct download.

---

#### 2. List Attachments for Message
Endpoint: GET /attachments/message/{message_id}  
Auth: Bearer JWT (required)  
Status Code: 200 OK

Path Parameters:
- message_id *(UUID)*: Message ID.

Response Body: Array of AttachmentResponse objects

Example:

[
  {
    "id": "aa0e8400-e29b-41d4-a716-446655440000",
    "message_id": "770e8400-e29b-41d4-a716-446655440000",
    "filename": "invoice.pdf",
    "content_type": "application/pdf",
    "size_bytes": 123456,
    "is_inline": false,
    "created_at": "2026-02-21T08:16:00Z"
  }
]


Response Schema Details:
- is_inline: true = embedded in email body, false = regular attachment.
- content_type: MIME type of the file.
- size_bytes: File size in bytes.

---

### Webhooks
Receive real-time events from Nylas (email provider).

#### 1. Webhook Challenge (Verification)
Endpoint: GET /webhooks/nylas  
Auth: None  
Status Code: 200 OK

Query Parameters:
- challenge *(string, required)*: Challenge string from Nylas.

Response Body: Plain text (media type: `text/plain`)

<challenge-value>


Details:
- Nylas calls this during webhook registration to verify ownership.
- Simply echo the challenge back as plain text.

---

#### 2. Receive Webhook Events
Endpoint: POST /webhooks/nylas  
Auth: Signature verification (header: `x-nylas-signature`)  
Status Code: 200 OK

Request Header:

x-nylas-signature: <computed-signature>


Request Body (example for message.created):

{
  "specversion": "1.0",
  "type": "message.created",
  "source": "nylas",
  "id": "webhook-delivery-id-abc123",
  "time": 1708937400,
  "data": {
    "object": {
      "id": "nylas-message-id-xyz789",
      "grant_id": "grant_abc123xyz",
      "thread_id": "nylas-thread-id",
      "subject": "New inquiry",
      "from": [{"email": "customer@external.com", "name": "John Customer"}],
      "to": [{"email": "support@acme.com", "name": "Support"}],
      "cc": null,
      "bcc": null,
      "date": 1708937400,
      "body": "Hello, I have a question...",
      "snippet": "Hello, I have a question...",
      "attachments": [],
      "folders": ["inbox"]
    }
  }
}


Response Body:

{
  "status": "ok"
}


Details:
- Signature verified using verify_nylas_webhook() utility.
- Payload processed by process_webhook_event().
- Event types: "message.created", "message.updated", "grant.expired", etc.

Error Cases:
- 401: Invalid or missing signature.

---

### SSE (Server-Sent Events)
Real-time event stream for the connected user.

#### 1. Event Stream (Streaming)
Endpoint: GET /sse/events  
Auth: Query parameter JWT (`?token=<jwt>`)  
Status Code: 200 OK (streaming)  
Media Type: text/event-stream

Query Parameters:
- token *(string, required)*: JWT Bearer token.

Response (Streaming Format - Server-Sent Events):
`
event: connected
data: {"message":"connected","user_id":123}

event: message.created
data: {"type":"message.created","thread_id":"880e8400-e29b-41d4-a716-446655440000","message_id":"770e8400-e29b-41d4-a716-446655440000","company_id":123}

event: message.created
data: {"type":"message.created","thread_id":"880e8400-e29b-41d4-a716-446655440001","message_id":"770e8400-e29b-41d4-a716-446655440001","company_id":123}

:



**Event Schema Details:**

- **Connection Event:**
  
  event: connected
  data: {"message":"connected","user_id":<user_id>}
  
  - Sent once when SSE stream establishes.

- **Data Events:**
  
  event: <type>
  data: {"type":"<type>","thread_id":"<uuid>","message_id":"<uuid>","company_id":<int>}
  
  - Event types: "message.created", "message.updated", "thread.created", "thread.updated", etc.
  - Sent for each email/thread event relevant to user's companies.

- **Keepalive (Heartbeat):**
  
  :
  
  - Sent periodically (every 15 seconds of inactivity) to keep connection alive.

**Details:**
- SSE is persistent; client connects and receives events in real time.
- Events come from Redis stream.
- Server-side filtering: Only events for user's `companies` list.
- Connection closes on token expiry or client disconnect.

---

## Schemas Reference

### CompanyInfo
```json
{
  "id": 123,
  "name": "Acme Corporation"
}


### EmailRecipient

{
  "name": "John Doe",
  "email": "john@example.com"
}

- name is optional; email is required and must be valid email format.

### MessageResponse

{
  "id": "770e8400-e29b-41d4-a716-446655440000",
  "thread_id": "880e8400-e29b-41d4-a716-446655440000",
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "internet_message_id": "<msg.1234567890@acme.com>",
  "from_address": "sender@example.com",
  "from_name": "Sender Name",
  "to_addresses": [{"name": "Recipient", "email": "to@example.com"}],
  "cc_addresses": null,
  "bcc_addresses": null,
  "subject": "Subject",
  "body_html": "<p>Email body</p>",
  "body_text": "Email body",
  "snippet": "Email body...",
  "direction": "inbound",
  "has_attachments": false,
  "sent_at": "2026-02-21T09:15:00Z",
  "created_at": "2026-02-21T09:16:00Z",
  "attachments": []
}


### AttachmentResponse

{
  "id": "aa0e8400-e29b-41d4-a716-446655440000",
  "message_id": "770e8400-e29b-41d4-a716-446655440000",
  "filename": "invoice.pdf",
  "content_type": "application/pdf",
  "size_bytes": 123456,
  "is_inline": false,
  "created_at": "2026-02-21T08:16:00Z"
}


### InboxEntryResponse

{
  "id": "990e8400-e29b-41d4-a716-446655440000",
  "user_id": 123,
  "message_id": "770e8400-e29b-41d4-a716-446655440000",
  "thread_id": "880e8400-e29b-41d4-a716-446655440000",
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "is_read": true,
  "is_starred": true,
  "is_archived": false,
  "read_at": "2026-02-21T11:00:00Z",
  "created_at": "2026-02-21T09:16:00Z"
}


### ThreadDetailResponse

{
  "id": "880e8400-e29b-41d4-a716-446655440000",
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "subject": "Question about pricing",
  "last_message_at": "2026-02-21T10:00:00Z",
  "message_count": 3,
  "do_not_delete": false,
  "created_at": "2026-02-21T08:00:00Z",
  "messages": [ /* array of MessageResponse */ ]
}


---

## Error Handling

All error responses follow HTTP standard error codes with detail messages:

400 Bad Request:

{
  "detail": "Email account with this email address/grant already exists."
}


401 Unauthorized:

{
  "detail": "Invalid or expired token"
}


403 Forbidden:

{
  "detail": "No access to this email account"
}


404 Not Found:

{
  "detail": "Message not found"
}


---

## Pagination Guide

All paginated endpoints follow this pattern:

Offset Calculation:

offset = (page - 1) * size


Example: Get page 2 with 25 items per page:

/messages/?page=2&size=25
# offset = (2 - 1) * 25 = 25 (skip first 25, get next 25)


Max Page Size: Most endpoints default to size: 25 with a max of 100.

Total Count: Always included in response:

{
  "total": 542,
  "page": 1,
  "size": 25
  // max_pages = ceil(542 / 25) = 22
}


---

## Quick Examples

### Send Email

curl -X POST https://api.example.com/messages/send/ \
  -H "Authorization: Bearer <jwt_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
    "to": [{"name": "Customer", "email": "customer@example.com"}],
    "subject": "Support Response",
    "body_html": "<p>Thank you!</p>",
    "body_text": "Thank you!"
  }'


### List Inbox with Filters

# Get unread messages from specific account, page 1
curl "https://api.example.com/inbox/?page=1&size=25&is_read=false&account_id=550e8400-e29b-41d4-a716-446655440000" \
  -H "Authorization: Bearer <jwt_token>"


### Get Unread Counts

curl https://api.example.com/inbox/unread-counts \
  -H "Authorization: Bearer <jwt_token>"


### Download Attachment

# 1. Get attachment download URL
curl https://api.example.com/attachments/aa0e8400-e29b-41d4-a716-446655440000/download \
  -H "Authorization: Bearer <jwt_token>"

# Response: {"download_url": "https://s3.amazonaws.com/...presigned..."}

# 2. Use the URL to download
curl https://s3.amazonaws.com/...presigned-url... -o file.pdf


### Stream Real-Time Events

# Connect to SSE stream with token as query parameter
curl "https://api.example.com/sse/events?token=<jwt_token>"

# Receives:
# event: connected
# data: {"message":"connected","user_id":123}
#
# event: message.created
# data: {"type":"message.created","thread_id":"...","message_id":"...","company_id":123}


---

## Authorization & Access Control

All endpoints enforce company-level access control:

1. User has list of companies (from JWT): ["companies": [1, 2, 5]]
2. Resources filtered by company:
   - Accounts: Only if account.company_id in user.companies
   - Messages: Only if message.email_account.company_id in user.companies
   - Threads: Only if thread.email_account.company_id in user.companies
   - Inbox: Only user's own inbox entries from accessible companies

3. Error responses:
   - 403 Forbidden: User lacks access to company/resource
   - 404 Not Found: Resource not accessible (to prevent information leakage)

---

## Schemas (summarised)**
- EmailAccountCreate (request)


{
  "company_id": 123,
  "email_address": "user@example.com",
  "display_name": "Support",
  "provider": "godaddy",
  "grant_id": "optional-grant-id"
}


- EmailAccountResponse (example)


{
  "id": "11111111-1111-1111-1111-111111111111",
  "company_id": 123,
  "email_address": "user@example.com",
  "display_name": "Support",
  "provider": "godaddy",
  "status": "active",
  "last_sync_at": "2025-12-01T12:00:00Z",
  "created_at": "2025-11-01T09:00:00Z",
  "company": { "id": 123, "name": "Acme Inc." }
}


- SendEmailRequest (used for sending and replying)


{
  "email_account_id": "22222222-2222-2222-2222-222222222222",
  "to": [{ "name": "Recipient", "email": "to@example.com" }],
  "cc": [{ "name": "CC", "email": "cc@example.com" }],
  "bcc": null,
  "subject": "Hello",
  "body_html": "<p>Hi</p>",
  "body_text": "Hi",
  "reply_to_message_id": null
}


- MessageResponse (example)


{
  "id": "33333333-3333-3333-3333-333333333333",
  "thread_id": "44444444-4444-4444-4444-444444444444",
  "email_account_id": "22222222-2222-2222-2222-222222222222",
  "internet_message_id": "<msg@example.com>",
  "from_address": "sender@example.com",
  "from_name": "Sender",
  "to_addresses": [{"name":"Recipient","email":"to@example.com"}],
  "cc_addresses": null,
  "bcc_addresses": null,
  "subject": "Hello",
  "body_html": "<p>Hi</p>",
  "body_text": "Hi",
  "snippet": "Hi...",
  "direction": "inbound",
  "has_attachments": false,
  "sent_at": "2025-12-01T12:01:00Z",
  "created_at": "2025-12-01T12:02:00Z",
  "attachments": []
}


- PaginatedMessageResponse shape

{
  "items": [ /* MessageResponse objects */ ],
  "total": 120,
  "page": 1,
  "size": 25
}


- ThreadSummary / ThreadDetailResponse (examples)

Thread summary item:

{
  "id": "44444444-4444-4444-4444-444444444444",
  "email_account_id": "22222222-2222-2222-2222-222222222222",
  "subject": "Support request",
  "last_message_at": "2025-12-01T12:00:00Z",
  "message_count": 3,
  "do_not_delete": false,
  "created_at": "2025-11-30T08:00:00Z"
}


Thread detail returns the same fields plus messages (array of `MessageResponse`).

- InboxItemResponse (example)


{
  "id": "55555555-5555-5555-5555-555555555555",
  "is_read": false,
  "is_starred": false,
  "is_archived": false,
  "read_at": null,
  "created_at": "2025-12-01T12:00:00Z",
  "message": {/* MessageSummary */},
  "thread_subject": "Subject",
  "thread_message_count": 2,
  "account_email": "support@example.com"
}


- AttachmentResponse (example)


{
  "id": "66666666-6666-6666-6666-666666666666",
  "message_id": "33333333-3333-3333-3333-333333333333",
  "filename": "file.pdf",
  "content_type": "application/pdf",
  "size_bytes": 12345,
  "is_inline": false,
  "created_at": "2025-12-01T12:00:00Z"
}


Endpoint reference

- Accounts (`/accounts`)
  - POST /accounts/ : Create/connect an email account.
    - Auth: Bearer JWT
    - Request body: EmailAccountCreate (see example above)
    - Response: EmailAccountResponse (201)
  - GET /accounts/ : List accounts for user's companies.
    - Auth: Bearer JWT
    - Response: list[EmailAccountResponse] (200)
  - GET /accounts/{account_id} : Get single account.
    - Auth: Bearer JWT
    - Response: EmailAccountResponse (200) or 404
  - PATCH /accounts/{account_id}/ : Update account fields.
    - Auth: Bearer JWT
    - Request: EmailAccountUpdate (partial)
    - Response: EmailAccountResponse (200)
  - DELETE /accounts/{account_id} : Disconnect account.
    - Auth: Bearer JWT
    - Response: 204 No Content

- Messages (`/messages`)
  - POST /messages/send/ : Send an email.
    - Auth: Bearer JWT
    - Request: SendEmailRequest (see example)
    - Response: MessageResponse (200)
  - GET /messages/ : Paginated message list.
    - Auth: Bearer JWT
    - Query params: page, size, account_id (optional)
    - Response: PaginatedMessageResponse (200)
  - GET /messages/{message_id} : Get single message.
    - Auth: Bearer JWT
    - Response: MessageResponse (200) or 404
  - POST /messages/{message_id}/reply : Reply to a message.
    - Auth: Bearer JWT
    - Request: SendEmailRequest
    - Response: MessageResponse (200)

- Inbox (`/inbox`)
  - GET /inbox/ : List inbox items (paginated).
    - Auth: Bearer JWT
    - Query params: page, size, is_read, account_id
    - Response: InboxListResponse (items: `InboxItemResponse`)
  - GET /inbox/unread-counts : Get unread counts per account
    - Auth: Bearer JWT
    - Response: list of UnreadCountResponse objects
  - PATCH /inbox/{inbox_id}/read : Mark entry as read
    - Auth: Bearer JWT
    - Response: InboxEntryResponse (200)
  - PATCH /inbox/{inbox_id}/unread : Mark as unread
    - Auth: Bearer JWT
    - Response: InboxEntryResponse (200)
  - PATCH /inbox/{inbox_id}/star : Toggle star
    - Auth: Bearer JWT
    - Response: InboxEntryResponse (200)
  - PATCH /inbox/{inbox_id}/archive : Archive entry
    - Auth: Bearer JWT
    - Response: InboxEntryResponse (200)
  - PATCH /inbox/threads/{thread_id}/read : Mark entire thread read
    - Auth: Bearer JWT
    - Response: { "updated": <count> }

- Threads (`/threads`)
  - GET /threads/ : List threads (paginated)
    - Auth: Bearer JWT
    - Query params: account_id, page, size
    - Response: ThreadListResponse
  - GET /threads/{thread_id} : Thread details with messages
    - Auth: Bearer JWT
    - Response: ThreadDetailResponse or 404
  - PATCH /threads/{thread_id} : Update thread (e.g., `do_not_delete`)
    - Auth: Bearer JWT
    - Request: ThreadUpdate (partial)
    - Response: ThreadDetailResponse

- Attachments (`/attachments`)
  - GET /attachments/{attachment_id}/download : Get presigned download URL
    - Auth: Bearer JWT
    - Response: { "download_url": "https://..." } or 404
  - GET /attachments/message/{message_id} : List attachments for a message
    - Auth: Bearer JWT
    - Response: list[AttachmentResponse]

- Webhooks (`/webhooks/nylas`)
  - GET /webhooks/nylas?challenge=<str> : Responds with the challenge as plain text (used for verification)
    - No auth
    - Response: plain text challenge
  - POST /webhooks/nylas : Receive Nylas webhooks
    - No Bearer auth; instead the request is verified using the x-nylas-signature header and verify_nylas_webhook utility.
    - Request body: arbitrary payload forwarded by Nylas (see `NylasWebhookPayload`)
    - Response: { "status": "ok" } on success or 401 on invalid signature

- SSE (`/sse/events`)
  - GET /sse/events?token=<jwt> : Server-Sent Events stream
    - Provide user token in query param token
    - Returns text/event-stream with events of form:

Example SSE event (server-sent format):

event: message.created
data: {"type":"message.created","thread_id":"...","message_id":"...","company_id":123}



Webhooks payload shapes
- NylasWebhookPayload (top-level)


{
  "specversion": "1.0",
  "type": "message.created",
  "source": "nylas",
  "id": "delivery-id",
  "time": 1670000000,
  "data": { /* object varies by event type */ }
}


- NylasWebhookMessageData (inside data.object for message events)


{
  "id": "nylas-msg-id",
  "grant_id": "grant-id",
  "thread_id": "nylas-thread-id",
  "subject": "Hello",
  "from": [{"email":"from@example.com","name":"Sender"}],
  "to": [{"email":"to@example.com","name":"Recipient"}],
  "date": 1670000000,
  "body": "...",
  "snippet": "...",
  "attachments": [{ /* ... */ }],
  "folders": ["inbox"]
}


Notes & next steps
- I extracted the API surface from app/api/* and the JSON shapes from app/schemas/*.
- If you want, I can:
  - Add live example curl commands for each endpoint (including auth header examples).
  - Generate an OpenAPI/Swagger-friendly JSON/YAML file derived from the Pydantic models.
  - Add sample unit/integration tests for a few endpoints.
