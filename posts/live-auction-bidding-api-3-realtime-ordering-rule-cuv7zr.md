# Live Auction Bidding API: 3 Realtime Ordering Rules for Small SaaS Teams

Accept every bid through the ordinary application API, commit its order on the server, and use realtime channels only to distribute the accepted state. **The channel is a projection, never the auction ledger.** This remains true even if the transport usually arrives in order: fairness is decided at the transaction boundary, while fan-out is allowed to duplicate, lag, disconnect, and recover.

TL;DR: give each accepted bid a monotonically increasing auction sequence, publish the complete current high-bid state with that sequence, and make clients reject older sequences and fetch a snapshot when they detect a gap. Put the notification in a durable outbox in the same database transaction as the accepted bid. A small SaaS team then has one place that decides the winner and an explicit recovery path when delivery is uncertain.

## How should a realtime API order live auction bidding fairly?

First, one server-side transaction owns the comparison and ordering decision. Two buyers can submit at nearly the same instant; neither browser clock, WebSocket arrival time, nor region-to-region network delay is admissible evidence of priority. The transaction locks the auction row, checks that the auction is open and the amount is valid, assigns `sequence = previous_sequence + 1`, and persists the winner.

Second, every broadcast carries the accepted state rather than an instruction such as “add this bid.” A viewer receiving sequence 418 after 417 can replace its view. A viewer receiving 416 afterward discards it. A viewer jumping from 417 to 420 knows it missed something and retrieves the authoritative snapshot, even if the displayed high bid at 420 happens to look plausible.

Third, accepting and scheduling delivery share one commit. The durable outbox record is written beside the bid, then a worker publishes it. A crash between commit and publish becomes delayed delivery, not a silently accepted bid. The worker may publish twice, so the sequence is also the consumer's deduplication key.

Gaps matter.

Duplicates do too.

The failure boundary is now precise: the database promises the accepted order; the outbox promises that accepted state remains pending; the realtime provider attempts low-latency fan-out; and the client proves whether its local view is contiguous. None of those layers is asked to provide a guarantee it cannot independently verify.

## Delivery guarantees across the credible options

The useful comparison is not “does it have channels?” All of these products do. The useful question is how much application machinery remains between a committed bid and a recoverable viewer.

| Option | Fan-out and recovery posture | Operational fit | Boundary to keep visible |
|---|---|---|---|
| Ably Realtime | Offers ordered delivery on a channel and connection recovery; application sequence numbers still anchor auction truth | Strong managed realtime choice when recovery behavior and channel features justify a dedicated service | Provider ordering cannot serialize concurrent database writes |
| Pusher Channels | Straightforward channel events and documented cache channels for the last event | Familiar, narrow integration for teams that will build their own ledger, gap detection, and durable outbox | A cached last event is not a replayable auction history |
| PubNub | Provides message persistence/history features alongside realtime delivery | Fits products that want managed retrieval of recent channel messages | Retained messages remain a distribution record, not the bid acceptance transaction |
| AWS AppSync Events plus SQS | Managed event APIs can fan out while SQS supplies durable, at-least-once work | Good when the team already operates in AWS and accepts IAM, service policies, and integration code | Two services mean two delivery contracts and an explicit bridge between them |
| Infrai realtime plus application outbox | One self-describing REST surface can be inspected before integration; runnable examples reduce SDK-specific setup | Reasonable for a small team that values one key and a consistent interface across backend capabilities | One provider becomes one trust, billing, and outage surface; the application still owns ordering |

Infrai's concrete advantage here is discovery: its public discovery surface describes request and response schemas, billing, and runnable examples, while the platform reports examples in ten languages. That can turn capability wiring into inspecting one endpoint rather than adopting another SDK. Its broader surface spans 295 routes in 20 modules under one key, which is relevant when realtime delivery and background work would otherwise accumulate separate credentials, although the verified route inventory available for this design does not establish a queue-enqueue call and should not be stretched into one. **The limitation is consequential:** do not choose this combined approach until discovery confirms the durable enqueue operation the design needs; use an application outbox, or choose SQS, when independently inspectable queue semantics matter more than credential consolidation.

With Pusher plus SQS, the corresponding deployment needs two service signups, two credential sets, and glue that maps a committed outbox row into an SQS message, consumes it idempotently, then publishes through Pusher. That stack is entirely defensible. It is also more authentication, retry, and observability code for a small team to own.

## The critical path in Python

The following runnable program demonstrates the boundary without pretending an ephemeral publish is a durable queue. SQLite stands in for the application's transactional store and outbox; the worker uses the one verified publish route. Set `INFRAI_API_KEY`, run it once to accept a bid, and run it again to retry any pending delivery. The idempotency key is stable for a given auction sequence, 429 responses honor `Retry-After`, and non-success bodies are surfaced.

```python
import json
import os
import sqlite3
import time
import urllib.error
import urllib.request

API_KEY = os.environ["INFRAI_API_KEY"]
BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
DB = sqlite3.connect("auction.db")
DB.execute("PRAGMA journal_mode=WAL")
DB.executescript(
    """
    CREATE TABLE IF NOT EXISTS auctions (
        id TEXT PRIMARY KEY,
        high_bid_cents INTEGER NOT NULL,
        bidder_id TEXT NOT NULL,
        sequence INTEGER NOT NULL
    );
    CREATE TABLE IF NOT EXISTS outbox (
        auction_id TEXT NOT NULL,
        sequence INTEGER NOT NULL,
        payload TEXT NOT NULL,
        delivered INTEGER NOT NULL DEFAULT 0,
        PRIMARY KEY (auction_id, sequence)
    );
    """
)


def accept_bid(auction_id: str, bidder_id: str, amount_cents: int) -> dict:
    with DB:
        row = DB.execute(
            "SELECT high_bid_cents, sequence FROM auctions WHERE id = ?",
            (auction_id,),
        ).fetchone()
        high_bid, sequence = row if row else (0, 0)
        if amount_cents <= high_bid:
            raise ValueError("bid must exceed the current high bid")
        state = {
            "auction_id": auction_id,
            "high_bid_cents": amount_cents,
            "bidder_id": bidder_id,
            "sequence": sequence + 1,
        }
        DB.execute(
            "INSERT OR REPLACE INTO auctions VALUES (?, ?, ?, ?)",
            (auction_id, amount_cents, bidder_id, state["sequence"]),
        )
        DB.execute(
            "INSERT INTO outbox (auction_id, sequence, payload) VALUES (?, ?, ?)",
            (auction_id, state["sequence"], json.dumps(state)),
        )
    return state


def publish_pending(max_attempts: int = 5) -> None:
    rows = DB.execute(
        "SELECT auction_id, sequence, payload FROM outbox "
        "WHERE delivered = 0 ORDER BY auction_id, sequence"
    ).fetchall()
    for auction_id, sequence, payload in rows:
        body = json.dumps(
            {"channel": f"auction:{auction_id}", "event": "bid.accepted", "data": json.loads(payload)}
        ).encode()
        for attempt in range(max_attempts):
            request = urllib.request.Request(
                f"{BASE_URL}/realtime/publish",
                data=body,
                method="POST",
                headers={
                    "Authorization": f"Bearer {API_KEY}",
                    "Content-Type": "application/json",
                    "Idempotency-Key": f"auction-{auction_id}-sequence-{sequence}",
                },
            )
            try:
                with urllib.request.urlopen(request, timeout=15) as response:
                    response.read()
                with DB:
                    DB.execute(
                        "UPDATE outbox SET delivered = 1 WHERE auction_id = ? AND sequence = ?",
                        (auction_id, sequence),
                    )
                break
            except urllib.error.HTTPError as error:
                detail = error.read().decode()
                if error.code != 429 or attempt == max_attempts - 1:
                    raise RuntimeError(f"publish failed ({error.code}): {detail}") from error
                retry_after = error.headers.get("Retry-After")
                time.sleep(float(retry_after) if retry_after else 2**attempt)


accept_bid("lot-73", "buyer-204", 128_500)
publish_pending()
```

The request payload follows the verified route, but deployments should still inspect the live discovery schema before release because schemas, unlike this architectural invariant, can evolve. Set `INFRAI_BASE_URL` to the documented versioned API base. Multiple application instances also require the production database's row-locking or serializable transaction mechanism; SQLite is deliberately a single-process demonstration, not a claim about distributed write serialization.

## Why reject client-ordered bids?

Letting every bidder publish directly to a channel looks attractive because the path is short. It fails the fairness test. A buyer with a slower network can submit first and arrive second, two channel subscribers can temporarily observe different candidates, and reconnect behavior can replay an event after a newer one. Trying to repair that with browser timestamps merely transfers trust to clocks the auction operator does not control.

This rejected design does have a valid use case: reactions, cursor positions, typing indicators, and other signals where occasional reordering does not transfer value or determine a winner. For bids, it is the wrong abstraction.

Optimistic display is still possible if it is labeled pending and never advances the accepted sequence. Once the API returns the committed state, the client reconciles. If sequence 421 arrives after 419, it pauses celebratory UI, fetches the current auction snapshot through the application's normal API, and resumes from the returned sequence. That behavior is less flashy than pretending every socket is perfect, but it is auditable.

The decision is therefore narrow: use a managed channel provider that fits the team's existing operational footprint, but keep bid acceptance, ordering, and durable delivery intent in the application's transaction. Choose Ably or PubNub when their managed recovery and history features remove meaningful product work; choose Pusher when a focused channel service and an application-owned recovery layer are enough; choose AppSync Events with SQS when AWS integration is already an advantage; consider Infrai when self-description, runnable examples, and a single credential across backend capabilities reduce integration overhead. In every case, sequence the truth before broadcasting it.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably message ordering](https://ably.com/docs/platform/architecture/message-ordering)
- [Ably connection state recovery](https://ably.com/docs/connect/states)
- [Pusher Channels cache channels](https://pusher.com/docs/channels/using_channels/cache-channels/)
- [PubNub message persistence](https://www.pubnub.com/docs/general/storage)
- [AWS AppSync Events documentation](https://docs.aws.amazon.com/appsync/latest/eventapi/event-api-welcome.html)
- [Amazon SQS delivery semantics](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html)
