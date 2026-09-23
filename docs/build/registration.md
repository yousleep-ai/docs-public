# Registration

An analysis enters the catalogue by registration. Today it is done with the platform
team; a self-service path is being designed. This page says what to send, what we do
with it, and what changes afterwards.

```mermaid
flowchart LR
    B[Build the image] --> C[Check it with yousleep-verify]
    C --> S[Send it to us]
    S --> R[We check it the same way,<br/>pin the digest, register it]
    R --> A[It appears in the catalogue]
```

## What to send

- **The image**, as a reference we can pull or as a `docker save` archive.
- **The configuration file**, the one `yousleep-verify` passed with.
- **The expected document**, `conformance/expected-events.json.gz`, recorded with
  `--record` on your final image.
- **The check's report**, `yousleep-verify --json`, so we see what you saw.

Send it through the portal's contact form under *Integrate an algorithm*, or to
[contact@yousleep.ai](mailto:contact@yousleep.ai). Nothing in it should be real
recordings: the check runs on a synthetic one, and so do we.

## What we do

1. Run the same check, on the platform's architecture (`linux/amd64`). It has to pass
   there; nothing else is registered.
2. Pin the digest the check reports, so the exact bytes that passed are what runs.
3. Copy the image into a registry the platform controls. A past analysis has to stay
   re-runnable, and that cannot depend on a registry we do not operate.
4. Set the fields that are ours: scheduling, availability, and the memory terms that
   scale with recording length, which we measure on long recordings.
5. Register the configuration. Its provenance and licence blocks are shown to users as
   written, so they are checked for completeness, not rewritten.

## What it looks like afterwards

Your analysis is listed in the catalogue with its provenance, what it cites and the
licence it is offered under, and every result it produces carries that attribution. Who
may run it, and on what terms, follows from the licence class in the configuration.

## What changes later

The check you run and the check we run are already the same tool. What is being
designed is the path between them: submitting the bundle above from the portal, a
quarantine where an image waits until the check has passed, and registration without a
person in the loop. When that exists, this page shrinks to one command.
