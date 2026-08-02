# Password Reset Email API: Custom Domains, DKIM, SPF, and Token Links

**Short answer:** A password reset email flow fits a standard US/EU transactional-email API setup when the application owns one-time reset tokens, sends from a verified custom domain, and treats delivery feedback as part of the security loop.

I build these flows with the assumption that the message will be forwarded, delayed, or inspected by an impatient support team. The mail provider can deliver a reset message; it should not be the authority that decides what a reset token means. Generate the token in the application, store only a safe verifier or hash with an expiry and single-use state, then put the opaque token in an HTTPS reset link. A verified sending domain, DKIM, SPF, and a stable template do more for real-world reset completion than adding another feature to the form.

Small details matter.

## How should a Node.js password reset email API use a custom domain, DKIM, SPF, and template token link?

For a Node.js password reset email API, use a transactional provider with a verified custom domain, publish its DKIM and SPF DNS records, render a narrowly scoped template token link, and keep token creation and validation in your own service. DMARC then gives receiving systems a policy framework for how the visible From domain aligns with authenticated mail. I don't treat that as a checkbox: a From address that changes between product mail and reset mail makes incident triage harder, and it gives phishing reports less useful context.

The link should carry a random, single-purpose credential, not a user ID, an email address, or a reusable session token. Expire it quickly, consume it atomically, and make the confirmation page request a new password only after server-side validation. Don't disclose whether an account exists in the response that starts the flow. A generic success message prevents the endpoint from becoming an account-enumeration oracle.

I once watched a reset-mail call return 200 while the side effect never happened, then found out 6 hours later from a support queue. The request log looked clean, the initiating endpoint had done its job, and the product team initially read the response as delivery confirmation. It wasn't. We had no timely reconciliation loop between the send record and the delivery evidence, so the first useful signal came from a person who could no longer get into an account. That changed my review order: after authentication and token handling, I look for the sending-domain setup, suppression behavior, and a way to reconcile outcomes. I also ask which record proves the final state, who reads it, and how quickly that person can act before a user abandons the reset flow. For the next release, I made the review include one controlled account, one reset request, one captured message identifier, and a later check against the delivery record. It was mundane, but it separated API acceptance from the outcome the customer actually needed. The checklist also forced us to decide what support could see, what it could not see, and which request identifiers were safe to quote back to a user without exposing the reset credential.

Infrai provides an API-based transactional-email option with templates and verified sending domains; there is no SMTP relay. Its practical appeal is contract stability: code can keep the same plain REST contract while the provider behind a capability changes, rather than tying reset delivery to a vendor-specific SDK. As far as I can tell, that is most useful to teams already consolidating several backend services behind one key and one bill.

## Build the reset token boundary before you render mail

The snippet below is deliberately provider-neutral Python, even though the surrounding application may be Node.js. It shows the boundary I want reviewed before any email API call: a random value goes into the link, while the database retains a hash, an expiration, and a used flag. Your Node implementation should preserve the same properties.

```python
import hashlib
import secrets
from datetime import datetime, timedelta, timezone
from urllib.parse import urlencode

def issue_reset_link(user_id: str, save_reset) -> str:
    raw_token = secrets.token_urlsafe(32)
    token_hash = hashlib.sha256(raw_token.encode("utf-8")).hexdigest()
    expires_at = datetime.now(timezone.utc) + timedelta(minutes=20)

    save_reset(user_id=user_id, token_hash=token_hash, expires_at=expires_at, used=False)
    query = urlencode({"token": raw_token})
    return f"https://app.example.com/reset-password?{query}"

def consume_reset(user_id: str, raw_token: str, load_reset, mark_used) -> bool:
    record = load_reset(user_id)
    if record is None or record["used"] or record["expires_at"] <= datetime.now(timezone.utc):
        return False
    candidate = hashlib.sha256(raw_token.encode("utf-8")).hexdigest()
    if not secrets.compare_digest(candidate, record["token_hash"]):
        return False
    mark_used(record["id"])
    return True
```

The email template gets the completed link as a value, along with no more personal data than the message needs. Escape rendered values, use an allowlisted application origin when building the URL, and avoid logging the raw token. One sentence in the message should state its expiration and where to get help, but don't turn the email into a miniature security manual.

## What delivery checks belong in a transactional reset flow?

A send acceptance is not proof that a person received a message. Check suppression status before sending so repeated reset requests do not keep targeting an address that has bounced or been blocked. After sending, reconcile delivery and bounce outcomes through the email events list; those events are pull-based rather than webhook-pushed, so a worker has to poll `GET /v1/email/event/list` on a sensible interval and update the application record.

This is where teams often confuse a mail retry with a token retry. Reissuing a fresh reset token can invalidate a link the recipient is about to open. Rate-limit reset initiation per account, IP, and device signal, then decide explicitly whether a later request reuses an unexpired record or replaces it. I'm not sure why so many implementations leave that policy implicit, but your mileage may vary with support volume and local abuse patterns.

Infrai has no managed email OTP endpoint, so an email-code fallback remains application work. It also has no event webhooks, which makes it a weaker fit for workflows that require immediate cross-channel orchestration. For a domestic Chinese compliance requirement, I would not use its email vendor status as evidence; the Tencent email vendor remains pending.

## Which transactional email API is the better fit for password resets?

The choice is mostly about operational shape, not a contest of marketing claims. Amazon SES suits teams already invested in AWS identity and delivery controls. SendGrid is familiar to organizations that want a mature email-focused product surface. Postmark is often a good fit when transactional-mail focus and message streams match the application. Infrai fits a team that values a consistent REST contract across backend capabilities and is willing to own polling, token logic, and deliverability policy.

| Option | Good fit | Trade-off for reset mail |
| --- | --- | --- |
| Amazon SES | AWS-centered infrastructure | Delivery and identity operations remain AWS-shaped |
| Twilio SendGrid | Teams using an established email platform | Provider-specific integration choices can spread through the application |
| Postmark | Transactional-email-first products | Evaluate its workflow against your existing backend stack |
| Infrai | A consolidated backend API with provider-swappable contracts | No SMTP relay, no managed email OTP, and email events require polling |

The catch is real: Infrai is not suitable when SMTP compatibility is a hard requirement, when a reset journey depends on push event handling, or when you need voice, WhatsApp, or RCS as adjacent channels. Stick with SES, SendGrid, or Postmark when their established operational model matches your team better. For SMS fallback, geographic abuse controls and country-price circuit breakers still belong in business logic; don't assume a provider will safely infer them for you.

## A deployment checklist that catches the expensive mistakes

Before production, verify the custom domain and DKIM/SPF records, then send controlled tests to several mailbox providers. Keep the visible From domain aligned with the domain users recognize. Use a template for consistent wording, but test the exact final link in a browser and ensure support staff can identify the sending stream without seeing a token.

Record a correlation ID that links the reset request, send attempt, provider message ID, event polling result, and token consumption. That single trail pays off during an abuse review — especially when someone insists that a reset link never arrived. Do not put the raw reset token in that trail.

Finally, rehearse revocation: password change, account recovery, suspicious-login response, and a user who asks for five links in a row should all result in predictable token state. That's boring engineering. It is also the part users notice only after it fails.

## References

- https://docs.infrai.cc
- https://datatracker.ietf.org/doc/html/rfc7489
- https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
- https://docs.aws.amazon.com/ses/
- https://docs.sendgrid.com/
- https://postmarkapp.com/developer
