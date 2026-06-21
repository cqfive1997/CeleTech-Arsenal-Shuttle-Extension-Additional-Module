# Third-Party Runtime Testing Checklist

Use this checklist before treating a third-party runtime integration as
release-ready. Run functional checks in RimWorld with DevMode enabled when
possible.

## Registration And XML

- [ ] Third-party runtime registers without a bound module XML key.
- [ ] Registration appears in DevMode diagnostics.
- [ ] No external state is created when no module binds the key.
- [ ] Module XML with an unregistered runtime key shows missing-runtime
      diagnostics without breaking the UI.
- [ ] Module XML with a registered runtime key appears in External Modules.
- [ ] Main-page external badge opens External Modules and selects the target.

## State And Migration

- [ ] Runtime sync creates an `ExternalModuleRuntimeState` envelope.
- [ ] `ownerPackageId`, `runtimeSystemKey`, `schemaVersion`, `initialized`, and
      values persist through save/load.
- [ ] `Initialize` runs once per state envelope.
- [ ] `Migrate` runs only when saved schema is older than current schema.
- [ ] Failed migration keeps the old schema and logs a useful message.
- [ ] Disabling the add-on does not cause missing-class save failure.

## Runtime Hooks

- [ ] `Reconcile` can clamp and repair the add-on's own key-value state.
- [ ] `Tick` follows `TickInterval` and stops while runtime is disabled.
- [ ] Runtime exceptions are isolated and do not stop built-in systems.
- [ ] `CollectPowerDemand` adds demand only while enabled.
- [ ] `CollectMassContribution` adds positive kg only while enabled.
- [ ] `TryConsumeStoredEnergyWd` fails safely for invalid values or unavailable
      sinks.
- [ ] `CanRemove` can block removal with a visible reason.
- [ ] `OnInstalled`, `OnRemoved`, and `OnArrived` run in the expected lifecycle
      order.

## Launch Hooks

- [ ] `PreLaunchValidate` runs only for matching registered runtime keys.
- [ ] It can block launch with a player-readable reason.
- [ ] It is read-only and does not create or migrate state.
- [ ] Exceptions fail closed and include runtime/module context in logs.
- [ ] Disabled runtime does not block launch.
- [ ] `OnLaunchSucceeded` runs only after successful launch and does not roll
      launch back.

## UI Panel Provider

- [ ] Registered provider appears only for its own runtime key.
- [ ] Context exposes detached module info, read-only state, runtime enabled
      flag, ticks, and optional command executor.
- [ ] Provider cannot cast state to `IShuttleExternalRuntimeStateStore`.
- [ ] Provider exceptions from `CanShow`, `GetPreferredHeight`, `DrawPanel`, or
      `CollectCommands` are guarded.
- [ ] Provider does not open independent windows or draw outside its rectangle.

## Panel Commands

- [ ] `CollectCommands` adds a valid command contribution.
- [ ] Host draws the button and sends `ShuttleExternalCommand`.
- [ ] Command handler receives writable target state.
- [ ] Disabled contribution shows a disabled button/reason.
- [ ] Cross-owner command keys are rejected.
- [ ] If `context.CommandExecutor` is used, it follows the same validation path
      as host-rendered commands.
- [ ] Handler failures do not fake-update UI state.

## Copied SDK Surfaces

- [ ] Feature keys are checked before optional SDK calls.
- [ ] Read snapshots are treated as stale immediately.
- [ ] Cargo query handles truncation/completeness flags.
- [ ] Cargo quote happens before consume/deposit.
- [ ] Cargo transaction failure reasons are handled without log spam.
- [ ] Launch quote is treated as destination-independent in SDK `1.7.0`.
- [ ] Launch rule provider blockers are understood as SDK-readiness-only in
      SDK `1.7.0`.

## Boundary Audit

- [ ] Sample/add-on code uses only public API namespaces.
- [ ] No code references `ShuttleController`.
- [ ] No code references `ShuttleRuntimeState`.
- [ ] No code references `ShuttleAssemblyState`.
- [ ] No provider writes runtime state directly from UI drawing code.
- [ ] No provider executes another owner's command.
- [ ] No runtime hook mutates cargo, holders, launch services, or assembly
      topology directly.

## Final Check Notes

- Test registration, XML binding, state envelopes, migration, enable/disable,
  UI providers, and command handlers as separate behaviors.
- External state should survive loading after the add-on is disabled, and the
  same runtime key should recover old values after the add-on is enabled again.
- UI reads state and routes writes through commands. Cargo writes go through SDK
  transactions. Launch checks stay read-only.
- Snapshots and quotes can become stale immediately; treat them as copied data,
  not locks or reservations.
