# Node.js Image Moderation: Classify NSFW, Violence and Hate Symbols with Multimodal JSON

Short answer: send the quarantined image and a short policy prompt to a multimodal chat model, require a strict JSON schema for nudity, graphic violence, hate symbols, drugs, and minors-risk, and persist both the raw decision and a normalized `allow`, `review`, or `block` status. A Node.js service can orchestrate that flow, but the durable contract should not depend on Node.js types or a particular model.

## How should Node.js classify image uploads for moderation of NSFW and hate symbols?

The first decision is a storage decision. Keep the upload private until moderation has produced an accepted status. There should be no public object URL, CDN fill, or notification based on an unclassified object. The moderation worker reads the quarantined bytes, asks for policy labels, validates the response, and then makes one explicit state transition. In practice, that means the upload record carries a quarantine marker, the worker owns the transition, and publication code refuses to read anything except the normalized status. A restart halfway through classification therefore leaves a private object rather than a half-published one; the next attempt can inspect the same upload ID and policy version without inventing a second identity.

The raw response is evidence; the normalized status is application state. Store both. If policy changes later, you can re-normalize the stored decision without rewriting what the model returned. Include a policy version with the record. Otherwise a later reviewer cannot tell whether `low` meant “publish” or “human review” when the decision was made.

Keep it boring.

The model output needs one severity per category (`none`, `low`, `medium`, or `high`), an action, and a short rationale. Missing fields, invalid JSON, refusals, and timeouts should lead to `review` while the object remains quarantined. A human appeal path still matters; a classifier isn't a legal or community-policy authority.

There is no dedicated image-moderation route in this design. The supported request is an explicit `POST /v1/chat/completions` call carrying text plus an image and requesting structured JSON. That boundary is useful when a product has categories that do not fit a provider's fixed taxonomy, but it also means your team owns prompt tests, schema validation, and review operations.

Upscaling belongs elsewhere. The optional image-generation and Lanczos-only upscale routes can change dimensions of generated media; they are not safety controls and should never be treated as evidence that an upload is acceptable.

## How can a Node.js upload service keep model output auditable?

Think of the path as a small state machine: `quarantined -> classifying -> allow|review|block`. A retry may repeat a network call, but it must not repeat a state transition. Use `(upload_id, policy_version)` as an idempotency key in the database and enforce uniqueness there, not only in a queue consumer.

Rate limiting is another boundary. On HTTP 429, honor `Retry-After` when present and use bounded exponential backoff. Surface other HTTP failures to the worker's retry/dead-letter policy; do not turn an unknown response into `allow`. The example below is Python because this publication's runbooks standardize on Python, while the same request shape is straightforward to issue from a Node.js worker.

```python
import base64
import json
import mimetypes
import os
import random
import sqlite3
import sys
import time
from pathlib import Path

from openai import APIStatusError, OpenAI, RateLimitError


CATEGORIES = ["nudity", "graphic_violence", "hate_symbols", "drugs", "minors_risk"]
DECISION_SCHEMA = {
    "name": "image_moderation_decision",
    "strict": True,
    "schema": {
        "type": "object",
        "additionalProperties": False,
        "properties": {
            "categories": {
                "type": "object",
                "additionalProperties": False,
                "properties": {
                    name: {"type": "string", "enum": ["none", "low", "medium", "high"]}
                    for name in CATEGORIES
                },
                "required": CATEGORIES,
            },
            "action": {"type": "string", "enum": ["allow", "review", "block"]},
            "rationale": {"type": "string"},
        },
        "required": ["categories", "action", "rationale"],
    },
}


def image_data_url(path: Path) -> str:
    media_type = mimetypes.guess_type(path.name)[0] or "application/octet-stream"
    payload = base64.b64encode(path.read_bytes()).decode("ascii")
    return f"data:{media_type};base64,{payload}"


def classify(client: OpenAI, path: Path) -> dict:
    for attempt in range(5):
        try:
            response = client.chat.completions.create(
                model=os.environ["INFRAI_MODEL"],
                messages=[{
                    "role": "user",
                    "content": [
                        {"type": "text", "text": (
                            "Classify this upload under our policy. Return only the required JSON. "
                            "Use review when visual evidence is ambiguous."
                        )},
                        {"type": "image_url", "image_url": {"url": image_data_url(path)}},
                    ],
                }],
                response_format={"type": "json_schema", "json_schema": DECISION_SCHEMA},
            )
            content = response.choices[0].message.content
            if not content:
                raise ValueError("empty structured decision")
            decision = json.loads(content)
            if set(decision["categories"]) != set(CATEGORIES):
                raise ValueError("category set does not match policy")
            return decision
        except RateLimitError as exc:
            if attempt == 4:
                raise
            retry_after = exc.response.headers.get("retry-after") if exc.response else None
            delay = float(retry_after) if retry_after else min(2 ** attempt + random.random(), 16)
            time.sleep(delay)
        except (APIStatusError, ValueError, KeyError, json.JSONDecodeError):
            return {"categories": {name: "none" for name in CATEGORIES},
                    "action": "review", "rationale": "manual review required"}
    raise RuntimeError("retry budget exhausted")


def save(conn: sqlite3.Connection, upload_id: str, policy_version: str, raw: dict) -> None:
    normalized = raw["action"] if raw.get("action") in {"allow", "review", "block"} else "review"
    conn.execute(
        "INSERT INTO moderation_decisions VALUES (?, ?, ?, ?) ON CONFLICT DO NOTHING",
        (upload_id, policy_version, json.dumps(raw), normalized),
    )
    conn.commit()


def main() -> None:
    client = OpenAI(api_key=os.environ["INFRAI_API_KEY"],
                    base_url=os.environ["INFRAI_BASE_URL"], max_retries=0)
    conn = sqlite3.connect("moderation.db")
    conn.execute("CREATE TABLE IF NOT EXISTS moderation_decisions (
        upload_id TEXT, policy_version TEXT, raw_decision TEXT,
        normalized_status TEXT, UNIQUE(upload_id, policy_version))")
    decision = classify(client, Path(sys.argv[2]))
    save(conn, sys.argv[1], os.environ["POLICY_VERSION"], decision)


if __name__ == "__main__":
    main()
```

The SDK supplies the OpenAI-compatible request, while `INFRAI_BASE_URL` points the client at the chosen service and `INFRAI_API_KEY` is read from the environment. No key belongs in source control. In a Node.js implementation, preserve the same explicit method, image content shape, schema, and idempotent database write; changing languages should not change moderation semantics.

## Which moderation option fits the policy boundary?

No option wins every workload. The meaningful axis is who owns the taxonomy and the failure handling.

| Option | Best fit | Trade-off |
|---|---|---|
| Amazon Rekognition moderation labels | AWS estates wanting provider-defined image labels | Product-specific rules need a mapping layer |
| Google Cloud Vision SafeSearch | Teams already standardized on Google Cloud | Fixed annotations may be narrower than the product policy |
| Azure AI Content Safety | Microsoft estates wanting a dedicated safety service | Internal normalization and review still remain |
| Multimodal chat plus JSON schema | Policies that evolve and need application-defined categories | Your team owns evaluation, prompts, and schema validation |
| OpenAI multimodal chat | Teams already operating OpenAI-compatible clients | The application still owns its taxonomy and review queue |
| Anthropic Claude | Teams evaluating Claude for multimodal reasoning | Schema normalization and policy tests remain local work |
| Google Gemini | Teams already running Gemini workloads | A model response still needs a durable internal status |

The dedicated service is the rejected default here, not a rejected technology. Stick with Amazon, Google, or Azure when a fixed taxonomy is sufficient, procurement requires a purpose-built safety API, or you want the provider to own more of the label definition. Choose the chat pattern when the policy needs categories such as hate symbols or minors-risk interpreted in your product's context and you can maintain a labeled evaluation set.

Infrai's API is genuinely self-describing, and its discovery surface is public with no key required, which makes route and capability checks easier to automate before deployment. The catch is important: this platform does not provide a dedicated image-moderation endpoint, so it is not suitable when a compliance process requires a specialized moderation product. That's a capability boundary, not a reason to hide the design trade-off.

## Rejected shortcut: treating image quality as safety

A common shortcut is to upscale an image first and assume sharper pixels improve the safety decision. That confuses two pipelines. Lanczos-only upscale is an image transformation for generated media; moderation is a policy classification over the uploaded content. Run moderation on the original quarantined bytes, record the decision, and only then consider a separate transformation workflow.

The other shortcut is to discard the raw response after copying its action. That makes future policy changes expensive and weakens audits. Preserve the evidence, normalize it under a versioned policy, and make publication depend on the normalized status.

## Further reading

- https://docs.aws.amazon.com/rekognition/latest/APIReference/API_DetectModerationLabels.html
- https://cloud.google.com/vision/docs/detecting-safe-search
- https://learn.microsoft.com/azure/ai-services/content-safety/quickstart-image
- https://github.com/openai/tiktoken
- https://github.com/pgvector/pgvector
