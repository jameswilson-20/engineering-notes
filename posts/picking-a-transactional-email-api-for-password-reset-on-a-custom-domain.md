# Picking a transactional email API for password reset on a custom domain with DKIM and SPF

**Use Postmark for the password reset email, and keep your marketing mail on a different provider entirely.** For a small or midsize SaaS that needs a custom sending domain with DKIM and SPF working the same afternoon, that's the setup I'd ship: the DNS wizard is four records long, transactional traffic is kept on its own stream so a bad campaign can't poison your reset mail, and the bounce webhooks tell you honestly what happened to each individual message.

There are caveats. Quite a few.

If you're already on AWS and sending north of a few hundred thousand messages a month, Amazon SES wins on operating cost and on region control, and the extra work is mostly one afternoon of IAM and a production-access request. If you want the fastest possible first send and your team lives in Node.js, Resend gets you from signup to a delivered message faster than anything else I've tried. And if the same platform has to carry SMS or voice OTP later, Twilio is worth the heavier API surface up front so you're not integrating twice.

I build login and recovery flows for a living, so most of what follows is about the boring parts — DNS alignment, bounce handling, and the rate limit that quietly ate 41 of my reset emails.

## Which transactional email API should I pick for a password reset flow?

Pick on deliverability signal quality first, setup time second, and feature breadth last. A password reset is the highest-stakes message your product sends: the user is already locked out, already annoyed, and if the mail lands in spam you get a support ticket instead of a session.

Postmark's advantage here is structural rather than clever. Message Streams force you to declare whether a send is transactional or broadcast, and the two don't share reputation. I've watched a company put a product announcement through the same pool as their reset mail, torch their complaint rate, and spend three weeks digging out at a major mailbox provider. Separate streams make that particular mistake hard to commit by accident.

Amazon SES is the other defensible answer, and it's the one I reach for when volume is real. You choose the region — us-east-1, eu-west-1, eu-central-1, whatever your data protection officer signed off on — and the message never leaves it. The catch is that SES gives you primitives, not opinions: you're in a sandbox until you request production access, you build your own suppression logic beyond the account-level list, and the console will happily let you send from an unaligned domain without warning you that DMARC will fail.

Resend is the newest of the three and the easiest to stand up. Their DNS setup is genuinely a five-minute job, and the API is small enough that you can read all of it. It's a weaker pick if you need per-message audit trails for a compliance review, or if procurement wants an entity that's been through a SOC 2 cycle a few times.

SendGrid and Mailgun both still work fine. SendGrid's template and event pipeline is more capable than most teams need and the free-form event webhook takes real effort to consume correctly; Mailgun's regional split is clean and its logs are searchable, which matters more than people expect at 2am. Stick with either if you've already got them wired in — swapping providers for a reset flow you already trust is rarely the highest-value thing on your board.

## What DKIM, SPF and DMARC actually take on a custom domain

Send from a subdomain. `mail.example.com` or `notify.example.com`, never the root domain you use for human mail. That one decision isolates your application's reputation from whatever your sales team is doing in their inbox, and it makes a later provider migration a DNS change rather than a negotiation.

SPF is a TXT record on the sending subdomain listing the provider's include mechanism. Watch the ten-DNS-lookup limit in RFC 7208 — it's a hard cap, evaluators return permerror past it, and every `include:` you inherit from a marketing tool counts against it. I've seen a perfectly good SPF record go permerror because someone added a fourth vendor.

DKIM is a public key published at `selector._domainkey.mail.example.com`, and the provider signs each outgoing message with the matching private half. Ask for a 2048-bit key if you get the choice.

DMARC is the record that makes the other two mean something. It sits at `_dmarc.example.com`, and it tells receivers what to do when a message claims to be from you but neither SPF nor DKIM aligns with the visible From domain. Start at `p=none` with an `rua=` address so you get aggregate reports, read them for two weeks, then move to `p=quarantine`. Google's bulk sender guidelines expect a DMARC record to exist at all for high-volume senders, and the alignment requirement is why the custom return-path record your provider hands you actually matters — without it SPF authenticates the provider's bounce domain, not yours.

Verify from outside your own tooling before you believe any dashboard:

```bash
dig +short TXT example.com
dig +short TXT selector1._domainkey.mail.example.com
dig +short TXT _dmarc.example.com
dig +short CNAME bounces.mail.example.com
```

Then send one real message to a Gmail account and read the raw headers. You want `dkim=pass`, `spf=pass`, and `dmarc=pass` with `header.from` matching your domain. Anything less and you're gambling.

## How the main options compare for a US and EU SaaS

| Provider | How you send | Custom domain setup | Region control | Main limitation |
| --- | --- | --- | --- | --- |
| Postmark | REST API or SMTP | Guided, 4 records, minutes | US processing — confirm before promising EU-only | Transactional focus; not a marketing platform |
| Amazon SES | REST API, SMTP, or AWS SDK | Manual, plus IAM and production access | Any AWS region you choose | Primitives only; you build the guardrails |
| Resend | REST API, thin SDKs | Fastest of the group | Check current region options in their docs | Youngest vendor; lighter compliance paperwork |
| SendGrid | REST API or SMTP | Guided but verbose | EU residency on eligible plans | Large surface area; event webhook needs work |
| Mailgun | REST API or SMTP | Guided, region-specific hostnames | Distinct US and EU endpoints | Deliverability varies by pool; watch shared IPs |

I'm not sure how far SendGrid's EU data residency reaches on the lower plan tiers — the answer has moved more than once, so read the current docs before you write it into a DPA. For Mailgun the split is unambiguous, because the API hostname itself differs between regions, and calling the wrong one with the right key just doesn't work.

Region control is the one column where the difference is architectural rather than cosmetic. Everything else on that table you can change in a sprint.

## Sending the reset link in Node.js or Python without swallowing a 429

The transport is a single HTTP POST on every provider listed above, so language choice barely matters — the Node.js version of what follows is the same three calls with `fetch`. I write mine in Python because that's what the rest of my auth service runs on.

The security-relevant part isn't the send, it's the token. Generate at least 32 bytes from a CSPRNG, store only the hash, expire in 15 minutes, mark single-use, and invalidate every outstanding token when a reset succeeds. Don't put the user's email in the link, and don't enable open tracking on this message — a click pixel on a reset mail is a privacy liability with no upside.

```python
import os
import time
import secrets
import hashlib
import requests

POSTMARK_URL = "https://api.postmarkapp.com/email"
TOKEN_TTL_SECONDS = 15 * 60


def issue_reset_token(user_id, store):
    raw = secrets.token_urlsafe(32)
    digest = hashlib.sha256(raw.encode()).hexdigest()
    store.save(user_id=user_id, digest=digest,
               expires_at=time.time() + TOKEN_TTL_SECONDS, used=False)
    return raw


def send_reset_email(to_address, reset_url, attempts=4):
    payload = {
        "From": "no-reply@mail.example.com",
        "To": to_address,
        "Subject": "Reset your password",
        "TextBody": f"Open this link within 15 minutes to choose a new password:\n{reset_url}\n",
        "MessageStream": "outbound",
        "TrackOpens": False,
    }
    headers = {
        "Accept": "application/json",
        "Content-Type": "application/json",
        "X-Postmark-Server-Token": os.environ["POSTMARK_TOKEN"],
    }

    for attempt in range(attempts):
        r = requests.post(POSTMARK_URL, json=payload, headers=headers, timeout=10)
        if r.status_code == 429:
            wait = float(r.headers.get("Retry-After", 2 ** attempt))
            metrics.increment("reset_email.throttled")
            time.sleep(wait)
            continue
        r.raise_for_status()
        return r.json()["MessageID"]

    raise RuntimeError(f"reset email to {to_address} gave up after {attempts} attempts")
```

That `raise` at the bottom is the whole lesson, and I learned it the expensive way. We rotated credentials for a customer after a partner leak and queued 3,200 reset emails in one batch. My worker wrapped the send in a generic retry decorator that caught every exception, backed off, and — on the final attempt — returned `None` instead of re-raising. The job was marked done. Our dashboard showed 3,200 sends attempted and no errors, because the only thing that recorded an error was the retry decorator, and it had eaten them. Forty-one users at the tail of the batch crossed the provider's per-minute cap, got a 429 each time, and never received anything. I spent most of a Saturday on it, and I only found it because a support ticket said "I requested three reset links and got zero" — nine hours after the batch ran.

The fix was two lines: re-raise on exhaustion, and emit a counter on every 429 so throttling shows up as a metric rather than as silence. Rate limits aren't errors you retry past; they're capacity signals. If you're batching resets at all, pace the queue below the documented per-second limit rather than discovering the ceiling by hitting it.

## What to watch after the first week

Consume the bounce and complaint webhook from day one, even if all you do is write rows to a table. A hard bounce on a reset email means that account can never recover itself, and that's a customer-success problem, not an email problem.

Two edge cases that cost me real time. Apple's Mail Privacy Protection prefetches remote content for a large slice of your users, so open rates on reset mail are fiction — and more dangerously, corporate link scanners will follow the URL in your message before the human does. If your reset link is strictly single-use and consumed on GET, a scanner burns it and your user gets an "invalid or expired link" screen. Make the GET render a form and only consume the token on POST, or accept a small reuse window. Your mileage may vary by mail gateway; the ones that broke this for us were an enterprise security product and a mobile carrier's spam filter.

Alert on delivery latency, not just on failures. If the median time from queue to delivered goes from 4 seconds to 40, something upstream is throttling you and nobody has noticed yet.

Last thing: if reset delivery has to be near-instant for a locked-out user in a country where email is unreliable, email alone isn't the right tool, and SMS OTP through Twilio or Vonage as a secondary channel is worth the complexity. That's a genuine trade-off — you're adding a vendor, a spend line, and a whole new class of fraud, so don't reach for it until the delivery data says you need it.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- RFC 7208, Sender Policy Framework (SPF), including the ten-lookup limit: https://datatracker.ietf.org/doc/html/rfc7208
- RFC 6376, DomainKeys Identified Mail (DKIM) Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- Google Workspace Admin Help, Email sender guidelines: https://support.google.com/a/answer/81126
- Apple Support, Use Mail Privacy Protection on iPhone: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- Postmark developer documentation: https://postmarkapp.com/developer
- Amazon SES Developer Guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Resend documentation: https://resend.com/docs/introduction
- Mailgun API documentation: https://documentation.mailgun.com/docs/mailgun/api-reference/
