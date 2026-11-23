# Module 3: Architecture and API Calls

## Learning objectives

By the end of the module you will be able to:

- build an LLM request through the API with the correct roles and the essential parameters;
- estimate the cost and latency of a call before running it;
- classify API errors and decide which ones to retry;
- implement a retry with exponential backoff and jitter;
- consume a streamed response and display it in real time;
- design an abstraction layer that lets you switch provider without rewriting the application.

## Premise: an LLM call is an expensive, unpredictable network call

If you come from traditional software, you are used to HTTP calls with a latency of a few milliseconds, a fixed cost and a deterministic response. An LLM call is different: it can take anywhere from a few hundred milliseconds to tens of seconds, the cost depends on the number of tokens, the response changes from one call to the next, and the provider can return capacity errors even when your request is valid.

Managing calls therefore means managing three things: costs, latency and failures.

## 1. Anatomy of a request

Most chat APIs have a similar structure. Even though field names change between providers, the concepts are these:

- **Model**: identifier of the version to use. It should be treated as a dependency: versioned, documented, replaceable.
- **Messages**: an ordered list of turns, each with a role.
  - `system`: persistent instructions and rules.
  - `user`: input from the user or the application.
  - `assistant`: previous responses from the model, when simulating a conversation.
- **Generation parameters**: temperature, top-p, maximum number of output tokens, stop sequences.
- **Tools (optional)**: definitions of functions that the model can ask to invoke (see module 7).
- **Response format (optional)**: expected JSON schema, see module 2.

Minimal example in Python, with a generic client (the name of the client and of the fields varies from provider to provider; adapt it to the documentation you use):

```python
import os

def chiedi_al_modello(client, domanda: str) -> str:
    risposta = client.chat(
        model=os.environ["LLM_MODEL"],   # the model is configuration, not code
        system="Answer in Italian, concisely.",
        messages=[{"role": "user", "content": domanda}],
        temperature=0.2,
        max_tokens=400,
    )
    return risposta.text
```

Note: the model name comes from an environment variable. Changing the model (or the provider) should not require a change to the application code.

### Exercise

Write the same prompt with and without `system`. Observe which rules the model follows better when they are in the system prompt compared to when they are in the first user message, and why.

## 2. Costs, latency and tokens

### Cost

The cost of a call is, in simplified form:

```
cost = (input_tokens × input_price) + (output_tokens × output_price)
```

Two details that are often overlooked:

- output usually costs more than input;
- every call resends the conversation history: a twenty-turn chat pays for the previous turns every time. Cost grows more than linearly with the length of the session.

Many providers offer discounts for prompt caching (reuse of identical prefixes) and for batch processing. Always check the up-to-date documentation: prices and conditions change often.

### Latency

Two metrics to measure separately:

- **Time-to-first-token (TTFT)**: how long it takes before the first piece of the response arrives. Dominated by the prompt prefill.
- **Throughput**: tokens per second during generation. It determines how long a long response takes.

For UX, TTFT matters most: a stream that starts quickly feels faster even if the total time is the same.

### Estimation exercise

For an application that summarizes documents, estimate: average input tokens (document plus system prompt), average output tokens, number of requests per day. Calculate the monthly cost using the prices of the provider you choose. Then ask yourself: which part of the cost could you reduce by trimming the context without losing quality?

## 3. Error handling

API errors fall into categories with different behaviors:

| Type | Example | Action |
|---|---|---|
| Request error | 400, malformed input, context too long | Do not retry: fix the request |
| Authentication | 401, 403 | Do not retry: check the credentials |
| Rate limit | 429, quota exceeded | Retry with backoff |
| Provider-side error | 500, 502, 503, 529 (overload) | Retry with backoff, then fall back |
| Timeout | The response does not arrive in time | Retry with caution, limit the size |

Warning: an aggressive retry against an overloaded provider makes things worse for everyone. This is why backoff is needed.

### Exponential backoff with jitter

```python
import random
import time

def con_retry(operazione, max_tentativi=5, base=1.0, massimo=30.0):
    for tentativo in range(max_tentativi):
        try:
            return operazione()
        except ErroreRitentabile:          # 429, 5xx, timeout
            if tentativo == max_tentativi - 1:
                raise
            attesa = min(massimo, base * (2 ** tentativo))
            time.sleep(random.uniform(0, attesa))   # full jitter
```

Jitter prevents many clients from all retrying at the same instant, creating synchronized spikes. If the provider returns an explicit wait header (for example `retry-after`), it takes precedence over the local calculation.

### Exercise

Simulate a service that fails with 70% probability and returns 429 or 503. Implement the retry above and measure: how many total calls are needed to get a response, and how much total time the client takes. Then compare with a retry without backoff.

## 4. Streaming

Streaming lets you receive the response while it is being generated, usually through Server-Sent Events or a similar protocol. The client receives events with small fragments of text that must be accumulated and displayed.

Points to watch:

- handle the connection closing abnormally midway through a response;
- show an intermediate state (cursor, indicator) if the response is slow to start;
- decide what happens if the user cancels: interrupting the request on the client side does not always stop the computation on the provider side, so a server-side timeout can also make sense;
- do not apply structured output validation until the response is complete.

Example flow in pseudocode:

```text
for each fragment received from the stream:
    accumulate the text
    send the fragment to the UI
when finished:
    validate the complete output (if structured)
    save to the log: tokens used, TTFT, total duration
```

### Use case to develop

A web page with a chat that shows the response in streaming. Implement the server-side flow that forwards the fragments to the browser, records TTFT and tokens used for each request, and handles cancellation by the user.

## 5. Provider abstraction

An application that calls a provider's APIs directly in a thousand places becomes expensive to change. A good practice is to define an internal interface:

```python
from typing import Protocol

class ModelloLLM(Protocol):
    def genera(self, system: str, messaggi: list[dict], **opzioni) -> "Risposta": ...
```

Each provider (OpenAI, Anthropic, an open source model served through Groq or Together AI) has an implementation of this interface. The rest of the application depends only on the interface.

Be careful, however, not to hide too much: the real differences (tool handling, message format, behavior with structured output, context limits) exist and must be tested. An abstraction that ignores these differences produces bugs that are hard to diagnose.

### Comparing providers: criteria, not verdicts

When you compare providers and models, evaluate at least:

- quality on your task (not on generic benchmarks): measure with your own dataset, module 6;
- cost per completed task, not per token;
- p50 and p95 latency in your scenario;
- availability, rate limits and data region;
- support for tools, structured output, streaming, context size;
- contractual terms on data usage.

Exercise: choose three models (one closed, one open served by a third-party provider, one smaller and cheaper) and define the comparison table with these criteria. You do not need to fill it in with real data: the point is to decide which measurements matter before looking at the results.

## 6. Observability

Every call should leave a trace: model used, prompt version, input and output tokens, TTFT, total duration, outcome, number of retries, estimated cost. Without this data you do not know how much a feature costs or where it breaks.

Practical tip: assign an identifier to each request and record it together with the response. When a user reports a problem, you will be able to reconstruct exactly what happened.

## 7. Cost optimization

The cost of an LLM feature is controlled through architecture choices, not just by negotiating the price.

- **Prompt caching**: if many requests share a long, identical prefix (system prompt, documentation, few-shot examples), many providers let you reuse its computation at a reduced cost. Rule of thumb: put the stable parts at the beginning of the prompt and the variable ones at the end.
- **Batch processing**: for non-interactive tasks (classifying 50,000 historical tickets) batch APIs cost less, in exchange for longer completion times.
- **The right model for the task**: a small model for classifying and routing, the large one only for complex generation.
- **Output control**: appropriate `max_tokens`, length instructions, structured output instead of free text when the recipient is a program.
- **Essential context**: in a long chat, summarize the history instead of resending it in full.

### Exercise

Go back to the cost estimate from section 2. Calculate the effect of three interventions: caching the system prompt, switching to a small model for 70% of the simple requests, reducing output tokens by 30%. Which intervention pays off most in your case?

## 8. Security of keys, data and budget

- **API keys**: never in the code or in the repository. Use environment variables in development and a secret manager in production. One key per environment (development, test, production) makes revocation easier.
- **Client-side keys**: a web or mobile application must never call the provider directly with the key. Calls go through your backend, which authenticates the user and applies the limits.
- **Budget**: set spending limits on the provider and per-user limits in the backend. A bug in a loop can generate thousands of calls in a few minutes.
- **Data sent**: everything in the prompt is sent to the provider. Check the contractual terms (retention, use for training, region) and consider masking personal data before sending. The topic is explored in depth in module 8.

### Exercise

Write a small backend (for example FastAPI) that exposes `/chat` and forwards the request to the provider. Add: reading the key from an environment variable, a limit of ten requests per minute per user, rejection of requests with input above a token threshold.

## Further reading

- [Documentation of the chosen providers](https://docs.claude.com): sections on rate limits, errors, streaming and pricing (always check the current version).
- [AWS Architecture Blog, "Exponential Backoff And Jitter"](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/): the theory of retry with backoff and jitter.
- [MDN, "Using server-sent events"](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events): how streaming responses from the server works.
- [OpenTelemetry, "Semantic conventions for Generative AI"](https://opentelemetry.io/docs/specs/semconv/gen-ai/): standard conventions for tracing calls to models.
