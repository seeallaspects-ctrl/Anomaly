# Skill: Fetch Calls

## Purpose
Pull call log records from the Google LSA API for a specific client account.

## Inputs
- Client account ID (from `config.json`)
- `target_month` — The month to fetch calls for (e.g., "February")
- `target_year` — The year (e.g., "2026")
- Date range is calculated as: first day of target month through last day of target month

## Outputs
For each call, return:
- `date` — Date the call occurred
- `phone_number` — Caller's phone number
- `service_type` — LSA service type label (set to "No category" if blank/missing)
- `audio_url` — URL to the call audio recording
- `charge_status` — From `lead_charged` boolean: `True` → "Charged", `False` → "Not Charged"
- `lead_status` — From `lead_status` enum: the enum name (e.g., "BOOKED", "DECLINED", "NEW", "ACTIVE", "EXPIRED")
- `call_duration` — Duration of the call in seconds
- `answered` — Whether the call was answered (boolean)

**Note:** Location is NOT available from the LSA API. It is extracted from the call transcript by the Analysis Agent downstream.

## GAQL Query

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
WHERE local_services_lead.lead_type = 'PHONE_CALL'
    AND local_services_lead.creation_date_time >= '{start_date}'
    AND local_services_lead.creation_date_time <= '{end_date}'
ORDER BY local_services_lead.creation_date_time ASC
```

### Field Mapping
| API Field | Output Field | Transform |
|---|---|---|
| `lead.creation_date_time` | `date` | Split on space, take `[0]` → `"2026-04-01"` |
| `lead.contact_details.phone_number` | `phone_number` | Raw, preserves `+` prefix |
| `lead.service_id` | `service_type` | snake_case → Title Case (or "No category" if blank) |
| `lead.lead_charged` | `charge_status` | `True` → `"Charged"`, `False` → `"Not Charged"` |
| `lead.lead_status.name` | `lead_status` | Enum name string (e.g., `"BOOKED"`) |

**Note:** Audio URL, call duration, and answered status fields should be obtained from the lead's phone call details resource. See `lsa_agent.md` for client setup.

## Rules
- **Lead type filter:** Only fetch leads where `local_services_lead.lead_type = 'PHONE_CALL'`. This must be included in the GAQL `WHERE` clause to exclude message leads before any processing begins.
- Default `service_type` to "No category" when the API returns blank, null, or missing
- **Service type formatting:** The API returns service types as snake_case identifiers (e.g., `pool_contractor_repair_tile`). Use a known display name map for common values. For any unmapped value, convert from snake_case to Title Case (replace underscores with spaces, capitalize each word).
- Return raw data — do not summarize or transform beyond the fields listed above
- Only return calls within the target month's date range
- Use GAQL `WHERE` clause to filter by `creation_date_time` between the first and last day of the target month
