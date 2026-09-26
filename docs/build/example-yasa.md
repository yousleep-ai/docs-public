# Example: packaging YASA

[YASA](https://github.com/raphaelvallat/yasa) is an open-source Python library for
sleep analysis by Raphael Vallat, published under the BSD-3-Clause licence. Its
automatic sleep staging is described in
[Vallat and Walker (2021)](https://doi.org/10.7554/eLife.70092). This page packages it
for the platform with `yousleep-init`, `yousleep-verify` and `yousleep-manifest`, step
by step, as an author outside youSleep would. It is tested with `yousleep-common` 33.4.0
and YASA 0.7.0.

## 1. Generate the project

```bash
uvx --from yousleep-common==33.4.0 yousleep-init yasa-staging --no-input \
    --name "YASA sleep staging" --developer "Raphael Vallat" \
    --source-url https://github.com/raphaelvallat/yasa --source-licence BSD-3-Clause \
    --channel EEG:1-1@100 --channel EOG:0-1@100 --channel EMG:0-1@100
cd yasa-staging
make verify
```

YASA scores from one EEG channel and uses an EOG and a chin EMG channel when they are
present. The EEG is therefore declared with a count of exactly one, and the other two
types with a count from zero to one, which makes them optional. The generated project
passes `make verify` before any change; the sampling rates are set in step 3.

## 2. Write the script

`analysis.py` replaces the generated placeholder loop. It does the following, in order:

1. **Selects the channels** the manifest names, and checks each header label against
   the manifest, so the wrong file is refused before any signal is read.
2. **Loads only those channels.** MNE reads the header first; the selected channels are
   picked and then loaded. A polysomnogram carries dozens of channels, and memory would
   otherwise grow with the file rather than with the selection.
3. **Passes subject values** only when the study records an age YASA accepts
   (0 < age < 120) and a sex of `male` or `female`. YASA's classifiers with
   demographics were trained on a binary sex and need both values; every other case
   runs the classifier without demographics.
4. **Names the classifier generation** it runs (0.5.0). YASA otherwise picks the newest
   classifiers it ships, so a release adding a generation would change results
   without a new analysis id; see [Versions](../components/common/analysis-authoring.md#versions).
5. **Writes one event per 30-second epoch**: the predicted stage with its probability,
   or every stage's probability when the user asks for it (step 3).

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
`N3`, `REM`) to the platform's labels. The helpers the script uses are:

| Helper | What it does |
|---|---|
| [`load_manifest`][yousleep_common.utils.manifest.load_manifest] | Reads and validates the manifest the platform writes |
| [`select_channels`][yousleep_common.utils.analysis.select_channels] | Returns the selected channels by index, and refuses a file whose labels differ |
| [`Event`][yousleep_common.models.events.Event] | One scored interval, with its label, probability and channels |
| [`save_event_blocks`][yousleep_common.utils.event_blocks.save_event_blocks] | Writes the events document, with the optional `device` block |

The platform sets the thread-count variables (`OMP_NUM_THREADS` and the others) to the
analysis's cores, and the libraries YASA uses read them, so the script sets no thread
limits itself.

## 3. Write the configuration

The generated configuration already holds the channel types and the provenance from the
command's flags. It is edited to set the sampling rates, add the subject values, the
five output labels, the full-probability parameter and the citation:

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
  base_system_memory_mib: 600   # step 6
inputs:
  recording:
    channel_types:
      - type: "EEG"
        number: {range: [1, 1]}
        requirements: {sampling_rate: {range: [81, "+inf"]}}
      - type: "EOG"
        number: {range: [0, 1]}
        requirements: {sampling_rate: {range: [60, "+inf"]}}
      - type: "EMG"
        number: {range: [0, 1]}
        requirements: {sampling_rate: {range: [60, "+inf"]}}
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

- **Sampling rates.** YASA refuses a recording at 80 Hz or below and resamples every
  channel to 100 Hz; the bounds are inclusive, so the EEG requires at least 81 Hz. It
  then filters each channel to 0.4-30 Hz, a band that a channel sampled below 60 Hz
  cannot carry, so the EOG and EMG require at least 60 Hz. A 1 Hz chin EMG, as some
  recordings carry, is left out and the run continues without it.
- **Full probabilities.** `supports_full_probabilistic_output` tells the platform the
  analysis can write every stage's probability. The user asks for it per run through
  the `store-all-probabilities` parameter, which the script reads from the manifest.
- **Data rights.** `data_rights: pending` records that no signed Analysis Declaration is
  on record. It grants no right to use the software; that follows from the software's
  licence, named in `source_licence`. [Registration](registration.md) describes what
  offering an analysis beyond its organisation requires.

`make check` validates the file.

## 4. Lock the dependencies and write the Dockerfile

`requirements.in` names the direct dependencies, and `requirements.txt` is compiled
from it for the platform. It pins 51 packages:

```text
yousleep-common==33.4.0
mne==1.13.2
yasa==0.7.0
```

```bash
uv pip compile requirements.in --python-version 3.12 \
    --python-platform x86_64-manylinux_2_28 -o requirements.txt
```

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

- **`libgomp1`.** LightGBM, which runs YASA's classifier, links against the OpenMP
  runtime, and the slim base image does not carry it. Without it the build-time import
  fails with `OSError: libgomp.so.1: cannot open shared object file`.
- **The build-time import** makes a missing system library fail the build rather than
  the first run.
- **`MPLCONFIGDIR`.** The platform runs the image with a read-only root filesystem and a
  writable `/tmp`. Matplotlib, which YASA imports, keeps a cache in the home directory
  and warns without a writable one.

## 5. Build for linux/amd64

`make build` runs `docker build --platform linux/amd64`, the platform's architecture. On
an arm64 machine the image builds and runs under emulation. The expected output is
recorded from this build: floating-point results differ between architectures, and an
arm64 build differs from the amd64 recording in 8 of 10 probabilities.

## 6. Verify

`make verify` builds the image and runs `yousleep-verify` on it. The fixture is a
five-minute synthetic recording with one channel of each declared type at its minimum
rate, 20 signals the manifest does not select, and subject values of age 40 and sex
`male`.

| Check | What it tests | YASA |
|---|---|---|
| `configuration` | The file validates, with provenance and at least one output label | Pass, 5 labels |
| `data-rights` | A signed declaration is referenced (warning only) | Warning: `pending` |
| `image` | The image is present, with its platform | Pass, linux/amd64 |
| `image-labels` | The licence and source labels are set (warning only) | Pass |
| `run` | The container exits 0 under the platform's confinement, and its peak memory | Pass, about 10 s under emulation, peak 425 MiB of 600 MiB |
| `document` | A readable events document is at the output path | Pass, 10 events in 1 block |
| `non-empty` | The document holds at least one event (warning only) | Pass |
| `labels` | Every label is declared under `outputs.events` | Pass |
| `channels` | Every channel is one the manifest named | Pass |
| `values` | Probabilities appear only on labels declared `probabilistic` | Pass |
| `times` | Every event ends within the recording | Pass |
| `determinism` | Two runs give the same events | Pass |
| `required-only` | A run with only the required channels and no subject values succeeds | Pass, EEG only |
| `expected` | The events match the recorded document (step 7) | Pass |

The `run` check reports the container's peak memory against the memory the
configuration claims. With the generated base of 512 MiB, YASA's peak of 425 MiB left
less than 30% headroom and the check warned, so `base_system_memory_mib` is set to 600.

## 7. Record the expected output

```bash
make record   # writes conformance/expected-events.json.gz
make verify   # PASS expected: matches the recorded document (10 events)
```

The recorded document is committed beside the configuration. A rebuild is compared
against it, so a change to the image that alters results is visible.

## 8. Run it on a real recording

```bash
make run RECORDING=SC4001E0-PSG.edf MANIFEST_ARGS="--age 40 --sex male"
```

With no channel chosen, the manifest tool selects channels as the platform does and
says what it left out: on a Sleep-EDF night from PhysioNet (22.1 hours, 100 Hz) it keeps
`EEG Fpz-Cz` and `EOG horizontal`, leaves out the second EEG because the analysis takes
one, and leaves out the 1 Hz chin EMG. The image scores the night's 2650 epochs in about
20 seconds under emulation. With `MANIFEST_ARGS="--param store-all-probabilities=true"`,
it writes 13,250 events, one per stage per epoch. `CHANNELS` chooses the channels
explicitly.

## 9. Register the analysis

The analysis is sent and registered as [Registration](registration.md) describes. The
platform team runs the same check on linux/amd64, pins the image digest, and measures
the memory that scales with recording length with `yousleep-verify --meter`, on
synthetic recordings of up to 48 hours at 128 to 512 Hz.

YASA resamples each channel with MNE's FFT method, which pads the signal to the next
power of two, so memory grows by about 100 to 200 MiB per hour at 256 to 512 Hz and
steps with recording length. The claim set at registration is

`800 + h·kHz·(450 + 90·n)` MiB

where `h` is the recording length in hours, `kHz` the highest native sampling rate of
the selected channels in kHz, and `n` the number of selected channels. The meter
confirms it:

| Hours | Channels | Rate | Measured peak (MiB) | Claim (MiB) | Headroom |
|---|---|---|---|---|---|
| 8 | 1 | 256 Hz | 1014 | 1906 | 88% |
| 24 | 3 | 256 Hz | 3189 | 5224 | 64% |
| 18.4 | 3 | 128 Hz | 1796 | 2495 | 39% |
| 24 | 3 | 512 Hz | 5755 | 9648 | 68% |

The 18.4-hour recording at 128 Hz is just past a power-of-two boundary, where the
padding is largest; every measured point has at least 30% headroom.

## Conclusion

Packaging YASA takes a script of under 100 lines, a configuration and a Dockerfile with
one system library. Every step is checked by the tool the platform runs: the author
sets the cores and the base memory from the reported peak, and the platform measures
the memory that scales with recording length. Packaging YASA also led to fixes in the
tools, which are in the versions this page uses.

## Get started

| Resource | What it covers |
|---|---|
| [Packaging an analysis](../components/common/analysis-authoring.md) | The contract, the three files and the configuration reference |
| [Starting a project](../components/common/init.md) | `yousleep-init` and the Makefile targets |
| [The manifest](../components/common/analysis-manifest.md) | What the container reads, and `yousleep-manifest` for hand runs |
| [The events document](../components/common/events-document.md) | What the container writes |
| [Checking an image](../components/common/conformance.md) | `yousleep-verify`, its checks, the recorded output and the meter |
| [Registration](registration.md) | What to send and what the platform team does with it |
| [analysis-example](https://github.com/yousleep-ai/analysis-example) | Two complete example analyses |
| [YASA](https://github.com/raphaelvallat/yasa) | The upstream library and its documentation |
| [Vallat and Walker, 2021](https://doi.org/10.7554/eLife.70092) | The paper describing YASA's sleep staging |
| [`yousleep-common` on PyPI](https://pypi.org/project/yousleep-common/) | The package with `yousleep-init`, `yousleep-verify` and `yousleep-manifest` |
