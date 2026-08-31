# Bounce Handling First: What to Compare in a Transactional Email and SMS Notification API

Keep the suppression list and the message templates inside your own service, then pick the transactional email and SMS notification API on the quality of the bounce data it hands back to you. The deciding constraint is not the per-message rate. It's whether, six months from now, you can explain why one customer stopped receiving order confirmations — and prove the address was dead before you ever queued the next one.

The system I have in mind is an ordinary storefront: order confirmation, shipment notice, refund receipt, plus an SMS for the delivery window. Addresses arrive from a checkout form, which means a steady trickle of `gmial.com`, mistyped corporate domains, and the work address someone left behind two jobs ago. Every one of those is a future bounce. Every bounce you fail to record is a small, permanent tax on the reputation of your sending domain.

## Acceptance is not delivery

When the API answers 202 with a message id, one thing has happened: the provider took custody. The receiving mail server can still reject the message seconds later, and that rejection comes back asynchronously — as a delivery status notification, or as a webhook that wraps the same information in the provider's own vocabulary.

The part worth storing is the enhanced status code defined in RFC 3463. Class 5 is permanent (5.1.1 is the classic bad destination mailbox), class 4 is transient: greylisting, mailbox over quota, a temporary policy block that clears in an hour. A complaint is a third thing entirely — the recipient pressed "report spam" and the mailbox provider relayed it through a feedback loop in ARF format. No retry is appropriate there, ever.

SMS has no bounce. Carriers return delivery receipts that are optional, occasionally wrong, and often late, so an invalid number usually surfaces as a carrier error code with no shared taxonomy behind it. Consent lives somewhere else again, in the STOP keyword handler and, for US A2P traffic, in the campaign registration your numbers are attached to. Treating both channels as one "send" primitive is how a suppression list quietly grows holes.

Two channels. Two failure vocabularies. One customer record.

## Where the suppression gap opens

The common shape of the bug — and I mean a design bug, not a provider defect — is that the provider owns the suppression list and the application owns nothing. It works beautifully until the day you move. The new provider starts with an empty list, your worker replays the queue, and a few thousand addresses that have been dead for a year get mail again. Reputation damage from that single afternoon outlasts whatever you saved on the migration.

Three narrower failures show up just as often. Storing only the normalized label ("bounce", "blocked", "dropped") and discarding the raw diagnostic means you can never re-classify a decision you got wrong. Keeping the email suppression state and the SMS opt-out state in separate systems means the two disagree about the same person. And skipping the check on transactional sends, on the theory that transactional mail is exempt, guarantees you keep hammering an address that the receiving server has already told you does not exist.

Suppressing a good address is the expensive mistake in the other direction. A carrier code you don't recognize, or a 4.x.x storm during someone else's incident, should never harden into a permanent block. Park unknowns in a review state and look at them weekly. I'm not sure any provider's classifier is right about every carrier and every locale, which is exactly why the raw code needs to survive in your own storage.

## How do you compare transactional email and SMS notification providers on bounce data?

Ask five questions, in this order, before anyone opens a pricing page:

- Does the bounce event carry the raw SMTP diagnostic and enhanced status code, or only a normalized label?
- Can you export the full suppression list yourself, on demand, without filing a support ticket?
- Are webhooks signed, and are they retried long enough to survive an hour of your own downtime?
- Is there a per-recipient status query, so a support agent can answer "did it arrive?" without a log dig?
- For EU recipients, where is the message content processed, and how long is it retained?

The shortlist people usually bring to this question — SendGrid, Postmark, Mailgun, Twilio, MessageBird — differs less in whether mail arrives than in how each one names a bounce, how much of the underlying SMTP conversation it exposes, and what it hands back when you leave. That difference is a normalization problem you solve once in your own code, not a ranking. Most of these shops run the storefront in Node.js and the fulfillment workers somewhere else; the vendor SDK, in either runtime, is a thin wrapper over an HTTP call and a signature check, so it should never be the thing that decides the architecture.

The authentication baseline is not negotiable and is nobody's product feature: SPF, DKIM, and a DMARC policy with a reporting address, plus one-click unsubscribe on anything a human could plausibly call bulk. Mailbox providers publish spam-complaint thresholds around 0.3%, and a transactional stream that shares a domain with a sloppy marketing stream inherits the consequences. Also stop counting opens as evidence of delivery. Apple's Mail Privacy Protection loads remote content on behalf of the user, so an open is a signal about the mail client, not about the human.

## Template ownership decides what a migration costs

Now the axis that actually splits these designs. Templates can live in the provider, in your repository, or in a split arrangement, and the choice determines how much of the bounce and suppression machinery you can carry with you.

| Template ownership | What the provider stores | Effect on bounce and suppression work | Reject it when |
|---|---|---|---|
| Provider-hosted | Layout, copy, merge fields | Suppression state tends to follow the templates into the provider's dashboard; export becomes a migration project | You have a second channel, a second region, or any plan to change transport |
| App-rendered (in git) | Nothing but the finished MIME body | Your service already owns rendering, so owning the verdict and the suppression table is a small addition | Non-engineers must change copy hourly and you have no deploy pipeline for content |
| Split: layout in git, strings in a CMS | Nothing | Same as app-rendered, with translation review handled outside the deploy cycle | The team is one person and the CMS is one more thing to run |

App-rendered templates are the boring, portable choice for a storefront, because the same worker that renders the MIME body is the natural place to enforce the send gate. The critical path is short: classify the event, write the verdict, refuse the next send. Here it is in Python, with SQLite standing in for whatever your fulfillment service already uses.

```python
import re
import sqlite3
from datetime import datetime, timedelta

# RFC 3463 enhanced status code, e.g. "550 5.1.1 <a@b.example>: unknown user"
STATUS = re.compile(r"\b([245])\.(\d{1,3})\.(\d{1,3})\b")
SOFT_STRIKES = 4                      # transient failures tolerated per address
SOFT_WINDOW = timedelta(days=30)


def verdict(kind: str, diagnostic: str) -> str:
    """kind is the provider's own label; diagnostic is the raw SMTP text."""
    if kind == "complaint":
        return "permanent"            # a spam report is never retried
    match = STATUS.search(diagnostic or "")
    if not match:
        return "review"               # unknown shape: park it, never guess
    return {"5": "permanent", "4": "transient"}[match.group(1)]


def record_event(db, address: str, kind: str, diagnostic: str, at: datetime) -> str:
    addr = address.strip().lower()
    call = verdict(kind, diagnostic)
    db.execute(
        "INSERT INTO bounce_event(address, kind, diagnostic, at) VALUES (?, ?, ?, ?)",
        (addr, kind, diagnostic, at.isoformat()),
    )
    if call == "review":
        return call
    if call == "transient":
        since = (at - SOFT_WINDOW).isoformat()
        (strikes,) = db.execute(
            "SELECT count(*) FROM bounce_event WHERE address = ? AND kind = 'soft' AND at >= ?",
            (addr, since),
        ).fetchone()
        if strikes < SOFT_STRIKES:
            return "deferred"
    db.execute(
        "INSERT INTO suppression(address, reason, code, since) VALUES (?, ?, ?, ?) "
        "ON CONFLICT(address) DO UPDATE SET reason = excluded.reason, code = excluded.code",
        (addr, kind, diagnostic[:200], at.isoformat()),
    )
    return "suppressed"


def deliverable(db, address: str) -> bool:
    """Called before every send, transactional ones included."""
    row = db.execute(
        "SELECT 1 FROM suppression WHERE address = ?", (address.strip().lower(),)
    ).fetchone()
    return row is None
```

Two operational notes that matter more than the code. Store the raw event before you decide anything, so a classification change can be replayed over history instead of applied going forward only. And make the send path idempotent with a unique key over (order id, event type, recipient), because the retry that duplicates a shipment notice is the same retry that duplicates a bounce record and pushes a good address over its strike count.

## The option I rejected, and when it's the right one

I rejected provider-hosted templates with the provider's suppression list as the source of truth. Not because it's badly built — it's usually excellent, and the editing experience beats anything you'll ship yourself — but because it puts two things you need during a migration behind someone else's export tooling, and it doesn't give the marketing team much that a CMS wouldn't.

The catch is that this rejection is conditional, and the condition is not "scale". A single-brand shop on one channel, with a marketing lead who rewrites copy weekly and no engineer to deploy it, should stick with provider-hosted templates and mirror only the suppression list into its own database. That's a genuinely good trade-off: you give up portability of the copy, which you rarely exercise, and you keep the part that costs you money when it's missing. Multi-region senders, anyone splitting email and SMS across different vendors, and teams whose templates are localized in the same review flow as their code should keep rendering in the application.

Own the verdict. Rent the transport.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- RFC 3463, Enhanced Mail System Status Codes: https://datatracker.ietf.org/doc/html/rfc3463
- RFC 3464, An Extensible Message Format for Delivery Status Notifications: https://datatracker.ietf.org/doc/html/rfc3464
- RFC 5965, An Extensible Format for Email Feedback Reports (ARF): https://datatracker.ietf.org/doc/html/rfc5965
- RFC 8058, Signaling One-Click Functionality for List Email Headers: https://datatracker.ietf.org/doc/html/rfc8058
- Google, Email sender guidelines: https://support.google.com/mail/answer/81126
- Apple, Use Mail Privacy Protection on iPhone: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
