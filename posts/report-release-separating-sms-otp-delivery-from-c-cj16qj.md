# Report Release: Separating SMS OTP Delivery from Carrier Filtering and Sender Registration

Separate SMS OTP delivery from report authorization before diagnosing carrier filtering in a healthtech workflow that sends a generated report as an email attachment. The deciding constraint is integration effort: a small implementation still needs to distinguish an accepted request, a verified login, and an authorized attachment send, because collapsing those events makes failures difficult to investigate and recovery dangerous.

TL;DR: investigate SMS OTP failures through sender eligibility, destination-specific requirements, delivery evidence, and anti-fraud decisions; release the report only after the application has independently established authorization. US A2P 10DLC registration is a concrete requirement for the applicable messaging route, not a universal explanation for missing codes across the US and EU.

This decision record uses a generated report named `report-042.pdf` and a login attempt named `attempt-017` as illustrative identifiers. They are example data, not patient records or measured results. The question is why an OTP fails to reach the user and what that failure permits the report workflow to do.

## Why can SMS OTP delivery fail after sender registration?

Very little by itself.

A user who cannot see a code has supplied an observation, not a carrier diagnosis, and a request accepted by a messaging interface doesn't establish that the person completed authentication. The useful investigation begins by finding the last event for which the application has evidence: was the request rejected before submission, accepted for sending, assigned a delivery outcome, or followed by successful verification? Treat an absent outcome as unknown until the relevant evidence arrives.

US A2P 10DLC documentation establishes a specific registration framework for application-to-person traffic using US ten-digit long codes. Check whether the actual sender and traffic fall within that framework before treating registration as either the cause or the cure. A registration record answers an eligibility question; it cannot, by itself, prove the delivery of an individual code. The reference also documents filtering and messaging limits, which is why a registered sender should not be modeled as a delivery guarantee.

Don't export that US rule into an EU-wide rule. The cited US registration documentation doesn't establish the requirements for every European destination, sender type, or route; an EU investigation needs the applicable destination and route documentation before assigning a registration-related cause. Keep that gap explicit in the incident record. “EU filtering” is too broad to be an actionable diagnosis.

Shared routes deserve skepticism.

Record whatever route information the messaging service actually exposes and ask whether it identifies a shared sender, a shared delivery path, or neither. Without attributable route evidence, shared infrastructure remains a hypothesis. Switching routes and seeing a different result would still leave several variables uncontrolled, so it should not become a retrospective claim that another tenant caused the failure.

## Decision: preserve authorization across uncertain delivery

The proposed invariants are application rules: the report belongs to an authorized account; the attachment recipient comes from that account's approved destination; and an OTP send event cannot substitute for successful verification. Persist those decisions alongside the report's immutable identifier. Keep generated report content out of delivery logs.

Unknown means pending.

For this design, the failure boundary between authentication and report dispatch is deliberately durable. Store the verified attempt and the authorized dispatch intent transactionally where the application's data layer supports it, then let a worker perform the external send. This introduces a queue or outbox and a worker to operate, but it keeps a browser retry from directly creating a second report-send operation, and it gives the team a place to reconcile an external response that was lost. It doesn't create an exactly-once email guarantee.

Anti-fraud controls belong in the evidence trail as their own decision. A locally rejected request has no carrier delivery outcome to inspect; a request accepted by the application may still encounter downstream controls. Record the component that made the decision when that information exists, and preserve an unknown classification when it doesn't. Repeatedly requesting another code without this distinction can obscure the original attempt and consume messaging capacity without resolving the user's login.

There are two different durability questions here. The application can choose to preserve a dispatch intent reliably within its own storage boundary, while external message delivery remains a separate outcome requiring external evidence. A durable row is not a delivered report. That distinction matters particularly when the generated file is later replaced: retries should refer to the already authorized report version rather than whatever file currently occupies a mutable storage key.

## Options and their integration costs

The comparison below evaluates proposed designs, not commercial products. The primary cost is the number of states and recovery paths the team must implement and maintain.

| Design | Integration effort | Failure boundary | Appropriate use |
| --- | --- | --- | --- |
| Send the attachment directly from the verified request handler | Fewest initial components | An external send and a local response can disagree after a timeout | A prototype that explicitly accepts manual reconciliation |
| Persist an authorized intent, then dispatch from a worker | Adds durable intent storage, worker ownership, and reconciliation | Authentication is recorded separately from external delivery | A report workflow that needs retryable processing and an inspectable decision history |
| Start report dispatch when the OTP request is accepted | Superficially short path, but no verified-user boundary | Messaging acceptance is mistaken for authorization | No valid use for releasing a protected report; acceptance can trigger non-sensitive progress UI |

The middle option is the decision here.

The trade-off is more integration work up front: its additional state lets the operator tell whether the user needs authentication recovery, the report needs generation, or a dispatch needs reconciliation. Combining those into a generic “notification failed” field reduces schema work while transferring the ambiguity into every support investigation. This design is not suitable for a disposable prototype whose team has explicitly accepted manual reconciliation and cannot operate a worker; the direct-send option has fewer components in that limited setting.

One limitation remains.

A worker that retries every ambiguous external send can still create duplicate email, regardless of how carefully the intent table deduplicates local jobs. Preserve an unknown submission state and reconcile it using the sending interface's documented capabilities; don't assume that every interface supports an idempotency key or a searchable message identifier.

## Critical path: authorize once, dispatch separately

The Python below is a policy example using application-defined objects, not an SDK integration. Its transaction interface must provide the documented atomic behavior in the implementation, and the unique dispatch key must be enforced by storage. It intentionally stops at durable intent creation, where the application's transaction boundary ends.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class ReportRequest:
    account_id: str
    report_version_id: str
    login_attempt_id: str


def authorize_report_email(store, request: ReportRequest):
    with store.transaction() as tx:
        attempt = tx.get_login_attempt(request.login_attempt_id)
        if attempt.account_id != request.account_id:
            raise PermissionError("Login attempt belongs to another account")
        if attempt.state != "verified":
            raise PermissionError("Verified login required")

        report = tx.get_report_version(request.report_version_id)
        if report.account_id != request.account_id:
            raise PermissionError("Report belongs to another account")
        if report.state != "ready":
            raise ValueError("Report is not ready")

        recipient = tx.get_approved_report_recipient(request.account_id)
        dispatch_key = (
            request.account_id,
            request.report_version_id,
            recipient.id,
        )
        return tx.insert_dispatch_intent_if_absent(
            key=dispatch_key,
            report_version_id=report.id,
            recipient_id=recipient.id,
            state="pending",
        )
```

The example defines one automatic send per report version and approved recipient. That is a policy choice, not a universal deduplication rule: a deliberate repeat send needs a separately authorized operation, or this key would suppress it. The recipient record and report version must remain resolvable with the intended meaning while the worker processes the intent. Changing an address must not silently redirect an already authorized report.

Successful OTP verification is only one input to authorization. The ownership checks remain necessary, and a production implementation must apply its session-validity and recipient-approval policies inside the same decision boundary. The example doesn't establish a complete health-data compliance posture.

## Rejected shortcut and operational acceptance

Direct sending from the request handler is rejected for this report workflow because an external email operation cannot be made atomic merely by placing application code inside a database transaction. It remains a reasonable prototype choice when manual reconciliation is an explicit requirement the team accepts. The acceptance-triggered shortcut has a narrower valid role: update a progress indicator after requesting an OTP, with no release of protected report data.

Before deployment, exercise the uncertain cases. Test a rejected SMS request, a missing delivery outcome, a verified attempt for the wrong account, two concurrent requests for `report-042.pdf`, and a worker that loses its response after submitting the email. Assert the authorization decision and stored state in each case; a test that only checks an HTTP success response misses the boundary this design exists to protect. Use synthetic reports and test recipients.

Operationally, correlate the login attempt, sender configuration, destination classification, documented delivery status, report version, and dispatch intent, while excluding OTP values and attachment contents from logs. Keep rejection, expiration, unknown submission, and confirmed outcomes distinguishable. Deploy schema changes before workers that require them, and assign ownership for reconciling unknown submissions before enabling automatic retries. Messaging spend is a consequence of retry policy here, not the decision's central argument.

The acceptance criterion is concrete: a missing SMS must leave a traceable authentication state and must not release the attachment. Registration evidence, carrier evidence, route evidence, and anti-fraud evidence should narrow the diagnosis only as far as they actually support. Choose the integration whose failure states the team can explain and operate.

## References

- US A2P 10DLC registration, messaging limits, and filtering context: https://www.twilio.com/docs/messaging/compliance/a2p-10dlc
- Email-sending service context; attachment handling and delivery behavior require the applicable interface documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
