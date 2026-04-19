# Skill: Analyze Call

## Purpose
Analyze a call transcript and produce structured data for the Google Sheet columns. Always reference `references/sheet_template_spec.md` for definitions and rules.

## Inputs
- `transcript` — Speaker-attributed transcript (from Transcribe Audio skill)
- `call_type` — The detected call type
- `call_duration` — Duration in seconds (from LSA metadata)
- `answered` — Whether the call was answered (from LSA metadata)

## Outputs

### location
- Extract the caller's location from the transcript if mentioned (city, area, neighborhood, zip code, address, etc.)
- Normalize to **City, State** format only (e.g., "Dallas, Texas", "Lake Worth, Florida") — do not include street addresses or zip codes
- If no location is found in the transcript, return `"Not Provided"`
- The LSA API does not provide location — this must come from the transcript only

### call_summary
- 3 sentences or less
- Must cover: what the call was about, what the caller wanted, whether answered, the outcome, the objective

### feedback_choice
- `"Positive"` or `"Negative"` only
- Positive = legitimate lead (service inquiry, quote request, appointment, follow-up)
- Negative = not a legitimate lead (spam, solicitation, scam, wrong number, robocall)

### phone_call_status
Classify using BOTH call metadata and transcript:

| Status | Criteria |
|---|---|
| Answered | Call was answered by a live person on the business side |
| Missed | Duration > 20 seconds AND not answered AND no voicemail. Genuine missed call — caller waited but nobody picked up. |
| Voicemail | Not answered, caller left a voicemail message |
| ABA10- | Duration < 10 seconds AND not answered AND no voicemail (misdial) |
| ABA10+ | Duration >= 10 seconds AND <= 20 seconds AND not answered AND no voicemail (brief intent) |

Classification guardrails:
- Do NOT select ABA10- if answered, went to voicemail, or duration >= 10s
- Do NOT select ABA10+ if answered, went to voicemail, duration < 10s, or duration > 20s
- Do NOT select Missed if answered, went to voicemail, or duration <= 20s (use ABA instead)
- Do NOT select Missed if voicemail was left (use Voicemail)

### feedback_reason
- One sentence explaining the Positive/Negative classification
- Must directly reference what happened in the call

## Rules
- Never output a value outside the allowed options for any field
- Never leave any output blank
- The transcript is the primary source for summary, sentiment, and reason
- Call metadata (duration, answered) is the primary source for status classification
