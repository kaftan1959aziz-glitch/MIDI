# Windows MIDI Services SDK — Build Repair Notes

This document records the build repairs and validation work carried out while preparing a more approachable and reproducible build of the Windows MIDI Services SDK.

> **Project status:** These are independent community buildability improvements and repair notes for the SDK source. This document does not represent an official Microsoft release, endorsement, or change in project ownership.

## Objectives

The work focused on resolving practical build failures while preserving the original SDK source, IDL definitions, existing include directives, and package declarations wherever possible.

The guiding principles were:

- Prefer small, evidence-based changes over broad project restructuring.
- Preserve source files, IDL definitions, package declarations, and existing dependencies.
- Make generated-file locations and include paths explicit where a project requires them.
- Avoid unnecessary changes to the original SDK architecture.
- Verify repairs through actual Visual Studio and MSBuild builds.
- Clearly distinguish successful incremental builds from complete clean builds.

## Repairs Completed

### 1. NuGet Restore for WinUI-Dependent Projects

Several projects initially failed during MIDL compilation because WinUI types could not be resolved.

The affected projects included:

- `miditroubleshooter`
- `midisettings`
- `midinetworksetup`
- `midisysextool`
- `midiscratchpad`
- `midisoundmodule`

The failures were traced to missing NuGet restore outputs, including `obj\project.assets.json` and generated NuGet props and targets, rather than missing package declarations.

Explicit MSBuild `Restore` operations were run for all six affected projects. The restores completed successfully, after which the WinUI-related MIDL failures were resolved.

Existing package references were retained. The restore failures were not addressed by removing dependencies from project configuration.

### 2. `mididiag` Generated-Header Include Paths

`mididiag` initially failed because the compiler could not locate the generated `WindowsMidiServices.h` header.

Investigation identified the generated IDL intermediate directory:

```text
$(SolutionDir)VSFiles\intermediate\IDL\$(Platform)\$(Configuration)\
```

The project configuration was updated to include this directory in both:

- Debug x64
- Release x64

This made the generated IDL header location explicit and consistent across the two tested configurations.

Both Debug x64 and Release x64 builds of `mididiag` subsequently succeeded.

### 3. NuGet Package Configuration

The Windows MIDI Services NuGet project contained package paths hard-coded to the Release configuration.

This prevented the package specification from consistently following the active build configuration.

The following changes were made:

- Added `NuspecProperties` to pass the active MSBuild configuration into the nuspec.
- Replaced hard-coded Release directory references with `$configuration$`.
- Applied the configuration-aware paths to the x64 WinMD, runtime DLL, PRI and PDB files.
- Applied the same approach to the ARM64EC runtime files.
- Updated the COM extension generated header and source paths to use the active configuration.
- Set `GeneratePackageOnBuild` to `false`, preserving the repository's explicit package-generation workflow.

The existing package identity, dependencies, package contents and target framework declarations were retained.

These changes address configuration-dependent package paths. A complete clean package-generation test remains outstanding.

### 4. vcpkg MSBuild Integration

The repository's `src/in-box` directory contained a `Directory.Build.props` file with project-wide vcpkg configuration, but the corresponding vcpkg MSBuild imports were incomplete.

The repository's vcpkg integration requires both the props and targets imports.

The following imports were added:

- `vcpkg.props` in `Directory.Build.props`
- `vcpkg.targets` in a new `Directory.Build.targets`

Both imports reference the repository's own vcpkg checkout rather than relying on an unrelated Visual Studio-bundled vcpkg installation.

This makes the intended vcpkg integration explicit and consistent for projects beneath `src/in-box`.

### 5. Generated MIDI Service Transport IDL Header Investigation

The AppSDK solution did not contain a project dependency on the standalone MIDI Service Transport project.

The transport project was built separately to investigate the missing generated `Midi2MidiSrvTransport.h` header.

MIDL generated the transport header, type library and interface source files in the transport project's intermediate directory.

The subsequent C++ compilation encountered a separate missing `MidiSrvRpc.h` dependency.

After the transport IDL header had been generated, a later AppSDK Release build no longer reported the original missing transport header.

This is evidence that the generated header was relevant to the observed failure. However, a clean-build dependency chain has not yet been established, and this investigation should not be interpreted as a completed dependency fix.

## Build Verification

The following results were observed on November 24, 2026.

| Project / Solution | Configuration | Result |
|---|---|---|
| `mididiag` | Debug x64 | Succeeded |
| `mididiag` | Release x64 | Succeeded |
| `Midi2-AppSDK.sln` | Release x64 | Succeeded incrementally |

The Release x64 AppSDK solution build reported:

```text
1 succeeded
59 up-to-date
0 failed
0 skipped
```

The `mididiag` Release build performed a full compilation because a usable previous IPDB/IOBJ was not available.

**Important qualification:** The AppSDK solution result was an incremental build, not a complete clean build. Projects reported as up-to-date were not all rebuilt.

## Remaining Warnings and Work

Some warnings remain outside the scope of these repairs:

- RC4005 macro redefinition warnings in resource compilation.
- C4702 unreachable-code warning originating in `fmt/base.h`.

These warnings were observed but not modified as part of this work.

Further validation would benefit from:

1. A clean Debug x64 build of the AppSDK solution.
2. A clean Release x64 build of the AppSDK solution.
3. Verification of the generated IDL dependency chain from a clean intermediate directory.
4. A complete NuGet package-generation test.
5. Verification of the relevant Azure Pipelines configuration and restore sources.
6. Reproduction of the build from a fresh repository clone.

## Scope and Preservation

The repairs were deliberately limited to build configuration, generated-file paths, package configuration and documentation.

No MIDI source implementation or IDL definitions were intentionally changed as part of this repair set.

Existing package references were preserved, including references that had previously appeared to be candidates for removal during restore troubleshooting.

The work aims to improve build reproducibility without redesigning the SDK or changing its intended architecture.

## Acknowledgements

Build investigation and repairs authored by **Khalfan Aziz**, with assistance from **ChatGPT (GPT-6)**.

Special thanks to coffee barrels, aspirin sacks and the occasional Jupiter-sized headache encountered along the way.

And remember: a generated header is only useful when the compiler knows where to find it!
