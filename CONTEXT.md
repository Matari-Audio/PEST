# PEST

PEST describes portable evidence about an already-built audio plug-in without depending on the framework that produced it.

## Language

**Plugin Artifact**:
One immutable CLAP or VST3 bundle supplied to PEST for testing.
_Avoid_: Build, binary under test, installed plug-in

**Plugin Release**:
One explicit release identity whose platform-specific Plugin Artifacts are tested and aggregated.
_Avoid_: Build number, latest version

**Universal Profile**:
The mandatory validator and REAPER behavior shared by every supported Plugin Artifact.
_Avoid_: Default tests, generic tests

**Plugin Scenario**:
A declarative, plug-in-specific workflow and its expected evidence.
_Avoid_: Custom test, script, hook

**Run Receipt**:
The authoritative immutable record of inputs, tools, environment, commands, seeds, outcomes, and linked evidence from one PEST run.
_Avoid_: Log, report, console output

**Execution Profile**:
The declared headless or desktop environment required to produce a class of evidence.
_Avoid_: Mode, CI job
