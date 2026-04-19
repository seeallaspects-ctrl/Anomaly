# Skill: Calculate Response Status

## Purpose
Determine how quickly the business responded to a customer's message by analyzing conversation event timestamps.

## Inputs
- `conversation_events` — Array of message events, each containing:
  - `timestamp` — UTC timestamp of the message
  - `sender_type` — `"CUSTOMER"` or `"BUSINESS"`
  - `message_text` — The text content of the message

## Outputs
- `response_status` — One of the following classification strings (must match the Google Sheet data validation dropdown values exactly):
  - `"Responded within few mins"` — Business replied within 30 minutes
  - `"Responded within few hours"` — Business replied after 30 minutes but within 24 hours
  - `"Responded after 24 hrs"` — Business replied after more than 24 hours
  - `"Unresponded"` — Business never replied

## Classification Logic
1. Find the **first message** where `sender_type = "CUSTOMER"` — this is the customer's initial message timestamp
2. Find the **first message** where `sender_type = "BUSINESS"` that occurs **after** the customer's first message — this is the business's first reply timestamp
3. Calculate the time delta between the customer's first message and the business's first reply
4. Classify based on the delta:
   - Delta ≤ 30 minutes → `"Responded within few mins"`
   - Delta > 30 minutes AND ≤ 24 hours → `"Responded within few hours"`
   - Delta > 24 hours → `"Responded after 24 hrs"`
5. If no `BUSINESS` message exists after the customer's first message → `"Unresponded"`

## Timestamp Format
Timestamps from the LSA API follow the format `"YYYY-MM-DD HH:MM:SS.ffffff"` (with microseconds).

Parse with:
```python
from datetime import datetime
dt = datetime.strptime(timestamp, '%Y-%m-%d %H:%M:%S.%f')
```

Delta calculation:
```python
delta_minutes = (business_reply_time - customer_message_time).total_seconds() / 60
```

## Rules
- All timestamps are in UTC — compare directly without time zone conversion
- Only consider the **first** customer message and the **first** business reply after it
- If there are no customer messages in the conversation, return `"Unresponded"`
- Never return a value outside the four allowed classification strings
