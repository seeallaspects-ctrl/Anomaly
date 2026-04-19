# Skill: Create Month Tab

## Purpose
Create a new Call Leads tab for a specific month by copying the template tab, renaming it, and clearing any existing data rows (keeping only the header).

## Inputs
- `spreadsheet_id` — The Google Sheet spreadsheet ID (from `config.json`)
- `template_tab_gid` — The GID of the template tab to copy (from `config.json`)
- `target_month` — The month to create (e.g., "February")
- `target_year` — The year (e.g., "2026")

## Process

### Step 1: Check if tab already exists
- Use the Sheets API `spreadsheets.get` to list all existing tabs
- Check if a tab named `Call Leads [Month] [Year]` already exists
- If it exists, skip creation and return the existing tab's GID and name

### Step 2: Copy the template tab
- Use the Sheets API `batchUpdate` with a `duplicateSheet` request
- Source: the template tab GID
- This creates a copy with a default name like "Copy of Call Leads March 2026"

### Step 3: Rename the copied tab
- Use the Sheets API `batchUpdate` with an `updateSheetProperties` request
- New name: `Call Leads [Month] [Year]` (e.g., "Call Leads February 2026")

### Step 4: Clear data rows (keep header)
- Use the Sheets API to clear all rows below row 1 (the header) in the new tab, range A2:N
- This ensures the new tab starts clean with only the header formatting intact

## Outputs
- `tab_name` — The name of the newly created tab (e.g., "Call Leads February 2026")
- `tab_gid` — The GID of the newly created tab
- `created` — Boolean indicating if a new tab was created (false if it already existed)

## Rules
- Never modify or delete the template tab
- The template tab is the source of truth for header formatting and structure
- Only clear data rows (row 2+) in the new tab — never touch row 1 (header)
- Tab name format is always: `Call Leads [Full Month Name] [4-digit Year]`
