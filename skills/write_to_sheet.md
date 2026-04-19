# Skill: Write to Sheet

## Purpose
Write a complete lead record row to the client's Google Sheet. Used by both Call Leads and Message Leads pipelines.

## Inputs — Call Leads
- Client `config.json` (for spreadsheet ID and tab info)
- Complete row data:
  - `date`
  - `phone_number`
  - `location`
  - `service_type`
  - `transcript`
  - `call_summary`
  - `feedback_choice`
  - `phone_call_status`
  - `feedback_reason`
  - `audio_url`
  - `charge_status`
  - `lead_status`

## Inputs — Message Leads
- Client `config.json` (for spreadsheet ID and tab info)
- Complete row data:
  - `date`
  - `phone_number`
  - `service_type`
  - `feedback_choice`
  - `response_status`
  - `charge_status`
  - `lead_status`

## Column Mapping — Call Leads (C:N)
Reference `references/sheet_template_spec.md` for exact order:

**NOTE:** Columns A and B are empty (spacing). Data starts at Column C.

| Column | Value |
|---|---|
| A | (empty — do not write) |
| B | (empty — do not write) |
| C | date |
| D | phone_number |
| E | location |
| F | service_type |
| G | transcript (stored permanently for re-analysis) |
| H | call_summary |
| I | feedback_choice |
| J | phone_call_status |
| K | feedback_reason |
| L | audio_url |
| M | charge_status ("Charged" or "Not Charged") |
| N | lead_status (e.g., "BOOKED", "DECLINED", "NEW") |

## Column Mapping — Message Leads (C:I)

| Column | Value | Data Validation |
|---|---|---|
| A | (empty — do not write) | — |
| B | (empty — do not write) | — |
| C | date | — |
| D | phone_number | — |
| E | service_type | — |
| F | feedback_choice | Dropdown: `"Positive"`, `"Negative"` |
| G | response_status | Dropdown: `"Responded within few mins"`, `"Responded within few hours"`, `"Responded after 24 hrs"`, `"Unresponded"` |
| H | charge_status ("Charged" or "Not Charged") | — |
| I | lead_status (e.g., "BOOKED", "DECLINED", "NEW") | — |

**Column F and G use data validation dropdowns.** Values written to these columns MUST exactly match the dropdown options listed above. After writing rows, data validation rules must be set on these columns via `setDataValidation` batchUpdate (see below).

## API Implementation

### OAuth Setup
```python
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

# First refresh the access token
token_resp = requests.post('https://oauth2.googleapis.com/token', data={
    'client_id': os.getenv('GOOGLE_CLIENT_ID'),
    'client_secret': os.getenv('GOOGLE_CLIENT_SECRET'),
    'refresh_token': os.getenv('GOOGLE_REFRESH_TOKEN'),
    'grant_type': 'refresh_token'
})
access_token = token_resp.json()['access_token']

creds = Credentials(
    token=access_token,
    refresh_token=os.getenv('GOOGLE_REFRESH_TOKEN'),
    token_uri='https://oauth2.googleapis.com/token',
    client_id=os.getenv('GOOGLE_CLIENT_ID'),
    client_secret=os.getenv('GOOGLE_CLIENT_SECRET')
)

sheets = build('sheets', 'v4', credentials=creds)
```

### Writing Rows
```python
# Append rows (works for both call and message leads)
sheets.spreadsheets().values().append(
    spreadsheetId=spreadsheet_id,
    range="'{tab_name}'!C:N",     # C:N for calls, C:I for messages
    valueInputOption='RAW',
    insertDataOption='INSERT_ROWS',
    body={'values': rows}          # rows = list of lists, one per lead
).execute()
```

### Applying Formatting

#### Call Leads — format all columns C:N
```python
sheets.spreadsheets().batchUpdate(
    spreadsheetId=spreadsheet_id,
    body={'requests': [{
        'repeatCell': {
            'range': {
                'sheetId': tab_gid,
                'startRowIndex': start_row,
                'endRowIndex': end_row,
                'startColumnIndex': 2,         # C
                'endColumnIndex': 14           # N (exclusive)
            },
            'cell': {
                'userEnteredFormat': {
                    'backgroundColor': {'red': 0, 'green': 0, 'blue': 0},
                    'textFormat': {
                        'foregroundColor': {'red': 1, 'green': 1, 'blue': 1},
                        'fontSize': 10,
                        'bold': False
                    }
                }
            },
            'fields': 'userEnteredFormat(backgroundColor,textFormat)'
        }
    }]}
).execute()
```

#### Message Leads — format C:E and H:I only, SKIP F:G
Columns F (Feedback Choice) and G (Response Status) have **pre-configured color-coded dropdown chips** set up in the sheet. Do NOT apply formatting to these columns — it overrides the chip colors. Apply formatting in two separate ranges that skip F and G:

```python
sheets.spreadsheets().batchUpdate(
    spreadsheetId=spreadsheet_id,
    body={'requests': [
        # Format C:E (columns 2-4, indices 2-5 exclusive)
        {
            'repeatCell': {
                'range': {
                    'sheetId': tab_gid,
                    'startRowIndex': start_row,
                    'endRowIndex': end_row,
                    'startColumnIndex': 2,     # C
                    'endColumnIndex': 5        # E (exclusive)
                },
                'cell': {
                    'userEnteredFormat': {
                        'backgroundColor': {'red': 0, 'green': 0, 'blue': 0},
                        'textFormat': {
                            'foregroundColor': {'red': 1, 'green': 1, 'blue': 1},
                            'fontSize': 10,
                            'bold': False
                        }
                    }
                },
                'fields': 'userEnteredFormat(backgroundColor,textFormat)'
            }
        },
        # Format H:I (columns 7-8, indices 7-9 exclusive)
        {
            'repeatCell': {
                'range': {
                    'sheetId': tab_gid,
                    'startRowIndex': start_row,
                    'endRowIndex': end_row,
                    'startColumnIndex': 7,     # H
                    'endColumnIndex': 9        # I (exclusive)
                },
                'cell': {
                    'userEnteredFormat': {
                        'backgroundColor': {'red': 0, 'green': 0, 'blue': 0},
                        'textFormat': {
                            'foregroundColor': {'red': 1, 'green': 1, 'blue': 1},
                            'fontSize': 10,
                            'bold': False
                        }
                    }
                },
                'fields': 'userEnteredFormat(backgroundColor,textFormat)'
            }
        }
    ]}
).execute()
```

**IMPORTANT — Do NOT set data validation on F or G.** The dropdown chips (with color-coded options) are pre-configured on the Message Leads tab. Writing a matching value into the cell is sufficient — the existing dropdown validation and chip colors will apply automatically. Calling `setDataValidation` would replace the color-coded chips with plain dropdowns.

### Deduplication Check
```python
# Read existing date + phone columns before writing
existing = sheets.spreadsheets().values().get(
    spreadsheetId=spreadsheet_id,
    range="'{tab_name}'!C:D"
).execute().get('values', [])

existing_keys = {(row[0], row[1]) for row in existing if len(row) >= 2}
# Skip any lead where (date, phone_number) is already in existing_keys
```

## Rules
- Never write a partial row — Call Leads: all columns C-N; Message Leads: all columns C-I
- Columns A and B are always empty — do not write to them
- Append below existing data — never overwrite existing rows
- Before writing, check if a row with the same date + phone number exists (deduplication)
- Call Leads: use the tab name from the client's `config.json` — tab names include month/year
- Message Leads: always use the `"Message Leads"` tab from `config.json` under `tabs.message_leads`
