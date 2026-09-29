# teupadel: e-mail and SES

!!! info "Source of truth / update here"
    Runbook for teupadel.com transactional e-mail (Amazon SES, `us-east-1`). Update (PT + EN) when the SES
    status, bounce handling or DMARC policy change. Follow-up items in the
    [backlog](teupadel-backlog.en.md) (6, 7 and 8).

## Current status (2026-09-29)

| Point | Status |
|---|---|
| Mode | **Sandbox** |
| Production access | **Denied by AWS on 2026-09-29** (case `179054498900338`). Reopen in 2 to 3 weeks |
| Usage | Transactional e-mail only: the login magic link (Google login sends no e-mail) |
| Sender | `TeuPadel <noreply@teupadel.com>` (`SES_SENDER`), configuration set `teupadel-transactional` |
| Domain | `teupadel.com` verified, with DKIM, SPF, DMARC and a custom MAIL FROM (per the text sent to AWS) |
| DMARC | `p=none` today; plan below |
| Magic link in production | **Never exercised** (backlog 6) |

## Verified addresses and testers

In the sandbox SES only delivers to verified identities. The API only sends the magic link to `@teupadel.com`
or verified identities; any other request keeps the `202` answer without sending e-mail (anti-enumeration).
To authorise a tester:

```bash
aws sesv2 create-email-identity --email-identity <email> --region us-east-1
# the person clicks the AWS e-mail; then confirm:
aws sesv2 list-email-identities --region us-east-1
```

The current identity list is not recorded here; query it with the second command.

## What the API does with bounces and complaints

Flow: SES -> SNS -> SQS queue `teupadel-ses-events` -> `ses_events.py` worker (API background task, only
active if `SES_EVENTS_QUEUE_URL` is set).

- **Permanent** bounce and complaint: written to `email_suppressions` (`ON CONFLICT DO NOTHING`).
- Transient bounces and `Delivery`: ignored.
- Unreadable message: deleted from the queue. DB error: the message stays in the queue (redelivery).
- Before any send the API checks `email_suppressions`; a suppressed address gets no e-mail.
- With 2 replicas both consume the same queue; SQS delivers each message to only one.
- Needs `sqs:ReceiveMessage`/`DeleteMessage` on the API's AWS credentials (IaC `iac-mail-routing`, module
  `aws-ses-smtp-user`).
- Other sending defences: 5 requests/hour per IP on `/auth/magic-link`, `EMAIL_LINKS_PER_HOUR` (5) per
  e-mail, disposable domains rejected, Turnstile on the request.

## DMARC plan and post-approval

1. After leaving the sandbox, send a real e-mail to Gmail and Outlook and validate SPF, DKIM and DMARC in
   the headers.
2. Configure sending as `contato@` (SMTP in `/teupadel/ses/gmail-smtp`).
3. Keep `p=none` and read the aggregate reports for 2 to 4 weeks.
4. With clean reports, raise to `p=quarantine`.
5. Monitor the bounce rate (alert > 2 %, not yet created, backlog 9) and the complaint rate.
6. Test receiving `contato@`, `suporte@` and `privacidade@` (backlog 24).

## Reopening the production request

Reopen the case in 2 to 3 weeks, with the bounce worker validated. Describe the real usage: transactional
only, low volume, sending only to people requesting login, bounce and complaint handling. Update the status
here and in the backlog with the date of the new answer.

## Appendix: text to reuse in the request

Text sent to AWS (in English, the language of the request). Adjust before resending: the line "Password
reset emails, if enabled" does not apply today because there is no password login, and it is worth citing
the already validated bounce worker.

> Hello,
>
> Thank you for following up.
>
> TeuPadel is a web application that uses Amazon SES exclusively for transactional email. We are requesting production access for the us-east-1 region.
>
> Our initial email types will be:
>
> - Email verification links
> - Passwordless login (magic) links
> - Password reset emails, if enabled
> - Account and security notifications
>
> These emails are sent only after a user explicitly requests an action in the application. We do not send marketing campaigns, newsletters, cold email, or messages to purchased or third-party lists.
>
> At launch, we expect low volume, typically tens of emails per day, with occasional short bursts. We expect to remain well below the default SES quota.
>
> Our sending domain, teupadel.com, is verified in SES. DKIM, SPF, DMARC, and a custom MAIL FROM domain are configured. Emails will be sent from noreply@teupadel.com, with replies directed to our support/contact address.
>
> We have configured SES bounce, complaint, and delivery notifications through SNS and SQS. We will use these events to monitor delivery, suppress addresses that bounce or generate complaints, and prevent further sending to problematic recipients.
>
> We do not currently send marketing emails. If we introduce optional marketing communication in the future, it will require explicit opt-in, include an unsubscribe mechanism, and be managed separately from transactional messages.
>
> We will monitor bounce and complaint rates and investigate delivery issues promptly. We will also test the SES mailbox simulator before launching.
>
> Website:
> https://teupadel.com
>
> Please let us know if you need any additional information.
>
> Kind regards,
> Cassio Oliveira
> TeuPadel
