# Validator profiles for PEST

Research date: 2026-09-02. This fixes the current baseline to pluginval
`v1.0.4`, clap-validator `0.4.1` (`152b982`), and VST3 SDK
`v3.8.1_build_84`. PEST should pin the downloaded/built bytes and verify the
tool-reported version before every run.

## Decision

Run every validator directly and under PEST's own process-tree supervisor.
Do not invoke Steinberg Validator through pluginval: that loses a separate
receipt and still needs the same outer timeout and output checks.

### pluginval (VST3)

```text
pluginval \
  --validate <artifact.vst3> \
  --strictness-level 10 \
  --random-seed <nonzero-u64> \
  --randomise \
  --timeout-ms 30000 \
  --sample-rates 44100,48000,96000 \
  --block-sizes 64,128,256,512,1024 \
  --output-dir <artifact-dir> \
  --output-filename pluginval.txt
```

Level 10 is the maximum; higher levels add longer fuzz/state/thread/real-time
tests. An explicit nonzero seed makes JUCE's test RNG and randomized order
replayable; zero means “generate a seed.” The sample-rate and block-size lists
above freeze pluginval's documented defaults instead of silently inheriting a
future change. Do not pass `--skip-gui-tests` in the full desktop profile.
`--repeat` is deliberately absent: pluginval says repeats do not recreate the
plug-in, while PEST already gives each seed a fresh supervised process.
([CLI contract](https://github.com/Tracktion/pluginval/blob/v1.0.4/Source/CommandLine.cpp#L320-L409),
[seed semantics](https://github.com/Tracktion/pluginval/blob/v1.0.4/Source/PluginTests.h#L24-L42),
[test selection/order](https://github.com/Tracktion/pluginval/blob/v1.0.4/Source/PluginTests.cpp#L164-L205))

Pass only on a normal exit code `0`; pluginval documents `1` for test errors.
Its 30-second timeout is an **output-inactivity** timeout, not a wall-clock
limit, and terminates the whole pluginval process. PEST therefore still needs
an outer wall-clock deadline and process-tree cleanup.
([documented exit codes](https://github.com/Tracktion/pluginval/blob/v1.0.4/README.md#running-in-headless-mode),
[timeout implementation](https://github.com/Tracktion/pluginval/blob/v1.0.4/Source/Validator.cpp#L23-L100))

Official `v1.0.4` assets exist for Linux, Windows, and macOS. The release's
Linux/Windows binaries are x86-64; its macOS build is universal arm64+x86-64
(the release workflow sets both architectures). Pin asset SHA-256 values in
PEST's tool lock, because the pluginval release metadata itself supplies no
digest.
([release assets](https://github.com/Tracktion/pluginval/releases/tag/v1.0.4),
[platform build matrix](https://github.com/Tracktion/pluginval/blob/v1.0.4/.github/workflows/build.yaml#L16-L33),
[universal macOS build](https://github.com/Tracktion/pluginval/blob/v1.0.4/.github/workflows/build.yaml#L85-L98))

### CLAP Validator

Run the complete deterministic suite out of process and request structured
output:

```text
CLAP_VALIDATOR_CONFIG=<known-empty-config.toml> \
clap-validator --verbosity info validate --json --jobs 1 <artifact.clap>
```

Do not pass `--include`, `--exclude`, `--only-failed`, or `--in-process`.
`CLAP_VALIDATOR_CONFIG` prevents an ambient ancestor
`clap-validator.toml` from disabling tests. Out-of-process mode gives every
test a clean child and crash containment; `--jobs 1` removes scheduling as a
variable. The validator applies a fixed 45-second timeout to each test.
([validation options](https://github.com/free-audio/clap-validator/blob/0.4.1/src/commands/validate.rs#L13-L87),
[discovery and sandbox policy](https://github.com/free-audio/clap-validator/blob/0.4.1/src/validator.rs#L87-L176),
[ambient config lookup](https://github.com/free-audio/clap-validator/blob/0.4.1/src/cli/config.rs#L9-L40))

Exit code alone is insufficient for PEST's fail-closed profile: CLAP Validator
returns success when there are warnings or skips. Parse stdout as JSON and
require every mandatory result to have `status.code == "success"`; fail
`warning`, `failed`, and `crashed`, and allow `skipped` only when the manifest
explicitly declares the tested capability not applicable. Missing/malformed
JSON or an empty result set also fails.
([status model](https://github.com/free-audio/clap-validator/blob/0.4.1/src/tests.rs#L45-L72),
[exit tally](https://github.com/free-audio/clap-validator/blob/0.4.1/src/validator.rs#L19-L60),
[exit decision](https://github.com/free-audio/clap-validator/blob/0.4.1/src/commands/validate.rs#L65-L87))

Use two fuzz modes:

```text
# Reproducible acceptance seeds; one supervised invocation per seed.
clap-validator --verbosity info fuzz --json \
  --plugin-id <id> --reproduce <u64-seed> <artifact.clap>

# Time-budgeted discovery; not reproducible as a whole.
clap-validator --verbosity info fuzz --json \
  --plugin-id <id> --jobs <n> --duration <duration> --limit <n> <artifact.clap>
```

`--reproduce` executes exactly one deterministic chunk and conflicts with
jobs/duration/limit. A discovery campaign seeds its orchestrator from current
time, does not accept a campaign seed, and retains only failing chunks, so its
entire executed sequence cannot be replayed. Record every emitted failing seed
and turn it into a fixed acceptance seed. JSON `[]` plus exit `0` means no
failure was found within the budget; it is not proof of exhaustive coverage.
([fuzz CLI and exit behavior](https://github.com/free-audio/clap-validator/blob/0.4.1/src/commands/fuzz.rs#L42-L113),
[campaign and replay mechanics](https://github.com/free-audio/clap-validator/blob/0.4.1/src/fuzz.rs#L43-L131),
[time-seeded orchestrator](https://github.com/free-audio/clap-validator/blob/0.4.1/src/fuzz/rng.rs))

The `0.4.1` release supplies Ubuntu 22.04 x86-64, Windows x64, and native or
universal macOS arm64 assets with publisher-provided SHA-256 digests.
([release assets and digests](https://github.com/free-audio/clap-validator/releases/tag/0.4.1),
[official build matrix](https://github.com/free-audio/clap-validator/blob/0.4.1/.github/workflows/build.yml#L21-L32))

### Steinberg VST3 Validator

Run both supported lifetime models because they cover different failure modes:

```text
validator -e <artifact.vst3>
validator -e -l <artifact.vst3>
```

`-e` enables the official extensive tests; `-l` creates a local plug-in
instance per test instead of sharing the default global instance. There is no
seed control or machine-readable output. Do not narrow with `-suite` or `-cid`.
([options and dispatch](https://github.com/steinbergmedia/vst3_public_sdk/blob/v3.8.1_build_84/samples/vst-hosting/validator/source/validator.cpp#L218-L459),
[extensive test construction](https://github.com/steinbergmedia/vst3_public_sdk/blob/v3.8.1_build_84/samples/vst-hosting/validator/source/validator.cpp#L620-L733))

A normal zero exit is necessary but **not sufficient**. Source inspection shows
that a failed test increments the failure count, but failed `setup()` and
`teardown()` only print diagnostics and do not increment it. PEST must fail on
nonzero/signal/exception **or** any line beginning `ERROR:`/`Error:` **or** the
exact diagnostics `Failed to setup test!` and `Failed to teardown test!`.
Preserve complete stdout/stderr; do not use `-q`.
([test runner accounting](https://github.com/steinbergmedia/vst3_public_sdk/blob/v3.8.1_build_84/samples/vst-hosting/validator/source/validator.cpp#L744-L815),
[final exit calculation](https://github.com/steinbergmedia/vst3_public_sdk/blob/v3.8.1_build_84/samples/vst-hosting/validator/source/validator.cpp#L408-L459))

Steinberg publishes the validator as SDK source rather than per-platform
release binaries. Build and hash it from the pinned recursive SDK tag on each
native runner. SDK `3.8.1` lists Windows x64, macOS Apple Silicon, and Ubuntu
24.04 x86-64 among supported targets.
([SDK tag](https://github.com/steinbergmedia/vst3sdk/tree/v3.8.1_build_84),
[supported platforms](https://github.com/steinbergmedia/vst3sdk/blob/v3.8.1_build_84/README.md#supported-platforms),
[validator build target](https://github.com/steinbergmedia/vst3_public_sdk/blob/v3.8.1_build_84/samples/vst-hosting/validator/CMakeLists.txt))

## Supervisor contract shared by all three

For each invocation PEST must create a new OS process group/job, capture stdout
and stderr separately, enforce an outer wall-clock deadline, terminate the
entire tree on timeout or abnormal exit, wait for reaping, and fail if any
descendant survives. Only then may it parse the tool-specific result above.

This is not redundant. CLAP Validator's current timeout branch returns a
timeout after `wait_timeout` without an explicit child kill, pluginval's
timeout is inactivity-based, and Steinberg Validator has no timeout option.
([CLAP sandbox source](https://github.com/free-audio/clap-validator/blob/0.4.1/src/cli/sandbox.rs#L40-L100),
[pluginval timeout source](https://github.com/Tracktion/pluginval/blob/v1.0.4/Source/Validator.cpp#L77-L99),
[Steinberg option set](https://github.com/steinbergmedia/vst3_public_sdk/blob/v3.8.1_build_84/samples/vst-hosting/validator/source/validator.cpp#L323-L406))

The receipt should record tool binary SHA-256, reported version, full argv,
environment overrides, artifact SHA-256, platform/architecture, start/end time,
outer-timeout outcome, raw exit status, parsed result, fuzz seed(s), and paths
to complete logs. A missing tool, version/hash mismatch, spawn failure,
timeout, abnormal termination, malformed output, failed cleanup, or parser
diagnostic above is a failed validator run.
