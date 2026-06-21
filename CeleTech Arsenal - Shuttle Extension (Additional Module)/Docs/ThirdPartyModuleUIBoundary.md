# Third-Party Module UI Boundary

Third-party UI is hosted by the main shuttle UI. Add-ons may draw inside the
External Modules detail host, while the surrounding page, module list,
navigation, and layout stay with the main UI.

## Allowed

- Register a panel provider through `ShuttleUIAPI.RegisterModulePanelProvider`.
- Draw inside the `Rect` passed to `DrawPanel`.
- Read detached module facts and read-only external runtime state from
  `ShuttleExternalModulePanelContext`.
- Declare host-rendered buttons through `CollectCommands`.
- Use `context.CommandExecutor` when it is present and the provider draws its
  own compact command control.
- Use translated labels and descriptions supplied by the add-on.

## Provider Boundary

- Opening independent RimWorld windows from a panel provider.
- Drawing outside the assigned rectangle.
- Injecting arbitrary controls into the main shuttle page.
- Loading arbitrary icon paths through the provider.
- Direct runtime state writes from UI drawing code.
- Calling internal UI host classes, controllers, or runtime buckets.
- Cross-owner runtime, panel, or command operation.

## Current Host Behavior

- The main page draws installed module icons and external badges.
- The External Modules page owns navigation, selection, command strip placement,
  provider chrome, and error guarding.
- Provider exceptions from `CanShow`, `GetPreferredHeight`, `DrawPanel`, or
  `CollectCommands` are guarded so the main UI remains usable.
- Provider command contributions are validated before display/execution.

## Authoring Note

A third-party provider draws its own content inside the rectangle supplied by
the main mod. The main page, module list, navigation, button layout, and error
guarding remain under the main mod's control.

For buttons, prefer `CollectCommands` so the host can draw and execute them. If
the provider draws its own compact button, use `context.CommandExecutor` when it
is available and keep the request on the controlled
`ShuttleExternalPanelCommandContribution` path.
