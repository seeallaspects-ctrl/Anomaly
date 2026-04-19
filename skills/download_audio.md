# Skill: Download Audio

## Purpose
Download a call audio file from an LSA-provided URL for transcription processing.

## Inputs
- `audio_url` — The direct URL to the audio recording from the LSA API

## Outputs
- Audio file data ready for transcription processing

## Rules
- Handle common audio formats (MP3, WAV, OGG, etc.)
- If the download fails or the URL is invalid, return an error status — do not retry indefinitely
- Audio is used for transcription only and is not stored permanently
