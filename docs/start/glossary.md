# Glossary

**Analysis**
: One run of one algorithm on one recording, with chosen parameters. It has a status,
  a result (an events document) and a place in the viewer and the reports.

**Analysis configuration**
: The file that registers an analysis with the platform: identification, purpose,
  provenance, licence, the image, resources, inputs, outputs and parameters. An author
  writes what only an author can know; the platform sets the rest at registration.

**API key**
: A credential for a machine, created on the account page once an administrator has
  enabled keys for the account, scoped and expiring. People sign in with an email and a
  password instead.

**Block**
: The unit of an events document: a run of consecutive events of equal duration
  sharing a channel set. A night of thirty-second epochs is one block.

**Channel**
: One signal of a recording, with the label the file gives it (`source_name`) and the
  name the user gives it (`name`). Analyses select channels by index, never by label.

**Conformance check**
: `yousleep-verify`: runs an image the way the platform does, on a synthetic recording,
  under the platform's confinement, and checks the document it writes.

**Configuration id**
: The slug of an analysis configuration, `u-sleep-research-v2` for example, derived
  from its name.

**Epoch**
: A fixed window of a recording, thirty seconds in sleep staging, that receives one
  label.

**Event**
: A labelled interval on a recording: a start, an end, a label from the EDF+
  vocabulary, the channels it applies to, and optionally a probability or a value.

**Events document**
: The one file an analysis writes: its events, encoded as blocks, plus optional
  provenance about the software that produced them.

**Hypnogram**
: The sequence of sleep stages over a night; in the platform, the staging events of a
  recording drawn against time.

**Manifest**
: The one file an analysis reads: where the recording is, which channels were selected
  and at which index, the parameters typed as declared, the resources the run has, and
  where to write the result.

**Organisation**
: The group an account belongs to. A project can be opened to every member or shared
  with named colleagues.

**Project**
: The container you organise and share: studies, their recordings, analyses and reports.

**Recording**
: An EDF file belonging to a study.

**Registration**
: How an analysis enters the catalogue: the platform team checks the image, pins its
  digest, copies it into the platform's registry and sets the fields that are theirs.

**Report**
: Statistics and figures over one study or over a project, with studies grouped and
  compared; exportable as figures and tables.

**Slot**
: A labelled input position an analysis declares, a left and a right EOG say, which the
  platform fills with one channel of the recording.

**Study**
: One subject's recording session and its metadata, inside a project.
