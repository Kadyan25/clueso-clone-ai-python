# Clueso Clone – Python AI Service (FastAPI Stub)

## Overview

This repository is a **Python AI service stub** for the Clueso clone. It is based on the reference [`ProductAI`](https://github.com/Tushar7436/ProductAI) project and represents the **intended AI layer** between the Node backend and external AI providers (LLM + TTS).

For this assignment:

- The **Node backend** currently uses a **mock AI** implementation for `/sessions/:id/process`.
- This Python service is documented as an **architectural boundary** and is not fully wired end‑to‑end yet, mainly due to audio dependencies on the local environment.

---

## Goals

- Expose a FastAPI HTTP API that the Node backend can call.
- Accept structured session data (events, text, metadata).
- Call external AI providers (LLM + TTS) to:
  - Generate a clear script for the recording.
  - Optionally generate audio (voiceover).
- Return structured JSON back to the Node backend.

Even if the full AI stack is not running, the repo shows how this layer would fit into the system.

---

## Tech Stack

- **Framework:** FastAPI
- **Runtime:** Python 3 (reference project was built for <= 3.11)
- **HTTP Server:** Uvicorn
- **AI (intended):**
  - Text generation via Gemini or similar LLM
  - Text‑to‑speech via ElevenLabs
- **Audio processing (intended):**
  - `pydub` and related libraries

> On the current machine (Python 3.13 on Windows), `pydub` expects `pyaudioop`, which is not available, causing import/runtime errors. This is the main reason full audio integration is deferred.

---

## Architecture & Integration Plan

### Role in the system

Intended flow:

1. Chrome extension → Node backend (session + events).
2. Node backend → Python AI service (structured request).
3. Python AI service → LLM + TTS → returns script + audio info.
4. Node backend → persists fields (`scriptText`, `audioFileName`) on the Session.
5. Frontend → displays script/audio for the user.

### Example Endpoint Design

These endpoints describe the **target contract** (even if implementation is partial).

**POST `/summaries`**

- **Request body example:**

{
"sessionId": 1,
"events": [
{ "timestamp": "2025-01-01T10:00:00Z", "url": "https://example.com", "type": "navigate" }
],
"userInstructions": "Explain this flow for a product manager."
}

text

- **Response example:**

{
"title": "Onboarding flow for Example App",
"short_summary": "This session shows how to sign up and configure the basic settings.",
"detailed_steps": [
"Open the Example App homepage.",
"Click on Sign Up and fill the registration form.",
"Confirm your email and log in for the first time."
],
"persona_variant": {
"pm": "From a PM perspective, this flow introduces the core onboarding steps.",
"developer": "From a developer perspective, note the API calls for signup and verification."
}
}

text

**POST `/tts` (optional)**

- Accepts `{ "sessionId": 1, "script": "..." }`.
- Returns `{ "audioFileName": "session-1-voiceover.mp3" }`.

These endpoints would be called from the Node backend when processing a session.

---

## Current Status

- Repo is a **local clone** of the reference `ProductAI` project.
- Project structure (typical FastAPI layout):

clueso-clone-ai-python/
app/
main.py # FastAPI entrypoint (app = FastAPI())
routes/... # Endpoint definitions
services/... # AI, TTS, audio utilities
requirements.txt
test_events.json # Example input data for testing
...

text

- Attempts to run:

python -m uvicorn app.main:app --reload --port 8000

text

- Current blockers on the local environment:
  - `ModuleNotFoundError: No module named 'pyaudioop'` raised inside `pydub/utils.py`.
  - Python 3.13 + Windows combination does not ship `audioop` / `pyaudioop`, so `pydub` cannot function as expected without additional workarounds.

Because of this, full TTS/audio is **not** wired, and the Node backend uses a mock implementation instead.

---

## How the Node Backend Would Use This

In the Node backend, the future implementation of `/sessions/:id/process` would:

1. Fetch the `Session` and related events from the database.
2. Call the Python service, for example:

// pseudo-code in Node
const response = await axios.post(
${PYTHON_LAYER_URL}/summaries,
{
sessionId: session.id,
events,
userInstructions: "Explain this as a product walkthrough."
},
{ timeout: PYTHON_SERVICE_TIMEOUT }
);

const { title, short_summary, detailed_steps } = response.data;

// Save on the Session model
session.scriptText = short_summary + '\n\n' + detailed_steps.join('\n');
session.audioFileName = null; // or from /tts endpoint

text

3. Update `status` to `READY` and return the updated session to the frontend.

The current mock logic in Node mirrors this shape so that switching to the real Python service later should be straightforward.

---

## Running Locally (Attempted Steps)

> These steps describe what was attempted; on the current environment they fail due to audio deps. They are preserved to show the intended setup.

1. **Create virtual environment**

cd clueso-clone-ai-python
python -m venv .venv

On Windows:
.venv\Scripts\activate

On macOS/Linux:
source .venv/bin/activate

text

2. **Install dependencies**

pip install -r requirements.txt

text

3. **Run the API server**

uvicorn app.main:app --reload --port 8000

text

4. **Test basic health endpoints**

- `GET http://localhost:8000/` (if defined)
- `POST http://localhost:8000/summaries` with a minimal JSON payload

On the current machine this fails with `ModuleNotFoundError: No module named 'pyaudioop'` coming from `pydub`.

---

## Environment & Compatibility Notes

- Reference project expects a Python version where `audioop` / `pyaudioop` is available (e.g. 3.10–3.11 on Linux/macOS).
- On Windows + Python 3.13:
  - `pydub` cannot import `pyaudioop`, which blocks full audio processing.
- External services (Gemini, ElevenLabs) require API keys set via environment variables (see original `ProductAI` README for exact names).

For this assignment, those keys are **not required**, because:

- The Node backend does not yet call this service in production.
- All “AI” behavior is mocked on the Node side for stability.

---

## Future Work

If time and environment allow, the next steps would be:

1. **Create a minimal audio‑free FastAPI stub**

   - Implement a simple `/summaries` endpoint that:
     - Accepts `{ sessionId, events, userInstructions }`.
     - Returns hard‑coded or lightly processed summary text.
   - Remove `pydub` / audio usage from this stub so it works on Python 3.13.

2. **Wire Node → Python**

   - In the Node backend, replace the mock logic in `/sessions/:id/process` with:
     - HTTP call to `PYTHON_LAYER_URL + '/summaries'`.
     - Fallback to local mock if the Python service is down.

3. **Gradually reintroduce audio**

   - On a compatible environment (e.g. Linux VM or container with Python 3.11), restore `pydub` and TTS logic.
   - Add a `/tts` endpoint and connect it to ElevenLabs.

---
