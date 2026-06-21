# CeleTech Arsenal - Shuttle Extension (Additional Module)

Release repository for the optional compatibility module pack for
`CeleTech Arsenal - Shuttle Extension`.

This repository contains the playable release files only. It does not include
the C# source project, build scripts, object files, or debug symbols.

## Install

Copy the folder below into your RimWorld `Mods` directory:

```text
CeleTech Arsenal - Shuttle Extension (Additional Module)
```

Required dependency:

- `LongRange.CeleTech.ShuttleExtension`

Optional compatibility targets:

- Dubs Bad Hygiene
- Bad Hygiene Lite

Load this mod after the main shuttle extension and after the relevant hygiene
mod when one is installed.

## Contents

- Playable release assets under
  `CeleTech Arsenal - Shuttle Extension (Additional Module)/`.
- SDK documents under
  `CeleTech Arsenal - Shuttle Extension (Additional Module)/Docs/`.
- Dubs Bad Hygiene integration notes in
  `CeleTech Arsenal - Shuttle Extension (Additional Module)/Docs/DBHIntegration.md`.

## SDK Entry Point

Third-party authors should start with:

```text
CeleTech Arsenal - Shuttle Extension (Additional Module)/Docs/ThirdPartyRuntimeSdk.md
```

For the conceptual shuttle architecture and SDK boundary model, read:

```text
CeleTech Arsenal - Shuttle Extension (Additional Module)/Docs/ShuttleArchitectureOverview.md
```

The SDK documents are included here so add-on authors can find the public
runtime, UI, command, cargo, and launch integration boundaries without needing
the main mod development workspace.
