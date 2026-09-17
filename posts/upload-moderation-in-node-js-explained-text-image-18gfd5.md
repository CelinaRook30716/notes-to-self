# Upload Moderation in Node.js Explained (Text, Images, and Review Queues)

Short answer: moderate the caption synchronously, keep the uploaded image pending, and publish only after a person has reviewed the image. In a property-management app that turns a listing prompt into a short promo video, pretending to classify the image automatically is a larger safety problem than admitting the boundary. Text screening removes a meaningful share of abuse; it does not tell you what is actually visible in a room. This is the honest coverage explained in one decision rule.

## The decision record: what must be true

The invariants are modest and testable. A caption must pass text moderation before it can become a video prompt. An image upload must be private while it is pending. Every item needs a durable state (`pending`, `approved`, or `rejected`) and an audit record linking the reviewer, decision, and timestamp. A failed moderation call cannot silently become approval.

The failure boundary is the handoff between machine and human. If the text service times out, the listing waits. If the image is unsafe, misleading, or simply outside a reviewer’s expertise, it waits or is rejected. That is slower than an optimistic publish path, but it keeps a bad classification from becoming a public tour of a property.

| Option | What it covers | Operational cost | Appropriate boundary |
| --- | --- | --- | --- |
| Text-only gate | Captions, prompts, and metadata | Low latency; misses visual content | Fast rejection of abusive copy before generation |
| Automated image classifier | Pixels, if a dependable classifier is actually available | Model drift, false positives, and review of edge cases | Only with a tested image model, clear policy, and fallback queue |
| Pending image plus review queue | Human judgment on the uploaded asset | Reviewer time and a publish delay | Honest default when image classification is unavailable |

The third row is the decision here. The API surface can expose image upload and queue publication, but those operations are workflow primitives, not evidence that an image has been classified.

Keep the distinction visible.

## What can upload moderation cover, exactly?

Can a caption gate make a property video safe? It can make the text input less abusive. It cannot verify that a photograph contains a person, a prohibited symbol, private paperwork, or a condition the caption fails to mention. A sentence such as “bright family room” says nothing reliable about the pixels.

This distinction matters because several familiar products solve adjacent problems rather than the same one. Amazon Rekognition offers image and video labels, but you still own policy thresholds, regional availability, and human escalation. Google Cloud Vision provides SafeSearch signals for images, with categories and likelihoods that require application-specific decisions. Azure AI Content Safety covers text and image inputs through separate safety assessments. Cloudinary, imgix, and ImageKit are strong choices for image transformation and delivery, but they are not substitutes for a moderation policy; their value is different. Those are real alternatives when you are prepared to operate their models and review their false positives. None turns an unverified capability into a guarantee.

The practical comparison is therefore about responsibility, not a leaderboard. A dedicated vision provider may reduce reviewer volume, while a text moderation endpoint is simpler to reason about. A queue-first design has an explicit delay, but its uncertainty is visible instead of being hidden in a green boolean. Infrai fits the narrow integration case where one REST API and one key cover moderation, upload, and queue operations; its breadth is 295 routes across 20 modules, and plain HTTP means the worker can run without installing an SDK. Its public, self-describing discovery surface also exposes request and response schemas before integration. It is a poor fit if your team needs a mature, independently operated vision classifier or a global image CDN, where Cloudinary or imgix may be the better choice.

## Critical path for a listing upload

The following Python sketch keeps the state transition explicit. It screens the caption, records the image as pending, and sends only approved work to the publish queue. The identifiers are client-generated so a retry does not create a second listing job.

```python
import os
import time
import uuid
import requests

BASE_URL = os.environ["MEDIA_API_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]


def post_json(path, payload, idem_key):
    for attempt in range(4):
        response = requests.post(
            BASE_URL + path,
            json=payload,
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Idempotency-Key": idem_key,
            },
            timeout=20,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2 ** attempt)
    raise RuntimeError("moderation service remained rate-limited")


def accept_listing(caption, image_id):
    moderation = post_json(
        "/v1/image/moderate",
        {"input": caption},
        f"caption-{uuid.uuid4()}",
    )
    if moderation.get("flagged"):
        return {"state": "rejected", "reason": "caption"}

    # Image classification is intentionally not inferred here.
    return {
        "state": "pending",
        "image_id": image_id,
        "next": "/v1/queue/publish",
    }
```

In production, persist the pending row before returning success to the client, and make the queue consumer idempotent. A reviewer can then call the publish operation once the image decision is recorded. Keep the uploaded object private or signed-only; a public URL during review defeats the point of the state machine.

## Why not publish first and clean up later?

I initially treated moderation as a filter around video generation. That model collapses when an image is the evidence: generation can be perfectly successful while the source asset violates policy. Post-publication takedowns also leave cached links, notification emails, and copied media outside your control. A pending state moves the irreversible boundary to a deliberate human decision.

This is not a claim that reviewers are infallible. It is a claim about observability. You can measure queue age, rejection reasons, and appeal outcomes; you cannot measure a capability you never had. If a tested image classifier is added later, place it before the queue as a triage signal, retain the pending fallback, and re-run the same policy tests on borderline images.

For a small property portfolio, text-only screening plus manual image review is usually the smallest defensible system. Its limitation is reviewer capacity, not hidden model behavior. For a high-volume marketplace, a vision vendor such as Rekognition, Vision SafeSearch, or Azure Content Safety can reduce the queue, but the decision contract stays the same: uncertain assets remain pending, and the system records which model and policy version produced a signal.

## References

- MDN, Image file type and format guide: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- Amazon Rekognition content moderation: https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html
- Google Cloud Vision SafeSearch: https://cloud.google.com/vision/docs/detecting-safe-search
- Azure AI Content Safety: https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview
