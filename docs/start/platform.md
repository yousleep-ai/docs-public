# The platform in five minutes

Six nouns carry everything the platform does. Learn them and every page here reads
easily.

| Term | What it is |
|---|---|
| **Project** | The unit you organise and share. One per cohort or team. |
| **Study** | One subject's night (or longer recording), with its metadata: age, sex, lights off and on. |
| **Recording** | The EDF file of a study, with its channels as the file names them and as you name them. |
| **Analysis** | One run of one algorithm on one recording, with the parameters you chose. |
| **Events** | What an analysis produces: labelled intervals on the recording, sleep stages above all. |
| **Report** | Statistics and figures over a study, or over a project with its studies grouped and compared. |

## Three ways in

<div class="grid cards" markdown>

-   :material-monitor:{ .lg .middle } **The portal**

    ---

    Create a project, upload recordings, run analyses, inspect signals with events
    overlaid, and read the reports. Everything below is available here first.

-   :material-code-braces:{ .lg .middle } **The API and SDK**

    ---

    The same operations as HTTP requests, or as a typed Python client that adds
    end-to-end workflows. For pipelines, integrations and cohorts too large to click
    through.

-   :material-package-variant:{ .lg .middle } **Your own analysis**

    ---

    An algorithm packaged as a container that obeys a small contract runs on the
    platform beside the ones we host, on your recordings or on everyone's.

</div>

## What an analysis is

An analysis in the catalogue is two things: a **container image**, and a
**configuration** that says what it is, who wrote it, what it needs from a recording
and what it produces. The platform reads the configuration to offer the analysis, to
check a recording against its requirements, to reserve the resources a run needs and to
present the result; the image never sees any of that except through the manifest it is
given.

```mermaid
sequenceDiagram
    participant You
    participant Portal
    participant Container
    You->>Portal: submit an analysis on a recording
    Portal->>Portal: check requirements, reserve resources
    Portal->>Container: start with --manifest-file (recording, channels by index, parameters)
    Container->>Container: read the recording, compute
    Container-->>Portal: one events document
    Portal-->>You: events in the viewer, statistics in the report
```

The container runs confined: no network, a read-only filesystem, an unprivileged user,
the cores and memory the configuration declares, and a time bound. Everything it needs
arrives as a file; everything it produces is a file.

## What is hosted today

**U-Sleep**, a published and peer-reviewed sleep-staging algorithm, in the
configurations listed in the portal's catalogue. Each configuration says who developed
it, what it cites and under which licence it is offered, and the catalogue shows that
beside every result.

## What to read next

- Using the platform from code: the [SDK overview](../components/common/index.md), then the
  [guide](../components/common/sdk-guide.md).
- Connecting a system that submits on behalf of many subjects: the
  [integration guide](../components/common/integration-guide.md).
- Bringing an algorithm: [packaging an analysis](../components/common/analysis-authoring.md).
- The words on this page and a few more: the [glossary](glossary.md).
