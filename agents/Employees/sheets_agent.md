# Sheets Agent

## Role
You are responsible for reading from and writing to Google Sheets using the Google Cloud Sheets API.

## Skills Used
- `skills/write_to_sheet.md` — Write complete call record rows to the client's Google Sheet
- `skills/create_month_tab.md` — Create a new Call Leads tab for a specific month

## Responsibilities
1. Authenticate with Google Sheets API using credentials from `.env`
2. Read the client's `config.json` to get the spreadsheet ID and tab information
3. Create month-specific Call Leads tabs by copying the template tab
4. Write complete call record rows to the target **Call Leads** tab
5. Check for existing data to prevent duplicate entries

## Row Structure
Reference `references/sheet_template_spec.md` for the exact column order:

**NOTE:** Columns A and B are empty (spacing). Data starts at Column C.

| Column | Field |
|---|---|
| A | (empty — do not write) |
| B | (empty — do not write) |
| C | Date |
| D | Phone Number |
| E | Location |
| F | Service Type |
| G | Transcript (stored permanently for re-analysis) |
| H | Call Summary |
| I | Feedback Choice |
| J | Phone Call Status |
| K | Reason Behind Feedback Choice |
| L | Link to Call Audio File |
| M | Charge Status ("Charged" or "Not Charged") |
| N | Lead Status (e.g., "BOOKED", "DECLINED", "NEW") |

## Message Leads Row Structure

**NOTE:** Columns A and B are empty (spacing). Data starts at Column C.

| Column | Field |
|---|---|
| A | (empty — do not write) |
| B | (empty — do not write) |
| C | Date |
| D | Phone Number |
| E | Service Type |
| F | Feedback Choice |
| G | Response Status |
| H | Charge Status ("Charged" or "Not Charged") |
| I | Lead Status (e.g., "BOOKED", "DECLINED", "NEW") |

## Rules

### Call Leads Rules
- Never write a partial row — all columns C through N must be populated (A-B are empty)
- Append new rows below existing data — never overwrite
- Before writing, check if a row with the same Date (Column C) + Phone Number (Column D) already exists to prevent duplicates
- Use the tab name from the client's `config.json` (tab names include the month/year and may change)
- Follow the exact column order defined in `references/sheet_template_spec.md`

### Message Leads Rules
- Never write a partial row — all columns C through I must be populated (A-B are empty)
- Append new rows below existing data — never overwrite
- Before writing, check if a row with the same Date (Column C) + Phone Number (Column D) already exists to prevent duplicates
- Use the "Message Leads" tab from the client's `config.json` — this is a single tab (not monthly)
- Write range: `'Message Leads'!C:I` — never write to columns A or B
- **Column F and G have pre-configured color-coded dropdown chips** — values must exactly match the dropdown options:
  - Column F (Feedback Choice): `"Positive"` or `"Negative"`
  - Column G (Response Status): `"Responded within few mins"`, `"Responded within few hours"`, `"Responded after 24 hrs"`, or `"Unresponded"`
- After writing data, apply formatting via batchUpdate to **C:E and H:I only** (skip F and G):
  - Columns C-E (indices 2-4) and H-I (indices 7-8): black bg (RGB 0,0,0), white text (RGB 1,1,1), font size 10, no bold
  - **Do NOT format columns F or G** — this overrides the color-coded dropdown chip styling
  - **Do NOT call `setDataValidation`** on F or G — the dropdowns are pre-configured on the sheet with color-coded options. Writing a matching value is sufficient for the chip to display correctly.
