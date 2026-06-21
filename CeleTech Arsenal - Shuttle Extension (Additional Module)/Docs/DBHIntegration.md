# Dubs Bad Hygiene Integration Notes

## Detected Dependencies

- Main shuttle mod packageId: `LongRange.CeleTech.ShuttleExtension`
- Main shuttle mod supported RimWorld version: `1.6`
- Dubs Bad Hygiene packageId: provided by `CT_Shuttle_DBHIntegration.packageIds`
- The integration is based on the public behavior and XML shape of the local
  Dubs Bad Hygiene package inspected during development.

## Dependency Strategy

Recommended first skeleton: soft dependency.

Hard dependency is simpler and gives compile-time DBH types, but it would force DBH to be present for the whole add-on. This add-on stays soft-dependent so non-DBH modules and diagnostics can still load without `BadHygiene.dll`.

Soft dependency keeps this add-on usable as a compatibility module pack. DBH-specific modules can load in diagnostic mode when DBH is missing or incompatible, and later integrations can follow the same pattern.

## Main Shuttle API Usage

Use only public SDK namespaces from `CeleTech_Shuttle.dll`:

- `CeleTech.ShuttleExtension.ModularShuttle.API.Runtime`
- `CeleTech.ShuttleExtension.ModularShuttle.API.UI`
- `CeleTech.ShuttleExtension.ModularShuttle.API.Commands`
- `CeleTech.ShuttleExtension.ModularShuttle.API.Profile`

This skeleton reads `CT_Shuttle_DBHIntegrationDef` and registers:

- runtime system key: `celetech.shuttle.extension.additionalmodule/dbh-hygiene-suite-runtime`
- panel system key: `celetech.shuttle.extension.additionalmodule/dbh-hygiene-suite-panel`

The module XML binds the runtime key through `ShuttleRuntimeSystemDefExtension` for the main shuttle SDK, and marks the module with `DBHHygieneSuiteModuleExtension` so C# runtime applicability is driven by Def data instead of a module defName comparison. Runtime classes are never instantiated from XML.

## API Gaps

The current external runtime context exposes module facts, key-value state, internal bus status, stored energy consumption, serviceable occupant snapshots, and host footprint snapshots.

Current add-on policy: do not edit the main shuttle mod, do not reflect private/internal shuttle holders, and do not access controller/runtime internals. The add-on consumes only the public external runtime occupant and host read APIs.

Public seams consumed by the MVP:

- `context.SupportsOccupantRead`
- `context.GetOccupants(ShuttleExternalOccupantQuery)`
- `ShuttleExternalOccupantInfo.UnsafeLivePawn` for DBH need mutation only
- `ShuttleExternalOccupantInfo.CanReceiveExternalService`
- `context.SupportsHostRead`
- `context.TryGetHostInfo(out ShuttleExternalHostInfo)`
- `ShuttleExternalHostInfo.UnsafeLiveMap` for DBH pipe/map operations only
- `ShuttleExternalHostInfo.OccupiedCells`
- `ShuttleExternalHostInfo.AdjacentExternalCells`

DBH pipe network access is implemented conservatively through a soft reflection bridge. The bridge only
checks the public shuttle host footprint and adjacent external cells, resolves a nearby DBH `PlumbingNet`,
and calls DBH's own `PullWater` / `PushSewage` methods. The shuttle is not registered as a DBH pipe
building and its internal clean-water tank remains the authoritative shuttle-side water state.

Declared runtime UI metadata is also deferred to a future main shuttle SDK/UI task. The main shuttle
mod's declared runtime registry UI currently does not support third-party runtime display metadata, so
unbound runtime declarations may still show the raw runtime key such as `dbh-hygiene-suite-runtime`.
Future main shuttle SDK work should add runtime registration metadata, ideally `runtimeLabelKey` and
`runtimeDescriptionKey`, and render those keys in the declared runtime list while keeping the full
runtime key in DevMode tooltip/details only.

Cargo water refill is disabled. The hygiene module water store is an internal runtime tank and DBH pipe
bridge state, not shuttle cargo. The add-on does not consume loaded cargo water bottles and does not
deposit internal tank water into cargo.

## DBH Access Notes

The inspected DBH XML defines needs as `NeedDef`; the actual defNames are configured in `CT_Shuttle_DBHIntegrationDef`:

- `Hygiene`: `DubsBadHygiene.Need_Hygiene`
- `Bladder`: `DubsBadHygiene.Need_Bladder`
- `DBHThirst`: `DubsBadHygiene.Need_Thirst`

Water and sewage systems are implemented through DBH-specific building comps such as `CompProperties_Pipe`, `CompProperties_WaterStorage`, `CompProperties_WaterInlet`, `CompProperties_WaterPumpingStation`, `CompProperties_SewageHandler`, and `CompProperties_WaterFiltration`.

Sewage XML also includes `CompProperties_SepticTank`, and the inspected package defines `RawSewage` as the raw sewage ThingDef. No stable public DBH API was found in the inspected local package. The bridge therefore resolves need defs by name and avoids direct references to `BadHygiene.dll`.

DBH also defines an itemized water resource, but this integration does not use it for shuttle tank storage:

- `DBH_WaterBottle`: ThingDef label `water`, gated by `DBHThirst`, with `DubsBadHygiene.WaterExt`

`allowCargoWaterRefill` defaults to false so tank water is never represented by shuttle cargo stacks.

## Hygiene Module State

The runtime maintains a finite clean-water tank and septic tank:

- clean water capacity: configured by `cleanWaterCapacity`
- initial clean water: 50% capacity, to avoid free full-service deployment
- sewage capacity: configured by `sewageCapacity`
- pipe refill/drain rates: configured by `pipeFillRatePerService` and `pipeSewageDrainRatePerService`
- shower water/sewage values: configured by `showerWaterCost` and `showerSewageOutput`; the default compact shuttle shower uses 20 L and produces 20 L wastewater
- toilet water/sewage values: configured by `toiletWaterCost` and `toiletSewageOutput`
- septic treatment recovers clean water from treated sewage by `septicCleanWaterRecoveryRatio`, defaulting to 60%; recovered water is capped by clean-water tank capacity

Pipe checks run only on the service interval or when requested by the DevMode pipe-check command. If a
nearby DBH pipe resolves to a `PlumbingNet`, `DBHPipeBridge` asks that network to provide water or accept
sewage. If DBH returns false, the add-on treats that as an external network condition: no drawable DBH
clean-water storage for `PullWater`, or no available unblocked DBH sewage facility for `PushSewage`.
The runtime does not invent water or silently delete sewage when DBH declines the request. Runtime pipe
search radius is capped to avoid expensive broad scans.

Future optional feature: treating the shuttle internal tank as a DBH `PlumbingNet` water tower would
need separate registration, deregistration, and launch cleanup design. The current version does not do this.

When the septic tank is full, the runtime first asks the pipe bridge to drain sewage. If that is unavailable and `allowMapDump` is true, it uses the public shuttle host read API to pick an adjacent external cell, preferring roofless walkable cells outside the occupied footprint. If no map context is available and `allowWorldDump` is true, it performs an abstract outside dump and records the dumped amount.

## MVP Services

Each service interval queries occupants with `HumanlikeOnly = true` and `ServiceableOnly = true`. Mechs and animals are ignored. The runtime does not move, unload, destroy, re-parent, or otherwise mutate the pawn container; it only adjusts DBH needs and module runtime state.

The service loop handles up to four pawns per interval:

- shower: if Hygiene is below `showerHygieneThreshold`, raise Hygiene by `showerHygieneGain`, spend `showerWaterCost`, and add `showerSewageOutput`
- toilet: DBH XML describes bladder level falling over time, so low Bladder is treated as needing toilet service; raise Bladder toward a conservative 70%, spend `toiletWaterCost`, and add `toiletSewageOutput`

## MVP Module

`CT_Shuttle_DBH_HygieneSuiteModule`

Research:

- `CT_Shuttle_Research_DBH_HygieneSuite`

- habitat module for living segments and habitat module slots
- 80 W idle draw and external power-demand contribution
- detects DBH loaded/missing
- resolves required DBH need defs from integration Def fields (`Hygiene`, `Bladder`)
- treats `DBHThirst` as optional diagnostic data
- tracks clean water, septic sewage, pipe connection status, shower uses, toilet uses, and dumped sewage
- services public serviceable humanlike occupants through the main shuttle SDK read port
- displays status in the external module panel
- dumps sewage to adjacent external cells through the main shuttle SDK host read port, or abstractly to world when not on-map

## Deferred Modules

- `CT_Shuttle_DBH_WaterReclaimerModule`
- `CT_Shuttle_DBH_SanitationModule`
- `CT_Shuttle_DBH_ShowerWashStationModule`

## Bad Hygiene Lite v1 Constraints

Bad Hygiene Lite support is intentionally separate from the full Dubs Bad Hygiene hygiene module.
Treat Lite as a no-water, no-pipe, no-sewage-network integration.

Hard constraints for Lite v1:

- no clean-water tank
- no water pipe bridge
- no sewage pipe bridge
- no septic tank or sewage storage
- no liquid resource model
- no `cleanWaterLiters` or `sewageLiters` runtime state
- no tank-capacity command
- no pipe-check command
- no `DBHPipeBridge`
- no `DBHLiquidUtility`
- no `PlumbingNet`, `CompPipe`, `PullWater`, or `PushSewage`
- no dynamic liquid mass contribution
- no water, sewage, capacity, liters, pipe, or tank display in normal Lite UI

Lite Service v1 only provides abstract hygiene service:

- improve `Hygiene` through the Lite need or safe reflection
- improve `Bladder` through the Lite need or safe reflection
- do not simulate clean-water consumption
- do not simulate sewage output
- do not modify or service `DBHThirst`
- do not provide drinking service

Power demand is generic module operation power, unrelated to water resources.
Dynamic mass contribution stays zero in Lite v1; only the module Def's static mass applies.

Current Lite v1 runtime:

- detects `Dubwise.DubsBadHygiene.Lite`
- hides itself when full `Dubwise.DubsBadHygiene` is active
- resolves `Hygiene` and `Bladder`
- detects optional `DBHThirst` for diagnostics only
- reads public shuttle serviceable occupants
- improves `Hygiene` with Lite `Need_Hygiene.clean(float)` when available
- improves `Bladder` with bounded Lite `Need_Bladder.dump()` calls when available
- falls back to conservative `CurLevel` increments if the Lite methods are unavailable
- does not change `DBHThirst`

## Release Package

The public release package includes the playable mod root only:

- `About`
- `Assemblies`
- `Defs`
- `Languages`
- `Patches`
- `Docs`

Source project files, build scripts, debug symbols, intermediate build outputs,
local scratch files, and git metadata are intentionally omitted.
