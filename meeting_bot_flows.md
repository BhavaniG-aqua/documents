# Meeting Bot — High-Level Flows

## 1. Internal Meeting Join (Meeting_Assistant)

Uses the **Teams Media SDK (.NET)** to join as a first-party bot with per-speaker unmixed audio access.

```mermaid
flowchart TD
    A[Calendar Event Created in Outlook] --> B[Graph Webhook Notification]
    B --> C[Python Backend Schedules Meeting]
    C --> D[Scheduler Triggers Join at start - 1 min]
    D --> E[.NET Media Bot Joins via Graph API]
    E --> F[Teams Streams Per-Speaker PCM via TCP AudioSocket]
    F --> G[.NET Forwards Audio via WebSocket]
    G --> H[Silero VAD - Turn Detection]
    H --> I[Groq Whisper - whisper-large-v3]
    I --> J[Transcript Saved with Speaker Labels]
    J --> K[Meeting Ends]
    K --> L[Groq LLM - llama-3.3-70b-versatile - Summary]
```

**Key characteristics:**
- Per-speaker unmixed audio (no diarization needed)
- Requires Azure VM with static public IP + TLS certificate for TCP media
- Bot appears as a named participant in the meeting
- Real-time transcription with speaker identification

---

## 2. External Meeting Join (teams-acs-poc)

Uses **Azure Communication Services (ACS) Web Calling SDK** in a Chromium worker to join as an anonymous ACS participant.

```mermaid
flowchart TD
    A[Calendar Event / Manual Teams URL] --> C[FastAPI Creates ACS Identity + Token]
    C --> D[ACS Joins Meeting via Join URL]
    D --> E[ACS Receives Mixed Meeting Audio]
    E --> F[PCM Forwarded to Python via WebSocket]
    F --> G[Silero VAD - Turn Detection]
    G --> H[Active-Speaker Inference from ACS Roster]
    H --> I[Groq Whisper - whisper-large-v3]
    I --> J[Transcript Saved with Inferred Speaker Labels]
```

**Key characteristics:**
- Mixed audio (single stream, speaker attribution via active-speaker inference)
- No TCP port / TLS certificate required (WebSocket-based audio delivery)
- Works for external/cross-tenant meetings without admin consent in the other tenant
- One Chromium worker per concurrent meeting

---

## Models Used

| Purpose | Model | Provider |
|---------|-------|----------|
| Speech-to-Text | `whisper-large-v3` | Groq |
| Meeting Summary | `llama-3.3-70b-versatile` | Groq |
