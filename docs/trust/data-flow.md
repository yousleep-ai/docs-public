# How data flows

A project's recordings are kept in youSleep's storage or, for an organisation
with its own site, in the organisation's storage. This is fixed when the project
is created.

## What is where

| Data | Where |
|---|---|
| Account data | Database, youSleep's server in Denmark |
| Recordings (pseudonymised EDF) | Cloudflare R2, EU region, or the organisation's storage |
| Cleaned headers, study data, annotations, run records | Database |
| Analysis results | Cloudflare R2, EU region |
| Backups: database, results, logs | Cloudflare R2, EU region |
| Messages to staff | Database, with the account. Email to contact@yousleep.ai: Google Workspace |

## Pseudonymisation

- The portal and the SDK clean the header of the copy being uploaded, on the
  user's computer: the patient and recording identification fields are blanked.
- Per project, the start date can also be removed, and file names replaced by
  codes youSleep issues. youSleep keeps no map of codes to names.
- The server checks every header twice, at registration and after upload, and
  deletes a file that fails. A refused header is not logged or kept.
- Signals, annotation text inside the file, and entered study data are not
  changed. They must not identify anyone.

## A recording in youSleep's storage

1. The user's computer cleans the header and registers the recording.
2. The file goes straight from the user's computer to storage, by a signed URL.
3. youSleep's runners read it once to check it and take its annotations
   (times and labels, and the text of any that match no known label).
4. Each analysis runs on youSleep's server in a container with no network. The
   result goes to storage; the runner deletes its copy of the recording.
5. The portal reads recordings from storage by signed URLs, a part at a time.

## A recording kept at a site

The signals never reach youSleep. youSleep keeps the cleaned header, the name or
code, size and fingerprints, the annotations' times and labels (no free text),
and the results. See the runner's
[Data and security](../components/runner/data-and-security.md).

## Deletion

- Deleted in the portal: removed from live systems at once, and from storage. A
  file registered in place at a site is only unregistered.
- An organisation over its free storage for 30 days loses its most recent
  uploads until it is within its limit.
- Deleted accounts, and accounts unused for 24 months, lose their account
  record, conversations, private projects and results.
- Deleted data leaves every backup within 35 days.
- Objects youSleep no longer tracks are deleted by the storage within 60 days.

What survives an account's deletion:
[Privacy Policy](https://portal.yousleep.ai/legal/privacy-policy), section 5.4.

## Who can read a recording

The user who uploaded it, members it was shared with in the organisation, and
youSleep's staff. Recordings are not sold, used for advertising, or used to train
models.
