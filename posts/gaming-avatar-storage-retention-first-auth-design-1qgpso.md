# Gaming Avatar Storage: Retention-First Auth Design for Signed and Public Delivery

Short answer: keep uploaded gaming avatars private by default, authorize the profile request in the application, and issue a short-lived signed download URL; publish a separate derivative only when the image is deliberately public. The deciding constraint is retention: a URL that is easy to cache is also easy to keep using after the account or image should be gone.

That answer applies to an avatar attached to a player account, a clan roster, or a private moderation view. It does not apply automatically to a public creator page. Access policy and delivery performance are related decisions, but they are not the same decision.

Start with deletion.

## The decision record: preserve the boundary before optimizing delivery

The system should maintain four invariants. The original upload has an owner and a lifecycle state in the application database. Its object key is opaque and immutable after creation. A read is authorized before a download capability is minted. Deletion removes the database pointer and the object, with a recorded state for retries. Those rules matter more than whether the last hop is a CDN.

| Option | Appropriate when | Retention consequence | Trade-off |
|---|---|---|---|
| Private object plus signed URL | A logged-in player or staff member must read the image | Access credentials expire; the object remains governed by the private lifecycle | A viewer may need a fresh URL after expiry |
| Public CDN URL | The product defines the avatar as public to anonymous readers | Copies can remain reachable in caches after the origin object is deleted | Revocation is a cache and distribution problem, not just an authorization check |
| Private original plus public derivative | The same upload serves both private account screens and public pages | The two artifacts can have different retention and deletion rules | Processing, consistency, and cleanup become application responsibilities |

The common mistake is to store a permanent URL in the player row and call the key “unguessable.” An opaque name reduces accidental discovery; it does not establish an authorization boundary. A signed URL is a time-limited capability, and the storage service checks its signature and expiry when the request arrives. AWS documents this model for presigned URLs, including the fact that the URL can be used by a client that does not hold the underlying storage credentials.

Retention must be explicit. “Delete the avatar” should mean that the current-avatar pointer is cleared, the object enters a deletion state, and a worker retries the physical delete until it has a confirmed result. A failed worker attempt is not permission to keep serving the old pointer. The application should make the serving decision from current metadata, not from the existence of an object discovered during a listing.

That distinction becomes important during an account deletion or a moderation action. Suppose a player replaces an avatar while a cleanup job is already scanning the old prefix. If the cleanup job treats “not current” as “safe to delete” without recording the upload ID and state transition, it can race with a new pointer or erase an object that a retry still expects. The fix is not a more clever URL. Store the identity of the upload, transition that exact record to `deleting`, and let the worker operate on that record. A repeated delete can then be harmless, while a repeated profile read sees the state and refuses to mint another capability. The queue may deliver the same job twice; the policy should still produce the same answer. This is why retention belongs in the architecture record instead of in a bucket setting added after launch.

## Should an auth app use signed URLs or public CDN delivery for avatar files?

For authenticated profile images, signed access is the conservative default. The profile endpoint checks the viewer's session and the target player's visibility rules, then creates a URL with a lifetime shorter than the expected session or page usefulness. The client may cache the image in memory, but the server does not treat a previously issued URL as a durable record.

Public delivery is a product decision. It is valid for an open leaderboard, a public creator page, or a game community directory where anonymous readers are intended to see the image. It is not a harmless latency setting for a private account. A copied public URL can be embedded outside the game, and an edge cache can outlive the database record that originally authorized the image.

The boundary is the feature.

The catch is operational: signed URLs do not make deletion instantaneous. A URL already issued before deletion may remain usable until its expiry, depending on the storage service's semantics. For a moderation takedown, choose a short enough lifetime for the risk, refuse the object at the application layer after the deletion state is recorded, and make the deletion worker observable. For a public derivative, define cache invalidation and a maximum stale window before launch. If the product cannot tolerate either window, it needs a delivery design with an explicit revocation check, not a stronger-sounding filename.

## A deletion-first path for uploads and downloads

The database should hold the object key, content type, owner, creation time, and lifecycle state. It should not hold a signed URL. Each new upload gets a new key, such as `players/{player_id}/avatars/{upload_id}.png`, and the current pointer changes only after validation succeeds. This prevents a partially accepted upload from becoming the image shown to every profile reader.

The sequence below is intentionally boring. Boring is good here. Authorization, signing, and deletion are separate transitions, so a retry cannot silently turn an old capability into a new one.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone


@dataclass
class Avatar:
    player_id: str
    object_key: str
    state: str
    content_type: str
    created_at: datetime


def avatar_for_profile(viewer_id: str, player_id: str, repository, signer):
    avatar = repository.current_avatar(player_id)
    if avatar is None or avatar.state != "active":
        return None

    if not repository.may_view_profile(viewer_id, player_id):
        return None

    expires_at = datetime.now(timezone.utc) + timedelta(minutes=5)
    return signer.presign_get(avatar.object_key, expires_at=expires_at)


def retire_avatar(player_id: str, repository, object_store):
    avatar = repository.mark_current_avatar_deleting(player_id)
    if avatar is None:
        return

    object_store.delete(avatar.object_key)
    repository.mark_avatar_deleted(avatar.object_key)
```

In production, the delete call and the state update need an idempotent worker boundary: a process can stop after deleting the object but before recording completion, so retrying must be safe. The sample leaves that persistence adapter abstract on purpose; the storage API, transaction model, and queue semantics are implementation choices, while the state transition is the part that should survive a provider change.

The upload path deserves the same discipline. Validate size, media type, and image decoding before changing the current pointer. Set the object's content type as metadata. Record the upload ID and owner in the database. Do not derive authorization from a storage listing, and do not assume that an image suffix proves that the bytes are an image.

## Failure modes that make a tidy URL design unsafe

The first failure mode is stale identity data. A player changes an avatar, but a database row still contains an old public URL or a client retains it in local storage. A new object key and a response that can be refreshed are easier to reason about than overwriting a fixed `current.png` name. The old object still needs lifecycle cleanup; changing the pointer is not deletion.

The second is asymmetric audience. A private original is reused for a public announcement, so a social preview fetches a credential intended for a signed-in page. Split the artifacts. A public derivative can be resized, stripped of unnecessary metadata, and assigned a separately reviewed retention policy.

The third is expiry confusion. A five-minute signed URL controls that particular delivery capability; it does not revoke a file that was already downloaded or copied. If the game client needs offline access, document that the client is creating a new retention surface. The URL mechanism cannot undo that copy.

The fourth is deletion without evidence. A background job logs an exception and moves on, while the profile endpoint still returns the old object key. Record `deleting`, retry with an idempotency key, count the age of each pending deletion, and alert on the oldest item rather than only on worker crashes. I would rather inspect one explicit state machine than guess from a green request metric.

There is also a capacity and cost trade-off. A derivative doubles some storage and processing work, and CDN caching changes the traffic pattern rather than eliminating the need for origin policy. AWS publishes storage pricing separately from the presigned URL mechanism; the relevant estimate must include stored bytes, requests, transfer, image processing, and any cache invalidation work. Your mileage may vary because player behavior and cache hit rates are workload facts, not properties of the word “CDN.”

## Rejected option and the limits of this recommendation

I reject “make every avatar public and let the key be hard to guess” for private account images. It optimizes the visible URL path while discarding the ability to ask the application whether this viewer may still see the image. It is suitable for an avatar whose public visibility is part of the game's design, such as an open community roster, and that decision should be recorded as policy rather than smuggled in as a caching tweak.

The private-default design is not suitable when anonymous access must remain stable across email, social previews, or static pages, or when the client must retain a usable image after account access ends. In those cases, choose a public derivative with a documented removal window, or use an application-mediated delivery layer that checks revocation on every request. Stick with signed private access when account deletion, player privacy, and staff-only moderation are the stronger requirements.

I'm not sure a single avatar policy can serve every surface of a modern game. An inventory of consumers is the fast way to settle it: list the profile page, roster, moderation console, email renderer, public web page, and offline client, then give each one an audience and a deletion deadline. The final choice should follow that table, not the cache dashboard.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://aws.amazon.com/s3/pricing/
