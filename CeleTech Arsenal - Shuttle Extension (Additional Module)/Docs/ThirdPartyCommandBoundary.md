# Third-Party Command Boundary

External commands are the supported write path for third-party runtime state.
They keep the write surface small on purpose: a command handler can see the
target command, detached module info, the target runtime's key-value state
store, current tick, internal bus power state, and the stored-energy consume
helper.

The command context keeps `ShuttleController`, `ShuttleRuntimeState`,
`ShuttleAssemblyState`, cargo backends, holders, launch services, and live
module objects outside the public boundary.

## Supported Flow

```text
Panel/provider UI
  -> ShuttleExternalPanelCommandContribution
    -> host command strip or context.CommandExecutor
      -> ShuttleExternalCommand
        -> registered IShuttleExternalCommandHandler
          -> target external runtime key-value state
```

Use this flow for player-facing state changes such as mode switches, toggles,
thresholds, scan requests, or other add-on-owned runtime settings.

## Provider Rules

- Prefer `CollectCommands` for ordinary buttons. The host draws the button and
  executes it through the command boundary.
- If `ShuttleExternalModulePanelContext.CommandExecutor` is present, a provider
  that draws its own compact control may execute the same
  `ShuttleExternalPanelCommandContribution` through that executor.
- Treat a missing `CommandExecutor` as "host-rendered commands only".
- Keep runtime state writes out of `DrawPanel`; route them through a command
  handler.
- Keep commands within the same owner package id as the target runtime.
- Use the host command strip or `context.CommandExecutor` instead of opening a
  separate window for command execution.

## Handler Rules

- Validate every argument.
- Re-check runtime enabled state and module binding through the supplied context.
- Clamp numeric input in the handler, even if the UI already clamps it.
- Return structured failure instead of throwing for expected invalid input.
- Avoid retry loops and log spam.
- Use `TryConsumeStoredEnergyWd` only for deliberate command-side energy costs.

## Authoring Note

When a third-party module needs to write state, wrap the change as a command and
let the main mod perform the `ExternalModuleRuntimeState` write. The main mod
validates the owner, runtime key, and module binding before calling the
registered command handler.

For ordinary buttons, prefer `CollectCommands` so the host can draw and execute
them. If a provider needs to draw a compact custom button and
`context.CommandExecutor` is available, it can pass the same
`ShuttleExternalPanelCommandContribution` to the executor. The executor still
uses the main mod's command boundary and leaves final state writes with the main
mod.
