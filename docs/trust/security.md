# Security overview

This page is for the people who assess youSleep before their organisation uses
it: an IT security reviewer, a data-protection officer, an architect. It states
how the platform is built and run today. The [data-flow sheet](data-flow.md)
follows a recording through the platform, and the [sub-processors](sub-processors.md)
page lists who else processes data.

!!! note "Who operates youSleep"
    youSleep is a research project, provided as a closed service on a
    request-access basis. No company has yet been incorporated to operate it;
    the [Privacy Policy](https://portal.yousleep.ai/legal/privacy-policy) says
    who acts as controller until one is. *The operating entity will be named
    here once it is formed.*

## Roles in the data

- **Your organisation is the controller** of the recordings uploaded or
  registered in it, what is entered about their studies, and the results of
  their analyses. youSleep processes that data on your organisation's behalf,
  as its processor, under section 4.5 of the
  [Terms and Conditions](https://portal.yousleep.ai/legal/terms-and-conditions).
- **youSleep is the controller** of the account data it needs to run the
  service: names, email addresses, organisation membership, usage.
- **The recordings are meant to be pseudonymised.** The portal and the SDK
  blank the identification fields of a recording's EDF header on the user's own
  computer before upload, and check every header again on the server
  ([Pseudonymisation](data-flow.md#pseudonymisation)).

## Hosting

| | |
|---|---|
| Application and database | One server in Denmark, operated by youSleep. Its data volumes are encrypted (LUKS2) |
| Recordings, results and backups | Cloudflare R2 object storage. Recordings and analysis results are stored in the EU region |
| Inbound traffic | Through Cloudflare only. The server accepts no inbound connection except through Cloudflare Tunnel, which also carries administrative SSH |
| Email | SMTP2GO, for transactional email only |

The server is administered over SSH with keys only; it refuses passwords and
the root account. An organisation can also keep its recordings and run its
analyses on its own infrastructure ([Your own site](#your-own-site)).

## Encryption

- **In transit.** Every connection is HTTPS, with TLS 1.2 or 1.3 and HSTS.
  Browsers, the SDK and runners reach the API at `api.yousleep.ai` and the
  portal at `portal.yousleep.ai`.
- **At rest.** The server's data volumes, which hold the database, are
  encrypted with LUKS2. Cloudflare encrypts the objects it stores in R2.
- **Transfers to storage** go by signed URLs: each is for one request on one
  object, and expires. Neither a browser nor a runner holds a storage
  credential.
- **An organisation's bucket key**, where it gives one, is stored encrypted
  (AES-256-GCM) under a key kept apart from the database and its backups.

## Accounts and access

- **Access is by request.** youSleep's staff approve each account before it
  can sign in, and an email address is confirmed before use.
- **Passwords** are stored as bcrypt hashes.
- **Sessions** are short-lived tokens: 30 minutes, renewed for up to 7 days.
  Changing the password ends every session the account had, and deleting the
  account asks for the password again.
- **API keys** are for machines. Each acts as one account with the
  permissions chosen for it, reaches only what that account owns, always
  expires (after at most a year), and can be revoked on its own. A key cannot
  manage the account, accept legal documents or create keys. youSleep stores
  only a hash of each key.
- **Projects are private by default.** A user can make a project visible to
  their organisation or share it with named members; sharing never reaches
  beyond the organisation.
- **The access log.** When a member opens a project they do not own, or
  downloads a recording or exports results, it is recorded. The project's
  owner and the organisation's owners and admins can read the log.
- **youSleep's staff** hold an administrator role, which the API checks on
  every administrative route. Every administrative action is recorded in an
  audit trail.
- **Rate limits** apply per account, per API key and, for routes used before
  sign-in, per client address.

## Your own site

An organisation can run its analyses on its own hardware with youSleep's
runner, and keep a project's recordings in its own S3-compatible storage. Then:

- **The runner only connects out**, over HTTPS, and opens no port. Its
  requests and youSleep's answers are signed, and the runner acts only on
  answers it can verify.
- **Analyses run with no network**, as an unprivileged user, on a read-only
  file system, with hard limits on memory, CPU and processes. The runner moves
  every file in and out.
- **The signals of a recording kept in your storage never reach youSleep.**
  youSleep's servers never connect to your storage: browsers on your network
  and your runner reach it with requests youSleep signs.

The runner's [Data and security](../components/runner/data-and-security.md)
page states exactly what passes between a site and youSleep, which keys exist
and who holds each, and what the runner must reach.

## The portal in the browser

- **A content security policy on every page.** Scripts run only from the
  portal itself: each page's own inline scripts carry a one-time code, and
  no other inline script runs. The portal cannot be framed by another site.
- **Further headers**: `X-Content-Type-Options: nosniff`, a strict referrer
  policy, and a permissions policy that turns off the camera, microphone,
  location and payment features.
- **No advertising or product-analytics cookies.** The cookies are those the
  service needs: authentication, session and CSRF protection, and Cloudflare's
  security cookies.

## Backups and retention

- **The database** is backed up nightly to Cloudflare R2. A backup is kept for
  35 days and cannot be deleted for its first 30, whatever credential asks.
- **Analysis results and logs** are copied nightly to a separate backup
  bucket. A copy is removed 30 days after its original is deleted.
- **Recordings are not backed up by youSleep**: the people who upload them
  hold the originals.
- **Deleted data ages out of every backup within 35 days.** Data deleted in
  the portal leaves the live systems at once.
- **Storage the platform no longer tracks** is deleted by the storage itself
  within 60 days.

The [Privacy Policy](https://portal.yousleep.ai/legal/privacy-policy) (section 5)
sets out what is kept and for how long, and what survives an account's
deletion.

## Monitoring and logging

- **Self-hosted monitoring.** Metrics, logs and alerts run on youSleep's own
  server; alerts are sent by email. No third-party analytics or error-tracking
  service receives data from the platform.
- **What logs leave out.** A refused recording header is neither logged nor
  kept. An API key's secret is never logged, and the runner's log leaves out
  the signatures of the URLs it is given.
- **Cloudflare** processes request metadata, such as IP addresses, for traffic
  security and DDoS protection, on its own retention schedule.

## Vulnerabilities and incidents

- **Reporting a vulnerability.** The disclosure policy is at
  [yousleep.ai/security](https://yousleep.ai/security), and
  [security.txt](https://yousleep.ai/.well-known/security.txt) names the
  contact.
- **Dependencies** are watched for known vulnerabilities in every repository,
  and updated through reviewed changes. Container images are pinned by digest,
  and a runner runs an analysis's image only by its digest.
- **Every change** goes through a pull request and passes automated tests
  and checks before it is merged and deployed.
- **A breach.** If a breach affects personal data youSleep controls, youSleep
  notifies the supervisory authority and the people affected as the GDPR
  requires. If it affects the data in an organisation's recordings, youSleep
  notifies the organisation's owner without undue delay, so that the
  organisation can meet its own obligations
  ([Privacy Policy](https://portal.yousleep.ai/legal/privacy-policy), section 8).

Questions about this page: [contact@yousleep.ai](mailto:contact@yousleep.ai).
