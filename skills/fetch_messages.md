# Skill: Fetch Messages

## Purpose
Pull message lead records from the Google LSA API for a specific client account.

## Inputs
- Client account ID (from `config.json`)
- `target_month` — The month to fetch messages for (e.g., "March")
- `target_year` — The year (e.g., "2026")
- Date range is calculated as: first day of target month through last day of target month

## Outputs
For each message lead, return:
- `date` — Date the message was received
- `phone_number` — Customer's phone number
- `service_type` — LSA service type label (set to "No category" if blank/missing)
- `charge_status` — From `lead_charged` boolean: `True` → "Charged", `False` → "Not Charged"
- `lead_status` — From `lead_status` enum: the enum name (e.g., "BOOKED", "DECLINED", "NEW", "ACTIVE", "EXPIRED")
- `conversation_events` — Array of message events, each containing:
  - `timestamp` — UTC timestamp of the message
  - `sender_type` — `"CUSTOMER"` or `"BUSINESS"`
  - `message_text` — The text content of the message

## GAQL Queries

Fetching message leads requires **two separate queries** because conversation events are in a different GAQL resource.

### Query 1: Lead Metadata
```sql
SELECT
    local_services_lead.id,
    local_services_lead.lead_type,
    local_services_lead.category_id,
    local_services_lead.service_id,
    local_services_lead.contact_details,
    local_services_lead.creation_date_time,
    local_services_lead.lead_status,
    local_services_lead.lead_charged
FROM local_services_lead
WHERE local_services_lead.lead_type = 'MESSAGE'
    AND local_services_lead.creation_date_time >= '{start_date}'
    AND local_services_lead.creation_date_time <= '{end_date}'
ORDER BY local_services_lead.creation_date_time ASC
```

### Query 2: Conversation Events
```sql
SELECT
    local_services_lead_conversation.id,
    local_services_lead_conversation.conversation_channel,
    local_services_lead_conversation.participant_type,
    local_services_lead_conversation.event_date_time,
    local_services_lead_conversation.message_details.text,
    local_services_lead.id
FROM local_services_lead_conversation
WHERE local_services_lead.lead_type = 'MESSAGE'
    AND local_services_lead.creation_date_time >= '{start_date}'
    AND local_services_lead.creation_date_time <= '{end_date}'
ORDER BY local_services_lead_conversation.event_date_time ASC
```

### Joining
Group Query 2 results by `row.local_services_lead.id` and attach to the matching lead from Query 1.

### Field Mapping
| API Field | Output Field | Transform |
|---|---|---|
| `lead.creation_date_time` | `date` | Split on space, take `[0]` → `"2026-04-01"` |
| `lead.contact_details.phone_number` | `phone_number` | Raw, preserves `+` prefix |
| `lead.service_id` | `service_type` | snake_case → Title Case (or "No category" if blank) |
| `lead.lead_charged` | `charge_status` | `True` → `"Charged"`, `False` → `"Not Charged"` |
| `lead.lead_status.name` | `lead_status` | Enum name string (e.g., `"DECLINED"`) |
| `conv.participant_type.value` | `sender_type` | `2` → `"BUSINESS"`, `3` → `"CUSTOMER"` |
| `conv.event_date_time` | `timestamp` | Raw UTC string |
| `conv.message_details.text` | `message_text` | Raw text (may be empty for non-text events) |

## Rules
- **Lead type filter:** Only fetch leads where `local_services_lead.lead_type = 'MESSAGE'`. This must be included in the GAQL `WHERE` clause to exclude phone call leads before any processing begins.
- Default `service_type` to "No category" when the API returns blank, null, or missing
- **Service type formatting:** The API returns service types as snake_case identifiers (e.g., `pool_contractor_repair_tile`). Use a known display name map for common values. For any unmapped value, convert from snake_case to Title Case (replace underscores with spaces, capitalize each word).
- Return raw data — do not summarize or transform beyond the fields listed above
- Only return messages within the target month's date range
- Use GAQL `WHERE` clause to filter by `creation_date_time` between the first and last day of the target month
- All timestamps in `conversation_events` are in UTC — do not convert time zones
