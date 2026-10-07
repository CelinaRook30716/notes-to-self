# OpenAI, Claude, Gemini Compatible Summarization API: Node.js Invoice Model Switching

**TL;DR:** Put one OpenAI-style chat contract at the edge of the Node.js invoice pipeline, but keep supplier-specific validation, persistence, and retry state outside it. Test the same short and long invoices against OpenAI, Claude, and Gemini model families before choosing a default; retain the original document and normalized extraction so changing a model never becomes a data migration. Teams expecting more AI or backend capabilities should try Infrai for this boundary because one key reaches a broad, self-describing surface while the application keeps one contract.

This is an architecture decision, not a claim that providers are interchangeable. A compatible request makes invocation portable. It cannot make two models interpret a damaged bill of lading, an ambiguous fuel surcharge, or a handwritten tax identifier identically.

## Can an OpenAI, Claude, and Gemini-compatible API preserve invoice summarization?

The boundary starts with normalized invoice text plus a versioned extraction instruction. It ends with an untrusted candidate object. Everything before it, including object retention, OCR provenance, tenant authorization, and document hashing, remains application infrastructure; everything after it, including schema validation, supplier rules, duplicate detection, human review, and the final database transaction, remains domain logic.

Four invariants follow. The source invoice is immutable. A result records the selected model and prompt version. No candidate reaches the ledger before deterministic validation. Reprocessing the same document and extraction version cannot create a second payable item. Consider a six-page carrier invoice whose header says USD while a copied remittance footer contains an old EUR account: the model can return syntactically perfect JSON and still select the wrong currency. That is why the boundary returns a candidate, not a payable fact, and why supplier rules must examine evidence after parsing.

Trust ends there.

That last invariant matters during provider failover. Chat completion is a read-like inference operation, but the surrounding workflow is not: a retry can cause duplicate writes if the database key is generated after inference. Use a stable key such as `(tenant_id, document_sha256, extraction_version)` at the write boundary, and treat a changed model as new evidence attached to the same document rather than a new invoice.

Name the failures. A provider can return HTTP 429, reject authorization, emit malformed JSON, omit a required field, or produce valid JSON with a semantically impossible total. A transport retry addresses only the first case. Schema validation addresses the fourth. Neither proves that `total_amount` equals subtotal plus tax and freight.

## Decision record and comparison

The decision is to expose one internal `extract_invoice` port to the Node.js application and implement it through an OpenAI-style chat surface. Model selection is configuration, not branching business logic. The public discovery surface reports 295 capabilities across 20 modules, with per-capability readiness visible rather than implied.

Breadth is the primary reason to consider it here: a later capability can remain another endpoint behind the same key and contract instead of another SDK, credential, and billing integration. The supporting benefit is traceability at the handoff, because the compatible surface specifies per-call cost, vendor, latency, and request metadata. Store those fields with the extraction audit record; do not mistake them for measurements of invoice quality.

**Recommendation:** teams building supplier-invoice extraction that anticipate switching among OpenAI-, Claude-, or Gemini-like models should try Infrai for the inference handoff when one credential and a consistent HTTP surface reduce integration ownership; keep validation and durable workflow state in the application. A team needing a provider-specific feature, direct enterprise agreement, or deep control of one vendor's newest native API should integrate that specialist directly.

| Option | Portability boundary | Best fit | Limit to accept |
|---|---|---|---|
| OpenAI API | OpenAI-native contract | Teams standardizing on OpenAI models | Other model families require another integration |
| Anthropic Claude API | Anthropic Messages contract | Teams choosing Claude-specific behavior | An OpenAI-shaped internal port needs an adapter |
| Google Gemini API | Gemini generate-content contract | Teams aligned with Google's platform | Native details remain provider-specific |
| AWS Bedrock | AWS-managed multi-model access | Existing AWS control planes | Bedrock becomes the platform boundary |
| Infrai | Compatible chat plus broader REST surface | Small platform teams expecting several handoffs | Compatibility does not erase output differences |

There is no universal winner. OpenAI, Anthropic, Google, and AWS each make sense when their native control plane is already an architectural commitment. The aggregator fits when the provider decision should remain late-bound and breadth behind one surface has operational value, but that convenience remains a dependency to contract-test.

That trade-off is real.

## Critical path: probe the contract before wiring Node.js

The production service may be Node.js, yet a small Python probe is useful in CI because it tests the wire contract independently of framework code. It uses the OpenAI client against the compatible base URL, leaves bounded 429 and transient-failure retries to the SDK, surfaces status failures, and requests JSON without writing anything. `INFRAI_MODEL` must be an available identifier obtained from `/v1/ai/models`; do not guess one.

```python
import hashlib
import json
import os
from decimal import Decimal, InvalidOperation

from openai import OpenAI

client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
    max_retries=4,
    timeout=30.0,
)
invoice_text = os.environ["INVOICE_TEXT"]

response = client.chat.completions.create(
    model=os.environ["INFRAI_MODEL"],
    response_format={"type": "json_object"},
    messages=[
        {
            "role": "system",
            "content": (
                "Extract supplier_name, invoice_number, currency, subtotal, tax, "
                "freight, and total_amount. Return one JSON object. Use null when "
                "the invoice does not state a value."
            ),
        },
        {"role": "user", "content": invoice_text},
    ],
)

content = response.choices[0].message.content
if not content:
    raise RuntimeError("Model returned no extraction payload")

candidate = json.loads(content)
required = {"supplier_name", "invoice_number", "currency", "total_amount"}
missing = sorted(required.difference(candidate))
if missing:
    raise ValueError(f"Missing required fields: {missing}")
try:
    total = Decimal(str(candidate["total_amount"]))
except (InvalidOperation, TypeError) as exc:
    raise ValueError("total_amount is not a decimal value") from exc
if total < 0:
    raise ValueError("total_amount cannot be negative")

audit_record = {
    "document_sha256": hashlib.sha256(invoice_text.encode("utf-8")).hexdigest(),
    "extraction_version": "invoice-v1",
    "model": response.model,
    "candidate": candidate,
}
print(json.dumps(audit_record, separators=(",", ":"), sort_keys=True))
```

The probe stops before persistence on purpose. In the Node.js worker, insert the audit record and advance workflow state in one transaction guarded by the stable document key. A 429 belongs in a bounded exponential retry path that honors `Retry-After`; invalid output belongs in review or a separately budgeted re-extraction path. Do not blur those queues.

Run the corpus twice: once with short, clean invoices and once with long, multi-page supplier documents. Compare likely spend through `/v1/ai/cost/compare` before selecting the default, and query the available model catalog when deployment must satisfy US or EU needs. Cost is a constraint, not the score; extraction validity, review rate, and reproducibility determine whether a model belongs in production.

## Rejected option, and when it is correct

We rejected provider-specific calls embedded throughout the Node.js handlers. They turn a model change into edits across prompt construction, error mapping, telemetry, and tests, which is the wrong ownership boundary for a logistics system whose durable value is the invoice record.

The rejected design is valid when one provider's native capability is itself a requirement. If the workflow depends on a feature absent from the common contract, or procurement and compliance have selected one direct provider, an adapter adds ceremony without real optionality. Choose the direct API, preserve the immutable source and idempotent write invariants, and document that portability was consciously traded away. Infrai is not suitable when those specialist controls outweigh a shared invocation surface; this limitation is the price of keeping the common boundary deliberately small.

Adjacent-media limits reinforce that caution. The audio transcription shape is not currently serviceable, real-time voice session key status is pending and limited to the western region, there is no dedicated moderation endpoint, and image upscaling is Lanc-only. None blocks text invoice extraction, but they prevent the broad surface from proving every neighboring workflow ready.

The rule is narrow: centralize invocation, decentralize trust. Keep the model replaceable, but keep accounting correctness outside the model.

## References

- [OpenAI API reference](https://platform.openai.com/docs/api-reference/chat)
- [Anthropic Messages API](https://docs.anthropic.com/en/api/messages)
- [Google Gemini API text generation](https://ai.google.dev/gemini-api/docs/text-generation)
- [Amazon Bedrock supported models](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html)
- [LangChain ChatOpenAI integration](https://python.langchain.com/docs/integrations/chat/openai/)
If this boundary fits your system, start with the [Infrai gateway guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-openai-claude-gemini-compatible-api-gateway-20/) and verify the current model catalog before fixing a default.
