# Skill: Classify Message Feedback

## Purpose
Classify a message lead as Positive or Negative by sending the message text and service type to the Claude API for analysis.

## Inputs
- `message_text` — The customer's message text (from the first CUSTOMER message in conversation_events)
- `service_type` — The service type associated with this lead (e.g., "Pool Repair Tile")

## Outputs
- `feedback_choice` — `"Positive"` or `"Negative"` only

## Classification Criteria

### Positive
The message is a legitimate lead — someone genuinely seeking the business's services:
- Service inquiry or question about services offered
- Quote or estimate request
- Appointment or scheduling request
- Follow-up on a previous service interaction
- Request for availability or pricing

### Negative
The message is NOT a legitimate lead:
- Spam or automated message
- Solicitation (selling services TO the business)
- Wrong number or misdirected message
- Irrelevant or nonsensical content
- Marketing or advertising pitch

## API Implementation

### Claude API Call
```python
import requests

resp = requests.post(
    'https://api.anthropic.com/v1/messages',
    headers={
        'x-api-key': os.getenv('ANTHROPIC_API_KEY'),
        'anthropic-version': '2023-06-01',
        'content-type': 'application/json'
    },
    json={
        'model': 'claude-sonnet-4-20250514',
        'max_tokens': 10,
        'messages': [{'role': 'user', 'content': prompt}]
    }
)
feedback_choice = resp.json()['content'][0]['text'].strip()
```

### Prompt Template
```
Classify this message lead as either "Positive" or "Negative".

Positive = legitimate lead (service inquiry, quote request, appointment, scheduling, follow-up, pricing question)
Negative = not a legitimate lead (spam, solicitation, wrong number, irrelevant content, marketing pitch)

Service Type: {service_type}
Customer Message: "{message_text}"

If ambiguous, lean toward Positive.

Respond with ONLY one word: Positive or Negative
```

## Rules
- Send the message text and service type to the Claude API for classification
- Only return `"Positive"` or `"Negative"` — never neutral, never blank
- Use the service type as context to help determine if the message is relevant to the business
- If the message is ambiguous, lean toward `"Positive"` (benefit of the doubt for potential leads)
- The Claude API call uses the `ANTHROPIC_API_KEY` from `.env`
- Model: `claude-sonnet-4-20250514`
- Max tokens: `10` (only need a single word response)
