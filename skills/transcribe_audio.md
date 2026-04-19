# Skill: Transcribe Audio

## Purpose
Convert a call audio file into a speaker-attributed transcript with call type detection.

## Inputs
- Audio file data (from Download Audio skill)

## Outputs
- `call_type` — One of: "Live Conversation", "Person-to-Voicemail", "Voicemail Only", "No Audio"
- `transcript` — Full speaker-attributed transcript in dialogue format

## Transcript Format
Each line must be attributed to a speaker:
- `Caller:` — The person calling in (the lead)
- `Business:` — The client's side of the conversation
- `Voicemail System:` — Automated voicemail greetings or system messages

Example (live conversation):
```
Caller: Hi, I need a quote for pool cleaning.
Business: Sure, what's your address?
Caller: 456 Oak Lane.
```

Example (voicemail):
```
Voicemail System: You've reached Bluewater Pool Care. Leave a message after the tone.
Caller: Hi, calling about a pool leak. Please call back at 555-9876.
```

## Call Type Detection Rules
- **Live Conversation** — Both Caller and Business speak
- **Person-to-Voicemail** — Voicemail greeting plays, caller leaves a message
- **Voicemail Only** — Voicemail greeting plays, caller does NOT leave a message (hangs up)
- **No Audio** — Audio is silent, too short, or unprocessable

## Rules
- Transcribe verbatim — do not paraphrase, summarize, or clean up speech
- Speaker diarization must clearly separate who is talking
- Detect voicemail greetings vs. live answers accurately
