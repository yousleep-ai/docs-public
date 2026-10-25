# Sub-processors

The service providers below process personal data on youSleep's instructions,
to provide their part of the service, and not for their own purposes. For the
data in an organisation's recordings, they are youSleep's sub-processors on the
organisation's behalf (section 4.5.7 of the
[Terms and Conditions](https://portal.yousleep.ai/legal/terms-and-conditions)).
This list follows section 11 of the
[Privacy Policy](https://portal.yousleep.ai/legal/privacy-policy).

| Provider | What it does | Personal data it processes | Where |
|---|---|---|---|
| **Cloudflare** | Object storage (Cloudflare R2), where uploaded recordings and analysis results are stored. Also DNS, DDoS protection and secure transport: every request to youSleep passes through Cloudflare | The recordings and results stored in R2. Request metadata, such as IP addresses, approximate location and access times | R2 storage in the EU region. Transport on Cloudflare's global network |
| **SMTP2GO** | Delivery of transactional email: verification, password resets, access and account notices | The recipient's email address, to deliver each email | See [International transfers](#international-transfers) |
| **Google (Google Workspace)** | Hosts the mailbox that receives what is sent to `contact@yousleep.ai`, and replies to youSleep's emails | What people send to that address | See [International transfers](#international-transfers) |
| **Stripe** | Integrated for a possible future paid offering, but **not active**: it processes no personal data | None | — |

## International transfers

Recordings and analysis results are stored in the European Union (Cloudflare
R2, EU region). Some providers operate globally distributed infrastructure.
Where personal data is processed outside the European Economic Area, youSleep
relies on appropriate safeguards under the GDPR, in particular the European
Commission's Standard Contractual Clauses or an applicable adequacy decision
(Privacy Policy, section 12).

## Not sub-processors

- **An organisation's own storage and runners** are operated by the
  organisation, not by youSleep or its providers.
- **GitHub** hosts youSleep's source code and container images. It receives no
  personal data from the service.

## Changes

A change to this list is made in the Privacy Policy first, which every user is
asked to accept again before continuing to use the service.
