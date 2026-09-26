# Tutorial: package YASA

This tutorial packages an existing, published sleep-staging method for the platform:
[YASA](https://github.com/raphaelvallat/yasa), an open-source Python library by
Raphael Vallat whose automatic sleep staging is described in
[Vallat and Walker (2021)](https://doi.org/10.7554/eLife.70092). You generate a project,
write a short script around YASA, describe it in a configuration, build an image, and
check it with the same tool the platform runs. The same steps apply to any method you
can call from Python.

| | |
|---|---|
| **You will build** | A container image and a configuration that run YASA's sleep staging on the channels the platform selects, ready to send for registration |
| **Time** | About an hour. The first image build takes 5 to 10 minutes, longer under emulation on an arm64 machine such as a Mac with Apple silicon |
| **You need** | Docker, uv (or Python 3.11 or later with pip), and a text editor |
| **Tested with** | `yousleep-common` 33.4.0 and YASA 0.7.0 |
| **Finished project** | [`yasa-sleep-staging/`](https://github.com/yousleep-ai/analysis-example/tree/main/yasa-sleep-staging) in the public examples repository |

## Before you start

| Tool | What it is for | Install |
|---|---|---|
| Docker | Builds and runs the image, as the platform does | [Get Docker](https://docs.docker.com/get-started/get-docker/) |
| uv | Runs the youSleep tools and locks the dependencies | [Installing uv](https://docs.astral.sh/uv/getting-started/installation/) |
| `yousleep-common` | Provides `yousleep-init`, `yousleep-verify` and `yousleep-manifest` | Nothing to install with uv: `uvx` fetches it on first use |

We suggest [uv](https://docs.astral.sh/uv/). `uvx --from yousleep-common==33.4.0 …` runs
a youSleep tool at an exact version without installing anything, and `uv pip compile`
locks dependencies for the platform's architecture from any machine. The commands
below use it.

!!! note "Without uv"
    Install the tools in a virtual environment and call them directly:

    ```bash
    python3 -m venv .venv && source .venv/bin/activate
    pip install yousleep-common==33.4.0
    ```

    Then run `yousleep-init …` where the tutorial shows `uvx --from yousleep-common==33.4.0 yousleep-init …`.
    The generated Makefile finds the installed tools by itself. Step 4 shows how to lock
    the dependencies without uv.

| Resource | What it covers |
|---|---|
| [YASA source](https://github.com/raphaelvallat/yasa) and [documentation](https://raphaelvallat.com/yasa/) | The library being packaged |
| [Vallat and Walker, 2021](https://doi.org/10.7554/eLife.70092) | The paper describing YASA's sleep staging |
| [Packaging an analysis](../components/common/analysis-authoring.md) | The contract between a container and the platform, and the configuration reference |
| [Starting a project](../components/common/init.md) | `yousleep-init` and the Makefile it writes |
| [The manifest](../components/common/analysis-manifest.md) | What the container reads |
| [The events document](../components/common/events-document.md) | What the container writes |
| [Checking an image](../components/common/conformance.md) | `yousleep-verify`, its checks and the recorded output |
| [Registration](registration.md) | What to send, and what the platform team does with it |
| [`yousleep-common` on PyPI](https://pypi.org/project/yousleep-common/) | The package with the tools and the helpers |

## 1. Generate the project

```bash
uvx --from yousleep-common==33.4.0 yousleep-init yasa-staging --no-input \
    --name "YASA sleep staging" --developer "Raphael Vallat" \
    --source-url https://github.com/raphaelvallat/yasa --source-licence BSD-3-Clause \
    --channel EEG:1-1@81 --channel EOG:0-1@81 --channel EMG:0-1@81
cd yasa-staging
make verify
```

YASA scores from one EEG channel and uses an EOG and a chin EMG channel when a
recording has them. `EEG:1-1` asks for exactly one EEG channel; `EOG:0-1` and `EMG:0-1`
make the other two optional. `@81` is the lowest sampling rate each channel may have:
YASA requires more than 80 Hz.

`make verify` builds the image and checks it. It passes on the generated project before
you change anything, so every later failure comes from your own change.

## 2. Write the script

Replace the placeholder method in `analysis.py`. The finished script
([`analysis.py`](https://github.com/yousleep-ai/analysis-example/blob/main/yasa-sleep-staging/analysis.py))
does five things, in order:

1. **Selects the channels** the platform chose for this run, and checks each header
   label against the manifest, so the wrong file is refused before any signal is read.
2. **Loads only those channels.** MNE reads the file's header first; the script picks the
   selected channels and loads them. A polysomnogram carries dozens of channels, and
   loading all of them makes memory grow with the file rather than with the selection.
3. **Passes subject values** only when the study records an age YASA accepts
   (0 < age < 120) and a sex of `male` or `female`. YASA's classifiers with demographics
   were trained on a binary sex and need both values; in every other case the script
   runs the classifier without demographics.
4. **Names the classifier generation** it runs (0.5.0). YASA otherwise loads the newest
   classifiers it ships, so a YASA release could change your results; see
   [Versions](../components/common/analysis-authoring.md#versions).
5. **Writes one event per 30-second epoch**: the predicted stage with its probability, or
   every stage's probability when the user asks for it (step 3).

```python
raw = mne.io.read_raw_edf(manifest.inputs.recording.path, preload=False, verbose="ERROR")
selected = select_channels(manifest, header_labels=raw.ch_names)  # refuses the wrong file
header = {}
for channel in selected:  # at most one EEG, one EOG and one EMG
    header.setdefault(channel.type, raw.ch_names[channel.index])
raw.pick([channel.index for channel in selected])
raw.load_data()  # reads the selected channels only

subject = manifest.inputs.metadata.subject if manifest.inputs.metadata else None
metadata = None
if subject and subject.age and 0 < subject.age < 120 and subject.sex in ("male", "female"):
    metadata = {"age": subject.age, "male": subject.sex == "male"}

staging = yasa.SleepStaging(raw, eeg_name=header["EEG"], eog_name=header.get("EOG"),
                            emg_name=header.get("EMG"), metadata=metadata)
model = "clf_eeg" + "".join(
    suffix
    for suffix, used in (("+eog", "EOG" in header), ("+emg", "EMG" in header),
                         ("+demo", metadata is not None))
    if used
)
classifier = Path(yasa.__file__).parent / "classifiers" / f"{model}_lgb_{CLASSIFIERS}.joblib"
hypnogram = staging.predict(path_to_model=str(classifier))  # stages per epoch, with .proba

store_all = bool(manifest.parameters.get("store-all-probabilities", False))
channels = [channel.name for channel in selected]
events = []
for i, stage in enumerate(hypnogram.hypno):
    probabilities = hypnogram.proba.iloc[i]
    for label in probabilities.index if store_all else [stage]:
        events.append(
            Event(start_time_ms=i * 30_000, end_time_ms=(i + 1) * 30_000,
                  label=STAGES[label], probability=float(probabilities[label]),
                  channels=channels)
        )

save_event_blocks(output, events, device={"software": "yasa", "version": yasa.__version__})
```

`CLASSIFIERS` is `"0.5.0"`, and `STAGES` maps YASA's stage names (`WAKE`, `N1`, `N2`,
`N3`, `REM`) to the platform's labels. The script uses four helpers from
`yousleep-common`:

| Helper | What it does |
|---|---|
| [`load_manifest`][yousleep_common.utils.manifest.load_manifest] | Reads and validates the manifest the platform writes |
| [`select_channels`][yousleep_common.utils.analysis.select_channels] | Returns the selected channels by index, and refuses a file whose labels differ |
| [`Event`][yousleep_common.models.events.Event] | One scored interval, with its label, probability and channels |
| [`save_event_blocks`][yousleep_common.utils.event_blocks.save_event_blocks] | Writes the events document, with an optional `device` block |

The script sets no thread limits: the platform sets the thread-count variables
(`OMP_NUM_THREADS` and the others) to the analysis's cores, and the libraries YASA uses
read them.

## 3. Describe it in the configuration

`yasa-staging.yaml` already holds the channel types and the provenance from the
command's flags. Add the subject values, the five output labels, the full-probability
parameter and the citation. The parts you edit look like this
([full file](https://github.com/yousleep-ai/analysis-example/blob/main/yasa-sleep-staging/yasa-staging.yaml)):

```yaml
provenance:
  origin: "third-party"
  developer: "Raphael Vallat"
  source_url: "https://github.com/raphaelvallat/yasa"
  source_licence: "BSD-3-Clause"
  data_rights:
    basis: "pending"
evidence:
  publications:
    - title: "An open-source, high-performance tool for automated sleep staging"
      url: "https://doi.org/10.7554/eLife.70092"
      name: "Vallat and Walker, 2021"
      by: "developer"
resources:
  cpu_cores: 1
  base_system_memory_mib: 600   # set from the peak in step 5
inputs:
  recording:
    channel_types:
      - type: "EEG"
        number: {range: [1, 1]}
        requirements: {sampling_rate: {range: [81, "+inf"]}}
      - type: "EOG"
        number: {range: [0, 1]}
        requirements: {sampling_rate: {range: [81, "+inf"]}}
      - type: "EMG"
        number: {range: [0, 1]}
        requirements: {sampling_rate: {range: [81, "+inf"]}}
    minimum_duration_seconds: 300   # YASA recommends at least five minutes
  metadata: ["age", "sex"]
outputs:
  supports_full_probabilistic_output: true
  events:
    - {label: "Sleep stage W", probabilistic: true}
    - {label: "Sleep stage N1", probabilistic: true}
    - {label: "Sleep stage N2", probabilistic: true}
    - {label: "Sleep stage N3", probabilistic: true}
    - {label: "Sleep stage R", probabilistic: true}
parameters:
  custom:
    - name: "Store all probabilities"
      key: "store-all-probabilities"
      type: bool
      default: false
      choices: [false, true]
      description: "Write every stage's probability per epoch instead of the predicted stage only."
```

- **Sampling rates.** YASA requires the recording to be sampled above 80 Hz, then
  resamples it to 100 Hz and filters it to 0.4-30 Hz. The bounds in a configuration are
  inclusive, so every channel requires at least 81 Hz. A channel below that, such as the
  1 Hz chin signal some recordings carry, is left out, and the run continues without it.
- **Full probabilities.** `supports_full_probabilistic_output` tells the platform the
  analysis can write every stage's probability. A user asks for it per run through the
  `store-all-probabilities` parameter, which the script reads from the manifest.
- **Data rights.** `data_rights: pending` records that no signed Analysis Declaration is
  on record. It grants no right to use the software; that follows from the software's
  licence, named in `source_licence`. [Registration](registration.md) describes what
  offering an analysis beyond its organisation requires.

Run `make check` to validate the file.

## 4. Pin the dependencies and write the Dockerfile

List the direct dependencies in `requirements.in`, and compile them into
`requirements.txt` for the platform, Python 3.12 on linux/amd64. The lock pins 51
packages.

```text
yousleep-common==33.4.0
mne==1.13.2
yasa==0.7.0
```

```bash
uv pip compile requirements.in --python-version 3.12 \
    --python-platform x86_64-manylinux_2_28 -o requirements.txt
```

!!! note "Without uv"
    Compile the lock inside a linux/amd64 container, so it matches the platform:

    ```bash
    docker run --rm --platform linux/amd64 -v "$PWD:/w" -w /w python:3.12-slim \
        sh -c "pip install pip-tools && pip-compile --strip-extras -o requirements.txt requirements.in"
    ```

Then write the Dockerfile
([full file](https://github.com/yousleep-ai/analysis-example/blob/main/yasa-sleep-staging/Dockerfile)):

```dockerfile
FROM python:3.12-slim
WORKDIR /app
LABEL org.opencontainers.image.source="https://github.com/raphaelvallat/yasa"
LABEL org.opencontainers.image.licenses="BSD-3-Clause"
RUN apt-get update \
 && apt-get install -y --no-install-recommends libgomp1 \
 && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir --no-deps -r requirements.txt \
 && python -c "import mne, yasa, lightgbm, yousleep_common"
ENV MPLCONFIGDIR=/tmp/matplotlib
COPY analysis.py .
ENTRYPOINT ["python3", "analysis.py"]
```

- **`libgomp1`.** LightGBM, which runs YASA's classifier, needs the OpenMP runtime, which
  the slim base image does not carry. Without it the import fails with
  `OSError: libgomp.so.1: cannot open shared object file`.
- **The import after installing** makes a missing system library fail the build rather
  than the first run.
- **`MPLCONFIGDIR`.** The platform runs the image with a read-only root filesystem and a
  writable `/tmp`. Matplotlib, which YASA imports, keeps a cache in the home directory
  and warns without a writable one.

## 5. Build and check the image

```bash
make verify
```

`make build` (part of `make verify`) builds for linux/amd64, the platform's architecture,
also on an arm64 machine. `yousleep-verify` then runs the image the way the platform
does, on a five-minute synthetic recording: one channel of each declared type at its
lowest rate, 20 other signals the manifest does not select, and a subject aged 40 and
male.

| Check | What it tests | YASA |
|---|---|---|
| `configuration` | The file validates, with provenance and at least one output label | Pass, 5 labels |
| `data-rights` | A signed declaration is referenced (warning only) | Warning: `pending` |
| `image` | The image is present, with its platform | Pass, linux/amd64 |
| `image-labels` | The licence and source labels are set (warning only) | Pass |
| `run` | The container exits 0 under the platform's confinement; its peak memory | Pass, peak 425 MiB of 600 MiB |
| `document` | A readable events document is at the output path | Pass, 10 events |
| `non-empty` | The document holds at least one event (warning only) | Pass |
| `labels` | Every label is declared under `outputs.events` | Pass |
| `channels` | Every channel is one the manifest named | Pass |
| `values` | Probabilities appear only on labels declared `probabilistic` | Pass |
| `times` | Every event ends within the recording | Pass |
| `determinism` | Two runs give the same events | Pass |
| `required-only` | A run with only the required channels and no subject values succeeds | Pass, EEG only |
| `expected` | The events match the recorded document (step 6) | Pass |

The `run` check reports the container's peak memory against the memory the
configuration claims, and warns when the peak leaves less than 30% headroom. YASA peaks
at 425 MiB here, so set `base_system_memory_mib` to 600; the generated 512 MiB is too
low.

## 6. Record the expected output

```bash
make record   # writes conformance/expected-events.json.gz
make verify   # PASS expected: matches the recorded document
```

Commit the recorded document with the project. Every later `make verify` compares the
image's output with it, so a change that alters results shows. Record it from the
linux/amd64 build: floating-point results differ between architectures, and an arm64
build does not match an amd64 recording.

## 7. Run it on a real recording

Download a night from the public
[Sleep-EDF Expanded](https://physionet.org/content/sleep-edfx/1.0.0/) database, for
example [`SC4001E0-PSG.edf`](https://physionet.org/files/sleep-edfx/1.0.0/sleep-cassette/SC4001E0-PSG.edf)
(48 MB, 22.1 hours at 100 Hz), and run the image on it:

```bash
make run RECORDING=SC4001E0-PSG.edf MANIFEST_ARGS="--age 40 --sex male"
```

`make run` selects channels as the platform would and says what it left out: here it
keeps `EEG Fpz-Cz` and `EOG horizontal`, leaves out the second EEG channel because the
analysis takes one, and leaves out the 1 Hz chin signal. The image scores the night's
2650 epochs in about 20 seconds under emulation and writes `output-events.json.gz`.

- `CHANNELS="EEG Fpz-Cz,EOG horizontal"` chooses the channels yourself.
- `MANIFEST_ARGS="--param store-all-probabilities=true"` writes every stage's
  probability: 13,250 events, five per epoch.

## 8. Send it for registration

Send the image, the configuration, the recorded document and the check's report as
[Registration](registration.md) describes. The platform team runs the same check on
linux/amd64, pins the image digest, and measures the memory that grows with recording
length with `yousleep-verify --meter`, on synthetic recordings of up to 48 hours at 128
to 512 Hz. You do not need to measure it yourself.

For YASA the measured claim is `800 + h·kHz·(450 + 90·n)` MiB, where `h` is the
recording length in hours, `kHz` the highest sampling rate of the selected channels in
kHz, and `n` the number of selected channels. YASA resamples each channel with an FFT
that pads the signal to the next power of two, so its memory grows by about 100 to
200 MiB per hour at 256 to 512 Hz and steps with recording length. At every point the
meter measured, the claim leaves at least 30% headroom above the peak.

## What you have built

You have a project that runs a published method on the platform's terms: it loads only
the channels it is given, declares what it needs and produces, passes the check the
platform runs, and records its own output so that a change to it shows. The same steps
package any method you can call from Python; start from
[Starting a project](../components/common/init.md) and the
[finished YASA project](https://github.com/yousleep-ai/analysis-example/tree/main/yasa-sleep-staging).
