# Practice Problem: Design a Real-Time AI Voice Assistant

> Difficulty: ⭐⭐⭐⭐ | Key Concepts: Streaming, latency, speech-to-text, TTS, state management

## Problem Statement

Design a real-time voice assistant for a healthcare clinic. Patients call to schedule appointments, check test results, and ask medical questions. The system must feel like talking to a human — low latency, natural speech, and context-aware responses.

## Requirements

### Functional
- Telephone-based (integrate with existing phone system via SIP/Twilio)
- Handle: appointment scheduling, test result inquiries, general medical FAQ
- Access patient records (after identity verification)
- Transfer to human nurse for clinical questions
- Support English and Spanish
- Record and transcribe all calls for compliance

### Non-Functional
- Voice-to-voice latency < 800ms (feels conversational)
- Handle 100 concurrent calls
- Available 24/7 with 99.9% uptime
- Cost < $0.50 per call (average 3-minute call)
- HIPAA compliant

## Constraints
- Patient data is PHI (Protected Health Information) — strict HIPAA requirements
- Must handle interruptions (patient talks while assistant is speaking)
- Background noise in calls (waiting rooms, cars)
- Cannot provide medical diagnoses or medication recommendations
- Must verify patient identity before accessing records

## Hints

1. 800ms total budget: STT + LLM + TTS. How do you fit?
2. How do you handle "barge-in" (patient interrupts the assistant)?
3. HIPAA: Can you use cloud STT/TTS? What about the LLM?
4. Identity verification by voice — what's secure enough?

## Solution Walkthrough

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│               Voice Pipeline                             │
│                                                         │
│  Phone ──→ Twilio ──→ WebSocket ──→ Voice Server        │
│                                        │                 │
│                           ┌────────────┤                 │
│                           ▼            │                 │
│                     ┌──────────┐       │                 │
│                     │  STT     │       │                 │
│                     │ Deepgram │       │                 │
│                     │ streaming│       │                 │
│                     └────┬─────┘       │                 │
│                          │             │                 │
│                     ┌────▼─────┐       │                 │
│                     │  Agent   │       │                 │
│                     │  (Haiku) │       │                 │
│                     │ streaming│       │                 │
│                     └────┬─────┘       │                 │
│                          │             │                 │
│                     ┌────▼─────┐       │                 │
│                     │  TTS     │       │                 │
│                     │ ElevenLabs│      │                 │
│                     │ streaming│       │                 │
│                     └────┬─────┘       │                 │
│                          │             │                 │
│                          ▼             │                 │
│                     Audio out ─────────┘                 │
└─────────────────────────────────────────────────────────┘
```

### Key Decisions

**Latency Budget (800ms total):**
```
STT (streaming):     150ms  (Deepgram Nova-2, streaming mode)
LLM (TTFT):          200ms  (Haiku — fastest available)
TTS (streaming):     150ms  (ElevenLabs Turbo v2, streaming)
Network + processing: 100ms
Buffer:              200ms

Total:               800ms ✓
```

**Why Haiku, not Sonnet:** Latency is everything in voice. Sonnet TTFT is 300-500ms — too slow. Haiku TTFT is 100-200ms. For appointment scheduling and FAQ, Haiku quality is sufficient.

**Streaming Pipeline:** Every component streams. STT streams partial transcripts → LLM generates as transcript arrives → TTS converts first sentence while LLM generates second. Pipeline parallelism cuts perceived latency 60%.

**Barge-In Handling:**
```
Patient starts talking while assistant speaks
  → VAD (Voice Activity Detection) detects speech
  → Immediately stop TTS playback
  → Buffer patient speech
  → When patient stops → process their input
  → Resume conversation from new context
```

**Identity Verification:**
1. Ask for date of birth + last 4 of phone number
2. Verify against patient database
3. Only then allow access to records
4. Never read full SSN, account numbers, or test values aloud — send to patient portal instead

**HIPAA Compliance:**
- STT/TTS: Use HIPAA-compliant tiers (Deepgram HIPAA, ElevenLabs Enterprise)
- LLM: Anthropic HIPAA BAA for Claude Haiku
- Call recordings: Encrypted at rest, stored in HIPAA-compliant storage
- PHI never logged in plain text — redact before logging
- All vendors must sign BAA (Business Associate Agreement)

**State Management:**
```python
class CallState:
    patient_verified: bool = False
    patient_id: str | None = None
    intent: str | None = None  # scheduling, results, faq
    conversation_history: list[Message]  # last 10 turns
    transfer_requested: bool = False
```

**Cost Breakdown:**
```
Average 3-minute call:

STT (Deepgram):    3 min × $0.0059/min = $0.018
LLM (Haiku):       ~2000 tokens total  = $0.002
TTS (ElevenLabs):  ~500 chars output   = $0.015
Twilio (phone):    3 min × $0.02/min   = $0.060

Total: ~$0.095 per call ✓ (well under $0.50)
```

**Escalation to Human:**
- Medical symptoms described → transfer to nurse
- Patient frustrated (3+ failed attempts) → transfer
- Explicit request: "talk to a person" → transfer
- Sensitive results (abnormal lab values) → transfer (never read aloud by AI)
