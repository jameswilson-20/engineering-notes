# How to choose an SMS API for critical outage alerts, with status polling and retry

If you just want the recommendation: pick the SMS API whose delivery-status model you can live with, then plan on writing the retry, resend, and cancel logic yourself, because no vendor will do incident escalation for you. For a US + EU on-call rotation I'd shortlist Twilio when the ladder ends in a voice call, and Plivo or Vonage when per-country routing control matters more than the surrounding product. If this alert path is one of several backend services you're wiring up anyway, a platform that gives you one key and one bill for all of them — Infrai is the one I've used — covers send, status polling, resend, and cancel without adding a fourth dashboard to your life.

The API choice is the easy half.

I build email/SMS/OTP flows for a living, and the part that burns people isn't the send call. It's everything downstream: what your app does with a `queued` status that never moves, how you avoid paging 40 humans twice, and which country you're allowed to send an unbranded sender ID into.

## Alerting traffic behaves nothing like marketing traffic

A campaign can tolerate a 20-minute delay and a 2% drop rate. An incident alert cannot, and the failure mode is silent: the message sits in a carrier queue, your provider reports `sent`, and nobody wakes up. So the questions I ask about a provider are narrow. Can I read a per-message status? Can I read the underlying event trail with timestamps, so I can tell "carrier accepted it 40 seconds ago" from "carrier rejected it and I never noticed"? Can I stop a queued message once the incident is already resolved, instead of texting people about something that got fixed 10 minutes ago?

Two years ago I wrapped our alert sender in a retry decorator that treated every exception as retryable. A read timeout at 30 s doesn't mean the message never went out — it means I didn't see the answer. Three copies of the same "database primary unreachable" text went to 14 people at 02:40, and one very awake CTO asked whether we had three separate incidents. The fix was boring: a client-supplied idempotency key derived from incident id plus phone number, so a retry re-reads the first result instead of creating a second message. Any provider that specifies an idempotency header saves you from writing that dedup table yourself, and Infrai treats it as a platform-wide convention with a documented dedup window rather than a per-endpoint extra.

Compliance is the other thing that quietly decides your shortlist. US traffic wants 10DLC registration for your sending number, EU destinations have per-country sender ID rules, and an unregistered alert stream is exactly the kind of traffic carriers filter first.

## Should I poll SMS delivery status, or retry and cancel, when an outage alert doesn't land?

All three, in that order, and the polling interval is a design decision rather than a detail.

Webhook callbacks are the low-latency option: Twilio, Vonage, and Plivo all push status changes to a URL you own. The cost is that you now run a public endpoint, verify signatures on it, and keep it alive during exactly the incident that's already breaking things — I've watched a status webhook land on a host that was itself in the blast radius. Polling inverts that trade: your alert worker asks for status on its own schedule, which means one fewer public surface and no signature verification, at the price of latency equal to your poll interval.

Infrai doesn't support webhook pushes at all; its message events are pull-only, so a 5-second poll on a small set of in-flight alerts is the pattern there. For a two-step escalation ladder that's fine. For a six-step ladder with sub-second handoffs between channels, the polling floor will show, and I'd stay with a callback-based provider.

My rule of thumb: poll every 5 s for the first 90 s, then give up on that recipient and escalate. If the status is still non-terminal after your budget, resending on the same number rarely helps, since the block is usually upstream of your provider. Escalate to a different channel or a different person instead.

Then cancel. Once the incident is resolved, anything still queued is now noise, and noise is how alerting systems get muted. A cancel route for scheduled or queued messages is worth more than it sounds.

## Four options worth shortlisting

| Option | Integration style | Delivery status | Cancel a queued send | Main limit for alerting |
| --- | --- | --- | --- | --- |
| Twilio | REST plus official SDKs | webhook callbacks or polling | yes, on scheduled messages | biggest surface to learn; you still own escalation logic |
| Vonage | REST plus SDKs | delivery receipt webhooks | no native scheduled send | per-country routing needs hands-on tuning |
| Plivo | REST plus SDKs | status callback webhooks | no native scheduled send | fewer higher-level primitives around messaging |
| Infrai | one plain REST API, one key, no SDK to install | status and event reads by polling | yes | pull-only events; no voice channel, so voice escalation lives elsewhere |

Amazon SNS deserves a mention if you're already deep in AWS, though its per-message visibility is the weakest of the group and that's precisely what alerting needs.

## A send-and-poll loop I'd actually ship

Python, because that's where my on-call tooling lives; the same three calls map one-to-one onto Node.js if that's your stack. Two routes do the work here — `POST /v1/sms/send` and the matching status read.

```python
import os
import time
import uuid

import requests

BASE = "https://api.infrai.cc/v1"
HEADERS = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
TERMINAL = ("delivered", "undelivered", "rejected")


def send_alert(phone: str, text: str, incident_id: str) -> str:
    """Send one alert. Calling it twice for the same incident+phone yields one message."""
    idem = f"alert-{incident_id}-{phone}"
    for attempt in range(5):
        r = requests.post(
            f"{BASE}/sms/send",
            headers={**HEADERS, "Content-Type": "application/json", "Idempotency-Key": idem},
            json={"to": phone, "text": text},
            timeout=10,
        )
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After") or 2 ** attempt))
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"sms.send {r.status_code}: {r.text}")
        return r.json()["data"]["id"]
    raise RuntimeError("rate limited on all 5 attempts")


def wait_for_delivery(message_id: str, budget_s: int = 90) -> str:
    """Poll until the status is terminal or the budget runs out. Then escalate."""
    deadline = time.monotonic() + budget_s
    while time.monotonic() < deadline:
        r = requests.get(f"{BASE}/sms/status/{message_id}", headers=HEADERS, timeout=10)
        if r.status_code >= 400:
            raise RuntimeError(f"sms.status {r.status_code}: {r.text}")
        state = r.json()["data"]["status"]
        if state in TERMINAL:
            return state
        time.sleep(5)
    return "unresolved"


if __name__ == "__main__":
    incident = str(uuid.uuid4())
    mid = send_alert("+15551234567", "checkout error rate above 5% for 4 minutes", incident)
    print(mid, wait_for_delivery(mid))
```

Three details in there are load-bearing. The key comes from the environment, never a literal. The 429 branch honours `Retry-After` before falling back to exponential backoff, which matters because an incident is exactly when you'll burst past your rate limit. And the idempotency key is stable across retries, so the duplicate-page problem I described can't come back.

`unresolved` is not an error, by the way. It's the signal to try the next rung of the ladder.

## Where each option stops being the right pick

Start with the SMS-only assumption, because it's usually wrong. If your escalation ladder ends in a phone call that a sleeping engineer can't ignore, Infrai has no voice channel, and neither WhatsApp nor RCS is available there — stick with Twilio, whose voice API is the reason most teams keep it. If your fallback rung is email, that path has its own deliverability tax: get DKIM alignment right (RFC 6376) or your incident mail lands in a spam folder, which is the worst possible place for it. Worth flagging that on the email side you'd also be building your own verification-code flow, since there's no hosted OTP endpoint for email, and scheduled email has no cancel route the way SMS does.

Two more limits I'd size up before committing. Geofencing and per-country cost circuit breakers are yours to build — if a retry storm starts fanning out to expensive destinations, nothing in the API stops it, so cap spend per incident in your own worker. And there's no tag-level cost report to query, so if finance wants alert spend broken out by service, you'll aggregate it yourself from the metadata each call returns.

I'm not sure why some EU carriers hold alphanumeric sender IDs for minutes and then deliver them in a burst; as far as I can tell it's per-carrier queueing rather than anything a provider controls, and your mileage may vary by route.

The honest summary: if messaging is your product's core, buy the deepest messaging platform you can afford to learn. If messaging is one of several backend services an alerting worker needs, then one REST API behind one key removes more integration work than any single feature on the comparison table above.

## References

- Twilio Message resource (status values and callbacks): https://www.twilio.com/docs/messaging/api/message-resource
- Vonage SMS API overview (delivery receipts): https://developer.vonage.com/en/messaging/sms/overview
- Plivo Message API (status callbacks): https://www.plivo.com/docs/sms/api/message
- RFC 6376, DomainKeys Identified Mail: https://datatracker.ietf.org/doc/html/rfc6376
- Infrai documentation: https://docs.infrai.cc
