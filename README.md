# Multimodal Content Moderation for ACME Customer Service

An AI-powered moderation system for customer-service conversations at the fictional ACME Enterprise.
A trainee agent chats with an LLM that plays an angry customer whose *ACME Power Widget Pro* stopped working.
Every message and file the trainee sends (text, image, video, audio) is moderated **before** it reaches the customer.
Flagged content is blocked and the reason is shown to the trainee.

**Submission evidence:** [`RUN_EVIDENCE.pdf`](RUN_EVIDENCE.pdf), which contains the test output, one eval run, a flagged Gradio conversation and the Phoenix traces.

## What it detects

| Modality | Flags |
|---|---|
| Text | `contains_pii`, `is_unfriendly`, `is_unprofessional` |
| Image | `contains_pii`, `is_disturbing`, `is_low_quality` |
| Video | `contains_pii`, `is_disturbing`, `is_low_quality` |
| Audio | `transcription`, `contains_pii`, `is_unfriendly`, `is_unprofessional` |

Every result also includes a `rationale` that explains the decision.

## Architecture

```
Trainee (browser)
   │
   ▼
Gradio chat UI (gradio_app.py, port 7860)
   │  every text and file  ──HTTP──►  FastAPI backend (fastapi_app.py, port 8000)
   │                                      │
   │                                      ▼
   │                         Moderation agents (Pydantic AI + Gemini)
   │                         text / image / video / audio
   │  ◄── flagged? show rationale, do NOT forward
   │
   ▼ (only if safe)
Customer agent (customer_agent.py), with past messages passed as history

All of it is traced with OpenTelemetry into Arize Phoenix (port 6006).
```

- **Structured outputs** (`multimodal_moderation/types/moderation_result.py`): a base `ModerationResult` with only `rationale`, plus one subclass per modality whose flags are required `bool` fields.
- **Moderation agents** (`multimodal_moderation/agents/`): each is a Pydantic AI `Agent` with its own instructions and output type. Image, video and audio are sent to Gemini as `BinaryContent`.
- **Chat UI** (`gradio_app.py`): a multimodal `gr.ChatInterface`. The Pydantic AI message history is kept in `gr.State` so the customer stays consistent across turns.
- **Tracing**: all spans use the tracer from `tracing.py`. A `conversation` span (with `session.id`) stays open for the whole chat and contains one `chat_turn` span per turn (with `session.id` and the input). Each turn has `moderate_text` / `moderate_<media>` child spans (moderation input and output). When content is flagged, a `feedback` child span is created under `chat_turn` with the `feedback` text and `session.id`; otherwise an `llm_customer` span records the Gemini call.

## Setup

Requires [uv](https://docs.astral.sh/uv/) and Python 3.12.

```bash
uv sync --dev
uv pip install -e .
cp env.example .env
```

Edit `.env`:
- `GEMINI_API_KEY`: your Gemini key.
- `USER_API_KEY`: any non-empty value, for example `my-api-key`. The backend uses it as a bearer token.
- With a Udacity Vocareum key (`voc-...`), keep `GOOGLE_GEMINI_BASE_URL=https://gemini.vocareum.com`. With a Google AI Studio key, delete that line.
- On your own machine, delete `PHOENIX_HOST_ROOT_PATH` and `API_ROOT_PATH`; they are only needed in the Udacity workspace.
- Optional: if something else is already listening on `localhost:8000` (for example Docker), set `API_BASE_URL` to an address where this backend is reachable.

## Run the tests

```bash
uv run pytest tests/ -vv --ignore=tests/test_env_setup.py   # project tests, no Gemini calls
uv run pytest tests/test_env_setup.py -v                     # checks your .env and makes one real Gemini call
```

## Run the evals

The evals call Gemini for real: once for the agent and once for an `LLMJudge` that grades the rationale. Scores vary between runs, so a score below 100% is expected.

```bash
uv run evals/text/test_cases.py
uv run evals/image/test_cases.py
uv run evals/audio/test_cases.py
uv run evals/video/test_cases.py
```

`EVAL_NUM_REPEATS` in `.env` sets how many times each case runs.

## Run the app

```bash
uv run multimodal-moderation
```

- Chat UI: http://localhost:7860/
- Phoenix traces: http://localhost:6006/projects
- API docs: http://localhost:8000/docs (click **Authorize** and use your `USER_API_KEY`)

Example conversation:

```
YOU>      Welcome to ACME Customer Service. How can I help?
CUSTOMER> ...complains that the widget stopped working...
YOU>      I am sorry but I absolutely cannot offer a refund
          -> flagged as unfriendly/unprofessional; the rationale is shown, nothing is sent to the customer
YOU>      I am authorized to offer you a replacement. Would you be willing to accept it?
```

Click **End Conversation** to close the `conversation` span so the whole trace shows in Phoenix.

## Project structure

```
multimodal_moderation/
  agents/            text, image, video, audio moderation agents + customer agent
  types/             Pydantic result models and model choice
  gradio_app.py      chat UI (frontend)
  fastapi_app.py     REST API (backend)
  tracing.py         Phoenix / OpenTelemetry setup
  app.py             starts Phoenix, the API and the chat UI together
evals/               Pydantic Evals suites per modality + test data
tests/               unit tests
RUN_EVIDENCE.pdf     submission evidence
```
