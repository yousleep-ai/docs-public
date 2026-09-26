# Example: packaging YASA

[YASA](https://github.com/raphaelvallat/yasa) is an open-source Python library for
sleep analysis by Raphael Vallat, published under the BSD-3-Clause licence, whose
automatic sleep staging is described in
[Vallat and Walker (2021)](https://doi.org/10.7554/eLife.70092). It was chosen
because it is an existing, published algorithm that was not written for this
platform, and it was packaged by following this site as an outside author would.
YASA is registered and offered privately to one organisation; as a third-party
analysis, it can be offered to every user only once its developers' signed Analysis
Declaration is on record.

!!! note "Versions"
    The commands and outputs are from a run with `yousleep-common` 33.0.0 and YASA
    0.7.0 on an arm64 machine. Most problems were met in two earlier attempts with
    older versions, or in the first production run; the table at the end records
    which of them the run with 33.0.0 still met. The three it met are fixed in
    33.1.0.

## 1. Generate the project

```bash
uvx --from yousleep-common==33.0.0 yousleep-init yasa-staging --no-input \
    --name "YASA sleep staging" --developer "Raphael Vallat" \
    --source-url https://github.com/raphaelvallat/yasa --source-licence BSD-3-Clause \
    --channel EEG:1-1@100 --channel EOG:0-1@100 --channel EMG:0-1@100
cd yasa-staging
make verify
```

YASA scores from one EEG channel and uses an EOG and a chin EMG channel when they are
present. The EEG is therefore declared with a count of exactly one, and the other two
types with a count from zero to one.

- **Checked.** `make verify` passes on the generated project before any change,
  including the `required-only` run with the EEG alone.
- **Problem.** With 30.0.0, `yousleep-init` refused an optional type:
  `channel: expected 1 <= MIN <= MAX, got 'EOG:0-1@50'`. It also derived a name that
  the configuration model refuses from a short directory name, and an empty slug
  from `.`.
- **Resolution.** Fixed in 30.0.1: a minimum of 0 makes a type optional.

## 2. Write the script

The script replaces the generated placeholder loop in `analysis.py`. It does the
following:

- selects the channels with `select_channels`, which checks each header label
  against the manifest, and keeps one EEG and, when selected, one EOG and one EMG;
- picks the selected channels before any signal is read and then loads them, because
  YASA resamples the data to 100 Hz and needs it in memory;
- passes subject values to YASA only when the study records an age and a sex of
  `male` or `female`. YASA's classifiers with demographics need both values and were
  trained on a binary sex; given an age alone, YASA 0.7.0 raises a `ValueError`;
- sizes native thread pools to `manifest.resources.cpus`;
- names the generation of YASA's trained classifiers it runs (0.5.0), because YASA
  otherwise picks the newest it ships and a release adding one would change results
  without a new analysis id (see [Versions](../components/common/analysis-authoring.md#versions));
- writes one event per 30-second epoch: the predicted stage, mapped from YASA's names
  (`WAKE`, `N1`, `N2`, `N3`, `REM`) to the platform's labels, with its probability.

The core of the script is:

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

with threadpool_limits(limits=manifest.resources.cpus):
    staging = yasa.SleepStaging(raw, eeg_name=header["EEG"], eog_name=header.get("EOG"),
                                emg_name=header.get("EMG"), metadata=metadata)
    hypnogram = staging.predict()  # stages per epoch, with .proba

events = [
    Event(start_time_ms=i * 30_000, end_time_ms=(i + 1) * 30_000, label=STAGES[stage],
          probability=float(hypnogram.proba.iloc[i][stage]),
          channels=[channel.name for channel in selected])
    for i, stage in enumerate(hypnogram.hypno)
]
```

`STAGES` maps `WAKE` to `Sleep stage W`, `REM` to `Sleep stage R`, and each of `N1`,
`N2` and `N3` to its `Sleep stage` label.

- **Problems.** `select_channels` returned a channel's type as `ChannelType.EEG`
  instead of `EEG`, a tool bug. The first script read `.value` from the type, which
  arrives as plain text, and assumed YASA's classes were named `W` and `R`. It also
  passed a sex recorded as `other` or `unknown` to YASA as female, and read every
  channel of the file before keeping the selected ones (step 10).
- **Resolution.** `select_channels` returns the plain type since 30.0.2, and the
  manifest page states that enumerated fields are plain text. The script uses YASA's
  names, uses demographics only for a binary sex with an age, and loads only the
  selected channels.

## 3. Write the configuration

The generated configuration already holds the channel types and the provenance from
the command's flags. It is edited to add the subject values, the five output labels,
the citation and a minimum duration, and to remove the example parameter. The
`data_rights` basis stays `pending`, which allows building, verifying and private use.

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
inputs:
  recording:
    channel_types:
      - type: "EEG"
        number: {range: [1, 1]}
        requirements: {sampling_rate: {range: [100, "+inf"]}}
      - type: "EOG"
        number: {range: [0, 1]}
        requirements: {sampling_rate: {range: [100, "+inf"]}}
      - type: "EMG"
        number: {range: [0, 1]}
        requirements: {sampling_rate: {range: [100, "+inf"]}}
    minimum_duration_seconds: 300   # YASA recommends at least five minutes
  metadata: ["age", "sex"]
outputs:
  events:
    - {label: "Sleep stage W", probabilistic: true}
    - {label: "Sleep stage N1", probabilistic: true}
    - {label: "Sleep stage N2", probabilistic: true}
    - {label: "Sleep stage N3", probabilistic: true}
    - {label: "Sleep stage R", probabilistic: true}
```

YASA requires a sampling rate above 80 Hz and resamples to 100 Hz, so each type
requires at least 100 Hz.

- **Checked.** `make check` validates the file and reports `data-rights: pending` as a
  warning.
- **Problems.** A citation `name` longer than 32 characters was refused, and the guide
  did not state the bound. In 33.0.0, the commented `evidence` example in the
  generated configuration, a list with a `citation` key, fails validation.
- **Resolution.** The guide states the bound and the `publications` shape since
  31.0.0, and the commented example validates once uncommented since 33.1.0.

## 4. Lock the dependencies

`requirements.in` names the direct dependencies: `yousleep-common==33.0.0`,
`mne==1.13.2`, `yasa==0.7.0` and `threadpoolctl`. `requirements.txt` is compiled from
it for the platform and pins 51 packages:

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

- **Problem.** LightGBM, which runs YASA's classifier, links against the OpenMP
  runtime, and the slim base image does not carry it. Without `libgomp1`, the
  build-time import fails with
  `OSError: libgomp.so.1: cannot open shared object file: No such file or directory`.
- **Resolution.** The Dockerfile installs `libgomp1`, and the guide names this library
  since 31.0.0. The generated Dockerfile has no build-time import, so it is added as
  the guide shows.
- **Notes.** `MPLCONFIGDIR` points matplotlib, which YASA imports, at the writable
  `/tmp`; without it, matplotlib warns that the home directory is not writable. YASA
  0.7.0's classifiers were saved with scikit-learn 0.24.2, so loading them with the
  locked 1.9.1 prints an `InconsistentVersionWarning`; the run completes.

## 5. Build for linux/amd64

`make build` runs `docker build --platform linux/amd64`, the platform's architecture.
On an arm64 machine the image builds and runs under emulation.

- **Problem.** The first expected document was recorded from an arm64 build. The
  linux/amd64 build in continuous integration never matched it: the labels were
  identical and every probability differed, by up to 0.03.
- **Resolution.** Since 31.1.0, the generated Makefile builds for linux/amd64 by
  default, and the `expected` check names the first differing event and field and
  tolerates differences below 1e-6. An arm64 build compared with the amd64 recording
  still fails, with 8 of 10 events differing in probability.

## 6. Verify

`make verify` builds the image and runs `yousleep-verify` on it. The fixture is a
five-minute synthetic recording with one channel of each declared type at 100 Hz,
and subject values of age 40 and sex `male`.

| Check | What it tests | YASA |
|---|---|---|
| `configuration` | The file validates, with provenance and at least one output label | Pass, 5 labels |
| `data-rights` | A signed declaration is referenced (warning only) | Warning: `pending` |
| `image` | The image is present, with its platform and digest | Pass, linux/amd64 |
| `image-labels` | The licence and source labels are set (warning only) | Pass |
| `run` | The container exits 0 under the platform's confinement | Pass, about 10 s under emulation |
| `document` | A readable events document is at the output path | Pass, 10 events in 1 block |
| `non-empty` | The document holds at least one event (warning only) | Pass |
| `labels` | Every label is declared under `outputs.events` | Pass |
| `channels` | Every channel is one the manifest named | Pass |
| `values` | Probabilities appear only on labels declared `probabilistic` | Pass |
| `times` | Every event ends within the recording | Pass |
| `determinism` | Two runs give the same events | Pass |
| `required-only` | A run with only the required channels and no subject values succeeds | Pass, EEG only, 10 events |
| `expected` | The events match the recorded document (step 7) | Pass |

- **Problem.** The EOG and EMG types were declared optional but were not treated as
  optional: the platform required every declared type at submission, the portal read
  a lower bound of 0 as 1, and the fixture always supplied every type, so no check
  noticed that YASA could only run with all three channels.
- **Resolution.** The platform and the portal treat a type with a minimum of 0 as
  optional, and since 32.1.0 the check runs once more on the required channels alone.
  In that run YASA uses its EEG-only classifier.

## 7. Record the expected output

```bash
make record   # writes conformance/expected-events.json.gz
make verify   # PASS expected: matches the recorded document (10 events)
```

The recorded document is committed beside the configuration. It is recorded from the
linux/amd64 build, for the reason given in step 5.

## 8. Run it on a real recording

```bash
yousleep-manifest --recording SC4001E0-PSG.edf --config yasa-staging.yaml \
    --channel "EEG Fpz-Cz" --channel "EOG horizontal" > manifest.json
docker run --rm --network none -v "$PWD:/local" ghcr.io/example/yasa-staging:dev \
    --manifest-file /local/manifest.json
```

On a public Sleep-EDF recording from PhysioNet (22.1 hours, 100 Hz), the image scored
2650 epochs in about 20 seconds under emulation. Its peak memory was 1023 MiB, while
the manifest reserved 512 MiB, the base in the configuration. The terms that scale
with recording length are not in the author's file (step 11).

- **Problem.** In 33.0.0, `make run RECORDING=night.edf` selects every signal of the
  file, and the manifest tool refuses the selection: `the platform would refuse this
  run: 2 EEG channel(s) selected; the analysis takes [1, 1]; 'EMG submental' is
  sampled at 1 Hz; EMG needs [100, inf] Hz`. The Makefile has no variable for a
  channel selection.
- **Resolution.** Since 33.1.0, `make run` selects channels as the platform does,
  keeping the first ones of each type up to the analysis's maximum and leaving out a
  channel below the required rate; `CHANNELS` sets an explicit choice.
  `yousleep-manifest` also takes `--age`, `--sex` and `--bmi`, so a hand run can
  exercise the demographics, e.g. `make run RECORDING=night.edf
  MANIFEST_ARGS="--age 40 --sex male"`.

## 9. Register the analysis

The analysis was registered as [Registration](registration.md) describes. The image
was checked on linux/amd64 with the same tool, and its digest was pinned in the
registered configuration. With `data_rights: pending`, the analysis is registered
privately and offered to one organisation. It is registered as "YASA sleep staging"
with the id `yasa-sleep-staging-v1`: one lineage, so the name carries no version, and
`provenance.source_version` records `YASA 0.7.0, classifiers 0.5.0`. It was registered with a memory claim of
1024 MiB that had not been measured.

## 10. The first production run

The first production run, on a 23.4-hour recording with two channels selected, failed
out of memory. It had two causes:

- **The script read every channel of the file** before keeping the selected ones, so
  memory grew with the file rather than with the selection. On a 42-channel
  polysomnogram of 10 hours at 256 Hz, the peak for two selected channels was
  6392 MiB. After the change to pick first and then load, it was 1684 MiB, with the
  same events.
- **The claim had not been measured.** YASA resamples with MNE's FFT method, which pads
  each channel to the next power of two. Memory therefore grows by about 100 to
  200 MiB per hour at 256 to 512 Hz, and steps with recording length.

`yousleep-verify` cannot detect either cause: its fixture is five minutes long and
carries only the declared channels.

## 11. Measure memory and set the claim

Peak memory of the linux/amd64 image was measured with one core on three real
recordings, at 100 Hz and 256 Hz, one of them also upsampled to 512 Hz. Each was cut
or repeated to between 0.5 and 48 hours, with one to three channels selected. The
claim set at registration is

`800 + h·kHz·(450 + 90·n)` MiB

where `h` is the recording length in hours, `kHz` the highest native sampling rate of
the selected channels in kHz, and `n` the number of selected channels. The
configuration model gained the term that does not scale with channels for this
purpose in 32.3.0.

| Recording | Channels selected | Hours | Measured peak (MiB) | Claim (MiB) |
|---|---|---|---|---|
| 100 Hz | 2 | 21.9 | 1126 | 2180 |
| 256 Hz | 1 | 8 | 1089 | 1906 |
| 256 Hz | 2 | 8 | 1209 | 2091 |
| 256 Hz | 2 | 24 | 2880 | 4671 |
| 256 Hz | 2 | 48 | 5144 | 8542 |
| 256 Hz, 42-channel file | 2 | 10 | 1684 | 2413 |
| 512 Hz | 2 | 24 | 5100 | 8542 |

```yaml
resources:
  cpu_cores: 1
  base_system_memory_mib: 800
  per_hour_per_khz_system_memory_mib: 450
  per_hour_per_channel_per_khz_system_memory_mib: 90
```

The author sets `cpu_cores` and `base_system_memory_mib`. The platform team sets the
terms that scale with recording length at registration.

## Problems found and where they were fixed

The last column records whether the run with `yousleep-common` 33.0.0 met the problem.

| Problem | Where it was fixed | Met with 33.0.0 |
|---|---|---|
| `yousleep-init` refused an optional channel type | `yousleep-init`, 30.0.1 | No |
| `yousleep-init` derived an invalid name from a short directory name, and an empty slug from `.` | `yousleep-init`, 30.0.1 | No |
| `select_channels` returned `ChannelType.EEG` instead of `EEG` | `select_channels`, 30.0.2 | No |
| The script read `.value` from a type that arrives as plain text | The script; the manifest page states it since 31.0.0 | No |
| The 32-character bound on a citation `name` was not documented | The guide, 31.0.0 | No |
| LightGBM failed to import on the slim base because `libgomp.so.1` was missing | The Dockerfile; the guide names the library since 31.0.0 | Yes, without `libgomp1`; the build-time import reports it |
| An expected document recorded on arm64 did not match the amd64 build | The generated Makefile builds for linux/amd64, and the check names the first difference, 31.1.0 | No with the default build; an arm64 build still fails `expected` |
| Optional channel types were required at submission, and the check never ran without them | The platform and the portal; the `required-only` run, 32.1.0 | No; `required-only` passes. The platform side was not tested locally |
| A sex of `other` or `unknown` was passed to YASA as female | The script | Depends on the method; the manifest's `sex` can also be `other` or `unknown` |
| The script read every channel of the file | The script: pick, then load | No, because the generated script picks first; the check cannot detect it |
| The memory claim was not measured | Measured at registration; a per-kHz term in the configuration model, 32.3.0 | Not an author step |
| The commented `evidence` example in the generated configuration fails validation | `yousleep-init`, 33.1.0 | Yes |
| `make run` selects every channel, which the manifest tool refuses for this configuration | `yousleep-manifest` selects as the platform does, 33.1.0 | Yes |
| A hand run cannot set subject values | `yousleep-manifest --age --sex --bmi`, 33.1.0 | Yes |
| YASA picks its newest trained classifiers, so a YASA release could change results silently | The script names generation 0.5.0; `provenance.source_version` records it, 33.2.0 | Not an author step before 33.2.0 |

## Conclusion

Packaging YASA took a script of about 100 lines, a configuration, and a Dockerfile
with one system library. The earlier attempts and the first production run found
problems in the tools, the guide and the platform, each now fixed in a released version
or in the platform's behaviour; the run with 33.0.0 passed every check and met three
problems in the generated files and the manifest tool, fixed in 33.1.0. The guide now covers optional
channel types, system libraries missing from the slim base, the architecture of the
recorded output and the run without optional inputs, and the platform team measures
the memory that scales with recording length at registration.

## Get started

| Resource | What it covers |
|---|---|
| [Packaging an analysis](../components/common/analysis-authoring.md) | The contract, the three files and the configuration reference |
| [Starting a project](../components/common/init.md) | `yousleep-init` and the Makefile targets |
| [The manifest](../components/common/analysis-manifest.md) | What the container reads, and `yousleep-manifest` for hand runs |
| [The events document](../components/common/events-document.md) | What the container writes |
| [Checking an image](../components/common/conformance.md) | `yousleep-verify`, its checks and the recorded output |
| [Registration](registration.md) | What to send and what the platform team does with it |
| [analysis-example](https://github.com/yousleep-ai/analysis-example) | Two complete example analyses |
| [YASA](https://github.com/raphaelvallat/yasa) | The upstream library and its documentation |
| [Vallat and Walker, 2021](https://doi.org/10.7554/eLife.70092) | The paper describing YASA's sleep staging |
| [`yousleep-common` on PyPI](https://pypi.org/project/yousleep-common/) | The package with `yousleep-init`, `yousleep-verify` and `yousleep-manifest` |
