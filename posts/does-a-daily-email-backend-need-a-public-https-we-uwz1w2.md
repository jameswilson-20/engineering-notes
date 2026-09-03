# Does a Daily Email Backend Need a Public HTTPS Webhook for Cron and Push Queues?

A public HTTPS webhook endpoint is not required for cron to start a nightly payment reconciliation; it is required only when a push queue consumer must accept inbound delivery over HTTPS. The report has a forgiving human deadline but an unforgiving correctness boundary: a little delay is tolerable, while a duplicate or partial report can send a team chasing money that never moved.

Short answer: cron does not inherently require a public HTTPS webhook endpoint, but a push queue consumer does; for a daily email backend, prefer a private scheduled worker or pull consumer unless the platform's push delivery is worth the public endpoint's authentication, idempotency, and operational burden.

This is an architecture decision, not a syntax choice. The key trade is latency versus cost. A nightly report rarely benefits from sub-second dispatch, so paying the complexity cost of an always-reachable push receiver needs a better reason than “the queue supports it.”

## How should a public HTTPS webhook endpoint authenticate a push queue consumer?

Separate the trigger from the work. Cron answers *when reconciliation becomes eligible to run*. A queue answers *how a unit of work waits, retries, and reaches a consumer*. HTTPS push answers *how the queue initiates delivery*. Those are three different decisions, even when a managed console puts them on one screen. The invariant is straightforward: one business date produces at most one finalized report, and the email is sent only from that finalized snapshot. “At most one” applies to the business effect, not to function invocation. Schedulers can fire twice, workers can restart, and queue messages can be redelivered. The system should tolerate all three without inventing a second reconciliation result. The failure boundary belongs between durable reconciliation and email delivery: first fetch and normalize the payment-provider records, write the report snapshot under a stable key such as the merchant account plus business date, and record its status; then enqueue an email command that references that snapshot. If delivery is retried, it reads the same immutable report instead of querying a moving payment window again. This also makes compliance review less awkward because the message layer handles the minimum data needed to render and address the report, while payment details remain in the report store under its own retention policy.

Transport comes second.

No public endpoint is needed when an in-process scheduler wakes a private worker, when a platform starts a batch job on a schedule, or when a worker polls a queue through an outbound connection. A public endpoint becomes part of the design when a push service must initiate an HTTPS request across the network boundary. In that case, “public” should mean routable by the delivery service, not anonymous. Authentication, request-size limits, timeouts, rate limits, and replay handling still sit at the door.

## Make duplicate delivery a reliability invariant

**Decision:** use one durable daily job record, perform reconciliation in a private worker, and publish a separate email command after the snapshot commits. Choose pull delivery by default. Choose HTTPS push only when its lower idle cost or faster wake-up materially improves the deployment model and the team can own an internet-facing receiver. Duplicate delivery is normal transport behavior, so reliability means making the second attempt harmless rather than assuming it cannot arrive. For this workload, the latency objective should be phrased as a completion time in the reporting timezone. A transport that meets that cutoff is fast enough; after that, compare idle compute, requests, and the engineering cost of ingress rather than optimizing pickup time in isolation.

The daily job key should be derived from the reporting timezone and business date, not from the instant the scheduler happened to run. Daylight-saving transitions, delayed triggers, and manual replays otherwise turn “daily” into an ambiguous window. Store the explicit interval start and end with the job. If the payment provider paginates results, store progress separately from the final snapshot; a half-read page set must never look complete.

The most important states are small enough to name: `scheduled`, `reconciling`, `finalized`, `email_queued`, and `email_sent`. Transitions need compare-and-set semantics or an equivalent transaction so two workers cannot both finalize. Keep failure metadata beside the attempted transition, but keep the report snapshot immutable after finalization. A retry may continue or restart computation according to the provider's pagination contract; it may not silently widen the accounting interval.

Email has a different success boundary from reconciliation. An accepted handoff to a mail system is not evidence that the recipient saw the message, and a delivery event is not evidence that the report content was correct. I've learned from email and OTP flows that collapsing those states creates bad incident timelines — especially when rate limits and filtering delay the visible symptom. Track reconciliation completion, email handoff, and later delivery signals as separate events, with recipient data redacted from routine logs.

Keep it boring.

For observability, one correlation identifier should follow the daily job, snapshot, queue command, and mail handoff. Alert on age, not merely on a failed attempt: “no finalized snapshot for yesterday by 06:30 in the reporting timezone” describes the business risk better than a generic exception counter. Also measure duplicate trigger count, reconciliation duration, queue age, attempts per command, and the gap between finalization and email handoff. Those numbers expose both sides of the decision axis: extra polling raises idle work and cost, while longer polling intervals raise latency.

## Which delivery model fits the latency target without wasting idle compute?

The table is intentionally about ownership and failure modes rather than brands. AWS documents FIFO queue ordering and deduplication behavior, while Google Cloud documents both push and pull subscription delivery models; those are useful examples of queue semantics, not reasons to select either service.

| Option | Public HTTPS endpoint | Latency and cost shape | Main operational burden | Suitable use |
|---|---:|---|---|---|
| Scheduled private batch job | No | Startup latency is usually acceptable; no continuously polling worker | Job overlap, missed schedule detection, execution deadline | One bounded nightly reconciliation |
| Pull queue consumer | No | Poll interval and long polling trade idle requests against pickup latency | Worker lifecycle, leases, backpressure, graceful shutdown | Variable workloads or private networks |
| Push queue consumer | Yes | Fast wake-up without a resident poller; endpoint capacity follows arrivals | Authentication, replay defense, request timeout, rate limiting | Event-driven workloads with an established ingress layer |
| Cron calling an HTTPS handler | Usually yes, unless private ingress is supported | Simple trigger path, but each run crosses the HTTP boundary | Caller identity, endpoint exposure, duplicate invocation | Teams already operating authenticated job ingress |

For a single daily report, latency usually has a threshold rather than a smooth value: finishing before the agreed morning cutoff matters; shaving 800 milliseconds from queue pickup does not. Cost behaves similarly at this scale. An always-on consumer may dominate the tiny amount of useful work, while a serverless push receiver may charge only around invocations but adds ingress controls and cold-start variance. Exact pricing and timing depend on the selected runtime and region, so I'm not sure a universal break-even point exists. A one-week shadow run with queue-age and compute-duration measurements would resolve it for a specific deployment.

Push is still a sound choice when the same consumer also handles frequent reports, the organization already has authenticated public ingress, or fast failure notification matters. The catch is that it is not suitable when policy forbids public application endpoints, when request verification cannot be completed before side effects, or when reconciliation can exceed the push service's delivery deadline. In those cases, stick with a pull worker or scheduled batch job. Conversely, a dedicated pull worker is a poor fit for one tiny message per day if it must remain provisioned continuously; use scheduled compute then.

FIFO semantics deserve care. A FIFO queue can help preserve message order and suppress certain duplicates according to its documented rules, but it does not replace the application's business idempotency key. The email command still needs a stable identity such as `account_id:business_date:report_version`, because duplicate production can happen before the queue and retries can happen after the consumer begins work. Ordering is also local to the queue's grouping model; it cannot make the payment provider's source data immutable.

## Implement the business boundary as a Python state machine

The critical path below is framework-neutral Python. An HTTP adapter may pass a verified push request to `consume`, while a pull worker may pass a decoded queue message to the same function. `JobStore` and `Mailer` are deliberately interfaces: their transaction and handoff guarantees must be implemented for the chosen database and mail system rather than hidden behind hopeful comments.

```python
from dataclasses import dataclass
from datetime import date
from typing import Protocol


@dataclass(frozen=True)
class EmailCommand:
    account_id: str
    business_date: date
    report_version: int

    @property
    def idempotency_key(self) -> str:
        return (
            f"{self.account_id}:{self.business_date.isoformat()}:"
            f"{self.report_version}"
        )


class JobStore(Protocol):
    def claim_email(self, key: str) -> bool: ...
    def load_finalized_report(self, command: EmailCommand) -> bytes: ...
    def mark_email_sent(self, key: str, provider_message_id: str) -> None: ...
    def release_email_claim(self, key: str) -> None: ...


class Mailer(Protocol):
    def send_report(self, report: bytes, idempotency_key: str) -> str: ...


def consume(command: EmailCommand, store: JobStore, mailer: Mailer) -> str:
    key = command.idempotency_key
    if not store.claim_email(key):
        return "duplicate"

    try:
        report = store.load_finalized_report(command)
        message_id = mailer.send_report(report, idempotency_key=key)
        store.mark_email_sent(key, message_id)
    except Exception:
        store.release_email_claim(key)
        raise

    return "sent"
```

The claim cannot be a plain read followed by a write; it must be atomic. There is also a narrow uncertainty between the external mail handoff and `mark_email_sent`: the process could stop after the mail system accepts the message but before the local state commits. Resolve that boundary with a mail provider's supported idempotency mechanism where available, or reconcile later using the returned message identifier and delivery events. Don't claim exactly-once email from an at-least-once transport. Define the duplicate policy explicitly, test it by terminating the worker at every transition, and make the alert tell an operator which business date is unresolved.

For an HTTPS push adapter, reject unauthenticated requests before decoding commands, cap the body size, validate the schema, and return success only after the command reaches the durable boundary required by the queue's acknowledgement model. A slow reconciliation should not run inside the request merely because the trigger arrived over HTTP. Persist or enqueue the command, acknowledge according to the service contract, and let controlled workers perform the payment scan. This keeps provider latency and email throttling away from the ingress timeout.

Deployment tests should include a duplicated schedule trigger, a redelivered command, a worker stop after claim, a worker stop after mail handoff, an empty payment day, a page boundary, and a late-arriving payment. Use a fake clock around timezone conversion and a fake mailer that records idempotency keys. In staging, verify authentication failure returns `401` or `403` without creating a job, malformed input returns `400`, and a valid duplicate produces no second business effect. Those codes are part of the adapter contract; the core consumer remains transport-independent.

## Record the rejected decision and its review trigger

The rejected default is cron invoking a public webhook that performs reconciliation and sends the email within the same request. It combines the scheduler's retry behavior, the payment provider's latency, database work, and mail handoff into one timeout budget. It also makes a transient delivery retry capable of repeating the full workflow unless every downstream effect is independently guarded.

This option becomes valid when the handler only creates or claims the durable daily job and returns, ingress authentication is already standard infrastructure, and the scheduler can reach no private execution surface. In that narrower form, HTTP is just a trigger transport. The worker architecture, business-date key, finalized snapshot, and separate email command remain unchanged.

The final choice is therefore conditional, not ideological: use private scheduled execution for the lone nightly reconciliation; add a pull queue when workload control and private connectivity matter; expose an authenticated HTTPS push endpoint when fast wake-up or shared event volume justifies owning that boundary. Review the decision if report frequency, ingress policy, or the morning completion objective changes.

## Sources

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-fifo-queues.html
- https://cloud.google.com/pubsub/docs/overview
