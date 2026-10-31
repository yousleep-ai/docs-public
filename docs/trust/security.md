# Security overview

youSleep is not yet operated by a company; the
[Privacy Policy](https://portal.yousleep.ai/legal/privacy-policy) says who is
controller until one is formed. *The operating entity will be named here then.*

## Roles

- Your organisation is the controller of its recordings, their study data and
  their results. youSleep is its processor
  ([Terms and Conditions](https://portal.yousleep.ai/legal/terms-and-conditions),
  section 4.5).
- youSleep is the controller of account data.

## Hosting

- Application and database: one server in Denmark, run by youSleep. Its data
  volumes are encrypted (LUKS2).
- Recordings, results and backups: Cloudflare R2, EU region.
- Inbound traffic, administrative SSH included, reaches the server only through
  Cloudflare Tunnel. SSH takes keys only, never root.
- Email: SMTP2GO, transactional only.

## Encryption

- In transit: HTTPS only, TLS 1.2 or 1.3, with HSTS.
- At rest: the server's data volumes are encrypted with LUKS2. Objects in R2,
  backups included, rely on R2's encryption at rest, as Cloudflare states.
- Files move to and from storage by signed URLs, one per request, each
  expiring. Browsers and runners hold no storage credentials.
- An organisation's bucket key is stored encrypted (AES-256-GCM).

## Access

- youSleep's staff approve every account; email addresses are verified.
- Passwords are stored as bcrypt hashes.
- Sessions last 30 minutes, renewable for up to 7 days. A password change ends
  all sessions. Deleting an account requires the password.
- API keys act as one account, reach only what it owns, expire within a year,
  and cannot manage the account or create keys. Only their hash is stored.
- Projects are private by default. Sharing stays within the organisation, and
  opening, downloading and exporting are logged for the project's owner, and for
  the organisation's owners and admins when the project is open to the
  organisation or kept in its bucket.
- Staff have an administrator role, checked on every administrative route.
  Administrative actions are recorded in an audit trail.
- Requests are rate-limited per account, per API key and, before sign-in, per
  client address.

## Sites

An organisation can run analyses on its own hardware and keep recordings in its
own storage. The runner connects out only, signs its requests and verifies
youSleep's answers, and runs each analysis with no network. Recordings kept at a
site never reach youSleep. Details:
[Data and security](../components/runner/data-and-security.md).

## Browser

- A content security policy on every page: scripts run only from the portal;
  inline scripts need a per-page nonce; the portal cannot be framed.
- `X-Content-Type-Options: nosniff`, a strict referrer policy, and a permissions
  policy that disables camera, microphone, location and payment.
- Cookies: authentication, session, CSRF protection and Cloudflare's security
  cookies. No advertising or analytics cookies.

## Backups

- Database: nightly, kept 35 days, undeletable for the first 30.
- Results and logs: copied nightly; a copy is removed 30 days after its original.
- Recordings are not backed up; the uploader holds the original.
- Deleted data leaves every backup within 35 days.

Retention: [Privacy Policy](https://portal.yousleep.ai/legal/privacy-policy),
section 5.

## Monitoring

- Metrics, logs and alerts run on youSleep's own server. No third-party
  analytics or error tracking.
- Refused recording headers are neither logged nor kept. API key secrets are
  never logged.
- Cloudflare processes request metadata, such as IP addresses, for security.

## Vulnerabilities and incidents

- Report a vulnerability: [yousleep.ai/security](https://yousleep.ai/security).
- Dependencies are monitored for known vulnerabilities in every repository.
  Container images are pinned by digest.
- Every change goes through a pull request with automated tests before deploy.
- Breaches are notified as the
  [Privacy Policy](https://portal.yousleep.ai/legal/privacy-policy) sets out
  (section 8): an organisation's owner is told without undue delay when its
  recordings are affected.

Contact: [contact@yousleep.ai](mailto:contact@yousleep.ai).
