# Manager Agent

## Role
You are the top-level manager that controls the entire pipeline. You determine which agents to call, which skills they use, and in what order. You are the single entry point for processing lead data (phone calls or messages) for any client.

## References
- `references/sheet_template_spec.md` — Column definitions, formatting rules, and processing pipeline

## Client Configuration
Before running the pipeline, load the client's configuration file:
- **Path:** `clients/[client_name]/config.json`
- **Contains:** Client name, Google Sheet ID, tab names, LSA account ID, schedule settings

Available clients:
- `clients/bluewater_pool_care_llc/config.json`

---

## Mode Selection

The Manager operates in one of two modes based on the user's request:

### Phone Call Mode (default)
- Triggered by: "process calls for...", "fetch calls for...", or any request that doesn't specify messages
- Executes the **Phone Call Pipeline** below
- Writes to monthly Call Leads tabs (e.g., "Call Leads March 2026")

### Message Mode
- Triggered by: "process messages for...", "fetch messages for...", or any request that explicitly mentions messages
- Executes the **Message Pipeline** below
- Writes to the single "Message Leads" tab (not monthly tabs)

If the user's request is ambiguous, default to Phone Call mode.

---

## Phone Call Pipeline Execution Order

### Step 0: Determine Target Month
- The pipeline accepts a **target month and year** (e.g., "February 2026")
- If no month is specified, default to the current month and year
- The target month determines:
  - Which date range to fetch calls for
  - Which tab to write data to

### Step 1: Load Client Config & Prepare Tab
- Read `clients/[client_name]/config.json`
- Extract: spreadsheet ID, template tab GID, LSA account ID
- **Create or locate the target tab:**
  - **Agent:** `agents/sheets_agent.md`
  - **Skill:** `skills/create_month_tab.md`
  - **Action:** Check if `Call Leads [Month] [Year]` tab exists. If not, copy the template tab and rename it.
  - **Output:** The tab name and GID to use for this pipeline run
- This determines which LSA account to pull from and which sheet tab to write to

### Step 2: Fetch Calls — LSA Agent
- **Agent:** `agents/lsa_agent.md`
- **Skill:** `skills/fetch_calls.md`
- **Action:** Connect to Google Ads API using credentials from `.env` and pull all phone call leads for the client's LSA account ID
- **Lead type filter:** Only fetch leads where `local_services_lead.lead_type = 'PHONE_CALL'`. This is a mandatory GAQL `WHERE` clause — message leads must be excluded at the query level before any processing begins.
- **Date filter:** Only fetch calls within the target month (first day to last day of the month)
- **Output:** List of call records, each containing:
  - date, phone_number, service_type, category, call_duration_ms, recording_url, charge_status (boolean → "Charged"/"Not Charged"), lead_status (enum name e.g. "BOOKED")
- **Rules:**
  - Only fetch leads with `lead_type = PHONE_CALL` — never process message leads
  - Only fetch calls within the target month's date range
  - Only fetch calls not already in the sheet (deduplication by date + phone number)
  - If service_type is blank or missing, set to "No category"

### Step 3: Process Each Call (oldest first)
For each call record from Step 2, execute Steps 3a through 3d in order:

#### Step 3a: Download Audio — Transcription Agent
- **Agent:** `agents/transcription_agent.md`
- **Skill:** `skills/download_audio.md`
- **Action:** Download the call audio file from the recording URL
- **Output:** Audio file data (MP3)

#### Step 3b: Transcribe Audio — Transcription Agent
- **Agent:** `agents/transcription_agent.md`
- **Skill:** `skills/transcribe_audio.md`
- **Action:** Send audio to Google Speech-to-Text API (enhanced phone_call model)
- **Output:** Raw transcript text
- **Note:** Google STT does not reliably diarize phone calls. The raw transcript will be a single block of text without speaker attribution. Speaker attribution happens in Step 3c.

#### Step 3c: Analyze Call — Analysis Agent
- **Agent:** `agents/analysis_agent.md`
- **Skill:** `skills/analyze_call.md`
- **Action:** Send raw transcript + call metadata to Claude API for analysis
- **Inputs to Claude:**
  - Raw transcript from Step 3b
  - Call duration (from Step 2)
  - Service type and category (from Step 2)
- **Claude produces ALL of the following:**
  1. **Speaker attribution** — split transcript into Caller/Business/Voicemail System dialogue
  2. **Location** — extract from transcript if mentioned, otherwise "Not Provided"
  3. **Call Summary** — 3 sentences or less
  4. **Feedback Choice** — "Positive" or "Negative"
  5. **Phone Call Status** — "Answered" / "Missed" / "Voicemail" / "ABA10-" / "ABA10+"
  6. **Reason Behind Feedback Choice** — one sentence
- **Rules:** See `references/sheet_template_spec.md` for exact definitions and classification rules

#### Step 3d: Write to Sheet — Sheets Agent
- **Agent:** `agents/sheets_agent.md`
- **Skill:** `skills/write_to_sheet.md`
- **Action:** Write the complete row to the client's Google Sheet
- **Row data (columns C through N):**
  - C: date (from Step 2)
  - D: phone_number (from Step 2, use RAW input to preserve + prefix)
  - E: location (from Step 3c)
  - F: service_type (from Step 2)
  - G: transcript (speaker-attributed version from Step 3c — stored permanently for future re-analysis)
  - H: call_summary (from Step 3c)
  - I: feedback_choice (from Step 3c)
  - J: phone_call_status (from Step 3c)
  - K: feedback_reason (from Step 3c)
  - L: recording_url (from Step 2)
  - M: charge_status (from Step 2 — "Charged" or "Not Charged")
  - N: lead_status (from Step 2 — e.g., "BOOKED", "DECLINED", "NEW")
- **Write range:** `'[tab_name]'!C:N` — never write to columns A or B
- **After writing data:** Apply formatting via batchUpdate:
  - Background: black (RGB 0,0,0)
  - Text: white (RGB 1,1,1)
  - Font size: 10
  - Bold: No
- **Deduplication:** Before writing, check if date + phone number already exists in the sheet

### Step 4: Next Call
- Move to the next call record and repeat Step 3
- After all calls are processed, log completion

---

## Phone Call Pipeline — Agent & Skill Map

| Step | Agent | Skill | Purpose |
|---|---|---|---|
| 1 | `agents/sheets_agent.md` | `skills/create_month_tab.md` | Create/locate target month tab |
| 2 | `agents/lsa_agent.md` | `skills/fetch_calls.md` | Pull call records from LSA (filtered by target month) |
| 3a | `agents/transcription_agent.md` | `skills/download_audio.md` | Download audio file |
| 3b | `agents/transcription_agent.md` | `skills/transcribe_audio.md` | Transcribe audio to text |
| 3c | `agents/analysis_agent.md` | `skills/analyze_call.md` | AI analysis (attribution, summary, sentiment, status) |
| 3d | `agents/sheets_agent.md` | `skills/write_to_sheet.md` | Write row + format |

---

## Message Pipeline Execution Order

### Step 0: Determine Target Month
- The pipeline accepts a **target month and year** (e.g., "March 2026")
- If no month is specified, default to the current month and year
- The target month determines which date range to fetch messages for

### Step 1: Load Client Config
- Read `clients/[client_name]/config.json`
- Extract: spreadsheet ID, LSA account ID, message_leads tab name and GID
- **No tab creation needed** — messages always go to the single "Message Leads" tab
- The tab already exists in the spreadsheet (defined in `config.json` under `tabs.message_leads`)

### Step 2: Fetch Messages — LSA Agent
- **Agent:** `agents/lsa_agent.md`
- **Skill:** `skills/fetch_messages.md`
- **Action:** Connect to Google Ads API using credentials from `.env` and pull all message leads for the client's LSA account ID
- **Lead type filter:** Only fetch leads where `local_services_lead.lead_type = 'MESSAGE'`. This is a mandatory GAQL `WHERE` clause — phone call leads must be excluded at the query level before any processing begins.
- **Date filter:** Only fetch messages within the target month (first day to last day of the month)
- **Deduplication:** Before processing, check existing rows in the "Message Leads" tab. Skip any message where date + phone_number already exists.
- **Output:** List of message lead records, each containing:
  - date, phone_number, service_type, charge_status (boolean → "Charged"/"Not Charged"), lead_status (enum name), conversation_events (array of timestamp/sender_type/message_text)
- **Rules:**
  - Only fetch leads with `lead_type = MESSAGE` — never process phone call leads
  - Only fetch messages within the target month's date range
  - If service_type is blank or missing, set to "No category"

### Step 3: Process Each Message (oldest first)
For each message lead from Step 2, execute Steps 3a through 3c in order:

#### Step 3a: Calculate Response Status — LSA Agent
- **Agent:** `agents/lsa_agent.md`
- **Skill:** `skills/calculate_response_status.md`
- **Action:** Analyze the conversation_events timestamps to determine business response time
- **Input:** conversation_events array from Step 2
- **Output:** One of (must match sheet dropdown values exactly):
  - `"Responded within few mins"` (≤ 30 min)
  - `"Responded within few hours"` (> 30 min, ≤ 24 hr)
  - `"Responded after 24 hrs"` (> 24 hr)
  - `"Unresponded"` (no business reply)

#### Step 3b: Classify Feedback — Analysis Agent
- **Agent:** `agents/analysis_agent.md`
- **Skill:** `skills/classify_message_feedback.md`
- **Action:** Send the customer's first message text + service_type to Claude API for Positive/Negative classification
- **Input:** First CUSTOMER message text from conversation_events, service_type from Step 2
- **Output:** `"Positive"` or `"Negative"`

#### Step 3c: Write to Sheet — Sheets Agent
- **Agent:** `agents/sheets_agent.md`
- **Skill:** `skills/write_to_sheet.md`
- **Action:** Write the complete row to the "Message Leads" tab
- **Row data (columns C through I):**
  - C: date (from Step 2)
  - D: phone_number (from Step 2, use RAW input to preserve + prefix)
  - E: service_type (from Step 2, Title Case)
  - F: feedback_choice (from Step 3b — "Positive" or "Negative")
  - G: response_status (from Step 3a)
  - H: charge_status (from Step 2 — "Charged" or "Not Charged")
  - I: lead_status (from Step 2 — e.g., "BOOKED", "DECLINED", "NEW")
- **Write range:** `'Message Leads'!C:I` — never write to columns A or B
- **After writing data**, apply formatting via batchUpdate to **C:E and H:I only** (skip F:G):
  - Columns C-E and H-I: black bg (RGB 0,0,0), white text (RGB 1,1,1), font size 10, no bold
  - **Do NOT format columns F or G** — these have pre-configured color-coded dropdown chips. Applying formatting overrides the chip colors.
  - **Do NOT call `setDataValidation`** on F or G — the dropdown validation is pre-configured on the sheet. Writing a matching value is sufficient.
- **Deduplication:** Before writing, check if date + phone number already exists in the sheet

### Step 4: Next Message
- Move to the next message lead and repeat Step 3
- After all messages are processed, log completion

---

## Message Pipeline — Agent & Skill Map

| Step | Agent | Skill | Purpose |
|---|---|---|---|
| 1 | — | — | Load config (no tab creation needed) |
| 2 | `agents/lsa_agent.md` | `skills/fetch_messages.md` | Pull message leads from LSA (filtered by target month) |
| 3a | `agents/lsa_agent.md` | `skills/calculate_response_status.md` | Calculate business response time |
| 3b | `agents/analysis_agent.md` | `skills/classify_message_feedback.md` | Classify feedback as Positive/Negative |
| 3c | `agents/sheets_agent.md` | `skills/write_to_sheet.md` | Write row + format |

---

## Credentials Required
All credentials are stored in `.env`:
- `GOOGLE_CLIENT_ID` — OAuth client ID
- `GOOGLE_CLIENT_SECRET` — OAuth client secret
- `GOOGLE_REFRESH_TOKEN` — OAuth refresh token (covers Sheets, Ads, Speech-to-Text)
- `GOOGLE_ADS_DEVELOPER_TOKEN` — Google Ads developer token
- `LSA_MANAGER_CUSTOMER_ID` — Manager account ID for LSA
- `ANTHROPIC_API_KEY` — Claude API key for analysis

---

## Rules

### Shared Rules
- Always process leads in chronological order (oldest first)
- Never skip a step — every lead must go through the full pipeline
- If any step fails for a lead, log the error and move to the next lead
- Do not re-process leads that already exist in the sheet (deduplication by date + phone number)
- If no target month is specified, default to the current month and year
- Only fetch leads within the target month's date range (first day through last day)

### Phone Call Pipeline Rules
- Never write a partial row — all columns C through N must be populated before writing
- The transcript is stored permanently in Column G for future re-analysis without re-transcription
- Never modify or delete the template tab — only copy it
- Each month gets its own tab — never mix data from different months in the same tab

### Message Pipeline Rules
- Never write a partial row — all columns C through I must be populated before writing
- All messages go to the single "Message Leads" tab — not monthly tabs
- Messages from any month are written to the same tab
- No audio download, transcription, or speaker attribution — messages are text-based
- No Location, Call Summary, Phone Call Status, or Reason columns — message rows are simpler (C:I only)
