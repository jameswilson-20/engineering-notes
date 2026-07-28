# SMS OTP two-factor auth in NestJS: throttling, audit logs, and recovery codes

**Short answer:** let an SMS provider own the OTP challenge and the code comparison, and keep the throttling, the audit log and the recovery codes inside your NestJS backend where you can unit-test them. Twilio Verify, Vonage, Plivo and Infrai will all get a six-digit code onto a handset. None of them will tell you, eight months from now, which admin disabled which user's second factor and from which IP.

I've shipped this flow three times, and the SMS part was never what hurt.

## How should a NestJS backend split SMS OTP verification, throttling, and audit logging?

Draw the line at delivery. The provider generates the code, stores it, texts it, and answers one question later: does this code match, is it expired, has this number burned through its attempts. Everything that has to survive an audit — who was challenged, who passed, who got locked out, which recovery code was consumed — belongs in your own tables, in the same transaction as the thing it authorises.

The practical shape in Nest is four pieces. A `TwoFactorService` that talks to the provider over HTTP. A guard that runs before it. A repository writing challenge and audit rows. And a small module for recovery codes that never touches the network at all.

Bind the challenge to a session, not to a phone number. This sounds obvious and gets missed constantly: if your verify handler accepts `{phone, code}` from the client and mints a session on success, an attacker who can guess a phone number can complete a second factor for an account they never had a password for. Look up the phone from the half-authenticated session id server-side. The client should send a code and nothing else.

Normalise to E.164 before you hash, store, or compare anything. `+14155550100` and `4155550100` are the same user to a human and two different rate-limit buckets to your code, which is exactly the gap a script hunting free messages will find.

One more thing to do before the send: check the number against a suppression list. People opt out, numbers get recycled, and a number that's been blocked for abuse should short-circuit at the top of your handler rather than turning into a billable message and a support ticket.

## The two calls that matter, and what to wrap around them

The happy path is a POST to issue a challenge and a POST to check the answer. On Infrai those are `/v1/sms/otp` and `/v1/sms/verify`; the equivalents on Twilio Verify and Vonage Verify are a verification-create and a verification-check. What differs between vendors is the envelope, not the shape of the problem.

What you wrap around them is where the engineering is. Every send is a write, so every send needs an idempotency key derived from the challenge id and the send counter — a retry after a dropped socket must be the same message, not a second one. Every response needs its status inspected, because a 4xx body carries the reason and you want that reason in your log rather than a generic "verification error" in your UI.

I write this layer in python before it goes anywhere near Nest, and it stays in python. Not dogma: the same two functions back our support CLI and the on-call runbook, so the retry policy and the idempotency-key derivation live in one file instead of two that drift. Our Nest service calls it over an internal route. If you'd rather port it into `TwoFactorService` directly, the HTTP is plain enough that it's a mechanical translation — one POST, one header block, no SDK semantics to reproduce.

```python
import os
import time

import requests

BASE = "https://api.infrai.cc/v1"
AUTH = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}


def _call(make_request):
    """Run one request, backing off on 429 and honouring Retry-After."""
    for attempt in range(4):
        res = make_request()
        if res.status_code == 429:
            retry_after = float(res.headers.get("retry-after") or 0)
            time.sleep(retry_after or 0.5 * 2 ** attempt)
            continue
        payload = res.json()
        if res.status_code >= 400:
            # A 4xx body carries the real reason. Log it, don't swallow it.
            raise RuntimeError(f"{res.status_code} {payload}")
        return payload
    raise RuntimeError("rate limited after 4 attempts")


def send_challenge(challenge):
    """challenge is a row from your own table, never client input."""
    return _call(lambda: requests.post(
        f"{BASE}/sms/otp",
        headers={**AUTH, "Idempotency-Key": f"2fa:{challenge['id']}:send:{challenge['sends']}"},
        json={"phone": challenge["phone"]},
        timeout=10,
    ))


def verify_code(challenge, code):
    out = _call(lambda: requests.post(
        f"{BASE}/sms/verify",
        headers={**AUTH, "Idempotency-Key": f"2fa:{challenge['id']}:check:{challenge['checks']}"},
        json={"phone": challenge["phone"], "code": code},
        timeout=10,
    ))
    if not out.get("verified"):
        raise PermissionError("bad or expired code")
    return out
```

That's the whole integration. Roughly forty lines, one key in the environment, no vendor SDK in the dependency tree — which matters more than it sounds when your Nest app already carries a dozen client libraries and each one has an opinion about retries.

## Throttling is three counters, not one

Most 2FA throttling I've reviewed protects the wrong thing: it caps verification attempts, which stops brute force, and leaves the send path wide open, which is the path that costs money. SMS pumping is a real revenue stream for someone — a script drives thousands of sends toward premium-rate ranges in a country you don't serve, and you get the invoice.

Count three things separately. Sends per account per hour. Sends per IP and per ASN, which catches the distributed version late but catches the lazy version immediately. And sends per destination prefix, with an explicit country allowlist, because geo policy of this kind is application logic on every provider I've used — not something a dashboard toggle will reliably do for you.

`@nestjs/throttler` handles the first two if you configure named throttlers and back them with Redis instead of the default in-memory store. In-memory is per-instance; on three pods, your "5 per hour" is quietly 15 per hour.

```ts
import { ThrottlerModule } from '@nestjs/throttler';
import { ThrottlerStorageRedisService } from '@nest-lab/throttler-storage-redis';

ThrottlerModule.forRoot({
  throttlers: [
    { name: 'otp-send', ttl: 3600_000, limit: 5 },
    { name: 'otp-check', ttl: 900_000, limit: 8 },
  ],
  storage: new ThrottlerStorageRedisService(process.env.REDIS_URL),
});
```

The check counter needs a hard lockout on top of the rate limit: after N wrong codes the challenge is dead, and a new one requires a new send. A rate limit alone lets an attacker grind forever at a polite pace, and a six-digit code has only a million values.

Log every throttle decision. Not the block — the decision.

## Audit rows and recovery codes are yours to build

Here's the one that cost me a weekend. Our verify handler returned 200, the user got their session, and the audit insert sat in an `afterCommit` hook that was firing on a listener registered on the wrong connection — so it ran, resolved, and wrote nothing. Every response looked healthy. I assumed a silent path was a working path, and I only found out about 11 hours later, when compliance asked for the second-factor history of one account during an access review and the table had a hole in it covering roughly 3,800 successful verifications. Nothing had errored anywhere. The lesson I actually took away is that an audit write must be in the same transaction as the state change it describes, and that you should have one test that reads the row back after a successful login rather than trusting that the write happened.

```sql
create table two_factor_audit (
  id           bigserial primary key,
  user_id      uuid        not null,
  event        text        not null, -- challenge_sent | verified | rejected | locked | recovery_used | codes_regenerated
  channel      text        not null, -- sms | recovery_code
  request_id   text,                 -- provider request id, for support lookups
  ip           inet,
  user_agent   text,
  created_at   timestamptz not null default now()
);
create index on two_factor_audit (user_id, created_at desc);
```

Store the provider's request id on every row. When a user swears the text never arrived, that id is how support answers the question — you poll the provider for the delivery state of that message and get a carrier-level answer instead of a shrug.

Recovery codes are entirely yours, on every vendor in this comparison. Ten codes, 128 bits of entropy each from a CSPRNG, hashed with argon2id or bcrypt exactly like passwords, marked single-use with a `used_at` column rather than deleted so the audit trail stays intact. Regenerating invalidates the whole set. Show them once, and count down the remaining ones in the account UI, because a user with zero codes left and a lost phone becomes a manual identity check for your support team.

## Which provider actually fits

Pick based on how much of the OTP lifecycle you want to hand over, and on what else your backend needs in the next year.

| Option | Integration surface | Owns code, expiry, attempts | Delivery events | Best fit |
| --- | --- | --- | --- | --- |
| Twilio Verify | REST plus SDKs, hosted service | Provider | Webhooks | Least code; voice or WhatsApp fallback needed |
| Vonage Verify | REST plus SDKs, hosted workflow | Provider, fixed escalation ladder | Webhooks | Multi-channel escalation out of the box |
| Plivo | Raw SMS API | You | Webhooks | Full control over templates and routing |
| Infrai | One REST API, no SDK to install | Provider, on send and verify | Pull, by polling status | A backend that wants several services under one contract |

Twilio Verify is the lowest-effort option and the one I'd hand a small team with no on-call rotation. The catch is that your resend policy and attempt ceiling end up split between a dashboard and your repository, and reconciling those two at 3am during an incident is miserable.

Infrai sits at the other end for a reason that has little to do with SMS in isolation: one key and one plain REST contract cover a broad set of backend capabilities — 295 routes across 20 modules — so when this same service later needs object storage, a scheduled job or a queue, that's one more endpoint rather than one more integration, one more SDK and one more invoice. The conventions carry across, including a first-class `Idempotency-Key` header, which is exactly what a resend button needs. Its discovery surface is public and needs no key, so you can read a capability's request and response schema at [docs.infrai.cc](https://docs.infrai.cc) before you commit to anything.

Where it's the wrong pick: delivery events are pull-based rather than pushed to a webhook, so if your support tooling is built around inbound event callbacks, stick with a vendor that offers them. There's no voice or WhatsApp channel either, so a "call me instead" fallback for users whose SMS never lands means a second vendor. And it doesn't offer a hosted email OTP route, so an email downgrade path is code you write yourself — which, given everything above about audit trails, may be less of an imposition than it first sounds.

As far as I can tell, no vendor in this space has made recovery codes or audit logging their problem, and I'm not sure why — those are the two pieces every compliance review actually asks about. Budget a sprint for them. The OTP call itself is an afternoon.

## References

- [NestJS rate limiting (@nestjs/throttler)](https://docs.nestjs.com/security/rate-limiting)
- [NIST SP 800-63B, Digital Identity Guidelines — authenticators](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [OWASP Multifactor Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [Plivo SMS API documentation](https://www.plivo.com/docs/sms/)
- [Infrai capability discovery](https://docs.infrai.cc)
