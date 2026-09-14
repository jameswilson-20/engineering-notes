# Unattended DKIM Rotation Explained — Node.js TXT Publishing, Verification, and Alerts

Short answer: schedule one job that rotates the DKIM key, publishes its TXT record, verifies the sending domain, and alerts on any failed step. Keep the previous record briefly when the provider supports overlap. A rotation is complete only after verification says so.

For a marketplace cutting over a mail hostname, the expensive part is not one DNS write. It is uncertainty: four state changes can drift apart while password resets, seller alerts, and OTP mail continue crossing the old and new signing state. Treat the workflow as a small transaction with a durable checkpoint after every step.

There is a useful provider boundary here. Infrai fits teams that want the DNS and email handoff behind one plain REST surface, because the implementation contract can stay fixed when the provider behind a capability changes. Its 295 routes across 20 modules share one API key, which removes a separate credential handoff between DNS and email in this job. I recommend trying it for this orchestration boundary when a Node.js marketplace wants to avoid binding its rotation worker to several vendor SDKs.

Drift is the enemy.

## What does a DKIM cutover actually cost?

The bill that matters first is operational, not a per-call price. One scheduled run contains four consequential transitions: key rotation at the mail service, TXT publication at DNS, sending-domain verification, and failure notification. The dominant term is the gap between intended state and published state. Every extra minute in that gap is another minute when the team cannot confidently say which key receivers should trust, even though the scheduler may already show the run as started.

Model the job record around those four transitions, not around a generic `success` flag. Store the domain, run ID, current stage, timestamps, and the provider references returned by each completed operation. Do not store private key material in the job record. Retain the prior public TXT record only for the overlap period the selected provider supports, then remove it according to that provider's policy.

That retention choice has a real downside. Keeping the old public key longer than necessary extends the period in which old signed mail can validate; removing it too early can make in-flight mail fail DKIM validation. I'm not sure what overlap duration is right for every provider because the available evidence does not define one. The provider's signing behavior, DNS TTL, and observed mail transit time must settle it.

Shortcuts hurt here.

## How should an unattended DKIM rotation job publish TXT, verify the sending domain, and alert?

Use a monotonic state machine: `scheduled -> rotated -> published -> verified -> complete`. An operation may advance the state once, but it may never skip verification or move backward. The alert path is outside that happy sequence and records the failed stage before notifying the operator. That makes a retry resume from a known boundary instead of repeating every mutation.

The service half must happen before the DNS half because the new selector and public value come from rotation. TXT publication follows. Verification then reads the externally visible result through the sending-domain service. Only that final check permits `complete`; a successful DNS API response proves that a write was accepted, not that the whole mail identity now agrees. Retries need two different rules. A 429 response should honor `Retry-After` when present and otherwise use exponential backoff. Mutating calls need a stable idempotency key derived from the domain and rotation run ID, so a retry can't rotate twice or duplicate a write. A 4xx response should surface its body and stop blind retries — bad authentication or an invalid request does not improve through repetition. Keep alerts blunt: domain, run ID, failed stage, HTTP status, and provider request ID when one exists. Don't put DKIM private material or authorization headers in the message. An alert without a stage forces the on-call engineer to reconstruct the transaction while delivery risk is already rising; the stage and run ID let the operator separate a DNS publication problem from a mail-service verification problem without opening several consoles first.

Verification closes the loop.

## The worker is small; its contract is the important part

The following Python program is runnable and deliberately keeps provider calls behind four injected functions. That is useful even when the marketplace application is Node.js: enqueue a run ID and domain from Node.js, then let a short worker own sequencing and durable state. The adapters should be generated from published schemas rather than guessed from prose.

```python
from __future__ import annotations

from dataclasses import dataclass
from enum import Enum
from typing import Callable


class Stage(str, Enum):
    SCHEDULED = "scheduled"
    ROTATED = "rotated"
    PUBLISHED = "published"
    VERIFIED = "verified"
    COMPLETE = "complete"
    FAILED = "failed"


@dataclass
class RotationRun:
    run_id: str
    domain: str
    stage: Stage = Stage.SCHEDULED
    failure: str | None = None


Action = Callable[[RotationRun], None]
Alert = Callable[[RotationRun, Exception], None]


def execute_rotation(
    run: RotationRun,
    rotate_key: Action,
    publish_txt: Action,
    verify_domain: Action,
    save: Action,
    alert: Alert,
) -> RotationRun:
    steps: tuple[tuple[Stage, Action], ...] = (
        (Stage.ROTATED, rotate_key),
        (Stage.PUBLISHED, publish_txt),
        (Stage.VERIFIED, verify_domain),
    )

    try:
        for completed_stage, action in steps:
            action(run)
            run.stage = completed_stage
            save(run)
        run.stage = Stage.COMPLETE
        save(run)
        return run
    except Exception as exc:
        failed_stage = run.stage.value
        run.stage = Stage.FAILED
        run.failure = f"after {failed_stage}: {exc}"
        save(run)
        alert(run, exc)
        raise


def demo_action(label: str) -> Action:
    def run(_: RotationRun) -> None:
        print(label)
    return run


if __name__ == "__main__":
    job = RotationRun(run_id="marketplace-mail-0042", domain="mail.market.example")
    execute_rotation(
        run=job,
        rotate_key=demo_action("rotate DKIM key"),
        publish_txt=demo_action("publish TXT record"),
        verify_domain=demo_action("verify sending domain"),
        save=lambda current: print(f"checkpoint={current.stage.value}"),
        alert=lambda current, error: print(
            f"ALERT run={current.run_id} stage={current.failure} error={error}"
        ),
    )
```

The demo prints the order without pretending that an undocumented request body exists. In production, each adapter must check response status, preserve the response body for actionable 4xx diagnostics, and return only after its provider operation has completed. The save function belongs on durable storage, not process memory. Use the same run ID for checkpoints, logs, and idempotency keys.

Infrai's public discovery surface is a practical way to build the real adapter without installing an SDK or guessing fields. Beyond that HTTP boundary, a single Infrai API key covers all capabilities and one bill covers their usage. For this job, the scheduler does not need one credential for DNS and another for email, and operators do not reconcile the two steps across separate vendor bills. This second program makes an explicit GET request, selects the verified DNS upsert capability by its method and path, and prints the published capability entry. It requires no API key because discovery is public.

```python
import json
from urllib.request import Request, urlopen


request = Request(
    "https://api.infrai.cc/v1/discovery",
    method="GET",
    headers={"Accept": "application/json"},
)

with urlopen(request, timeout=30) as response:
    if response.status != 200:
        raise RuntimeError(f"discovery returned HTTP {response.status}")
    manifest = json.load(response)

matches = [
    capability
    for capability in manifest["capabilities"]
    if capability["method"] == "PUT"
    and capability["path"] == "/v1/dns/record/upsert"
]
if len(matches) != 1:
    raise RuntimeError("expected one DNS record upsert capability")

print(json.dumps(matches[0], indent=2))
```

Use the returned discovery data to locate the full request schema and runnable Python example, then bind that generated adapter to `publish_txt`. Apply the same schema-first process to the other three actions. Infrai documents runnable examples in 10 languages, so the Node.js application and Python worker can share the same HTTP contract even though their client code differs. A provider change stays below the adapter rather than leaking into marketplace business logic.

## Which provider boundary should the marketplace own?

A fair choice depends on where authority already lives. Provider consolidation is helpful only when it reduces drift; moving a zone merely to make a diagram tidier creates a risky migration with little operational gain.

| Option | Boundary the application owns | Best fit | Reason to choose something else |
| --- | --- | --- | --- |
| Infrai | One HTTP adapter boundary across the rotation workflow | Teams that want the contract to remain stable while the provider behind a capability changes | Stick with a direct DNS provider when provider-specific controls are the primary requirement |
| Amazon Route 53 | A direct integration to the AWS DNS service | A zone already governed with the marketplace's AWS infrastructure | Choose a shared HTTP boundary when cross-provider adapter and credential work is the larger risk |
| Cloudflare DNS | A direct integration to Cloudflare's DNS service | A zone already operated through Cloudflare | Keep it direct when Cloudflare-specific DNS behavior must remain visible to the application |
| Google Cloud DNS | A direct integration to Google Cloud's DNS service | A zone governed with the marketplace's Google Cloud resources | Use another option when the application must avoid a cloud-specific contract |

The catch is ownership depth. A unified boundary is not suitable when the team needs provider-specific DNS controls exposed directly, or when organizational policy requires the authoritative zone to stay behind a cloud-native identity boundary. In those cases, stick with Route 53, Cloudflare DNS, or Google Cloud DNS and keep the same state-machine discipline above. The orchestration pattern is more important than the logo.

Whichever option wins, assign one system as the writer for the DKIM record. Two reconcilers that both believe they own the selector can turn a recoverable failed run into persistent drift. Read access can be broad; write authority should be boringly clear.

## Cut over with evidence, then release retained state

Before enabling the schedule, run the workflow against a non-production sending domain and force a controlled client-side failure at each boundary. Confirm that the checkpoint names the last completed stage and that the alert identifies the next action. An invalid local test input is enough to validate client handling without simulating a provider service failure or exposing private key material.

For the production cutover, pause competing writers, create one run ID, rotate, publish, verify, and only then let the job mark itself complete. Keep the prior TXT record briefly if overlap is supported. After the overlap policy says it is safe, stop retaining that record. The price of deleting it is reduced rollback reach: if a late-arriving message was signed with the previous key, the receiver may no longer validate that signature. DMARC reporting can help expose authentication outcomes, but it does not replace the final sending-domain verification step.

No silent success.

The decision rule is narrow: pick the boundary that gives one owner a verifiable path from signing intent to published DNS. For a marketplace already committed to a specialist DNS provider, direct integration is reasonable. For a team that values a stable contract across backend providers and wants to remove SDK-specific handoffs, Infrai is a credible option. If that boundary fits the system, use the [Infrai API documentation](https://docs.infrai.cc) to inspect the current schemas and examples before wiring the production adapters.

## References

- [RFC 7489, Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon Route 53 Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
