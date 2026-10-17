# SMTP Relay Appliance — Specification

Status: draft, 2026-10-16. Written to become `docs/SPEC.md` in `smtp-relay-appliance`.

A small Debian 13 appliance, built on `appliance-core`. It accepts mail from clients on a local
network, including old ones that can't do modern authentication. It queues each message durably
and submits it to Microsoft 365 **as the sender's own mailbox**, with app-only OAuth, so a copy
lands in that mailbox's Sent Items. For every message it records **capture events** to a local
outbox, which a publisher can forward to an event broker.

## 1. What it does

| | |
|---|---|
| **Accepts** | SMTP from allowlisted hosts (no login needed) and from authenticated clients (STARTTLS plus SMTP AUTH) |
| **Checks** | The client is allowed, and the `From` address is an allowed sender |
| **Adds** | A `Message-ID` when the message has none; it never rewrites an existing one |
| **Records** | `mail.issued` to the outbox **before** answering the client's `DATA`; later `mail.handed_off` and `mail.forward_failed` |
| **Delivers** | SMTP submission to Microsoft 365 with XOAUTH2 and an app-only token, authenticated as the `From` mailbox |
| **Keeps** | Retries on temporary failure. On permanent failure, a dead letter kept for a set time |
| **Doesn't** | Store message content beyond the queue and dead letters; accept mail for local delivery; relay for anyone outside its allowlists |

**v1** delivers by sending, through the `m365-smtp-oauth` upstream (§5).

**Later:**
- a `draft` route, which puts the message in the sender's Drafts through Graph;
- a `graph-sendmail` upstream, as a fallback;
- a generic `smarthost` upstream, for non-Microsoft 365 users;
- a publisher that forwards the outbox to NATS JetStream.

The event model carries `route` from day one, so adding `draft` changes no event type.

## 2. Components

```text
client ──SMTP──▶ Postfix smtpd ──milter──▶ smtprelay-milter ──▶ outbox/ (mail.issued)
                     │
                 queue (Postfix)
                     │
                 Postfix smtp ──XOAUTH2──▶ smtp.office365.com:587
                     │                        ▲
                 maillog ──▶ smtprelay-follow ─┘ (mail.handed_off, mail.forward_failed ──▶ outbox/)
                                    smtprelay-token: keeps a fresh app token for sasl-xoauth2
```

| Component | Role |
|---|---|
| **Postfix** | Listener, durable queue, retries, deferral and bounce handling. Stock Debian package |
| **sasl-xoauth2** | Cyrus SASL plugin that gives Postfix's SMTP client the XOAUTH2 mechanism. Debian package `sasl-xoauth2` |
| `smtprelay-milter` | Milter at end-of-message: adds a missing `Message-ID`, writes `mail.issued`, then lets Postfix answer. If the outbox write fails, it answers `451` (temporary), so no message is accepted unrecorded |
| `smtprelay-token` | Gets an app-only token by the client-credentials grant, with a certificate or a secret. Writes it in the form sasl-xoauth2 reads, and refreshes it before expiry |
| `smtprelay-follow` | Follows Postfix's log by queue ID. Turns each upstream acceptance into `mail.handed_off` and each failed attempt into `mail.forward_failed` |
| `smtprelay-sconfig` | The appliance's TUI/CLI on `appliance-core`'s `tui` and `kvstate`: network, certificate, upstream, allowlists, senders, status |
| `smtprelay-health` | One status check for operators and monitoring (§9) |

The milter, the follower and the token helper are the only new code. Postfix and sasl-xoauth2
handle queueing, retries and SMTP.

## 3. Listener

| Item | Behaviour |
|---|---|
| Ports | `25` and `587`, both offering STARTTLS. No implicit-TLS port in v1 |
| Certificate | Supplied by the operator (`sconfig`), with the chain. A self-signed one is generated only as a placeholder at first boot, and `health` flags it |
| **Allowlisted hosts** | Addresses or networks listed in `allowed_hosts`. No login. STARTTLS offered, not required. The client is identified by its address |
| **Authenticated clients** | Everyone else: STARTTLS **required**, then SMTP AUTH (`PLAIN`/`LOGIN` over TLS). One credential per client in a local SASL database. The client is identified by its login |
| Everyone else | Refused (`554`) |
| Size limit | `message_size_limit`, default 35 MB, Microsoft 365's limit for SMTP submission (to confirm in the tenant test) |

**Senders.** The `From` header's address, and the envelope sender, must each be in
`allowed_senders`, otherwise `553` at `MAIL FROM` or `550` after `DATA`. The upstream scope (§5)
enforces the same list a second time. An allowed sender sends only as itself: a message is
submitted as its `From` mailbox.

**No open relay.** `mynetworks` is empty. Clients reach the relay only through `allowed_hosts` and
SASL, and recipients may be anywhere. A lab scenario proves an outside host is refused.

## 4. Message-ID

- **Never rewritten.** It is the link between the capture event and the copy in the mailbox.
- **Missing:** the milter adds `Message-ID: <{UUIDv7}@{relay hostname}>` before recording, so the
  event and the delivered message carry the same value.
- **Normalized for events:** the header's value, with the angle brackets and surrounding
  whitespace removed, case kept.

## 5. Upstream: `m365-smtp-oauth`

| Item | Value |
|---|---|
| Server | `smtp.office365.com:587`, STARTTLS, verified against the public CA store |
| Auth | XOAUTH2. The token is app-only, from the tenant's token endpoint, scope `https://outlook.office365.com/.default` |
| As whom | The `From` address: Postfix's sender-dependent authentication chooses it, so `user=` is the message's sender and the Sent Items copy lands in that mailbox |
| Credential | **Certificate** (preferred): the key is generated on the appliance and never leaves it, and only the public certificate goes to the app registration. A client **secret** is supported but expires |
| Recipients | Taken from the SMTP envelope, so `Bcc` recipients in the envelope only are delivered |

**Tenant setup** (the operator's, documented in `docs/M365-SETUP.md`):
1. An app registration with the certificate (or secret), and its service principal registered in
   Exchange Online.
2. **Permission through Exchange's RBAC for Applications only:** the `Application SMTP.SendAsApp`
   role, assigned with a management scope that selects the allowed senders (a mail-enabled group
   or an administrative unit). Don't also consent the permission in Entra. A grant there applies to
   every mailbox and defeats the scope.
3. SMTP AUTH on, **for the allowed sender mailboxes only**.
4. Shared mailboxes work as senders and need no licence.

**Acceptance test against a real tenant.** Run before production, and after any change to the
upstream. Send one message with a known Message-ID, a recipient in the envelope only, and a body
over 4 MB. Then check:
1. the copy in the sender's Sent Items is found by that Message-ID;
2. the recipient's copy and message trace show the same Message-ID;
3. the envelope-only recipient received it;
4. a message from a mailbox outside the scope is refused.

If check 1 or 2 fails, the Message-ID link breaks, and the `graph-sendmail` upstream becomes v1's
candidate. It is subject to the same test.

## 6. Queue, retries, dead letters

- **Temporary failure (4xx, timeouts):** Postfix retries with its backoff, up to
  `maximal_queue_lifetime` (default 2 days). Each failed attempt publishes `mail.forward_failed`
  with `retrying: true`.
- **Permanent failure (5xx, or the lifetime runs out):** the message moves to the dead-letter
  store, `/var/lib/smtprelay/dead/`. It is kept for `dead_letter_days` (default 30) and then
  deleted. `mail.forward_failed` is published with `retrying: false`, and `health` reports the
  count. The relay doesn't send bounces: whoever needs to know subscribes to the event.
- **A message accepted from the client is never dropped silently.** It is delivered, or retried,
  or a dead letter.

## 7. Capture events

The events record what the relay witnessed and nothing more. A message's later fate (delivered,
bounced, a draft sent or discarded) is for observers of the mailbox to record, not the relay.

### Outbox

- One JSON file per event in `/var/lib/smtprelay/outbox/`, written atomically: write to a
  temporary file, `fsync`, then rename.
- A publisher moves sent files to `outbox/sent/`. v1 has no publisher, and the files stay until
  one exists or an operator collects them.
- Every event has a **source key**, so a republished event is recognized as a duplicate.

### Types (`type_version` 1)

| Type | When | Source key | Payload |
|---|---|---|---|
| `mail.issued` | Before the reply to `DATA` | `mail:{message_id}` | `message_id`, `from`, `to[]`, `cc[]`, `subject` (decoded), `attachments[]` (`name`, `media_type`), `client`, `relay`. **No body, no attachment content, no hash** |
| `mail.handed_off` | The upstream accepted the message | `mail:{message_id}:handed_off` | `route` (`send`; `draft` later), `upstream_reply` |
| `mail.forward_failed` | Each failed attempt | `mail:{message_id}:forward_failed:{attempt}` | `attempt` (from 1), `error` (the upstream's reply, or the local error), `retrying` |

**Envelope:**
- `issued_by` is `observer:smtp-relay` and `module` is `smtp-relay@{VERSION}`.
- `occurred_at` is when the relay accepted the message, or when the attempt ended.
- `correlation_id` is a new UUID per message, which every event about that message reuses.
- `causation_id` is the `mail.issued` event's id.

A deployment maps these names to its own event store's conventions in the publisher.

A message sent again by a client after a timeout has the same Message-ID, and so the same source
key: a duplicate, not a second message.

## 8. Configuration

All settings are kept in `appliance-core`'s `kvstate` and changed through `smtprelay-sconfig`.
Each change is applied by regenerating Postfix's maps.

| Key | Meaning |
|---|---|
| `hostname` | FQDN, used in the greeting and in added Message-IDs |
| `tls_cert`, `tls_key` | Listener certificate |
| `allowed_hosts` | Networks and addresses that send without logging in |
| `clients` | SASL logins (managed in the TUI, stored as hashes) |
| `allowed_senders` | Addresses the relay submits as |
| `upstream` | `m365-smtp-oauth` (v1) |
| `m365_tenant_id`, `m365_client_id` | App registration |
| `m365_credential` | `certificate` or `secret` |
| `message_size_limit`, `maximal_queue_lifetime`, `dead_letter_days` | Limits |

The release identity is written to `/etc/smtp-relay-appliance.release` (`appliance-core`
`lib/update.sh`).

## 9. Health

`smtprelay-health` prints one line per check and exits non-zero on any failure:
- Postfix is running, and the milter and follower are up;
- the token was refreshed and is valid for more than 10 minutes;
- the listener certificate isn't self-signed and isn't within 30 days of expiry, and the same for
  the app certificate;
- the oldest deferred message is younger than a threshold;
- no dead letters arrived since the last acknowledgement;
- the outbox backlog is below a threshold.

The output is meant to be scraped by any monitoring system.

## 10. Lab

Scenarios run with `lab-kit` on the lab network (`10.10.10.0/24`). Microsoft 365 is replaced by a
**fake upstream**: a test SMTP server that requires STARTTLS and XOAUTH2, checks the token's
`user=` against the `From`, and can be told to answer 4xx or 5xx. Scenarios:
1. An allowlisted host sends without logging in: delivered, `mail.issued` then `mail.handed_off`.
2. An authenticated client: delivered, and `client` is the login.
3. An outside host, or a disallowed `From`, is refused, and no event is written.
4. A message without a Message-ID gets one, and the delivered copy and the event carry the same
   value.
5. Record before reply: with the outbox unwritable, the client gets `451` and nothing is queued.
6. A 4xx upstream: retried, with `mail.forward_failed` and `retrying: true` per attempt, then
   delivered.
7. A 5xx upstream: a dead letter, `retrying: false`, and `health` fails.
8. An envelope-only recipient reaches the fake upstream.
9. A resend after a timeout produces the same source key.

## 11. Open

- The size limit and Message-ID behaviour are confirmed only by the tenant test (§5).
- The `draft` route needs Graph (`Mail.ReadWrite`), and it waits until it's known whether Outlook
  keeps a draft's Message-ID when a person edits and sends it.
- The publisher's target and its credentials, and how events map to a store's subjects, are a
  deployment's choice.
