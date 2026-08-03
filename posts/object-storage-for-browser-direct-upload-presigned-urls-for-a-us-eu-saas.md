# Object Storage for Browser Direct Upload: Presigned URLs for a US/EU SaaS App

## TL;DR

For a SaaS app serving the US and EU, choose object storage by testing residency, upload-policy enforcement, multipart behavior, and recovery semantics; don't choose it from a feature matrix. Use a short-lived presigned upload so the browser sends bytes directly, while the application owns authorization, object naming, finalization, and cleanup.

The URL is the small part.

## How should a SaaS app choose object storage for browser direct upload?

I would record the decision this way: adopt browser direct-to-storage upload with an application-issued presigned request, keep one independently governed storage location per required residency boundary, and treat the database record as the application's source of truth for whether an upload is usable. A Node.js control plane is perfectly capable of issuing the request even though the data plane bypasses it. The choice of server runtime shouldn't determine the storage contract.

Before comparing implementations, I write down invariants. A tenant can authorize only its own namespace. A ticket expires quickly and authorizes one intended operation. The client can't choose a physical bucket or promote an object into the readable namespace. Bytes assigned to the EU stay within the selected EU boundary; the same rule applies to the US. An object isn't visible to the product merely because a browser received a successful upload response. Finally, every retryable application operation has an idempotency key.

Those invariants expose the real failure boundaries. The application may issue a ticket and then disappear. The upload may finish while the finalization call is lost. Finalization may run twice. A large transfer may lose one part. A user may abandon a tab, leaving an unclaimed object. CORS can reject a browser request before storage evaluates the signature. None of these is exotic — they are normal distributed-system outcomes, and a credible evaluation has to reproduce them.

| Decision | Prefer it when | Cost or limitation |
|---|---|---|
| Direct upload with a presigned request | Normal user files can land in a private quarantine namespace | Validation happens after arrival; browser CORS must be configured |
| Application-proxied upload | Policy requires inspection before any third-party persistence | Application bandwidth, timeouts, and scaling enter the byte path |
| Single-part transfer | Objects are comfortably below the tested request-size and timeout limits | A dropped connection restarts the whole transfer |
| Multipart transfer | Large objects need resumable, independently retried parts | Initiation, part tracking, completion, and abandoned-upload cleanup add state |

I won't name a universal winner because there isn't one. The right implementation is the one whose documented behavior and failure tests preserve these invariants with the least operational machinery your team can actually maintain.

## Where are the consistency and durability boundaries?

The database and object store do not participate in one transaction, so I design the workflow as a small state machine: `pending`, `uploaded`, `validated`, then `available`, with `rejected` and `expired` as terminal cleanup states. The browser first asks the application for an upload intent. The application chooses the residency location and an opaque object key, stores both, and returns a time-limited presigned request. After transfer, finalization checks object metadata against the intent before changing state. A background reconciler handles intents and objects that got separated by a crash or a closed tab.

This is where my skepticism about durability claims becomes practical. A durability figure doesn't tell me whether a successful write is immediately readable from the path used by finalization, what concurrent overwrite wins, how deletion is observed, or what cross-location replication does during a partition. I want those semantics in documentation, then I want tests. As far as I can tell, teams get into trouble less often because storage loses bytes than because their application assigns the wrong meaning to an acknowledged write.

Use immutable keys. Overwriting `tenant/logo.png` creates awkward races between upload, validation, caches, and readers; writing a random key and updating a database pointer after validation makes the transition explicit. Keep the original filename as metadata, never as authority. Record expected content type and length when creating the intent, verify the observed metadata during finalization, and don't infer successful validation from a client callback alone.

For US/EU placement, bind residency to the tenant or workspace before issuing the upload. Don't accept a region string from the browser. Also distinguish residency from disaster recovery: an EU primary with an EU recovery copy is a different policy from an EU primary replicated to the US. If legal or contractual language matters, the architecture record should name the permitted locations and the team that approves changes; “multi-region” is too vague to be an invariant.

## What belongs on the critical path?

The synchronous path should authorize, create an intent, issue a narrowly scoped upload request, and later finalize that same intent. Scanning, media processing, and retention work can follow asynchronously, but the object stays unavailable until the required checks pass. I use a quarantine namespace because it gives cleanup and access policy an obvious boundary — no public object ACLs, no client-selected destination, and no promotion based only on possession of a key.

This Python sketch is intentionally storage-neutral. `signer` represents the adapter under evaluation, while the application-controlled fields show the contract I care about. A production implementation also authenticates the caller and performs database transactions around state changes.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from uuid import uuid4


@dataclass(frozen=True)
class UploadIntent:
    intent_id: str
    tenant_id: str
    location: str
    object_key: str
    expected_bytes: int
    content_type: str
    expires_at: datetime


def begin_upload(store, signer, tenant, expected_bytes, content_type):
    if expected_bytes <= 0 or expected_bytes > tenant.max_upload_bytes:
        raise ValueError("file size is outside tenant policy")
    if content_type not in tenant.allowed_content_types:
        raise ValueError("content type is outside tenant policy")

    intent = UploadIntent(
        intent_id=str(uuid4()),
        tenant_id=tenant.id,
        location=tenant.storage_location,
        object_key=f"quarantine/{tenant.id}/{uuid4()}",
        expected_bytes=expected_bytes,
        content_type=content_type,
        expires_at=datetime.now(timezone.utc) + timedelta(minutes=5),
    )
    store.insert_pending(intent)
    request = signer.presign_upload(
        location=intent.location,
        object_key=intent.object_key,
        content_type=intent.content_type,
        exact_bytes=intent.expected_bytes,
        expires_at=intent.expires_at,
    )
    return {"intent_id": intent.intent_id, "upload": request}


def finalize_upload(store, storage, tenant_id, intent_id):
    intent = store.get_for_tenant(tenant_id, intent_id)
    if intent.state in {"uploaded", "validated", "available"}:
        return intent

    observed = storage.stat(intent.location, intent.object_key)
    if observed.bytes != intent.expected_bytes:
        raise ValueError("uploaded size does not match the intent")
    if observed.content_type != intent.content_type:
        raise ValueError("uploaded type does not match the intent")

    return store.mark_uploaded_once(intent.intent_id, observed.version)
```

The crucial operation is `mark_uploaded_once`: back it with a uniqueness constraint or conditional state transition, not a hopeful `if` in application memory. Log the intent ID, tenant ID, location, object version, byte count, state transition, and latency. Alert on old pending intents, repeated finalization attempts, validation rejection rates, incomplete multipart sessions, and quarantine growth. Those signals describe correctness; aggregate upload traffic alone does not.

## How do retries, multipart uploads, and cleanup fail?

I've shipped the naive retry bug once. A mobile client repeated the same finalize operation after a slow response, our handler performed an unconditional insert, and I hit 412 duplicate asset rows before the processing queue exposed the pattern; the storage write was fine, but our control plane had mistaken “response not observed” for “operation did not happen.” I fixed the model by making the upload intent the idempotency key and enforcing one successful state transition in the database. Retrying then returns the existing result.

Multipart upload widens that problem. The application must track an upload identifier and part identities, permit retries without confusing old and new attempts, complete only the intended set, and abort abandoned sessions. The AWS multipart overview documents a three-step lifecycle: initiate, upload parts, and complete; after initiation, uploaded parts continue to incur storage charges until the upload is completed or aborted. It also warns that stopping part uploads alone does not stop an in-progress operation, so cleanup needs to account for concurrency rather than blindly assuming an abort instantly erased every active transfer.

Test it badly.

Throttle a part, terminate the tab, retry one part, submit finalization twice, expire the signing request before transfer, and attempt a content type or size outside policy. Change CORS to remove the application origin and confirm the browser fails without creating an available record. Run reconciliation with a pending database row but no object, then with an old quarantine object but no row. For multipart transfers, confirm completion uses exactly the recorded parts and that the scheduled cleanup aborts stale sessions according to your retention rule.

I'm not sure why teams so often load-test throughput while leaving these state transitions untested; your mileage may vary, but I get more confidence from a dozen destructive workflow tests than from another dashboard showing peak megabytes per second. Measure time to issue a ticket, upload completion rate by size band and location, finalization latency, stale-intent count, cleanup age, and bytes stranded outside `available`. Cost matters too: model stored bytes, request volume, transfer paths, recovery copies, scanning, and CDN behavior with your own traffic distribution rather than one sample object.

## Why reject proxying, and when should you still use it?

For the stated browser direct-to-storage workflow, I reject application proxying because it puts the Node.js service, its load balancer, and its timeout policy into every byte transfer. It also couples upload capacity to API capacity. A presigned design leaves the application responsible for the security decision while allowing storage to handle the transfer, which is the cleaner failure boundary for ordinary SaaS attachments.

The catch is real: direct upload is not suitable when policy requires synchronous inspection before bytes may reach the storage service, when clients cannot reach the storage endpoint, or when the team cannot safely operate post-upload quarantine and reconciliation. Stick with a proxy or controlled ingestion gateway in those cases. Small internal tools may also reasonably proxy modest files because one observable path is worth more than architecture they don't need.

I would reject multipart as the default for small files, too. It creates several persisted operations where one used to exist, so introduce it only after realistic object sizes and network conditions show that restart-from-zero is unacceptable. Likewise, don't build active cross-boundary replication merely because it sounds safer; first decide whether the copied data is legally allowed in the destination and whether recovery requires a second location at all.

The final selection exercise is a proof, not a ranking: implement one thin adapter, run the same expiry, policy, retry, CORS, multipart, reconciliation, and residency tests against each serious candidate, then preserve the results in the architecture record. Include exit cost: protocol compatibility helps, but migrations still move bytes, metadata, event wiring, lifecycle rules, access policy, and audit evidence. Choose the system whose limits your team can state plainly.

No magic here.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://firebase.google.com/docs/storage
