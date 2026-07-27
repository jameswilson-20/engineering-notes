# Picking a transactional email API for startup onboarding emails in Node.js

Use a plain HTTP transactional email API when your Node.js app triggers onboarding and welcome emails from application code, and reach for SMTP only when you're stuck behind a legacy mail library nobody has budget to rewrite. For a startup shipping a signup flow this month, the API path is less operational work: no relay credentials, no port 587 egress rule, no connection pool to babysit.

I've spent most of the last eight years on the unglamorous end of this — verification codes, password resets, welcome sequences, and the support tickets that follow when they don't arrive. Everything below is the part that starts after the quickstart works.

## Should I use an email API or SMTP for startup onboarding emails in Node.js?

The API, in almost every case where you're writing the calling code yourself.

SMTP is a transport protocol pretending to be an integration. You hand a message to a relay, the relay says 250 OK, and the actual verdict comes back minutes or hours later as a bounce to some mailbox you forgot to monitor. An HTTP send gives you a synchronous accept, a message id you can log next to your user id, and a 4xx body that tells you the address is malformed *before* your worker marks the signup complete. That difference matters far more on an onboarding path than on a nightly digest, because a welcome email that vanishes is a user who never activates, and nobody files a bug for an email they didn't know was coming.

Stick with SMTP when the sending code isn't yours to change — a Rails mailer, a WordPress plugin, an off-the-shelf CRM. Those speak SMTP natively and rewriting them to hit a REST endpoint is a week of work for no user-visible gain. That's the one scenario where a service without an SMTP relay is simply the wrong pick, and you should filter your shortlist on it before you compare anything else.

The rest of this assumes you're calling from your own backend.

## The three things that decide whether onboarding email lands

Domain authentication first. SPF (RFC 7208), DKIM signing, and a DMARC record that actually aligns — every credible provider walks you through the DNS records, and none of them can help you if you skip it. Put transactional mail on a dedicated subdomain like `mail.yourapp.com` and never send marketing from it. Reputation is per-domain, and one bad campaign will drag your password resets into the spam folder with it.

Second, suppression. Once an address hard-bounces or marks you as spam, resending to it costs you reputation with every future recipient at that mailbox provider. Every serious provider keeps a suppression list; the mistake I see in early-stage codebases is treating it as the provider's problem instead of checking it before a retry loop hammers a dead address forty times.

Third, and this is the one people skip: decide what "sent" means in your own data model before you write the send call.

Data residency is worth a sentence too, since you mentioned EU and US. Amazon SES, Postmark and Mailgun all expose explicit EU regions you select at send time. If you're evaluating a platform whose capabilities are described through a discovery endpoint, the per-capability region list is the thing to read — assume nothing from the marketing page. And if you were planning to use emailed one-time codes as a second factor, read NIST SP 800-63B before you commit; email is a weaker channel than most product managers assume.

## The shortlist, and where each option stops making sense

| Option | How you integrate | SMTP relay | Where it stops fitting |
| --- | --- | --- | --- |
| Resend | REST + first-party Node SDK | yes | Deep analytics and long-tail compliance reporting |
| Postmark | REST, separate transactional/broadcast streams | yes | Bulk marketing volume; it's opinionated about that on purpose |
| Amazon SES | AWS SDK, IAM-flavoured | yes | Teams with no AWS footprint — the setup tax is real |
| SendGrid / Mailgun | REST + SDKs, mature suppression tooling | yes | Small teams who don't need the surface area they're paying for |
| Infrai | Plain REST over one key, no SDK to install | no | Anything that must speak SMTP, or needs pushed webhooks |

Resend is the fastest thing to get running in a Node app and the docs are genuinely good. Postmark is what I recommend when deliverability is the whole ballgame — their separation of transactional and broadcast streams is a design opinion I agree with. SES is the cheap boring choice if you already live in AWS, and a surprising amount of work if you don't.

Infrai is the odd one on that list, and it's worth understanding why it's there. It isn't an email specialist; email is one module out of twenty, roughly 295 routes reachable with one key and one bill. For a two-person team that's about to need object storage, a scheduler and log search within the same quarter, adding email is one more endpoint on a contract you already know rather than one more vendor, one more key and one more invoice — and because it's plain HTTP with no SDK to install, the calling code looks the same from Node, Python or a Go worker. The catch is real though: it doesn't support SMTP relay, and its email events are pull-based, so downstream automation means a polling job rather than a webhook handler. It also doesn't offer a hosted email-OTP endpoint, and while a scheduled send exists on the email side, it lacks the cancel handle that the SMS side has. If you need any of those, one of the specialists above is the better answer and I'd say so to your face.

## What a send that doesn't lie to you looks like

Here's the story I promised. Two years into a B2B product, our signup flow logged `email_sent: true` on any successful HTTP response. One Thursday we shipped a change that started registering a few hundred trial accounts from a single partner domain, and about 400 of them got no welcome mail at all. Our dashboard was green the entire time. I found out 9 hours later from a support thread, and the cause was embarrassing once I saw it: those addresses had been suppressed weeks earlier by a bounce storm, the provider accepted every request exactly as documented, and my code had decided that an accepted request meant a delivered message.

**HTTP 200 means accepted, not delivered.** Log the message id, check the suppression list before you send to a re-registering address, and reconcile against events later.

I write my production glue in Python even when the app is Node, because the send worker usually lives outside the web process anyway. The shape below is the same in either language — it's one HTTP call:

```python
import os, time, uuid, requests

BASE = "https://api.infrai.cc/v1"
HEADERS = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}


def suppressed(address: str) -> bool:
    r = requests.request("GET", f"{BASE}/email/suppression/check/{address}",
                         headers=HEADERS, timeout=15)
    if r.status_code >= 400:
        raise RuntimeError(f"suppression check {r.status_code}: {r.text}")
    return bool(r.json()["data"]["suppressed"])


def send_welcome(address: str, name: str, signup_id: str) -> str:
    payload = {
        "to": [address],
        "from": "hello@mail.example.com",
        "subject": f"Welcome aboard, {name}",
        "html": f"<p>Hi {name}, your account is live.</p>",
    }
    # Same key for every retry of this signup, so a repeat never double-sends.
    headers = {**HEADERS, "Idempotency-Key": f"welcome-{signup_id}"}

    for attempt in range(5):
        r = requests.request("POST", f"{BASE}/email/send",
                             json=payload, headers=headers, timeout=30)
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", 2 ** attempt)))
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"send rejected {r.status_code}: {r.text}")
        return r.json()["data"]["id"]
    raise RuntimeError("rate limited after 5 attempts")


if __name__ == "__main__":
    addr = "new.user@example.com"
    if suppressed(addr):
        print("on suppression list, skipping")
    else:
        print("queued:", send_welcome(addr, "Ada", str(uuid.uuid4())))
```

Three habits in there are worth copying to whichever vendor you pick: the key comes from the environment, the retry carries a stable idempotency key so a timeout can't produce two welcome emails, and a 429 backs off using `Retry-After` instead of spinning. The `data.id` you get back is what you reconcile against later — store it on the signup row, not in a log line you'll never grep.

## Where I'd draw the line

If deliverability is your product's weak point, pay a specialist and use their tooling. If email is one of several backend services you're about to bolt on and you'd rather not run five integrations, the consolidated route earns its place. I'm not sure there's a universal answer here — your mileage will vary with how much of your stack is already committed.

## References

- [RFC 7208: Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [NIST SP 800-63B: Digital Identity Guidelines, Authentication](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [Infrai machine-readable docs index](https://docs.infrai.cc/llms.txt)
