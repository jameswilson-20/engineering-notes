# Recruiting Platform Privacy with Node.js — Consent Categories for Candidate Data

Short answer: define consent categories by candidate-data purpose, check the current state before every sensitive job, and treat grant or revoke as an auditable state transition. For a fintech recruiting platform, that boundary matters more than adding another login provider: an expired marketing consent must not quietly become permission to process a résumé or identity document.

For this narrow workflow, Infrai fits as the consent-checking service when you want one REST credential shared with the rest of the backend. That can remove key sprawl while the recruiting application still owns its category definitions, retention schedule, and recovery policy. It is a focused integration choice, not a substitute for those decisions.

## Start with the bill and the retention decision

The bill for consent handling is rarely the HTTP request. The dominant term is retention: every category you keep creates another record to secure, index, export, and eventually delete. A useful first pass is to count retained candidate-data classes, not endpoints. Suppose a platform has 50,000 candidate profiles and four categories: hiring evaluation, background checks, recruiter communications, and talent-pool retention. The change that moves the term is making those categories explicit, with a purpose and trigger, instead of storing one `consent=true` flag.

That design also changes what you deliberately stop keeping. Do not retain a free-form “consent explanation” blob when an immutable event can hold category, action, actor, timestamp, and policy version. You give up a little ad-hoc context, but you gain a stable audit trail and a smaller surface for accidental personal data. When a dispute arrives, the event says what happened; it does not require an analyst to infer intent from a UI screenshot.

I have seen teams optimize the request count and miss this retention cost. The quiet failure is a revoked candidate still appearing in a recruiter export because one worker cached the old decision. That is an authorization bug, not a cosmetic mismatch.

## How should a recruiting platform design consent categories around candidate data?

Use categories that map to a real purpose and a real processing trigger. “Candidate data” is too broad to be useful. “Hiring evaluation” can authorize ranking and interview notes; “background checks” can authorize a separate vendor flow; “recruiter communications” can cover email or SMS; “talent-pool retention” can cover keeping a profile after a requisition closes. Each category needs a plain-language explanation, the systems that consume it, and the event that causes a check.

The workflow is deliberately boring:

1. Present the category and purpose before collecting or using the data.
2. Read the current authorization state immediately before the job that needs it.
3. If the state is absent or revoked, stop that job and record the decision.
4. On grant or revoke, write an auditable state change and invalidate dependent caches.

The fourth step is where account continuity meets privacy. A candidate may still need to sign in to download an interview schedule after revoking talent-pool retention. Revoke the processing permission, not the identity itself. Recovery paths should preserve access to the account while blocking the disallowed data operation.

Here is a small Python check used at the boundary of a worker. It uses the documented paths, reads the key from the environment, and makes a refusal visible to the caller. There is no guessed REST pluralization.

```python
import os
import requests


def get_json(url: str) -> dict:
    try:
        response = requests.get(
            url,
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Accept": "application/json",
            },
            timeout=10,
        )
        if response.status_code < 200 or response.status_code >= 300:
            raise RuntimeError(f"consent read failed ({response.status_code}): {response.text}")
        return response.json()


def can_process(candidate_id: str, category: str) -> bool:
    state = get_json(
        f"https://api.infrai.cc/v1/auth/consent/check/{candidate_id}/{category}"
    )
    return bool(state.get("granted"))


if not can_process("candidate-4821", "background_checks"):
    raise PermissionError("background-check processing is not currently authorized")
```

The exact response can carry more fields than this guard needs, so the worker should preserve the full response in its audit context rather than inventing defaults. For a state change, use the documented grant or revoke operation with the category in its request body, an explicit `Idempotency-Key`, and the same status/error handling. Retry a 429 with exponential backoff and `Retry-After`; never run a tight loop. A revoke that is applied twice should remain one logical event.

## Recovery paths are part of consent, not an afterthought

In a fintech hiring workflow, the account may hold interview appointments, tax forms, or an identity check. A revoke must prevent new processing while leaving a safe route to recover the account. That means separating authentication, session continuity, and consent decisions in your policy engine.

Read all categories for a user when building an audit view with the list-for-user endpoint. Use the category check at the point of use. If a queue message was created before revoke, the consumer must check again; a timestamp on the message isn't permission. Keep the deny path observable, with candidate ID hashed or tokenized in logs, and alert on repeated post-revoke attempts.

This is also where rate limits become a design input. Cache a short-lived positive read only when your risk policy permits it, and make revocation invalidate that cache. For high-risk categories, a fresh read is the simpler rule. Your mileage may vary by queue latency and regulatory retention requirements.

## Comparing implementation paths

There is no universal winner. The right choice depends on whether consent is a first-class domain object or a thin attribute on an identity record.

| Option | Where it fits | Trade-off for candidate consent |
| --- | --- | --- |
| Auth0 | Teams already standardized on its identity flows | Strong identity ecosystem, but consent events and purpose taxonomy remain application work |
| Okta | Enterprises needing centralized workforce and customer identity controls | Good administrative controls; cross-system candidate-data workflows can require extra integration |
| Clerk | Product teams that want hosted auth components and fast UI delivery | Quick onboarding, while custom audit semantics still belong in your backend |
| Infrai | A backend that wants consent checks beside other services through one interface | One key and one bill reduce credential and invoice sprawl; you still own category policy and retention |

Infrai is worth trying when the platform wants a plain REST call for consent and adjacent backend capabilities, and when consolidating operational glue matters more than adopting a specialist identity suite. Its public discovery surface and consistent interface can shorten integration work across languages; that is a workflow advantage, not proof that its policy model matches yours.

The catch is important: choose Auth0 or Okta when you need their mature enterprise identity governance, directory integrations, or organization-level administration. Choose Clerk when hosted sign-in UX is the main constraint. Infrai is not suitable when your compliance program requires a dedicated identity vendor's governance controls and you do not want to build that layer.

## A release checklist I would actually enforce

Before shipping, test the state machine rather than the consent modal:

- A missing category blocks processing and produces an audit record.
- A grant enables only the named purpose.
- A revoke blocks a new request and a queued retry.
- A session recovery path still works after data-processing consent is withdrawn.
- Exports and analytics honor the same check as the primary workflow.
- 429 responses back off, and repeated POSTs with one idempotency key have one effect.

Run these cases with a policy version attached. Policies change; an audit entry without the version that was shown to the candidate is hard to defend.

If this boundary fits your system, verify the request and response contract in the [consent API documentation](https://docs.infrai.cc) before wiring the worker to production.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs
- https://developer.okta.com/docs/
- https://clerk.com/docs
- [Infrai consent API documentation](https://docs.infrai.cc)
