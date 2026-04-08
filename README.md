# VoxProbe — Adversarial QA for Voice AI Agents

VoxProbe is an adversarial testing harness for production voice AI agents. It places real outbound phone calls in which an LLM-powered "patient" persona runs scripted probing scenarios against a target voice agent, records the full two-sided transcript, and automatically flags behavioral bugs (hold loops, missing escalations, weekend bookings, identity-verification loops, etc.) using pattern-based detectors.

I built this as a personal exploration of a problem I kept running into while playing with voice agents: **how do you actually QA a non-deterministic, voice-driven AI?** Manually calling and probing edge cases is slow, hard to reproduce, and impossible to grade consistently. VoxProbe turns that into a one-click test you can re-run after every prompt or model change.

The included demo configures the patient bot for a medical-office scheduling domain (12 scenarios covering edge cases, escalation paths, and standard flows) — but the framework is domain-agnostic. Swap in a different set of scenarios and you can probe any voice agent.

## How it works

```
┌──────────────┐    POST /call     ┌──────────────┐   PSTN call   ┌────────────┐
│  React UI    │ ────────────────▶ │  FastAPI     │ ────────────▶ │  Target    │
│  Dashboard   │                   │  Backend     │               │  Voice     │
│              │ ◀──── transcript ─│  + Mongo     │ ◀──transcript │  Agent     │
└──────────────┘                   └──────┬───────┘               └────────────┘
                                          │
                                          ▼
                                  ┌────────────────┐
                                  │ Vapi (STT +    │
                                  │ GPT-4o-mini +  │
                                  │ TTS + telephony)│
                                  └────────────────┘
```

When a test starts, the FastAPI backend POSTs a single payload to Vapi containing a scenario-specific system prompt, persona, and opening line. Vapi handles the entire voice stack — telephony, speech-to-text, the patient-bot LLM (GPT-4o-mini), and text-to-speech — running the call autonomously. When the call ends, a background asyncio poller fetches the transcript from Vapi's API, runs the bug-detection pass against it, and persists everything to MongoDB. The React dashboard pulls from the backend to display call history, full two-sided transcripts, and a triaged bug report tracker.

I chose Vapi because it collapses the entire voice stack into one API call — no separate Twilio + transcription + TTS wiring. The transient-assistant pattern (a fresh assistant per call, configured inline) means every scenario gets its own system prompt without managing persistent assistants on Vapi's dashboard.

## Tech stack

- **Backend**: Python / FastAPI / asyncio
- **Frontend**: React 19 / Tailwind CSS / shadcn/ui
- **Database**: MongoDB (via Motor async driver)
- **Voice pipeline**: Vapi AI (telephony + STT + LLM + TTS)
- **LLM**: OpenAI GPT-4o-mini (driving the patient bot, via Vapi)
- **Voice**: OpenAI TTS (alloy)

## Features

- **12 demo scenarios** covering edge cases, escalation paths, and standard flows
- **9 automatic bug-detection patterns** — hold loops, missing tickets, weekend bookings, failed escalations, record lookup failures, excessive verification, missing symptom triage, and more
- **Full transcript recording** with both patient and agent sides preserved
- **Background transcript polling** that fires automatically after each call
- **Triaged bug reports** with severity levels (critical / high / medium / low) and recommendations
- **Re-run detection** endpoint to apply new patterns retroactively to historical calls
- **React dashboard** with Dashboard, Transcripts, and Bug Reports tabs

## Setup

### Prerequisites

- Python 3.10+
- Node.js 18+
- MongoDB (local or cloud)
- A [Vapi](https://vapi.ai) account with a phone number
- [ngrok](https://ngrok.com) (or any tunnel) to expose localhost for Vapi webhooks

### Backend

```bash
cd backend
pip install -r requirements.txt
cp .env.example .env
```

Fill in `.env`:

| Variable | Description |
|----------|-------------|
| `MONGO_URL` | MongoDB connection string |
| `DB_NAME` | Database name |
| `VAPI_API_KEY` | Your Vapi API key |
| `VAPI_PHONE_NUMBER_ID` | Your Vapi phone number ID |
| `BACKEND_URL` | Public URL for webhooks (your ngrok URL) |
| `TARGET_NUMBER` | E.164 phone number of the agent under test |

Start the server:

```bash
uvicorn server:app --host 0.0.0.0 --port 8000
```

### Frontend

```bash
cd frontend
npm install --legacy-peer-deps
cp .env.example .env
npm start
```

### Expose backend for webhooks

```bash
ngrok http 8000
```

Copy the `https://...ngrok-free.app` URL into `backend/.env` as `BACKEND_URL`.

## API endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/scenarios` | List all test scenarios |
| GET | `/api/bug-patterns` | List auto-detection patterns |
| POST | `/api/call` | Initiate an outbound test call via Vapi |
| GET | `/api/calls` | List all call records |
| GET | `/api/calls/{id}` | Get a specific call |
| GET | `/api/calls/{id}/transcript` | Get transcript + auto-detected bugs |
| GET | `/api/calls/{id}/vapi-transcript` | Force-fetch transcript from Vapi |
| POST | `/api/rerun-bug-detection` | Re-run detection across all historical calls |
| POST | `/api/bugs` | Create a manual bug report |
| GET | `/api/bugs` | List all bug reports |
| DELETE | `/api/bugs/{id}` | Delete a bug report |
| GET | `/api/config/status` | Check Vapi configuration status |

## Demo scenarios (medical-office domain)

| Category | Scenario | What it tests |
|----------|----------|---------------|
| Edge Case | Sunday Appointment Trap | Booking on a closed day |
| Edge Case | Availability Timeout Probe | Hold loop / timeout handling |
| Edge Case | Off-Topic Guardrail Test | Scope boundaries |
| Escalation | Urgent Medication Refill | Emergency escalation paths |
| Escalation | Speak to Human Request | Human handoff handling |
| Standard | Ticket Number Demand | Documentation completeness |
| Standard | Beyond One Week Availability | Alternative scheduling |
| Standard | Insurance Verification | Coverage questions |
| Standard | Cancel and Rebook Same Call | State management |
| Standard | Interruption Handling | Context switching |
| Standard | Backache Triage | Symptom clarification |
| Standard | No Insurance Scenario | Self-pay handling |

Each scenario is just a Python dict with `name`, `persona`, `goal`, `opening`, and `probing_instructions` fields — adding new ones (or swapping in a non-medical domain) is a few lines.

## Bug detection patterns

| Pattern | Severity | Trigger |
|---------|----------|---------|
| Infinite Hold Loop | Critical | "please hold" / "let me check" 3+ times without results |
| Documented Without Reference | High | Claims documentation but no ticket number |
| Weekend Appointment Booked | High | Confirms Saturday/Sunday appointment |
| No Alternative Timeframe | Medium | Can't check availability, no alternatives offered |
| Technical Error Without Escalation | High | System error without concrete escalation path |
| Failed Escalation — No Alternative | High | Live transfer unavailable, no alternative contact given |
| Patient Record Not Found — No Recovery | Medium | Can't find record, doesn't offer to create one |
| Excessive Identity Verification | Medium | Asks for name/DOB/phone 4+ times before helping |
| No Symptom Clarification | Medium | Patient mentions symptoms, agent skips triage questions |

Three detection styles are supported:
- **Simple match** — regex fires on a single agent utterance
- **Threshold** — pattern must occur N+ times across the conversation
- **Requires-missing** — pattern fires *only if* a remediation phrase (e.g. "phone number", "alternative", "new patient") is absent from the entire agent transcript

## Sample findings

See [BUG_REPORT.md](./BUG_REPORT.md) for an example bug report generated by running VoxProbe against a real production voice agent in the medical-scheduling space.

## Testing

```bash
python backend_test.py
```

## Why I built this

Voice AI is shipping into production fast — medical scheduling, customer support, appointment booking — but the QA tooling lags far behind. You can't write a unit test for "the agent gracefully escalates when a refill request errors out." VoxProbe is my attempt to close that gap with adversarial LLM testing: one AI agent finding bugs in another, with every call captured as a reproducible artifact.
