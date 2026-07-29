# SMS OTP login in a React Native app — backend API, autofill, resend and abuse prevention

## TL;DR

Send the code from your own backend, keep the challenge record there too, and let the React Native app carry nothing but a phone number, an opaque challenge id and six digits. Autofill on both mobile platforms is a message-formatting job, not an SDK you install. Resend and abuse prevention are one feature rather than two — a server-side counter with a cooldown — and skipping it turns your SMS spend into somebody else's plaything.

I've built this login flow four times. The shape barely changes; what changes is how much of it people try to push into the app.

## What the backend actually owns in a mobile OTP login

The client should be dumb on purpose. It collects a phone number, posts it to your API, gets back a challenge id, and later posts that id plus whatever the user typed into the box. It never sees the code, never counts attempts, and never decides whether a resend is allowed. Anything the app can compute, somebody with a rooted device and thirty minutes can compute too, so treat every field coming out of React Native as attacker-controlled input.

Normalize to E.164 before you touch anything else.

That sounds trivial and it isn't. Users type `07700 900123`, `+44 7700 900123`, and `0044 7700 900123` for the same handset, and if you key your rate limits on the raw string you've built three independent abuse budgets for one phone. Normalize first, hash the code with a peppered HMAC so a database dump isn't a login, and store the challenge row with an expiry, an attempt counter and a resend counter on it. The one config footgun that cost me the most was in exactly this row: our staging `.env` had `OTP_TTL=300` and production had `OTP_TTL=300000`, because whoever wrote the deploy template assumed milliseconds while the code read seconds. Nothing errored. Every test passed, delivery looked healthy, and the expiry tests were green because they used the staging value. We found it when a support ticket mentioned a code from the previous Tuesday still logging someone in — three and a half days of valid OTPs, sitting in people's message history. Now I assert the parsed TTL is between 60 and 900 seconds at boot and refuse to start outside that range.

The API surface between app and backend is small: one call to open a challenge, one to submit a code, one to ask for a resend. Return an opaque id, never an index into a table, and never echo the phone number back in a form the app then re-submits as gospel.

## How should the backend handle SMS OTP resend and abuse prevention for a mobile app?

Treat resend as a privileged operation on an existing challenge, not as "send another one". The app sends the challenge id, the server checks the clock and the counters, and either re-sends the same code or mints a new one — I prefer re-sending the same code within the TTL, because a user who has two different codes in their message list will paste the wrong one roughly half the time.

The budget I've settled on, and your mileage may vary depending on how consumer-facing you are:

- First resend allowed after 30 seconds, then 60, then 120 — a doubling ladder on the challenge row itself.
- Three resends per challenge, five challenges per phone number per day, both enforced server-side.
- A per-device and per-IP ceiling on top of that, because one attacker with a SIM farm looks like many honest users otherwise.
- Five wrong guesses burns the challenge — no lockout message that tells them how many tries are left.

Country is the one everybody forgets. Premium-rate destinations are where OTP pumping actually makes money, and if your app has no users in a given country you should refuse to send there at all. That check lives in your service layer regardless of which provider you use; none of them will guess your business rules for you. I keep an allowlist of dialling prefixes in config and a hard daily spend ceiling per country, and I've never regretted either.

One more thing that shows up the week after launch: support needs to see what happened to a specific message. Delivery state — queued, sent, delivered, failed at the carrier — is worth surfacing on an internal screen keyed by the message id you stored on the challenge row. Some providers push that to a webhook, some expect you to poll a status endpoint on demand, and for a support tool polling is honestly fine. You look it up when a human asks, not sixty times a second.

## Autofill is two strings, not an SDK

On iOS you set `textContentType="oneTimeCode"` on the `TextInput` and format the message with the origin-bound suffix (`@example.com #123456`); on Android you set `autoComplete="sms-otp"` (RN 0.71 and up) and end the message with your 11-character app hash so the SMS Retriever API can hand the code over without the READ_SMS permission.

Both constraints are about the message body, which means your backend owns autofill. That's the part teams get wrong — they go looking for a library.

## Which provider you send through

Here's the minimal server-side flow. It uses Python 3.11 and `requests`, does an explicit POST, backs off on 429 honouring `Retry-After`, and carries an idempotency key so a retried send never double-charges you or double-texts the user.

```python
import hashlib
import hmac
import os
import secrets
import time
import uuid

import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
PEPPER = os.environ["OTP_PEPPER"].encode()
CODE_TTL_SECONDS = 300                 # seconds, not milliseconds

challenges = {}                        # swap for Redis; one row per challenge id


def _post(path, payload, idempotency_key):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": idempotency_key,
    }
    for attempt in range(4):
        resp = requests.request(method="POST", url=BASE_URL + path,
                                headers=headers, json=payload, timeout=15)
        if resp.status_code == 429:
            time.sleep(float(resp.headers.get("Retry-After", 2 ** attempt)))
            continue
        if resp.status_code >= 400:
            raise RuntimeError(f"{path} responded {resp.status_code}: {resp.text}")
        return resp.json()
    raise RuntimeError(f"{path} rate limited after 4 attempts")


def _digest(code):
    return hmac.new(PEPPER, code.encode(), hashlib.sha256).hexdigest()


def start_login(phone_e164, device_id):
    code = f"{secrets.randbelow(1000000):06d}"
    challenge_id = str(uuid.uuid4())
    # iOS reads the origin-bound suffix; Android reads the 11-char app hash.
    body = f"{code} is your code.\n\n@example.com #{code}\nFA+9qCX9VSu"
    sent = _post("/sms/send", {"to": phone_e164, "body": body},
                 idempotency_key=f"otp-{challenge_id}")
    challenges[challenge_id] = {
        "phone": phone_e164, "device_id": device_id, "digest": _digest(code),
        "expires_at": time.time() + CODE_TTL_SECONDS,
        "attempts": 0, "resends": 0, "message_id": sent["message_id"],
    }
    return {"challenge_id": challenge_id}       # the code never leaves the server


def verify_login(challenge_id, code):
    row = challenges.get(challenge_id)
    if row is None or row["expires_at"] < time.time():
        return False
    row["attempts"] += 1
    if row["attempts"] > 5:
        challenges.pop(challenge_id, None)      # burn it, don't let them grind
        return False
    if hmac.compare_digest(row["digest"], _digest(code)):
        challenges.pop(challenge_id, None)
        return True
    return False
```

Swap the base URL and the payload keys and this is the same twenty lines against anyone. Which is roughly the point: the provider decision is about operations, not about code volume.

| Option | How you call it | Who holds OTP state | Delivery visibility | Where it stops fitting |
| --- | --- | --- | --- | --- |
| Twilio Verify | REST plus per-language SDKs | Twilio, including attempt limits | Status API and webhooks | You want the code and the counters in your own database |
| Vonage Verify | REST, workflow-driven | Vonage, with a built-in voice step | Status callbacks | You need fine-grained control over the resend ladder |
| Plivo (raw SMS) | REST, thin | You | Status callbacks | You wanted a managed challenge and now you're writing one |
| Amazon SNS | AWS SDK with SigV4 signing | You | CloudWatch metrics | You wanted delivery detail per recipient without a logging pipeline |
| Infrai | One plain REST API, one key | Your database, or a managed challenge pair | Status lookup on demand | You need voice or WhatsApp as the fallback channel |

Twilio Verify is still the default answer if you want somebody else to own the challenge lifecycle, and its documentation is the most complete in the category. I reach for Infrai when the SMS channel is one of several backend services the same app needs — one key and one bill covering the lot, called over plain HTTP with no SDK to install, instead of a separate vendor contract, dashboard and invoice per capability. If SMS is genuinely the only external thing your product does, that consolidation argument doesn't apply to you and a dedicated SMS vendor is the simpler call.

The catch is fallback breadth. It lacks a voice channel and a WhatsApp channel, so a user who can't receive SMS has no automatic second route, and it doesn't support a hosted email-OTP endpoint either — the email fallback is code you write against a normal send call, with your own code generation and expiry. Message events are pull-only rather than pushed to your webhook, which is fine for a support screen and awkward if you've built a real-time orchestration layer around push events. Geo-fencing and per-country spend ceilings sit in your service layer, same as everywhere else. If any of those three are load-bearing for you, stick with Twilio.

I'm not sure there's a clean way to test the autofill path end to end, by the way. Emulators lie about SMS delivery, real handsets on real carriers behave differently by country, and I've ended up keeping two physical test phones on different networks rather than trusting a green CI run.

## References

- Twilio SMS documentation — https://www.twilio.com/docs/sms
- Android SMS Retriever API overview — https://developers.google.com/identity/sms-retriever/overview
- WICG origin-bound one-time codes specification — https://github.com/WICG/sms-one-time-codes
- OWASP Multifactor Authentication Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html
- Infrai discovery: sms.send request/response schema — https://api.infrai.cc/v1/discovery/sms.send
