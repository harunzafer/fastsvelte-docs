---
description: "FastSvelte AI integration guide — the LLMClient interface, OpenAI setup, the sample note copilot, streaming responses, and swapping LLM providers."
keywords: "fastsvelte ai, llm integration, openai, responses api, streaming, copilot, fastapi ai, ai saas starter kit"
---

# AI Integration

FastSvelte ships a working, end-to-end AI feature so you can build your own on top of it instead of from scratch: a small provider-agnostic `LLMClient` seam, an OpenAI implementation, and a sample **note copilot** (Improve / Summarize) that streams its output to the UI. Every call is metered and billed — see **[AI Usage & Credit Billing](ai-billing.md)**.

## Setup

Configure the OpenAI key and model in `backend/.env`:

```bash
FS_OPENAI_API_KEY="sk-proj-your-openai-api-key"
FS_OPENAI_MODEL="gpt-5-mini"
```

`FS_OPENAI_MODEL` defaults to **`gpt-5-mini`**. The model name flows from settings into the client provider in `backend/app/config/container.py`:

```python
openai_client = providers.Singleton(
    OpenAIClient,
    model=settings.openai_model,
    temperature=0.1,
    api_key=settings.openai_api_key,
)
```

Whatever model you set **must have a row in the `model_price` table**, or usage-cost calculation fails — see [AI Usage & Credit Billing](ai-billing.md#model-pricing).

## The `LLMClient` interface

App code never imports the OpenAI SDK directly. It depends on a small `Protocol` in `backend/app/service/llm_client.py`:

```python
class LLMClient(Protocol):
    async def structured(
        self, messages: list[dict], model: Type[T]
    ) -> tuple[T, TokenUsage]: ...

    def stream(
        self, messages: list[dict]
    ) -> AsyncIterator[str | TokenUsage]: ...
```

- **`structured()`** returns a parsed Pydantic model plus the call's `TokenUsage`. Usage travels with every response because every call is billable.
- **`stream()`** yields plain-text chunks and, as its **final item**, the call's `TokenUsage` (captured from the terminal stream event). The caller accumulates text for the UI and forwards that final `TokenUsage` to billing.

This seam is the whole point: swap providers by writing another `LLMClient`, and nothing else in the backend changes.

## OpenAI implementation

`backend/app/service/openai_client.py` implements `LLMClient` against the OpenAI **Responses API** (`client.responses.parse` for structured output, `client.responses.stream` for streaming) — not the older Chat Completions surface. Token usage is read from the response and returned as `TokenUsage(provider, model, input_tokens, output_tokens, total_tokens)`.

## The sample copilot

`backend/app/service/copilot_service.py` exposes two actions, **Improve** and **Summarize**, each accepting an optional `tone`. They build a system+user message pair and return `llm_client.stream(...)`.

They're surfaced on the note routes in `backend/app/api/route/note_route.py`:

| Endpoint | Body | Role | Response |
|----------|------|------|----------|
| `POST /notes/{id}/copilot/improve` | `{ "tone"?: string }` | `MEMBER` | streamed `text/plain` |
| `POST /notes/{id}/copilot/summarize` | `{ "tone"?: string }` | `MEMBER` | streamed `text/plain` |

Each handler loads the note (404 if missing), checks the org has AI capacity before streaming (estimated from the note length), then returns a `StreamingResponse`. The stream is wrapped by the billing service so the final usage is recorded automatically — see [AI Usage & Credit Billing](ai-billing.md).

## Streaming to the frontend

The frontend consumes the `text/plain` stream incrementally in `frontend/src/lib/api/copilotStream.ts`, appending chunks as they arrive so the user sees output render live. The copilot toolbar wires the Improve / Summarize buttons to these calls.

## Adding your own AI action

1. Add a method to `CopilotService` (or a new service) that builds messages and calls `llm_client.stream(...)` or `.structured(...)`.
2. Add a route that checks capacity, then wraps the stream with `AiUsageBillingService.stream_and_record(...)` (streaming) or calls `record_llm_usage(...)` after a `structured()` call.
3. Regenerate the typed API client and wire the UI.

Billing is not optional plumbing you add later — route every call through `AiUsageBillingService` so usage is metered consistently.

## Using a different provider

OpenAI is the shipped implementation. Claude and Gemini are **documented recipes**, not code in the kit: implement the `LLMClient` interface against the provider's SDK (its own structured-output and streaming APIs, mapping the final usage into `TokenUsage`), then point the `openai_client` provider in `container.py` at your new class. No route or service code changes.

## Next steps

- **[AI Usage & Credit Billing](ai-billing.md)** — how calls are metered, the monthly allotment, credit packs, and usage reporting.
- **[Stripe Integration](billing.md)** — the subscription billing the AI allotment and credit purchases build on.
