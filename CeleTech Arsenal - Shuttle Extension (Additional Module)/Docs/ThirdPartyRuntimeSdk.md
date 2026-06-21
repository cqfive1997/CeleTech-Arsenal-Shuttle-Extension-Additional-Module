# Third-Party Runtime SDK

This is the main authoring guide for third-party shuttle runtime modules.
It covers runtime registration, XML binding, external state, lifecycle hooks,
commands, UI panels, and the boundaries that keep add-ons compatible with the
core shuttle system.

For copied read snapshots, cargo transactions, launch read/quote, and launch
rule providers, read `Docs/ExternalSDK_P1_Usage_Guide.md` as well.

## Current Scope

Supported authoring surfaces:

- Static profile contributors through `ShuttleProfileAPI`.
- External runtime systems through `ShuttleRuntimeAPI`.
- Key-value external runtime state saved by the main mod.
- Map-side runtime hooks such as `Initialize`, `Migrate`, `Reconcile`, `Tick`,
  `CollectPowerDemand`, `CollectMassContribution`, `CanRemove`, `OnInstalled`,
  `OnRemoved`, and `OnArrived`.
- Launch hooks: `PreLaunchValidate` and `OnLaunchSucceeded`.
- External command handlers through `ShuttleCommandAPI`.
- External Modules page panel providers through `ShuttleUIAPI`.
- Host-rendered panel command contributions.
- Optional inline panel command execution through
  `ShuttleExternalModulePanelContext.CommandExecutor`.

Outside the current scope:

- Direct access to `ShuttleController`, live assembly state, live runtime state,
  cargo backends, holders, or launch services.
- Arbitrary main-page UI injection or independent windows from panel providers.
- Cross-owner command or panel control.
- XML class-name activation for runtime systems.
- Cargo or launch mutation from runtime hooks. Use the copied SDK transaction
  and launch surfaces where supported.

## Document Map

| Need | Read |
| --- | --- |
| Safe copied read/cargo/launch SDK | `Docs/ExternalSDK_P1_Usage_Guide.md` |
| UI host rules | `Docs/ThirdPartyModuleUIBoundary.md` |
| Command routing rules | `Docs/ThirdPartyCommandBoundary.md` |
| Runtime test checklist | `Docs/ThirdPartyRuntimeTestingChecklist.md` |
| SDK changelog | `Docs/ExternalSDK_P1_Changelog.md` |

## Versions And Feature Checks

The runtime, command, and UI facades expose their own API versions:

```csharp
Version runtimeVersion = ShuttleRuntimeAPI.APIVersion;
Version commandVersion = ShuttleCommandAPI.APIVersion;
Version uiVersion = ShuttleUIAPI.APIVersion;
```

For copied SDK surfaces and optional host services, prefer feature keys:

```csharp
bool hasRuntime = ShuttleExternalSDK.IsFeatureSupported(
    ShuttleExternalSDKFeatureKeys.ExternalRuntime);
bool hasCommands = ShuttleExternalSDK.IsFeatureSupported(
    ShuttleExternalSDKFeatureKeys.ExternalCommands);
bool hasPanels = ShuttleExternalSDK.IsFeatureSupported(
    ShuttleExternalSDKFeatureKeys.ExternalPanels);
bool hasCopiedSnapshots = ShuttleExternalSDK.IsFeatureSupported(
    ShuttleExternalSDKFeatureKeys.ExternalReadSnapshots);
```

Treat a missing feature or unavailable provider as a normal compatibility case.
Keep reflection into internal controller or cargo state out of normal SDK
integrations.

## Namespaces And Reference

Reference the main mod assembly:

```text
Assemblies/CeleTech_Shuttle.dll
```

Use public namespaces only:

```csharp
using CeleTech.ShuttleExtension.ModularShuttle.API.Profile;
using CeleTech.ShuttleExtension.ModularShuttle.API.Runtime;
using CeleTech.ShuttleExtension.ModularShuttle.API.Commands;
using CeleTech.ShuttleExtension.ModularShuttle.API.UI;
using CeleTech.ShuttleExtension.ModularShuttle.API.SDK;
using CeleTech.ShuttleExtension.ModularShuttle.Profile.Extensions;
```

Third-party mods should stick to public API namespaces and avoid
`Runtime.Modules`, `Core`, `AssemblyState`, `RuntimeState`, `UI.External`, and
concrete controller types.

## Load Order

Third-party mods should load after the main shuttle mod:

```xml
<loadAfter>
  <li>LongRange.CeleTech.ShuttleExtension</li>
</loadAfter>
```

## Key Rules

All public registration keys use:

```text
ownerPackageId/local-key
```

Examples:

```text
my.cool.mod/alien-scanner-profile
my.cool.mod/alien-scanner-runtime
my.cool.mod/alien-scanner-panel
my.cool.mod/set-scan-mode
```

Rules:

- Use your real package id as `ownerPackageId`.
- Keep local keys stable after release.
- Keys are trimmed but not lower-cased.
- Duplicate registration is first-wins.
- Commands and panels may only operate on runtime keys owned by the same
  package id.

## Profile Contributor

Profile contributors add static profile facts while the main mod builds a
derived shuttle profile. Keep runtime state, cargo, launch, UI, and save-data
changes out of profile contributors.

Register:

```csharp
ShuttleProfileAPI.RegisterProfileContributor(
    "my.cool.mod",
    "alien-scanner-profile",
    new AlienScannerProfileContributor());
```

Bind XML:

```xml
<li Class="CeleTech.ShuttleExtension.ModularShuttle.Extensions.ShuttleProfileContributorDefExtension">
  <contributorKey>my.cool.mod/alien-scanner-profile</contributorKey>
</li>
```

Use `ModuleView` and `ParentSegmentView`. Live `ShuttleModule`,
`ShuttleSegment`, `ModuleDef`, and `ParentSegmentDef` are not public SDK data.

If XML declares a `contributorKey`, it replaces the built-in profile contributor
for that module. If you need battery, cargo, navigation, or other built-in
profile effects, add the equivalent contributions explicitly.

## Runtime Registration

Register an external runtime system from a static bootstrap class:

```csharp
ShuttleRuntimeAPI.RegisterRuntimeSystem(
    "my.cool.mod",
    "alien-scanner-runtime",
    new AlienScannerRuntimeSystem(),
    "MyCoolMod_AlienScanner_Runtime_Label",
    "MyCoolMod_AlienScanner_Runtime_Description");
```

Bind module XML to the full runtime key:

```xml
<li Class="CeleTech.ShuttleExtension.ModularShuttle.Extensions.ShuttleRuntimeSystemDefExtension">
  <runtimeSystemKey>my.cool.mod/alien-scanner-runtime</runtimeSystemKey>
</li>
```

XML is a key binding only. It does not instantiate runtime classes.

Prefer inheriting from `ShuttleExternalModuleRuntimeSystemBase` instead of
implementing `IShuttleExternalModuleRuntimeSystem` directly. The base class lets
future SDK versions add optional hooks without breaking your implementation.

## Runtime State

External runtime state is saved by the main mod as an envelope:

- `ownerPackageId`
- `runtimeSystemKey`
- `schemaVersion`
- `initialized`
- string-string `values`

Use simple key-value state:

```csharp
context.State.SetInt("scanCount", scanCount + 1);
context.State.SetFloat("heat", heat);
context.State.SetBool("installed", true);
context.State.SetString("mode", "deep-scan");
```

Rules:

- Keys are trimmed and case-sensitive.
- Empty keys fail safely.
- `SetString(key, null)` removes the value.
- Int, float, and bool use invariant-culture string storage.
- Store values in the external state envelope instead of saving third-party
  runtime state classes with `Scribe_Deep`.

Writable state is available in:

- `ShuttleExternalRuntimeContext.State`
- `ShuttleExternalCommandContext.State`

Read-only state is available in:

- `ShuttleExternalPowerDemandContext.State`
- `ShuttleExternalMassContributionContext.State`
- `ShuttleExternalLaunchValidationContext.State`
- `ShuttleExternalLaunchContext.State`
- `ShuttleExternalModulePanelContext.State`

## Runtime Hooks

Map-side hooks:

| Hook | Can write state? | Purpose |
| --- | --- | --- |
| `Initialize` | Yes | Set first-run defaults for one state envelope. |
| `Migrate` | Yes | Upgrade old schema values. Schema updates only after success. |
| `Reconcile` | Yes | Clamp and align state after load, profile changes, or binding changes. |
| `CollectPowerDemand` | No | Add copied power demand through the context. |
| `CollectMassContribution` | No | Add transient mass that belongs to this runtime. |
| `Tick` | Yes | Run periodic behavior according to `TickInterval`. |
| `CanRemove` | Read-only by convention | Block unsafe removal with a player-readable reason. |
| `OnInstalled` | Yes | React after the module is installed. |
| `OnRemoved` | Cleanup only | Stop external effects before the state record is cleaned up. |
| `OnArrived` | Yes | Re-sync map-side adapters after arrival. |

Launch-side hooks:

| Hook | Can write state? | Purpose |
| --- | --- | --- |
| `PreLaunchValidate` | No | Return `false` with a reason to block launch. |
| `OnLaunchSucceeded` | No | Observe successful launch and clean map-only data. |

`TickInterval <= 0` disables ticking, but it does not disable reconcile or
lifecycle hooks.

## Power And Mass

Use `CollectPowerDemand` for ongoing internal power demand. Use
`TryConsumeStoredEnergyWd` from runtime or command contexts when a runtime action
spends shuttle-owned stored energy.

Use `CollectMassContribution` for stateful contents that are not normal cargo
but should count against load, range, and launch readiness:

```csharp
public override void CollectMassContribution(
    ShuttleExternalMassContributionContext context)
{
    float cleanWaterKg = context.State.GetFloat("cleanWaterLiters", 0f);
    float sewageKg = context.State.GetFloat("sewageLiters", 0f);

    context.AddMassKg(cleanWaterKg, "MyAddon_CleanWaterMass", "clean water");
    context.AddMassKg(sewageKg, "MyAddon_SewageMass", "sewage");
}
```

Avoid creating hidden cargo objects for the same contents. UI display and mass
contribution should use the same helper so shown mass and launch mass do not
drift apart.

## Host And Occupant Reads

Runtime contexts may expose read-only host and occupant DTOs. These are for
integration logic, not ownership transfer.

Use host and pawn references for read-only service checks. Keep occupant
containment changes, such as moving, unloading, destroying, re-parenting, or
reserving pawns, outside this read surface. Use role flags and
`CanReceiveExternalService` when selecting pawns for external service work.

## Command API

Register a command handler:

```csharp
ShuttleCommandAPI.RegisterCommandHandler(
    "my.cool.mod",
    "set-scan-mode",
    new SetScanModeCommandHandler());
```

The full command key is:

```text
my.cool.mod/set-scan-mode
```

`ShuttleExternalCommand` contains:

- `CommandKey`
- `RuntimeSystemKey`
- `ModuleInstanceID`
- string-string `Arguments`

The command adapter validates that the command owner matches the runtime owner
and that the target module binds the target runtime key. Command handlers can
write only the target external runtime state store.

## UI Panel Provider

Register a panel provider:

```csharp
ShuttleUIAPI.RegisterModulePanelProvider(
    "my.cool.mod",
    "alien-scanner-panel",
    new AlienScannerPanelProvider());
```

Panel providers draw only inside the External Modules page detail host. They
receive:

- `PanelKey`
- `RuntimeSystemKey`
- detached `Module`
- read-only `State`
- `RuntimeEnabled`
- `TicksGame`
- optional `CommandExecutor`

Providers should stay inside the host rectangle, avoid independent RimWorld
windows, route state changes through commands, and avoid internal UI host
classes.

## Panel Commands

The preferred pattern is host-rendered commands:

```csharp
sink.AddCommand(new ShuttleExternalPanelCommandContribution(
    "set-deep-scan",
    "Deep Scan",
    "Switch scanner to deep scan mode.",
    "my.cool.mod/set-scan-mode",
    args,
    context.RuntimeEnabled,
    context.RuntimeEnabled ? null : "Runtime is disabled.",
    10));
```

The provider declares the button. The host draws it, validates ownership, creates
`ShuttleExternalCommand`, and routes it to the registered command handler.

If a panel draws its own compact control and `context.CommandExecutor` is not
null, it may use the same contribution object:

```csharp
string disabledReason;
if (context.CommandExecutor != null &&
    context.CommandExecutor.CanExecute(contribution, out disabledReason))
{
    context.CommandExecutor.Execute(contribution);
}
```

This is still the controlled command path, not a shortcut for mutating state
from `DrawPanel`.

## Metrics

The main mod measures external runtime cost. Third-party code does not report
its own timings.

Current diagnostics may include:

- latest external power demand watts,
- latest external mass contribution rows in DevMode,
- tick rolling average ms,
- tick lifetime/session peak ms,
- cumulative stored-energy consumption,
- reconcile and power-demand pass timing in DevMode.

Metrics are in-memory diagnostics. They are not saved and are not part of the
third-party state contract.

## Main Mod Ownership Boundary

The public SDK boundary leaves these operations to the main mod:

- storing `ShuttleController` references;
- changing `ShuttleRuntimeState` or `ShuttleAssemblyState`;
- changing live `ShuttleModule` or `ShuttleSegment` instances;
- saving third-party runtime state classes;
- opening arbitrary RimWorld windows from panel providers;
- drawing into arbitrary main-page rectangles;
- writing runtime state from panel drawing code;
- operating commands or panels across another owner package id;
- instantiating runtime systems from XML class names;
- changing cargo, holders, launch services, or power systems through internal
  reflection or Harmony patches for normal SDK work.

Use profile contributors, runtime systems, command handlers, panel providers,
and copied SDK facade methods instead.

## Minimal Release Checklist

- [ ] Your add-on loads after `LongRange.CeleTech.ShuttleExtension`.
- [ ] You compile against `Assemblies/CeleTech_Shuttle.dll`.
- [ ] Every key uses `ownerPackageId/local-key`.
- [ ] Runtime and command keys are stable after release.
- [ ] XML binds full runtime/contributor keys.
- [ ] Runtime state uses the main mod key-value envelope only.
- [ ] `Migrate` handles old schema values.
- [ ] `PreLaunchValidate` is read-only and fails closed when required state is
      missing.
- [ ] Panel UI reads only detached DTOs and read-only state.
- [ ] Panel buttons use `CollectCommands` or `context.CommandExecutor`.
- [ ] Command handlers validate arguments and handle disabled/unavailable state.
- [ ] Your integration stays out of controller, assembly state, cargo backend,
      holders, and launch services.
- [ ] The sample or your add-on passes `Docs/ThirdPartyRuntimeTestingChecklist.md`.

## Quick Reference

The recommended integration path for a third-party module is:

1. Register runtime, profile, command, and panel objects from C#.
2. Bind XML to full keys such as `my.cool.mod/alien-scanner-runtime`.
3. Store runtime values in the main mod's key-value envelope instead of saving a
   custom runtime state class.
4. Read snapshots/state from UI; route writes through commands or cargo
   transactions.
5. Declare ordinary provider buttons through `CollectCommands`. If the host
   supplies `context.CommandExecutor`, the provider may also use the same
   `ShuttleExternalPanelCommandContribution` for its own compact controls.
6. Use public facades and commands instead of depending on controller,
   `AssemblyState`, `RuntimeState`, cargo backend, holder, or launch service
   internals.
