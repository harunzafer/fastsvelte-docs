---
description: "Swap FastSvelte's OpenAI integration for Claude (Anthropic), Gemini, or any provider by implementing the LLMClient interface — a complete drop-in Claude recipe."
keywords: "fastsvelte llm provider, swap openai, claude anthropic, gemini, litellm, llmclient, ai saas starter kit"
---

# Swapping the LLM Provider

FastSvelte ships OpenAI as the default LLM, but no route, service, or billing code ever imports a provider SDK directly — they depend only on a small `LLMClient` seam (see [AI Integration](../features/ai.md#the-llmclient-interface)). To switch providers you write one class implementing that interface and repoint the dependency-injection container at it. Nothing else in the backend changes.

## The contract

`backend/app/service/llm_client.py`:

```python
class LLMClient(Protocol):
    async def structured(self, messages: list[dict], model: Type[T]) -> tuple[T, TokenUsage]: ...
    def stream(self, messages: list[dict]) -> AsyncIterator[str | TokenUsage]: ...
```

- **`structured()`** returns a parsed Pydantic model plus the call's `TokenUsage`.
- **`stream()`** yields plain-text chunks and, as its **final item**, the call's `TokenUsage` (the caller accumulates text for the UI and forwards that final usage to billing).

`messages` is an OpenAI-style `list[dict]` (roles `system` / `user` / `assistant`); each adapter is responsible for translating it into its own provider's request shape.

## Claude (Anthropic) — a complete recipe

Add `anthropic` to the backend dependencies, then create `backend/app/service/claude_client.py`:

```python
from typing import AsyncIterator, Type, TypeVar

from anthropic import AsyncAnthropic, NOT_GIVEN
from pydantic import BaseModel

from app.model.llm_usage_model import TokenUsage
from app.service.llm_client import LLMClient

T = TypeVar("T", bound=BaseModel)
PROVIDER = "anthropic"


def _split_system(messages: list[dict]) -> tuple[str | object, list[dict]]:
    # Anthropic takes the system prompt as a top-level arg, not a role in `messages`.
    system = next((m["content"] for m in messages if m["role"] == "system"), NOT_GIVEN)
    chat = [m for m in messages if m["role"] != "system"]
    return system, chat


class ClaudeClient(LLMClient):
    def __init__(self, model: str, max_tokens: int = 4096, api_key: str | None = None):
        self.client = AsyncAnthropic(api_key=api_key)
        self.model = model
        self.max_tokens = max_tokens

    async def structured(self, messages: list[dict], model: Type[T]) -> tuple[T, TokenUsage]:
        system, chat = _split_system(messages)
        response = await self.client.messages.parse(
            model=self.model,
            max_tokens=self.max_tokens,
            system=system,
            messages=chat,
            output_format=model,         # Pydantic class -> response.parsed_output
        )
        return response.parsed_output, self._usage(response.usage)

    async def stream(self, messages: list[dict]) -> AsyncIterator[str | TokenUsage]:
        system, chat = _split_system(messages)
        async with self.client.messages.stream(
            model=self.model,
            max_tokens=self.max_tokens,
            system=system,
            messages=chat,
        ) as stream:
            async for text in stream.text_stream:
                yield text
            final = await stream.get_final_message()
        yield self._usage(final.usage)

    def _usage(self, usage) -> TokenUsage:
        # Anthropic reports input/output separately; total is the sum.
        return TokenUsage(
            provider=PROVIDER,
            model=self.model,
            input_tokens=usage.input_tokens,
            output_tokens=usage.output_tokens,
            total_tokens=usage.input_tokens + usage.output_tokens,
        )
```

The shape mirrors the shipped `OpenAIClient`: `structured()` uses the SDK's structured-output helper (`messages.parse` with a Pydantic `output_format`), and `stream()` yields text deltas then a final `TokenUsage`. The only provider-specific work is lifting the `system` role out of `messages` (Anthropic takes it as a top-level argument) and supplying `max_tokens`, which Anthropic requires.

## Wire it in

Point the dependency-injection container at the new class. In `backend/app/config/container.py`, swap the `openai_client` provider for a `claude_client` and inject it wherever `openai_client` was wired (the `llm_client=...` argument):

```python
claude_client = providers.Singleton(
    ClaudeClient,
    model=settings.anthropic_model,      # e.g. "claude-opus-4-8"
    api_key=settings.anthropic_api_key,
)
```

Add `anthropic_api_key` / `anthropic_model` to `Settings` (mirroring the existing OpenAI fields), which read `FS_ANTHROPIC_API_KEY` / `FS_ANTHROPIC_MODEL` from the environment.

## Add the pricing row

Cost is computed from the `model_price` table, so **add a row for the Claude model under provider `anthropic`** or cost calculation fails — see [AI Usage & Credit Billing](../features/ai-billing.md#model-pricing). Current Claude pricing (per 1M tokens, input / output):

| Model | Input / 1M | Output / 1M |
|-------|-----------|-------------|
| `claude-opus-4-8` | $5.00 | $25.00 |
| `claude-sonnet-4-6` | $3.00 | $15.00 |
| `claude-haiku-4-5` | $1.00 | $5.00 |

## Other providers

**Gemini, LiteLLM, or any other provider** follow the exact same shape: one class implementing `structured()` + `stream()`, mapping the provider's usage fields into `TokenUsage`, wired via `container.py`, with a matching `model_price` row. The copilot, billing, routes, and UI are untouched.

## Next steps

- **[AI Integration](../features/ai.md)** — the `LLMClient` seam, the sample copilot, and streaming.
- **[AI Usage & Credit Billing](../features/ai-billing.md)** — how every call is metered and billed.
