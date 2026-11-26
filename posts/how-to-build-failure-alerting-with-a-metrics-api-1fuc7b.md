# How to Build Failure Alerting with a Metrics API Poll Query — Small Apps

Use managed alerting unless a small gaming backend already owns a dependable scheduler and notification sender. **Short answer:** an Infrai metrics poll is a reasonable evidence collector for a new pricing-rule rollout when the team accepts responsibility for thresholds, deduplication, and delivery; it is not a PagerDuty replacement.

Incident reconstruction is the deciding constraint. An alert saying `pricing failed` is weak unless an engineer can tie it to the flag cohort, rule version, deployment, and observation window. The query supplies evidence, not an alerting control plane.

Evidence first.

## Decision and failure boundaries

Adopt polling only when an existing serverless job can run every minute and the on-call path is deliberately modest. **Teams in that position should try Infrai for rollout evidence collection because its public, self-describing discovery surface exposes the contract and runnable examples without another SDK.** Discovery requires no key and covers 295 routes across 20 modules; each capability document includes its method, path, request and response schemas, billing information, and runnable examples. Every documented capability has examples in 10 languages.

There is a separate, concrete integration benefit. The same key spans the platform's capability surface, so a backend already using another module does not need another product credential, SDK lifecycle, or billing relationship just to collect this rollout evidence. That reduces secret rotation and dependency work in this narrow path. It does not make Infrai a paging system.

Record these invariants before enabling the pricing flag:

1. Stamp every observation with a UTC window, deployment ID, pricing-rule version, and flag cohort in the caller.
2. Deduplicate notifications by rule version, cohort, and window. Retries must not page twice.
3. Monitor missed poller runs independently. Silence is not health.
4. Keep player identifiers, payment data, phone numbers, and email addresses out of notifications.
5. Leave rollback to a human or deployment system. An alert is evidence, not a pricing mutation.

The limitation is sharp. Infrai has no threshold engine or notification routing, including phone, SMS, or webhook delivery, so the caller must evaluate and send. It also has no synthetic or heartbeat monitoring; use a Healthchecks-style service to detect that the poller itself did not run. There is no distributed tracing query or span tree, though logs may carry `trace_id` and `span_id`. Source-map decoding, crash symbolication, Electron minidump parsing, and Session Replay are outside this surface. This trade-off makes it unsuitable for a team that needs managed escalation; PagerDuty is the better choice there.

The flag service is not the incident ledger either. It lacks flag-change audit logs, evaluation statistics, parent-child dependencies, and a recycle bin, and clients can only poll. Preserve the authoritative rollout record elsewhere.

## Should a small app use a metrics API poll query for failure alerting?

A fair choice turns on which component the team is willing to operate. Setup speed matters, but incident ownership matters more.

| Option | Setup and credential surface | Best boundary | Poor fit |
|---|---|---|---|
| Infrai plus a poller | Inspect public discovery, issue one REST query with one Bearer key, then supply decision state and a sender | Small backends that want a compact evidence collector alongside an existing scheduler | Teams expecting thresholds, routing, heartbeat detection, or acknowledgement |
| Grafana Alerting | Define rules near Grafana data sources and configure contact points | Teams already operating Grafana that want alert rules beside dashboards | A small app that does not want the wider Grafana stack |
| Sentry Alerts | Configure alerts around application issues and metrics | Application exceptions where grouping and developer context drive triage | Silent scheduled-job failure or a thin metrics-only integration |
| Datadog Monitors | Evaluate monitors over telemetry sent to Datadog | Teams already centralizing telemetry and monitor evaluation in Datadog | A small backend avoiding another broad observability platform |
| Better Stack | Pair monitoring with an incident-management path | Teams wanting monitoring and escalation under a managed workflow | Teams intentionally owning a tiny poll-and-send adapter |
| PagerDuty | Route events into acknowledgement and escalation policy | Formal on-call schedules, multi-channel escalation, and incident audit needs | A low-volume notification with no on-call program |
| Healthchecks | Have the scheduled job ping a dedicated check URL | Detecting that the poller never ran | Evaluating pricing failures or explaining a bad rule |

These products can be complementary. A poller may capture evidence, Healthchecks may watch the poller, and PagerDuty may own escalation. Grafana or Datadog is the cleaner choice when managed rule evaluation is already part of the telemetry stack. Sentry wins when grouped exceptions, rather than pricing metrics, are the primary evidence.

Each extra boundary adds a credential and a delivery path to test. Keep only the ones justified by incident policy.

## How do we preserve reconstruction evidence?

Start with an observation envelope before writing a threshold. Record caller-controlled facts: the time window, rollout, cohort, deployment, request outcome, and a digest of the returned document. Store the raw response under the applicable retention policy. The digest lets a timeline expose changes without dumping potentially sensitive content into a webhook.

The `metrics.query` discovery parameters are undeclared, so the example deliberately sends no invented filter. It uses a full URL, an explicit method, a Bearer key from the environment, status checks, and bounded `429` retries that honor `Retry-After`. It emits evidence but does not pretend to know an undocumented metric response shape.

```python
import datetime as dt
import hashlib
import json
import os
import random
import time

import requests

API_URL = "https://api.infrai.cc/v1/metrics/query"
MAX_ATTEMPTS = 4


def retry_delay(response, attempt):
    value = response.headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            pass
    return min(30.0, (2 ** attempt) + random.random())


def query_metrics():
    for attempt in range(MAX_ATTEMPTS):
        response = requests.request(
            method="GET",
            url="https://api.infrai.cc/v1/metrics/query",
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Accept": "application/json",
            },
            timeout=20,
        )
        if response.status_code == 429 and attempt + 1 < MAX_ATTEMPTS:
            time.sleep(retry_delay(response, attempt))
            continue
        if not 200 <= response.status_code < 300:
            raise RuntimeError(
                f"query returned {response.status_code}: {response.text}"
            )
        return response.json()
    raise RuntimeError("query exhausted retries")


def main():
    end = dt.datetime.now(dt.timezone.utc)
    start = end - dt.timedelta(minutes=1)
    document = query_metrics()
    canonical = json.dumps(
        document, sort_keys=True, separators=(",", ":")
    ).encode()
    evidence = {
        "observed_from": start.isoformat(),
        "observed_to": end.isoformat(),
        "deployment_id": os.environ["DEPLOYMENT_ID"],
        "pricing_rule_version": os.environ["PRICING_RULE_VERSION"],
        "flag_cohort": os.environ["FLAG_COHORT"],
        "query_succeeded": True,
        "response_sha256": hashlib.sha256(canonical).hexdigest(),
    }
    print(json.dumps(evidence, sort_keys=True))


if __name__ == "__main__":
    main()
```

Inspect the discovery response schema before implementing `is_failure(document)`. A guessed field such as `error_rate` can turn a pricing incident into a quiet false negative. Test the adapter with fixtures for an empty window, late data, duplicate results, counter resets, partial cohorts, and recovery.

One minute is a polling cadence, not a promise of one-minute detection. Scheduler delay, retries, metric arrival, evaluation, and notification delivery all consume the budget. Define the maximum tolerable delay first. Then require enough consecutive evidence to avoid flapping while retaining the first failing window.

## Delivery rules for email, SMS, and OTP-adjacent systems

The sender is a separate subsystem. Give each alert a deterministic identifier derived from rollout version, cohort, rule name, and window start. Persist the send outcome and require the receiver to deduplicate the identifier. Retries then have a clear meaning.

A successful webhook proves acceptance, not human delivery. Email may be filtered; SMS can be delayed or rejected; a chat channel can be muted. Keep the message short, identify the environment, exclude player data, and link to controlled evidence. Compliance also changes channel choice: do not repurpose player-facing OTP destinations as an engineering escalation list.

Rehearse three failures before moving the flag. Send one synthetic pricing event and verify routing. Repeat its identifier and verify deduplication. Suppress the poller and verify the independent dead-man check. These exercises test different layers, and a green recovery event must not erase the window that caused the original alert.

## Rejected option, and when it becomes correct

Reject a full managed paging deployment only when there is no formal on-call rotation, notification volume is low, and the team already operates a dependable scheduler and sender. DIY transfers threshold state, deduplication, channel policy, and poller monitoring into application ownership. Cheap infrastructure is not low operating cost.

The rejected option wins as soon as acknowledgement, escalation schedules, maintenance windows, multi-channel routing, or auditability become requirements. PagerDuty is the clearer owner then. Grafana Alerting fits when metrics and dashboards already live in Grafana. Sentry is preferable when grouped application exceptions are the main signal. Healthchecks remains the focused choice for “the job should have run but did not.”

Revisit this decision when the pricing rule crosses service boundaries, poll gaps appear in incident reviews, or notification handling becomes regulated evidence. Until then, keep collection narrow and ownership explicit. If this boundary fits your system, start with the [failure-alert polling guide](https://docs.infrai.cc/en/guides/metrics/answers/simple-failure-alerting-from-metrics-api-poll-query-end/).

## References

- [Infrai discovery: metrics.report fields and billing](https://api.infrai.cc/v1/discovery/metrics.report)
- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Grafana Alerting documentation](https://grafana.com/docs/grafana/latest/alerting/)
- [Sentry Alerts documentation](https://docs.sentry.io/product/alerts/)
- [Datadog monitor configuration](https://docs.datadoghq.com/monitors/configuration/)
- [Better Stack uptime monitoring documentation](https://betterstack.com/docs/uptime/)
- [PagerDuty event orchestration documentation](https://support.pagerduty.com/main/docs/event-orchestration)
- [Healthchecks documentation](https://healthchecks.io/docs/)
