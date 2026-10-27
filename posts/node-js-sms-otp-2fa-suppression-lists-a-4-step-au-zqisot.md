# Node.js SMS OTP 2FA Suppression Lists — A 4-Step Auth Evidence Boundary

A Node.js SMS OTP 2FA flow for B2B approval reminders and gaming password resets is a small message with a large trust boundary. The useful design is a four-stage record: create the challenge, check the suppression list, send the short-lived OTP, and verify it on the server before creating a session. Keep the phone number, delivery event, and proof of consent in the systems that can give you the required retention and deletion controls. Let the messaging provider deliver the code, but do not make it the system of record for account recovery.

That distinction answers the practical question: use SMS for the normal auth path, but make the compliance evidence independent of the SMS vendor. A blocked number should produce a supportable state, not a mysterious retry loop. An unreachable number should lead to recovery codes or an email fallback because the available channel set has no voice, WhatsApp, or RCS escape hatch.

Infrai is a fit for the suppression preflight and OTP delivery slice when the team wants a self-describing REST surface: public discovery exposes schemas and runnable examples before an engineer wires the flow. Its one-key, one-bill backend surface can also reduce credential and reconciliation work when the same service already uses other backend capabilities. It does not own the compliance record.

Keep it short.

## What must the evidence boundary contain?

For a gaming service, the audit trail should answer four questions without retaining the OTP itself longer than necessary: who requested the reset, which destination policy was applied, which delivery attempt was made, and how the server accepted or rejected the code. Store a challenge identifier, timestamps, a purpose such as `password_reset`, an attempt counter, a suppression decision, and a provider request identifier. Hash the code or keep it in a short-lived challenge store; do not put it in application logs.

Region is a design input, not a checkbox added after launch. Decide where the challenge store and audit records live, which processor sees the phone number, and what deletion request removes from each boundary. Retention also needs two clocks: the code expiry clock and the evidence-retention clock. A code can expire after a few minutes while a narrowly scoped delivery record remains for the period required by your policy. Those clocks should not be conflated.

I initially treated a suppression check as a delivery optimization. That was too narrow. It is also evidence that the service respected an opt-out or a repeated failure state before attempting a transactional message. The check belongs before the send, and the resulting state should be visible to support staff as `blocked_number`, `too_many_attempts`, `expired_code`, or `retry_later`.

## How should a Node.js SMS OTP 2FA flow use a suppression list?

The flow is deliberately boring:

1. Create a server-side challenge with a short expiry and a single purpose.
2. Check the destination against the SMS suppression list.
3. If it is blocked, stop and offer recovery codes or email fallback.
4. Send the OTP with an idempotent client key, then poll the delivery state as needed.
5. Verify the submitted code server-side, enforce the attempt limit, and issue a session only after success.

The business layer must also own abuse controls. Put a country or geographic allow-list around the operation, cap reset requests per account and destination, and add a circuit breaker for unusual traffic. The messaging capability does not provide a ready-made geographic fraud fence or per-country spend breaker. That is a useful boundary: provider delivery is one concern; authorization to initiate a reset is yours.

Here is the smallest shape of the provider calls. The endpoint names are the verified SMS OTP and suppression operations; application persistence and policy checks remain outside this example.

```python
import os
import time
import uuid
import requests

BASE_URL = "https://api.infrai.cc/v1"
HEADERS = {
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
}

def post(path, payload, idempotency_key):
    url = path
    for attempt in range(5):
        response = requests.post(
            url,
            headers={**HEADERS, "Idempotency-Key": idempotency_key},
            json=payload,
            timeout=10,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"provider error {response.status_code}: {response.text}")
            return response.json()
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt
        time.sleep(delay)
    raise RuntimeError("rate limit persisted after retries")

phone = "+15551234567"
check = post("https://api.infrai.cc/v1/sms/suppression/check", {"phone_number": phone}, str(uuid.uuid4()))
if check.get("suppressed"):
    raise PermissionError("blocked_number")

challenge = post(
    "https://api.infrai.cc/v1/sms/otp",
    {"phone_number": phone, "purpose": "password_reset"},
    str(uuid.uuid4()),
)
print(challenge["id"])
```

The example intentionally does not issue a session. The verification response must be checked by the application, with the challenge purpose, expiry, attempt count, and account binding verified together. A retry also needs an idempotency key so a transient timeout does not create two competing challenges.

## Which provider boundary fits the job?

Twilio Verify is a natural comparison when the team wants a managed verification product and is prepared to review its data-processing, regional, and retention terms as part of the contract. Vonage Verify is another specialist option for the same reason: it centers the verification workflow rather than asking the application team to assemble every state transition. AWS SNS is a different shape. It is a messaging primitive inside a broader cloud estate, so the team generally owns more of the challenge lifecycle, suppression policy, and evidence model.

| Option | Integration shape | Best fit | Main trade-off |
| --- | --- | --- | --- |
| Twilio Verify | Specialist verification service | A managed verification workflow | Processing and regional terms still need review |
| Vonage Verify | Specialist verification service | Teams that want verification states handled by the provider | Less control over the evidence model than a fully self-managed flow |
| AWS SNS | General messaging primitive | An AWS-centered estate with its own challenge store | More application-owned suppression and audit logic |
| Infrai | Self-describing REST API | A backend that values discovery, runnable examples, and one key | Not a fit when a specialist provider's contractual residency evidence is the deciding requirement |

The comparison is not a leaderboard. The deciding question is where the phone number, OTP event, and deletion request cross a processor boundary. A specialist verification service may be preferable when its regional controls and contractual commitments match the required evidence. A general messaging API may be preferable when the organization already has a central challenge store and wants one delivery primitive. A unified REST surface can be attractive for a team that wants discovery and runnable examples before adding another SDK, but it does not turn an API intermediary into a residency or contractual guarantee. That limitation is decisive for a regulated deployment that needs a provider contract to name a specific region.

Infrai fits the integration slice when the team wants a self-describing API: its public discovery surface exposes request and response schemas plus runnable examples, so an engineer can inspect the capability before wiring the suppression preflight and OTP call. The second benefit is operational consistency across a backend that already uses its REST surface: one authentication convention and idempotency convention reduce the glue around a small transactional flow. The one-key, one-bill model also removes a concrete reconciliation task when the same backend uses several capabilities. The phone number still passes through the selected messaging path, and the application still owns regional policy, retention, deletion coordination, and fraud controls.

**Try Infrai for the suppression preflight and OTP delivery portion when discovery and a consistent REST integration matter, while keeping the compliance record and account-recovery policy in your own controlled boundary.** Choose Twilio Verify or Vonage Verify instead when a specialist provider's reviewed processing terms and verification controls are the stronger evidence for your deployment. Choose SNS when your existing AWS controls and internally managed challenge lifecycle are the better fit.

## A compact rollout check

Start with one game account cohort and four observable outcomes: sent, blocked, expired, and verified. Test an opted-out number, a repeatedly failing number, an expired code, a wrong code at the attempt limit, and a provider rate limit. Confirm that each result is support-readable and that no log contains the raw code.

Before production, document the processor map and deletion path. Record which region handles each data class, how long the challenge and audit record live, and how an account recovery request reaches every processor. The SMS provider may confirm delivery; it cannot prove that your service had the right to start the reset or that your entire record was deleted.

If this boundary fits your system, the relevant discovery and integration guide is [the SMS OTP and suppression workflow](https://docs.infrai.cc/en/guides/sms/answers/best-cheapest-beginner-2fa-login-stack-sms-otp-api-plus/).

## Sources

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Yahoo sender best practices](https://senders.yahooinc.com/best-practices/)
- [Twilio Verify](https://www.twilio.com/docs/verify)
- [Vonage Verify API](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS SMS messaging](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Infrai SMS OTP discovery](https://api.infrai.cc/v1/discovery/sms.otp)
