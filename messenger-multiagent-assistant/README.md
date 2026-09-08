# Messenger Multi-Agent AI Assistant

An n8n-based conversational AI system that receives Messenger/Instagram messages
(text, voice, or image), normalizes them into a common text format, and routes them
through a central orchestrator agent that delegates tasks to specialist sub-agents.

## Architecture

**Trigger & Input Handling**
- A Webhook node receives platform messaging events and handles the webhook
  verification handshake (`hub.challenge`)
- A Switch node routes incoming messages by type: text, audio attachment, or
  image attachment
- Audio is transcribed via OpenAI Whisper; images are analyzed via OpenAI vision —
  both are normalized into plain text before reaching the orchestrator, so the
  agent always reasons over text regardless of input channel

**Orchestrator**
- A GPT-5-mini agent acts as the single point of contact for the user, holding a
  per-user windowed conversation memory (keyed by sender ID) so it can resolve
  references like "reply to that same email" across turns
- Delegates to specialist tools rather than handling tasks directly, and never
  exposes internal tool/agent names to the end user

**Specialist Sub-Agents (Tools)**

| Agent | Responsibility |
|---|---|
| `Mail_Controller` | Send, read, delete, reply, and mark Gmail messages read/unread |
| `social_media` | Log and update rows in a Google Sheets content tracker |
| `Searching` | Look up facts via Wikipedia or live results via Google (SerpApi) |
| `Think` | Internal reasoning/planning step before acting |

**Output**
- If the original message was voice, the reply is synthesized back to audio and
  sent as a voice note
- Otherwise, the reply is sent back as a formatted text message
- The agent always replies in the same language the user wrote or spoke in

## Tech Stack
- **Orchestration**: n8n
- **LLM**: OpenAI GPT-5-mini
- **Speech-to-text**: OpenAI Whisper (transcription)
- **Text-to-speech**: OpenAI TTS
- **Vision**: OpenAI image analysis
- **Email**: Gmail API
- **Content tracking**: Google Sheets API
- **Web search**: SerpApi (Google Search) + Wikipedia

## Setup

1. Import `workflow.json` into your n8n instance (Workflows → Import from File)
2. Create and attach credentials for:
   - OpenAI (chat, transcription, vision, TTS)
   - Gmail (OAuth2)
   - Google Sheets (OAuth2)
   - SerpApi (API key)
3. Copy `.env.example` to `.env` and fill in your own values, or configure the
   equivalent credentials directly in n8n's credential manager
4. Set the Webhook node's path/URL and connect it to your Messenger/Instagram
   app's webhook configuration
5. Activate the workflow

## Notes
- `workflow.json` in this repo has all credential IDs and webhook IDs stripped
  and replaced with placeholders — you must reconnect your own credentials after
  importing.
- The Google Sheet used for `social_media` logging is not included; point the
  `Append or update row in sheet` node at your own sheet.
