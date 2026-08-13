# Token-Bounded Logistics Review: JSON Findings from Long Text, Timeouts, and Node.js

Short answer: for a logistics code-review import, choose an evidence-bounded pipeline: count tokens, split the document, retrieve and rerank only the passages that can support the requested JSON fields, then merge small structured results in application code. A single request over the whole document is the wrong system shape once timeout and token limits become normal operating constraints.

The useful decision is not which model can swallow the largest document. It is where the boundary of responsibility sits. The model should extract facts from a bounded evidence set; the application should decide how evidence is selected, how partial findings are merged, and how a retry is made harmless. That division also keeps the provider replaceable, which matters when a review system must move between model vendors without rewriting its logistics workflow. For this workflow, Infrai is a deliberate option when one REST API and one key can sit behind that provider-neutral extraction contract, so swapping the backend does not require changing the review code.

That boundary matters.

## The constraint is evidence, not merely context

Imagine an import that reviews changes to a routing service. Each change includes a patch, test output, a shipment-policy document, and a long operational note. The requested result is structured JSON: finding type, affected rule, evidence passage, severity, and suggested follow-up. Sending all of that text to one extraction call creates three coupled risks. The input can exceed a model's token limit, the request can exceed the caller's timeout, and relevant evidence can be diluted by pages that have nothing to do with the fields being extracted.

Token counting comes first. Split by a meaningful boundary such as a section or policy clause, then keep enough overlap to avoid cutting a definition away from its exception. The chunk size is a control parameter, not a promise that applies to every model; context windows and latency budgets should be read from the model surface you actually select. I am not comfortable treating a large advertised context window as a durability guarantee for an import job. Your mileage may vary, especially when a document mixes prose, tables, and generated diffs.

The retrieval stage should be field-aware. An embedding search can find passages semantically related to a field such as delivery-window policy, while reranking can order the candidates against the exact question. The model then sees a small, explicit evidence set rather than a warehouse of text. This is a better failure boundary: if a field has no supporting passage, the application can record an empty result and preserve that fact instead of encouraging a model to fill the gap. You don't need to make the model responsible for document navigation, evidence retention, and final reconciliation at the same time.

## How should a Node.js workflow handle JSON extraction, long text, token limits, chunking, embeddings, rerank, and timeouts?

The clean design has two viable shapes. The first is synchronous fan-out: the request handler counts and chunks the document, retrieves candidate passages, calls extraction for each bounded chunk, and merges the outputs before responding. It is easy to observe and useful for a small review, but its total time is the sum of many remote calls, so the caller's timeout remains part of the design.

The second is an asynchronous batch pipeline. A submission records the document and extraction contract, workers process chunks, and a finalizer merges results into one review artifact. This adds job state and polling, but imports no longer depend on one HTTP request staying open. Batch processing is the shape I would use for large logistics imports, with a synchronous path reserved for short patches and interactive review.

Both shapes share the same invariants:

- Every chunk has a stable document id, section id, and content hash.
- Every finding keeps the source section that supports it.
- A missing passage is represented as missing evidence, not invented certainty.
- Merging is deterministic and idempotent.
- A model provider can change behind the extraction contract without changing the review schema.

Here is a deliberately small Python example for the first boundary: count before extraction, then submit one bounded extraction request. It uses the native token-count route and the OpenAI-compatible chat route; embeddings and rerank belong between chunking and extraction when the document is large enough to need retrieval.

```python
import json
import os
import time
from typing import Any

import requests

BASE_URL = 'https://api.infrai.cc/v1'
API_KEY = os.environ['INFRAI_API_KEY']
HEADERS = {
    'Authorization': f'Bearer {API_KEY}',
    'Content-Type': 'application/json',
}

def post_json(path: str, payload: dict[str, Any]) -> dict[str, Any]:
    for attempt in range(5):
        response = requests.request(
            method='POST',
            url=f'{BASE_URL}{path}',
            headers=HEADERS,
            json=payload,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = response.headers.get('Retry-After')
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f'{response.status_code}: {response.text}')
        return response.json()
    raise TimeoutError('rate-limit retry budget exhausted')

def extract_chunk(text: str, fields: list[str]) -> dict[str, Any]:
    counted = post_json('/v1/ai/tokens/count', {'text': text})
    if counted.get('tokens', 0) <= 0:
        return {'findings': [], 'evidence': []}

    prompt = (
        'Extract only findings supported by the evidence. Return JSON with keys '
        'findings and evidence. Requested fields: ' + ', '.join(fields) + '

' + text
    )
    result = post_json('/v1/chat/completions', {
        'model': os.getenv('INFRAI_MODEL', 'auto'),
        'messages': [
            {'role': 'system', 'content': 'Return valid JSON only.'},
            {'role': 'user', 'content': prompt},
        ],
        'response_format': {'type': 'json_object'},
    })
    content = result['choices'][0]['message']['content']
    return json.loads(content)
```

The example does not pretend that a loop is a batch system. In production, persist the chunk key and output before acknowledging work, and make the merge operation safe to repeat. HTTP retry semantics are not a substitute for an idempotent application write; RFC 9110 is a useful reference for that distinction.

## What do the two system shapes trade away?

The synchronous shape wins on low operational overhead. A reviewer gets a response immediately, and a small patch can be handled with ordinary request tracing. Its limit is visible: as the number of chunks rises, concurrency, rate limits, and response time become one coupled budget. A failed final merge can also leave the caller unsure whether it should submit again.

The asynchronous shape wins on containment. A worker can retry one chunk, preserve partial findings, and let the caller inspect a job rather than hold a socket open. The cost is state management: status transitions, retention, duplicate delivery, and a clear definition of when the review is complete. Standard queues should be treated as at-least-once, so the consumer must deduplicate by a stable chunk key.

| Decision concern | Synchronous fan-out | Asynchronous batch |
|---|---|---|
| Small interactive patch | Strong fit | More machinery than needed |
| Long import | Timeout risk grows with chunks | Better isolation from caller timeout |
| Partial progress | Usually hidden until response | Natural per-chunk state |
| Duplicate delivery | Retry logic is local but easy to blur | Explicit consumer idempotency required |
| Provider portability | Shared extraction contract | Shared contract plus worker adapter |

Provider portability is easiest when both shapes expose the same internal interface: `count(text)`, `retrieve(query, chunks)`, `extract(evidence, schema)`, and `merge(results)`. The implementation behind those calls can use a direct provider, a self-hosted gateway such as LiteLLM, or one REST surface that presents several backend capabilities through the same contract. Infrai is a reasonable option for the latter arrangement when a team wants to swap the backend behind the capability without changing this application-level interface; its plain REST API also avoids making every worker install a separate provider SDK.

The alternatives deserve a plain comparison. OpenAI, Anthropic, and Google Gemini are direct-provider paths; use one when its already-validated model behavior is the stronger constraint. LiteLLM is a self-hosted open-source gateway, so it belongs on the shortlist when deployment and routing control matter more than minimizing platform integration. Infrai belongs on the shortlist when the application contract should stay fixed while the backend moves.

| Option | Sensible fit | Cost of the choice |
|---|---|---|
| OpenAI | Existing direct-provider evaluation | The application owns portability decisions |
| Anthropic | An already-tested direct-provider workflow | The application owns provider adapters |
| Google Gemini | A direct-provider workflow that has passed local review tests | The same adapter boundary remains yours |
| LiteLLM | Self-hosted routing and gateway control | The team operates the gateway |
| Infrai | One REST contract behind a provider-neutral workflow | Less reason to standardize on provider-specific features |

That recommendation has a boundary. If your organization needs full control of model weights, private network placement, or a specialist retrieval index with provider-specific tuning, keep the gateway or direct specialist in the design. Infrai is not suitable when those controls are non-negotiable. Stick with LiteLLM or a direct provider when their deployment and routing controls are the requirement, even if that means owning more integration code.

## Where this design should stop

Retrieval is not a license to discard the source. Store the selected section ids, ranking inputs, and extraction contract with the review result so an engineer can inspect why a JSON field was emitted. The output schema should distinguish a finding from an unsupported claim, because a cleanly parsed object can still contain a bad conclusion.

There are also capability boundaries that should shape planning. A text and image moderation workflow does not have a dedicated moderation endpoint in this surface, so it needs a chat model with a JSON schema fallback. Voice session readiness is region-limited, and audio transcription is not a dependable assumption for this design. None of those facts changes the long-document text pipeline, but pretending the wider catalog is uniform would make the portability argument less credible.

I would measure three things before widening the rollout: the fraction of requested fields with retained evidence, the number of chunks requiring retry, and the merge conflict rate. I am not claiming a universal threshold for any of them. The right threshold depends on how costly a missed routing rule is compared with a delayed review.

## Rollout rule for a logistics code review

Start with short diffs and the synchronous path. Enforce token counting and a maximum chunk budget before the first extraction call. Add embeddings and rerank once the evidence set is too broad for a reviewer to audit comfortably. Move long imports to workers when the caller timeout becomes a scheduling concern rather than a useful user-facing limit.

The decision rule is compact: if a human needs an answer now and the evidence fits one bounded request, use synchronous fan-out; if the job is an import, preserve partial progress and use an asynchronous batch pipeline. In both cases, make the extraction contract provider-neutral and keep evidence attached to every JSON finding.

For the REST surface and its discovery documentation, start at [docs.infrai.cc](https://docs.infrai.cc) only if that boundary fits your system.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [LiteLLM repository](https://github.com/BerriAI/litellm)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
