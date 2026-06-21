# Shuttle Architecture Overview

This document gives third-party authors a conceptual map of the modular shuttle
system. It is not a source-code tour. The goal is to explain why the public SDK
has narrow read/write boundaries and where add-ons are expected to connect.

## Core Idea

The shuttle is treated as one coordinated vehicle with installed segments,
modules, cargo, occupants, runtime services, and launch state. Individual
modules can add capability, but the main shuttle mod remains responsible for
keeping the vehicle internally consistent.

At a high level:

```text
RimWorld thing / comp
  -> ShuttleController
    -> Assembly state
    -> Derived profile
    -> Runtime state
    -> Runtime systems
    -> Commands and read models
    -> UI / SDK snapshots
```

Third-party add-ons should think in terms of declaration, snapshots, and
controlled requests:

- declare static capability through profile contributors;
- declare runtime behavior through registered runtime systems;
- read copied snapshots and detached DTOs;
- write add-on state through commands;
- use documented cargo/launch SDK transactions where available.

## Host And Controller

The RimWorld map object owns the shuttle comp. The comp owns the
`ShuttleController`.

The controller coordinates the shuttle's internal model:

- installed segments and modules;
- profile rebuilds;
- runtime system scheduling;
- cargo and launch services;
- read model construction;
- command routing;
- save/load reconciliation.

The controller is intentionally not a public SDK object. External code should
not store or call it directly. Public APIs expose smaller, stable views and
request paths instead.

## Three State Layers

The shuttle model is split into three major layers.

### Assembly State

Assembly state is the saved installation graph:

- which segments exist;
- which modules are installed;
- which slots they occupy;
- stable module instance IDs;
- static per-instance configuration.

Assembly state answers "what is installed?" It is changed by main mod commands,
not by third-party runtime hooks.

### Derived Profile

The profile is a rebuilt snapshot derived from assembly state and registered
contributors. It answers "what capabilities does this shuttle currently have?"

Examples:

- power capacity and generation;
- cargo capacity;
- range and launch cost inputs;
- crew and service capability;
- module labels, warnings, and requirements;
- add-on profile metrics or requirements.

The profile is derived data. It is rebuilt when installation facts or relevant
static configuration change. Third-party profile contributors add facts to this
layer, but they do not mutate assembly or runtime state.

### Runtime State

Runtime state is saved mutable state for behavior that changes during play:

- stored energy and cooldowns;
- module runtime values;
- add-on key-value state envelopes;
- service counters;
- mode switches;
- temporary contents that contribute mass.

Third-party runtime modules receive a key-value state envelope owned and saved
by the main mod. They should store primitive values there instead of saving
custom runtime state classes.

## Runtime Systems

Runtime systems are scheduled by the main mod. They can reconcile state, tick,
collect power demand, collect mass contribution, validate removal, and respond
to lifecycle events.

Built-in runtime systems handle core shuttle behavior. Third-party runtime
systems are registered through the public runtime API and bound from XML by
stable keys.

External runtime systems are useful for add-on-owned behavior such as:

- a scanner mode and scan cooldown;
- a hygiene suite water tank;
- an auxiliary service module;
- a custom warning or requirement;
- an external panel with controlled commands.

Runtime hooks are not general-purpose access to the controller, cargo backend,
launch service, or assembly graph. They operate through the context supplied by
the main mod.

## Commands And Writes

The main mod uses commands for player-facing writes. Commands make state changes
auditable and keep UI code from writing directly into model objects.

For third-party runtime state, the supported write path is:

```text
Panel/provider UI
  -> ShuttleExternalPanelCommandContribution
    -> host command strip or context.CommandExecutor
      -> ShuttleExternalCommand
        -> registered IShuttleExternalCommandHandler
          -> target external runtime key-value state
```

This command boundary validates owner package IDs, runtime keys, module
bindings, runtime availability, and command arguments before the handler writes
state.

Cargo and launch state have their own SDK surfaces. Use cargo quote/consume or
quote/deposit transactions where available, and treat launch read/quote as
copied readiness data unless a future API explicitly exposes more.

## Read Models And Snapshots

The shuttle UI and public SDK use copied read models rather than live internal
objects. This keeps add-ons compatible with save/load, despawn, launch transfer,
and UI refresh timing.

Common read surfaces include:

- detached module facts;
- read-only external runtime state;
- copied cargo rows and query results;
- launch readiness and issue rows;
- integration health diagnostics;
- host and occupant snapshots for supported services.

Snapshots can become stale immediately after they are read. Re-read before
making decisions that depend on current cargo, power, cooldown, occupant, or
launch state.

## UI Hosting

The main shuttle UI owns page layout, navigation, module selection, command
strip placement, error guarding, and refresh timing.

Third-party panels are hosted inside the External Modules detail area. A panel
provider can draw inside its assigned rectangle and declare commands. It should
not assume ownership of the surrounding page or open separate windows to modify
state.

This model lets add-ons present custom controls without making the main shuttle
UI dependent on arbitrary external layout code.

## Cargo And Launch Boundaries

Cargo and launch are coordinated systems with high consistency requirements:

- cargo rows may represent holders, filters, assignments, regions, or transfer
  timing;
- launch readiness depends on mass, power, cooldown, damage, occupants, cargo,
  and destination rules;
- transfer and recovery can temporarily make mutation unsafe.

For that reason, the public SDK exposes copied cargo read/query data, bounded
cargo transactions, launch read/quote, and advisory launch rule rows. Direct
holder, transporter, cargo backend, or launch service mutation is outside the
public boundary.

## External SDK Boundary

The public SDK is intentionally shaped around small stable contracts:

- registration facades;
- feature keys;
- copied DTOs;
- key-value external state;
- command handlers;
- hosted UI providers;
- bounded cargo transaction providers;
- read-only launch quote/read providers.

This keeps third-party modules usable across main mod updates and makes failure
states easier to diagnose. If an integration needs a capability that is outside
the current SDK, treat it as a candidate for a new public API rather than
reaching into internal objects through reflection or Harmony patches.

## Practical Authoring Model

For a third-party module, the usual flow is:

1. Define XML for a shuttle module and bind it to stable full keys.
2. Register C# profile, runtime, command, and panel objects during mod startup.
3. Store runtime values in the main mod external state envelope.
4. Use runtime hooks for add-on-owned behavior.
5. Use panel commands for player-facing state changes.
6. Use copied SDK snapshots for read decisions.
7. Use cargo transactions only through documented SDK quote/consume/deposit
   surfaces.
8. Keep launch checks read-only unless a future SDK explicitly exposes a write
   or execution path.

## Document Map

| Need | Read |
| --- | --- |
| Runtime modules, state, commands, panels | `Docs/ThirdPartyRuntimeSdk.md` |
| Copied SDK snapshots, cargo, launch | `Docs/ExternalSDK_P1_Usage_Guide.md` |
| Command routing rules | `Docs/ThirdPartyCommandBoundary.md` |
| UI provider rules | `Docs/ThirdPartyModuleUIBoundary.md` |
| Release verification | `Docs/ThirdPartyRuntimeTestingChecklist.md` |
| SDK version changes | `Docs/ExternalSDK_P1_Changelog.md` |
