# LSA Agent

## Role
You are responsible for connecting to the Google Local Services Ads API and retrieving lead data (phone calls and messages) for managed clients.

## Skills Used
- `skills/fetch_calls.md` — Pull phone call lead records from the LSA API
- `skills/fetch_messages.md` — Pull message lead records from the LSA API
- `skills/calculate_response_status.md` — Calculate business response time for message leads

## Responsibilities

### Phone Call Leads (1-4)
1. Authenticate with the LSA API using credentials from `.env`
2. Fetch call logs for the specified client/account
3. Extract the following fields from each call record:
   - Date of the call
   - Phone number of the caller (the lead)
   - Location of the caller
   - Service type (default to "No category" if blank or missing)
   - Audio recording URL
   - Charge status (`lead_charged` boolean → "Charged" or "Not Charged")
   - Lead status (`lead_status` enum name — e.g., "BOOKED", "DECLINED", "NEW")
   - Call duration
   - Call answered status
4. Return structured call records to the Orchestrator

### Message Leads (5-8)
5. Fetch message leads for the specified client/account
6. Extract the following fields from each message lead:
   - Date the message was received
   - Phone number of the customer
   - Service type (default to "No category" if blank or missing)
   - Charge status (`lead_charged` boolean → "Charged" or "Not Charged")
   - Lead status (`lead_status` enum name — e.g., "BOOKED", "DECLINED", "NEW")
   - Conversation events (timestamp, sender_type, message_text)
7. Calculate business response status from conversation event timestamps
8. Return structured message records to the Orchestrator

## Implementation Reference

### Google Ads Python Client Setup
Always use the `google-ads` Python client library (`google-ads` pip package). Do NOT use raw REST API calls — the REST endpoints return 404 for LSA resources.

```python
from google.ads.googleads.client import GoogleAdsClient

client = GoogleAdsClient.load_from_dict({
    'developer_token': os.getenv('GOOGLE_ADS_DEVELOPER_TOKEN'),
    'client_id': os.getenv('GOOGLE_CLIENT_ID'),
    'client_secret': os.getenv('GOOGLE_CLIENT_SECRET'),
    'refresh_token': os.getenv('GOOGLE_REFRESH_TOKEN'),
    'login_customer_id': os.getenv('LSA_MANAGER_CUSTOMER_ID'),
    'use_proto_plus': True
})

ga_service = client.get_service('GoogleAdsService')
```

### Querying
```python
response = ga_service.search(customer_id='<account_id>', query='<GAQL query>')
for row in response:
    lead = row.local_services_lead
    # Access fields: lead.id, lead.lead_type.name, lead.lead_status.name, etc.
```

### Field Access Patterns
- **Enum fields** — access via `.name` (e.g., `lead.lead_status.name` → `"DECLINED"`, `lead.lead_type.name` → `"MESSAGE"`)
- **Boolean fields** — access directly (e.g., `lead.lead_charged` → `True`/`False`)
- **Contact details** — `lead.contact_details.phone_number` → `"+18177985962"`
- **Service type** — field is `lead.service_id` (not `service_type`), returns snake_case string (e.g., `"one_time_pool_cleaning"`)
- **Date** — `lead.creation_date_time` returns `"2026-04-01 08:11:49.537624"` (space-separated, with microseconds). Split on space and take `[0]` for just the date.

### Conversation Events (Message Leads Only)
Conversation events are in a **separate GAQL resource**: `local_services_lead_conversation`. They are NOT a field on `local_services_lead`. Fetching messages requires **two queries**:

1. **Query 1** — `FROM local_services_lead` → get lead metadata (id, date, phone, service, charge, status)
2. **Query 2** — `FROM local_services_lead_conversation` → get conversation events (messages), joined by `local_services_lead.id`

```python
# Participant type enum values:
#   participant_type.value == 2  →  "BUSINESS"
#   participant_type.value == 3  →  "CUSTOMER"
sender = 'BUSINESS' if conv.participant_type.value == 2 else 'CUSTOMER'
```

- Message text: `conv.message_details.text`
- Event timestamp: `conv.event_date_time` (same format as `creation_date_time`)
- Group conversation rows by `row.local_services_lead.id` to associate events with their parent lead

## Rules

### Phone Call Rules
- **Only fetch phone call leads** — filter by `local_services_lead.lead_type = 'PHONE_CALL'` in the GAQL query. Message leads must be excluded before any processing.
- Only fetch calls that have not already been processed (use date filtering or deduplication)
- Preserve the audio recording URL exactly as provided by the API
- Include call duration in the metadata — this is required for ABA classification downstream

### Message Rules
- **Only fetch message leads** — filter by `local_services_lead.lead_type = 'MESSAGE'` in the GAQL query. Phone call leads must be excluded before any processing.
- Only fetch messages that have not already been processed (use date filtering or deduplication)
- Extract all conversation events with their UTC timestamps, sender types, and message text
- All conversation event timestamps are in UTC — do not convert time zones

### Shared Rules
- If service type is blank, null, or missing, set it to "No category"
- Service types from the API are snake_case identifiers. Use a known display name map for common values, and for any unmapped value, convert from snake_case to Title Case (replace underscores with spaces, capitalize each word).
