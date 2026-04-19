# Google Sheet Template Specification

## Overview

This reference defines the standard column mapping for the Call Leads and Message Leads sheets used across all clients. Every agent and skill that reads or writes to the sheet MUST follow this specification exactly.

---

## Sheet Structure

The Google Sheet contains a fixed set of tabs plus dynamically created month tabs:

| Tab | Purpose |
|---|---|
| **Dashboard** | Weekly aggregated summary (auto-calculated from Call Leads) |
| **Call Leads [Month Year]** (template) | The original template tab — never modified, only copied |
| **Call Leads [Month Year]** (per month) | Dynamically created tabs for each month's call data |
| **Message Leads** | Non-call leads (messages/texts) |

### Month Tab Creation

When the pipeline runs for a specific month:
1. The system checks if a `Call Leads [Month] [Year]` tab already exists
2. If not, it copies the template tab (defined in the client's `config.json` under `template_tab`)
3. The copy is renamed to `Call Leads [Month] [Year]` (e.g., "Call Leads February 2026")
4. All data rows are cleared from the copy (header row 1 is preserved)
5. The pipeline then writes call data for that month to the new tab

**Template tab rules:**
- The template tab is the source of truth for header formatting and column structure
- Never modify or delete the template tab
- Never write call data to the template tab
- The template tab's GID is stored in the client's `config.json` under `google_sheet.template_tab.gid`

---

## Call Leads — Column Definitions

**NOTE:** Columns A and B are intentionally left empty (spacing/formatting). Data begins at Column C.

### Column A: (Empty)
- Reserved — do not write to this column.

### Column B: (Empty)
- Reserved — do not write to this column.

### Column C: Date
- **Source:** LSA API
- **Processing:** Direct pull
- **Description:** The date the call occurred.
- **Format:** Date value as provided by LSA API.

### Column D: Phone Number
- **Source:** LSA API
- **Processing:** Direct pull
- **Description:** The phone number of the lead (the person calling the client).

### Column E: Location
- **Source:** Transcript → AI extraction
- **Processing:** Two-step fallback
- **Description:** The caller's geographic location in **City, State** format.
- **Format:** `"City, State"` (e.g., "Dallas, Texas", "Lake Worth, Florida")
- **Rule:**
  1. First, check the transcript for any mention of location (city, area, neighborhood, zip code, address, etc.)
  2. Extract and normalize to **City, State** format only — do not include street addresses, zip codes, or other details
  3. If no location is found in the transcript, set the value to `"Not Provided"`
- **Note:** The LSA API does not expose a location field. This must be extracted from the call transcript by the Analysis Agent.

### Column F: Service Type
- **Source:** LSA API
- **Processing:** Direct pull with fallback
- **Description:** The service type associated with the call as labeled by LSA.
- **Fallback Rule:** If the field is blank or missing, set the value to `"No category"`.

### Column G: Transcript
- **Source:** LSA API audio file → Google Speech-to-Text → AI speaker attribution
- **Processing:** Multi-step (download audio → transcribe → attribute speakers via Claude)
- **Description:** The full transcript of the call with speaker attribution.
- **Format:** Speaker-attributed dialogue (e.g., "Caller: ...", "Business: ...", "Voicemail System: ...")
- **Rule:** This column is stored permanently. It eliminates the need to re-download and re-transcribe audio for any future re-analysis. If no audio is available, set to an empty string.

### Column H: Call Summary
- **Source:** Transcript (Column G) → AI Summarization
- **Processing:** AI-generated from the transcript
- **Description:** A summary of the call in **3 sentences or less** that communicates:
  - What the call was and what it was about
  - What the caller was looking for
  - Whether or not the call was answered
  - The overall outcome of the call
  - The overall objective of the call

### Column I: Feedback Choice
- **Source:** Transcript → AI Sentiment Analysis
- **Processing:** AI classification
- **Description:** Sentiment analysis of the call based on the transcript.
- **Allowed Values (dropdown):**
  - `Positive` — The call was a legitimate lead (e.g., quote request, service inquiry, appointment scheduling).
  - `Negative` — The call was not a legitimate lead (e.g., spam, solicitation, scam, wrong number).
- **Rule:** Only `Positive` or `Negative`. Never neutral. Never blank.

### Column J: Phone Call Status
- **Source:** LSA API data + Transcript analysis
- **Processing:** Rule-based classification using call duration, answered status, and voicemail detection
- **Description:** The outcome status of the phone call.
- **Allowed Values (dropdown):**

| Status | Definition | Criteria |
|---|---|---|
| `Answered` | The call was picked up by the client. | Call was answered by a live person on the client side. |
| `Missed` | The call rang and was dropped without being answered or going to voicemail. | Call duration > 20 seconds AND not answered AND no voicemail left. This indicates a genuine missed call — the caller waited but nobody picked up. |
| `Voicemail` | The call was missed but a voicemail was left. | Call was not answered AND a voicemail message was recorded by the caller. |
| `ABA10-` | Abandoned under 10 seconds. | Call duration < 10 seconds AND not answered AND no voicemail. Low intent / misdial. |
| `ABA10+` | Abandoned between 10 and 20 seconds. | Call duration >= 10 seconds AND <= 20 seconds AND not answered AND no voicemail. Brief intent — caller hung up quickly. |

- **Classification Rules:**
  - Do NOT select `ABA10-` if the call was answered, went to voicemail, or lasted 10+ seconds.
  - Do NOT select `ABA10+` if the call was answered, went to voicemail, lasted under 10 seconds, or lasted over 20 seconds.
  - Do NOT select `Missed` if the call was answered, went to voicemail, or lasted 20 seconds or less (use ABA instead).
  - Do NOT select `Missed` if a voicemail was left (use `Voicemail` instead).

### Column K: Reason Behind Feedback Choice
- **Source:** Transcript → AI Analysis
- **Processing:** AI-generated explanation
- **Description:** A **single sentence** explaining why the call was classified as Positive or Negative in Column I.
- **Examples:**
  - `"Caller requested a quote for weekly pool cleaning service."` (Positive)
  - `"Caller was a telemarketer attempting to sell SEO services."` (Negative)
  - `"Caller inquired about pool pump repair availability."` (Positive)
  - `"Robocall with automated message about extended warranties."` (Negative)

### Column L: Link to Call Audio File
- **Source:** LSA API
- **Processing:** Direct pull
- **Description:** The direct URL to the audio recording of the call as provided by the LSA API.

### Column M: Charge Status
- **Source:** LSA API (`lead_charged` field)
- **Processing:** Boolean mapping
- **Description:** Whether Google charged the client for this lead.
- **Mapping:**
  - `True` → `"Charged"`
  - `False` → `"Not Charged"`

### Column N: Lead Status
- **Source:** LSA API (`lead_status` enum)
- **Processing:** Direct pull of enum name
- **Description:** The current status of the lead in Google LSA.
- **Allowed Values:**
  - `BOOKED` — Lead was booked by the client
  - `DECLINED` — Lead was declined by the client
  - `NEW` — Lead is new / unactioned
  - `ACTIVE` — Lead is currently active
  - `EXPIRED` — Lead expired without action
  - `DISABLED` — Lead was disabled
  - `CONSUMER_DECLINED` — Consumer declined the lead
  - `WIPED_OUT` — Lead was wiped out

---

## Data Row Formatting

All data rows (row 2 and below) MUST be formatted as follows after writing:
- **Background color:** Black (RGB: 0, 0, 0)
- **Text color:** White (RGB: 1, 1, 1)
- **Font size:** 10
- **Bold:** No
- **Call Leads range:** Apply to columns C through N for each new row — all columns C–N must be populated
- **Message Leads range:** Apply to columns C through I for each new row — all columns C–I must be populated

This formatting must be applied via the Google Sheets `batchUpdate` API after writing row data. Do NOT inherit formatting from the header row.

---

## Processing Pipeline (Per Call)

The following pipeline is executed for each call record:

```
1. PULL call metadata from LSA API
   → date, phone number, service type, audio URL, charge status, lead status

2. DOWNLOAD audio file from the audio URL

3. TRANSCRIBE audio with:
   → Speaker diarization (distinguish caller vs. business)
   → Voicemail detection (identify voicemail greetings, live conversation, messages left)
   → Call flow awareness (answered live, went to voicemail, missed entirely)
   → Format: attributed dialogue (e.g., "Caller: ...", "Business: ...", "Voicemail System: ...")

4. ANALYZE transcript to determine:
   → Speaker attribution (split raw transcript into Caller / Business / Voicemail System dialogue)
   → Location (extract from transcript as City, State — or "Not Provided")
   → Phone Call Status (Answered / Missed / Voicemail / ABA10- / ABA10+)
   → Feedback Choice (Positive / Negative)
   → Reason Behind Feedback Choice (one sentence)
   → Call Summary (3 sentences or less)

5. WRITE complete row to the Call Leads sheet (including the transcript in Column G)
```

**IMPORTANT:** The transcript is stored permanently in Column G. This allows future re-analysis (e.g., fixing locations, re-classifying calls) without needing to re-download audio or re-transcribe.

---

## Transcription Quality Requirements

- **Speaker Attribution:** Every line of the transcript must be attributed to a speaker (Caller, Business, Voicemail System).
- **Call Type Detection:**
  - Person-to-person live conversation
  - Person-to-voicemail (caller leaves a message)
  - Full voicemail / no answer (no live interaction)
- **Dialogue Format:** Transcripts must read as structured dialogue, not raw text blocks.

---

## Dashboard Tab Reference

The Dashboard tab auto-calculates weekly summaries from Call Leads data:

| Column | Description |
|---|---|
| Week | Week number (Week 1, Week 2, etc.) |
| Week Start | Start date of the tracking week |
| Week End | End date of the tracking week |
| Answered | Count of "Answered" calls in the week |
| ABA10- | Count of "ABA10-" calls in the week |
| ABA10+ | Count of "ABA10+" calls in the week |
| Voicemail | Count of "Voicemail" calls in the week |
| Missed | Count of "Missed" calls in the week |
| Response Rate | Percentage calculated from the week's data |

---

## Message Leads — Column Definitions

The Message Leads tab stores data for message/text leads from Google LSA. Shared columns follow the same definitions as their Call Leads counterparts. The one unique column is **Response Status**.

**NOTE:** Columns A and B are intentionally left empty (spacing/formatting). Data begins at Column C.

### Column A: (Empty)
- Reserved — do not write to this column.

### Column B: (Empty)
- Reserved — do not write to this column.

### Column C: Date
- Identical to Call Leads Column C.
- **Source:** LSA API
- **Processing:** Direct pull
- **Description:** The date the message was received.
- **Format:** Date value as provided by LSA API.

### Column D: Phone Number
- Identical to Call Leads Column D.
- **Source:** LSA API
- **Processing:** Direct pull
- **Description:** The phone number of the lead (the person who sent the message).

### Column E: Service Type
- Identical to Call Leads Column F.
- **Source:** LSA API
- **Processing:** Direct pull with fallback
- **Description:** The service type associated with the message lead as labeled by LSA.
- **Fallback Rule:** If the field is blank or missing, set the value to `"No category"`.
- **Formatting Rule:** snake_case identifiers from the API are converted to Title Case (replace underscores with spaces, capitalize each word). Use a known display name map for common values.

### Column F: Feedback Choice
- Identical to Call Leads Column I.
- **Source:** Message content → AI classification
- **Processing:** AI classification
- **Description:** Whether the message represents a legitimate lead.
- **Allowed Values (dropdown):**
  - `Positive` — The message was a legitimate lead (e.g., service inquiry, quote request, appointment scheduling).
  - `Negative` — The message was not a legitimate lead (e.g., spam, solicitation, wrong number).
- **Rule:** Only `Positive` or `Negative`. Never neutral. Never blank.

### Column G: Response Status
- **Source:** LSA API (calculated from conversation event timestamps)
- **Processing:** Time delta calculation between the customer's initial message and the client's first reply
- **Description:** How quickly the client responded to the lead's initial message. This measures the client's responsiveness.
- **Allowed Values (dropdown):**

| Status | Definition | Criteria |
|---|---|---|
| `Responded within a few minutes` | Client replied promptly. | Time between the customer's initial message and the client's first reply is **≤ 30 minutes**. |
| `Responded within a few hours` | Client replied within a same-day timeframe. | Time between the customer's initial message and the client's first reply is **> 30 minutes AND ≤ 24 hours**. |
| `Responded after 24 hours` | Client replied late. | Time between the customer's initial message and the client's first reply is **> 24 hours**. |
| `Unresponded` | Client never replied. | No reply from the client found in the conversation data. |

- **Calculation Steps:**
  1. From the LSA conversation data for the lead, identify the timestamp of the **customer's first message** (the initial inbound message)
  2. Identify the timestamp of the **client's first reply** (the first outbound response from the business)
  3. Calculate the time difference between these two timestamps
  4. Classify into the appropriate bucket based on the thresholds above
  5. If no client reply exists in the conversation data, classify as `Unresponded`

### Column H: Charge Status
- Identical to Call Leads Column M.
- **Source:** LSA API (`lead_charged` field)
- **Processing:** Boolean mapping
- **Description:** Whether Google charged the client for this lead.
- **Mapping:**
  - `True` → `"Charged"`
  - `False` → `"Not Charged"`

### Column I: Lead Status
- Identical to Call Leads Column N.
- **Source:** LSA API (`lead_status` enum)
- **Processing:** Direct pull of enum name
- **Description:** The current status of the lead in Google LSA.
- **Allowed Values:**
  - `BOOKED` — Lead was booked by the client
  - `DECLINED` — Lead was declined by the client
  - `NEW` — Lead is new / unactioned
  - `ACTIVE` — Lead is currently active
  - `EXPIRED` — Lead expired without action
  - `DISABLED` — Lead was disabled
  - `CONSUMER_DECLINED` — Consumer declined the lead
  - `WIPED_OUT` — Lead was wiped out

---

## Processing Pipeline (Per Message)

The following pipeline is executed for each message lead record:

```
1. PULL message metadata from LSA API
   → date, phone number, service type, charge status, lead status
   → Filter: only leads where lead_type = 'MESSAGE'

2. PULL conversation events from LSA API
   → Retrieve all message events for the lead (customer messages + client replies)
   → Extract timestamps for response time calculation

3. CALCULATE Response Status
   → Identify timestamp of customer's first message
   → Identify timestamp of client's first reply (if any)
   → Classify: ≤30min / ≤24hr / >24hr / Unresponded

4. DETERMINE Feedback Choice
   → Analyze message content to classify as Positive or Negative

5. WRITE complete row to the Message Leads tab
```

**Note:** Unlike Call Leads, message leads do not require audio download, transcription, or AI-driven call analysis. The pipeline is simpler: pull metadata, calculate response time, classify feedback, and write.
