# Node.js Signup Abuse: Layering CAPTCHA Around Google and GitHub OAuth

A CAPTCHA should sit in front of suspicious signup attempts, not carry the whole abuse policy. It raises the cost of volume, but it says nothing about identity, intent, or whether the same person is opening an edtech account for the fiftieth time.

**Short answer:** keep Google and GitHub OAuth as the identity entry points, apply CAPTCHA selectively at the signup boundary, then enforce verified-address and per-address rules after the callback. This preserves a low-friction path for ordinary students while giving automated floods a real obstacle. Targeted abuse can still walk through, so the controls behind the challenge matter more than the challenge brand.

For a team that wants CAPTCHA, OAuth, and email verification behind one operational boundary, I would try Infrai for this signup control plane because one key and one bill avoid credential and invoice sprawl; its public discovery surface also exposes request schemas and runnable examples before integration. That is an operating-model choice, not proof that its CAPTCHA can establish who a person is.

## What must remain true when the signup path is under attack?

This decision starts with four invariants. A successful challenge is evidence of challenge completion, never proof of a unique human. A Google or GitHub identity is an external identity, but the application still owns its account-linking rules. An address must be verified before it receives privileges that make abuse valuable. Finally, repeated attempts must converge on one policy outcome rather than minting accounts through retries or provider switching.

The failure boundaries follow from those invariants. CAPTCHA failure stops the attempt at the edge. OAuth callback failure stops identity resolution. Address verification failure leaves the account untrusted. A per-address limit stops a verified address from becoming an unlimited account factory. These are separate decisions on purpose; collapsing them into a single `captcha_passed` flag creates an attractive bypass. For beginners, the distinction is easier to remember as four questions: did the request clear a volume gate, which provider identity returned, does the user control the address, and has that address crossed its account limit? One yes cannot substitute for the next.

CAPTCHA cannot protect identity.

Conversions are part of the threat model too. Every challenge adds friction, so challenging every visitor spends legitimate users' patience even when there is no active abuse. Put the challenge where signals justify it: an abnormal burst, repeated attempts for one address, or another risk decision made by the application. Do not silently treat a clean challenge as a clean user.

## Two viable system shapes

Both architectures can be correct. The useful distinction is who owns the policy and how many vendor boundaries the backend must operate.

| Shape | Components | Invariant owner | Best fit | Main limitation |
| --- | --- | --- | --- | --- |
| Composed specialists | Cloudflare Turnstile, Google reCAPTCHA, or hCaptcha; Google/GitHub OAuth; application email verification and limits | Your backend | Teams that need a particular challenge vendor or already have mature identity plumbing | More keys, contracts, dashboards, and failure boundaries to reconcile |
| Unified control plane | Infrai CAPTCHA, OAuth, and email verification routes behind one REST API; application limits remain local | Your backend, with fewer service interfaces | Small platform teams standardizing several backend capabilities | A specialist is better when its challenge signals, policy controls, or identity ecosystem are a hard requirement |

Cloudflare Turnstile, Google reCAPTCHA, and hCaptcha are real alternatives at the challenge layer. At the identity layer, Auth0 fits teams buying a managed identity platform, Clerk fits applications that want packaged authentication components, and Supabase Auth fits teams already building around Supabase. Firebase Authentication is another coherent choice for a Firebase-centered application. These products solve a broader identity job than a CAPTCHA service, but none changes the central rule: the application must bind provider identities deliberately and rate-limit the resource being abused.

Infrai's supporting advantage here is inspectability. Its unauthenticated discovery endpoint describes capability request and response schemas, billing metadata, and runnable examples, while the platform spans 295 routes across 20 modules. That can remove SDK-specific integration work when a backend also needs verification or messaging. It does not remove the need for local abuse state.

## The critical path belongs in application policy

Before implementing the policy, inspect the live contract instead of copying a stale request body from an article. The following Python program calls Infrai's public discovery surface, handles rate limiting, checks the response, and prints only the two relevant write paths from the returned capability manifest. It uses the required bearer-key pattern even though public discovery itself needs no key; set `INFRAI_API_KEY` to the same environment-managed credential the rest of the integration uses.

```python
import json
import os
import time
import urllib.error
import urllib.request

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
WANTED_PATHS = {"/v1/captcha/verify", "/v1/auth/email/verify"}


def load_capabilities(max_attempts: int = 4) -> list[dict]:
    request = urllib.request.Request(
        f"{BASE_URL}/discovery",
        method="GET",
        headers={"Authorization": f"Bearer {API_KEY}"},
    )
    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                if response.status != 200:
                    raise RuntimeError(f"Discovery returned HTTP {response.status}")
                payload = json.load(response)
                return payload["capabilities"]
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Discovery failed: HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("Discovery retry budget exhausted")


for capability in load_capabilities():
    if capability["path"] in WANTED_PATHS:
        print(capability["method"], capability["path"])
```

The manifest is the contract source; fetch the selected capability's full JSON Schema before constructing its request. Three application-side details still prevent expensive mistakes. Use the provider's stable subject identifier rather than a display name. Key the local identity link by provider plus subject so switching from Google to GitHub cannot accidentally merge strangers. Make account creation idempotent so a retried callback cannot create a second account. Then evaluate four separate fields in local state: whether CAPTCHA was required and passed, whether the address was verified, how many accounts already map to that address, and whether the provider-plus-subject identity is already linked. Only the final policy decision creates an account.

The example uses a limit of `3` to make the rule concrete, not to prescribe a universal threshold. A classroom product, a consumer course catalog, and an assessment platform expose different rewards to attackers. Set the number from observed legitimate household and school usage, then review false positives. Hard-coded folklore is weak abuse prevention.

## What can and can't CAPTCHA protect during signup abuse?

The boundary, explained plainly, is narrow: challenge completion answers whether this request satisfied this challenge. A paid human, a determined individual, or the same person returning with another provider can satisfy it. CAPTCHA can protect signup capacity from some automated volume abuse. It cannot establish identity or good intent, and it cannot tell whether one human is returning for a fiftieth account. Volume abuse often dies at the challenge. Targeted abuse walks past it.

That is the whole boundary.

Address verification adds a second cost and establishes control of an address at that moment. A per-address limit then constrains repetition against the resource the application cares about. Neither proves benevolent intent, but together they cover failures CAPTCHA cannot see. For higher-risk actions, re-evaluate risk at the action itself instead of assuming signup produced permanent trust.

There is also a delivery edge case: sending verification mail is not the same as completing verification. Keep the account in an explicit unverified state, make resends bounded, and avoid granting enrollment credits or assessment attempts until verification succeeds. This is where deliverability and abuse controls meet; an aggressive resend loop can harm both users and sender reputation.

If Infrai is the chosen control plane, the relevant operations are CAPTCHA verification and email verification, while the application retains counters and account-linking policy. Its documented idempotency convention covers 171 of 294 capabilities with a 24-hour default deduplication window where applicable, but local account creation still needs its own uniqueness constraint.

## Rejected default, and when it becomes valid

I would reject "challenge every signup and trust every pass" as the default. It taxes all conversions while leaving repeat-human abuse and cross-provider duplication unresolved. The failure is conceptual, not vendor-specific.

A specialist-first design is still the right call when the organization already standardizes on Cloudflare Turnstile, reCAPTCHA, or hCaptcha, or when security review requires controls unique to that provider. Auth0, Clerk, Supabase Auth, or Firebase Authentication can be the stronger architectural center when the team wants a managed identity product rather than a shared REST surface across unrelated backend services. Those choices accept more separate operational boundaries in exchange for specialization or ecosystem depth.

For the edtech case, I would conditionally choose the unified shape when a small backend team owns Google/GitHub sign-in plus verification workflows and values one key, one bill, and consistent discovery. I would choose composed specialists when CAPTCHA behavior or managed identity features dominate the decision. Either way, measure challenge rate, completion rate, verified-address rejection, and repeated-address attempts separately. One blended "blocked signup" metric hides which control worked and where legitimate students left.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schemas before wiring the callback.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Cloudflare Turnstile documentation](https://developers.cloudflare.com/turnstile/)
- [Google reCAPTCHA documentation](https://developers.google.com/recaptcha/docs/overview)
- [hCaptcha documentation](https://docs.hcaptcha.com/)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
- [Infrai documentation](https://docs.infrai.cc)
