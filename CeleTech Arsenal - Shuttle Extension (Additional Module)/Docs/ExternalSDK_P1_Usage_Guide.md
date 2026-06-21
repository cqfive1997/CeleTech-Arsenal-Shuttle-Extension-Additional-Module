# External SDK P1 Usage Guide

## Overview

This guide is the P1 External SDK reference for third-party mod authors.
It covers feature discovery, safe host access, read snapshots, profile
expressions, cargo read/query, cargo quote/consume, cargo quote/deposit, and
cargo transaction diagnostics, destination-independent launch read/quote, and
advisory launch rule providers.

This guide stays focused on the SDK surfaces that are available now. It leaves
actual launch validation hooks, transfer APIs, reservation APIs, live cargo
handles, UI providers, pawn services, destination-specific launch, launch
execution, and combat/power/job providers to future design work.

## Document Map

Start here if you need copied read data, cargo transactions, launch read/quote,
or launch rule providers. Use the related documents below for the authoring
surfaces that sit outside this copied-data SDK:

| Need | Read |
| --- | --- |
| Register an external runtime module, state, lifecycle hooks, power/mass hooks, commands, or panels | `Docs/ThirdPartyRuntimeSdk.md` |
| Check UI host and panel rules | `Docs/ThirdPartyModuleUIBoundary.md` |
| Check external command routing rules | `Docs/ThirdPartyCommandBoundary.md` |
| Verify a third-party integration before release | `Docs/ThirdPartyRuntimeTestingChecklist.md` |
| Review SDK version changes | `Docs/ExternalSDK_P1_Changelog.md` |

In short: use this guide for safe copied SDK data and bounded cargo/launch
facades; use `ThirdPartyRuntimeSdk.md` when your add-on installs a runtime
module into shuttle slots.

## Supported Version

Current SDK API version:

```text
1.7.0
```

Check the runtime version before using optional surfaces:

```csharp
Version sdkVersion = ShuttleExternalSDK.APIVersion;
ShuttleExternalSDKInfo info = ShuttleExternalSDK.GetSDKInfo();
```

P1 stability notes:

- Public SDK DTOs are copied data snapshots, not live Verse objects.
- Public DTOs keep the shuttle controller, cargo backend, command executor,
  `ThingOwner`, `Thing`, `Pawn`, `Map`, `CompTransporter`, `ThingDef`,
  `ThingFilter`, and `TransferableOneWay` outside the SDK boundary.
- Snapshots can become stale immediately after they are read.
- Quote before you mutate cargo.
- Read every transaction result as structured data and handle each failure path.

## Feature Matrix

Use `ShuttleExternalSDK.IsFeatureSupported(...)` or
`ShuttleExternalSDK.GetSDKInfo().IsFeatureSupported(...)` before using optional
surfaces.

Supported in `1.7.0`:

| Feature key | Since | Provider / type | Access | What it does | Major caveat |
| --- | --- | --- | --- | --- | --- |
| `external.sdk.discovery` | P1 baseline | `ShuttleExternalSDK`, `ShuttleExternalSDKInfo` | Read-only | SDK version, mod version, compatibility, and feature discovery. | Feature checks are still required for optional surfaces. |
| `external.integration.health` | P1 baseline | `IShuttleExternalIntegrationHealthProvider` | Read-only | Coarse integration registration/runtime health. | Support data, not gameplay truth. |
| `external.read_snapshots` | P1 baseline | `IShuttleExternalReadSnapshotProvider` | Read-only | Broad copied host, assembly, profile, runtime, power, cargo, launch, and health snapshot. | Snapshots can become stale immediately. |
| `external.profile.capabilities` | `1.2.0` | `ShuttleExternalProfileCapabilitySnapshot` | Read-only | Data-only profile capability rows. | Does not grant behavior rights. |
| `external.profile.metrics` | `1.2.0` | `ShuttleExternalProfileMetricSnapshot` | Read-only | Data-only numeric/text profile metric rows. | Derived profile facts only. |
| `external.profile.requirements` | `1.2.0` | `ShuttleExternalProfileRequirementSnapshot` | Read-only | Data-only profile warning/error/blocker requirement rows. | Not a cargo or launch API. |
| `external.profile.contribution_sources` | `1.2.0` | `ShuttleExternalProfileContributionSourceSnapshot` | Read-only | Source tracing for external profile expressions. | No live contributor object is exposed. |
| `external.cargo.read` | `1.3.0` | `IShuttleExternalCargoReadProvider` | Read-only | Copied cargo snapshot rows and summary values. | Includes read-only assigned/blocked/refrigerated/queued states. |
| `external.cargo.query` | `1.3.0`; truncation fields `1.3.1` | `ShuttleExternalCargoQuery` | Read-only | Query copied cargo rows by def/category/stuff/quality flags. | Respect truncation and completeness flags. |
| `external.cargo.transaction` | `1.4.0` | `IShuttleExternalCargoTransactionProvider` | Write-capable | Discovery seam for quote/consume/deposit. | Write scope is only the supported normal loaded cargo paths. |
| `external.cargo.consume` | `1.4.0` | `TryConsume`, `ShuttleExternalCargoConsumeRequest` | Write-capable | Consume normal loaded cargo. | Excludes refrigerated, queued, assigned, blocked, pawn, and corpse cargo. |
| `external.cargo.consume.quote` | `1.4.0` | `TryQuoteConsume` | Read-only | Dry-run consume quote. | Quote can become stale before consume. |
| `external.cargo.deposit` | `1.5.0` | `TryDeposit`, `ShuttleExternalCargoDepositRequest` | Write-capable | Deposit internally-created plain item stacks into normal loaded cargo. | No live item handles, stuff, quality, partial, pawn, corpse, or refrigerated deposit. |
| `external.cargo.deposit.quote` | `1.5.0` | `TryQuoteDeposit` | Read-only | Dry-run deposit quote. | Quote can become stale before deposit. |
| `external.cargo.transaction.diagnostics` | `1.5.1` | `IShuttleExternalCargoTransactionDiagnosticsProvider` | Read-only | Bounded non-persistent transaction diagnostics. | Support data only; use cargo read/query for current cargo truth. |
| `external.launch.read` | `1.6.0` | `IShuttleExternalLaunchReadProvider` | Read-only | Copied launch readiness, payload, cost, cooldown, and issues. | Not a command, targeter, or launch permission API. |
| `external.launch.quote` | `1.6.0` | `TryQuoteLaunch` | Read-only | Destination-independent launch quote for current shuttle state. | Does not validate a world target. |
| `external.launch.readiness` | `1.6.0` | `ShuttleExternalLaunchReadinessSnapshot` | Read-only | Copied readiness fields and blocker/warning counts. | Actual launch validation remains authoritative. |
| `external.launch.issues` | `1.6.0` | `ShuttleExternalLaunchIssueSnapshot` | Read-only | Copied launch issue rows mapped from readiness facts. | Issue rows are diagnostics/readiness facts, not write handles. |
| `external.launch.rules` | `1.7.0` | External rule issue rows | Read-only/advisory | External rule warnings/blockers in Launch Read / Quote. | SDK readiness only; not actual launch enforcement. |
| `external.launch.rule_provider` | `1.7.0` | `IShuttleExternalLaunchRuleProvider` | Read-only/advisory | Register copied launch rule providers. | No targeter, destination, participant, or launch execution integration. |

Some lower-level runtime, command, panel, and unsafe compatibility keys may also
exist for legacy/module authoring surfaces. This guide focuses on the P1 copied
SDK surfaces above.

## Feature Discovery

Prefer feature checks over version-only checks:

```csharp
ShuttleExternalSDKInfo info = ShuttleExternalSDK.GetSDKInfo();

bool canReadCargo = info.IsFeatureSupported(
    ShuttleExternalSDKFeatureKeys.ExternalCargoRead);
bool canConsumeCargo = info.IsFeatureSupported(
    ShuttleExternalSDKFeatureKeys.ExternalCargoConsume);
bool canDepositCargo = info.IsFeatureSupported(
    ShuttleExternalSDKFeatureKeys.ExternalCargoDeposit);
bool canReadDiagnostics = info.IsFeatureSupported(
    ShuttleExternalSDKFeatureKeys.ExternalCargoTransactionDiagnostics);
bool canReadLaunch = info.IsFeatureSupported(
    ShuttleExternalSDKFeatureKeys.ExternalLaunchRead);
bool canQuoteLaunch = info.IsFeatureSupported(
    ShuttleExternalSDKFeatureKeys.ExternalLaunchQuote);
bool canRegisterLaunchRules = info.IsFeatureSupported(
    ShuttleExternalSDKFeatureKeys.ExternalLaunchRuleProvider);
```

Feature checks are especially important for optional host seams:

- Cargo read: `IShuttleExternalSDKCargoHost`
- Cargo transaction: `IShuttleExternalSDKCargoTransactionHost`
- Cargo transaction diagnostics:
  `IShuttleExternalSDKCargoTransactionDiagnosticsHost`
- Launch read/quote: `IShuttleExternalSDKLaunchReadHost`

Use the facade methods where possible. They perform the optional host cast and
provider null checks for you.

## Host Access Contract

Third-party integrations should obtain SDK services through safe host seams.
A shuttle host comp may implement `IShuttleExternalSDKHost`; optional surfaces
are then discovered through `ShuttleExternalSDK` facade methods.

For normal SDK use, avoid:

- reflecting or casting into `CompModularShuttleCore.Controller`;
- Harmony-patching internal controller/backend/holder fields;
- caching providers indefinitely when the host may despawn or be destroyed;
- treating snapshot rows as write handles.

Basic read snapshot access:

```csharp
using CeleTech.ShuttleExtension.ModularShuttle.API.SDK;

public void InspectShuttle(IShuttleExternalSDKHost sdkHost)
{
    if (sdkHost == null)
    {
        return;
    }

    ShuttleExternalReadSnapshot snapshot;
    if (!ShuttleExternalSDK.TryGetReadSnapshot(sdkHost, out snapshot) ||
        snapshot == null ||
        !snapshot.Available)
    {
        return;
    }

    string hostLabel = snapshot.Host != null ? snapshot.Host.HostLabel : null;
    int cargoUnits = snapshot.Cargo != null ? snapshot.Cargo.LoadedUnitCount : 0;

    // Use copied snapshot data only. Re-read later if you need fresh state.
}
```

If a facade call returns `false`, treat the surface as unavailable for that host
at that moment.

## Read Snapshot Example

`ShuttleExternalReadSnapshot` is a broad read-only snapshot. It contains copied
host, assembly, profile, runtime, power, cargo, launch summary, and integration
health data.

```csharp
ShuttleExternalReadSnapshot snapshot;
if (ShuttleExternalSDK.TryGetReadSnapshot(sdkHost, out snapshot) &&
    snapshot != null &&
    snapshot.Available &&
    snapshot.Profile != null)
{
    float cargoCapacity = snapshot.Profile.CargoMassCapacityKg;
    int externalRequirements =
        snapshot.Profile.ExternalRequirementCount;
}
```

Snapshot rules:

- `Available == false` means the snapshot could not be built.
- A snapshot is not a lock or reservation.
- `HostStableId` and id fields are copied identifiers, not live handles.
- `Launch` is a read summary in this SDK version, not a launch quote or launch
  participant API.

## Launch Read And Quote

Launch read/quote is read-only. It returns copied readiness, payload, cost,
cooldown, and issue data for the shuttle's current state. It does not expose
launch commands, target selection, arrival actions, cargo holders, pawns, or
live game objects.

Main types:

- `IShuttleExternalLaunchReadProvider`
- `ShuttleExternalLaunchReadSnapshot`
- `ShuttleExternalLaunchReadinessSnapshot`
- `ShuttleExternalLaunchPayloadSnapshot`
- `ShuttleExternalLaunchCostSnapshot`
- `ShuttleExternalLaunchCooldownSnapshot`
- `ShuttleExternalLaunchIssueSnapshot`
- `ShuttleExternalLaunchRuleDiagnosticsSnapshot`
- `IShuttleExternalLaunchRuleProvider`
- `ShuttleExternalLaunchRuleProviderRegistry`
- `ShuttleExternalLaunchQuoteRequest`
- `ShuttleExternalLaunchQuoteResult`

Destination-independent quote example:

Safe pattern:

1. Check feature keys.
2. Get the launch read provider through the facade or optional host seam.
3. Read the current launch snapshot.
4. Quote launch with `IncludeDestinationQuote == false`.
5. Inspect `DestinationQuoteSupported` and `DestinationQuotePolicy`.
6. Inspect `RuleDiagnostics` if external launch rules are present.
7. Re-read before acting; a quote is not a reservation.

```csharp
if (!ShuttleExternalSDK.IsFeatureSupported(
        ShuttleExternalSDKFeatureKeys.ExternalLaunchRead) ||
    !ShuttleExternalSDK.IsFeatureSupported(
        ShuttleExternalSDKFeatureKeys.ExternalLaunchQuote))
{
    return;
}

ShuttleExternalLaunchReadSnapshot launch;
if (ShuttleExternalSDK.TryGetLaunchReadSnapshot(sdkHost, out launch) &&
    launch != null &&
    launch.Available &&
    launch.Readiness != null &&
    launch.Cost != null)
{
    bool canLaunchNow = launch.Readiness.CanLaunchNow;
    float requiredEnergy = launch.Cost.RequiredEnergyWd;
    float storedEnergy = launch.Cost.StoredEnergyWd;
    int sdkOnlyRuleBlockers = launch.ExternalRuleBlockerCount;
}

ShuttleExternalLaunchQuoteResult quote;
ShuttleExternalLaunchQuoteRequest request =
    new ShuttleExternalLaunchQuoteRequest(false);

if (ShuttleExternalSDK.TryQuoteLaunch(sdkHost, request, out quote) &&
    quote != null &&
    quote.Available)
{
    float requiredEnergy = quote.Cost.RequiredEnergyWd;
    float payloadMass = quote.Payload.PayloadMassKg;
    bool destinationQuoteSupported = quote.DestinationQuoteSupported;
    ShuttleExternalLaunchRuleDiagnosticsSnapshot ruleDiagnostics =
        quote.RuleDiagnostics;
}
```

Destination-specific quote is intentionally deferred in `1.7.0`. If
`IncludeDestinationQuote == true`, the SDK returns a copied result with
`DestinationSpecific == false`, `DestinationQuoteSupported == false`, and an
unavailable/deferred reason. It does not open a world targeter and does not
mutate launch, cargo, holder, pawn, power, cooldown, or arrival state.

Snapshot rules:

- `CanOpenLaunchFlow` means the current read model has no blocking issues for
  opening the vanilla launch flow. Callers should still use the normal command
  flow.
- `CanLaunchNow` is a copied readiness fact at read time. Re-read before acting.
- Cost values are based on current shuttle profile, runtime power, and cargo
  snapshot only.
- Issue rows are copied diagnostics/readiness facts. Registered launch rule
  providers may add SDK-only issue rows.
- External launch rule blockers can make SDK `CanLaunchNow` false, but
  `ExternalRulesAffectSdkReadinessOnly == true` means they are not enforced by
  the actual launch validator in SDK `1.7.0`.
- External runtime prelaunch callbacks are not invoked by launch read/quote.
  The internal launch flow remains authoritative at execution time.

## Launch Rule Provider

Launch rule providers add copied advisory, warning, or blocker rows to SDK
Launch Read / Quote snapshots. They receive a copied context built from the
base launch read snapshot before external rule rows are appended.

Feature-check before registering or relying on rule rows:

```csharp
if (!ShuttleExternalSDK.IsFeatureSupported(
        ShuttleExternalSDKFeatureKeys.ExternalLaunchRuleProvider))
{
    return;
}
```

Minimal provider example:

```csharp
using System.Collections.Generic;
using CeleTech.ShuttleExtension.ModularShuttle.API.SDK;

public sealed class MyLaunchRuleProvider : IShuttleExternalLaunchRuleProvider
{
    public ShuttleExternalLaunchRuleProviderInfo GetProviderInfo()
    {
        return new ShuttleExternalLaunchRuleProviderInfo(
            "my.cool.mod",
            "launch-checks",
            "My launch checks",
            "1.0.0",
            true,
            "Adds copied SDK launch warnings.");
    }

    public bool TryEvaluateLaunchRules(
        ShuttleExternalLaunchRuleContext context,
        out ShuttleExternalLaunchRuleResult result)
    {
        List<ShuttleExternalLaunchRuleIssue> issues =
            new List<ShuttleExternalLaunchRuleIssue>();

        if (context != null &&
            context.Payload != null &&
            context.Payload.PayloadMassKg > 500f)
        {
            issues.Add(new ShuttleExternalLaunchRuleIssue(
                "payload-heavy",
                ShuttleExternalLaunchRuleSeverity.Warning,
                false,
                null,
                "Payload is heavy for this integration.",
                null,
                null,
                "payload-heavy",
                null));
        }

        result = new ShuttleExternalLaunchRuleResult(
            true,
            null,
            null,
            "my.cool.mod",
            "launch-checks",
            ShuttleExternalLaunchRuleEvaluationStatus.Succeeded,
            issues,
            null);
        return true;
    }
}
```

Registration:

```csharp
string rejection;
bool registered =
    ShuttleExternalLaunchRuleProviderRegistry.TryRegister(
        new MyLaunchRuleProvider(),
        out rejection);
```

Provider rules:

- Use stable `OwnerPackageId` and `LocalProviderKey` values.
- The registry is deterministic; duplicate full provider keys are rejected and
  the first registration wins.
- Provider instances are not saved.
- SDK `1.7.0` does not expose rule provider enable/disable config. Register
  only providers you intend to evaluate; `EnabledByDefault` is metadata for
  discovery and future configuration.
- Context fields are copied SDK DTOs only.
- Provider exceptions are caught per provider and become bounded diagnostic
  issue rows/counters without failing the whole launch read snapshot.
- Issue/message counts and string lengths are capped.
- Rule providers affect SDK Launch Read / Quote issue rows only in `1.7.0`.
  Actual launch validation, target selection, transfer, cargo handoff, energy
  spending, cooldown writes, and arrival logic stay with the main launch flow.
- Rule issue rows use `SourceKind == ExternalLaunchRule` plus
  `FromExternalLaunchRuleProvider == true`, `EnforcedInActualLaunch == false`,
  and `AffectsSdkReadinessOnly == true`.

Rule diagnostics:

- Launch Read and Quote expose `RuleDiagnostics` plus mirrored summary fields:
  `ExternalRulesEvaluated`, `ExternalRuleIssueCount`,
  `ExternalRuleBlockerCount`, and `ExternalRulesAffectSdkReadinessOnly`.
- `RuleDiagnostics.ProviderRecords` identifies the provider key, owner package
  id, local provider key, display name, evaluation status, issue count,
  blocker count, and a bounded message for each evaluated provider record.
- `ProviderExceptionCount` and `InvalidResultCount` are support diagnostics.
  They are reported as SDK diagnostics; external runtime failure state and
  actual launch validation remain unchanged.
- `RecordsTruncated == true` means provider records or issue rows hit a safety
  cap. Re-read later for current data, and treat the snapshot as incomplete
  support data.
- External rule issue rows include provider/rule attribution through
  `IssueKey`, `SourceId`, and `ReferenceId`; these are copied strings, not
  provider instances or live game references.
- Keep providers cheap, deterministic, and read-only. Targeter, world/map,
  arrival, cargo transfer, holder transfer, energy, cooldown, and launch
  execution work should stay outside rule providers.

## Profile Expression Example

Profile expressions are data-only facts contributed during profile build. Cargo,
launch, UI, pawn, combat, power, and job mutation remain separate API surfaces.

The expression sink is available through profile contribution context:

```csharp
using CeleTech.ShuttleExtension.ModularShuttle.API.SDK;
using CeleTech.ShuttleExtension.ModularShuttle.Profile.Extensions;

public void Contribute(ShuttleProfileContributionContext context)
{
    if (context == null || context.ExternalExpressions == null)
    {
        return;
    }

    string rejection;
    context.ExternalExpressions.TryAddMetric(
        "my.cool.mod",
        "scanner-range",
        "MyCoolMod_ScannerRange_Label",
        "MyCoolMod_ScannerRange_Tooltip",
        12f,
        "tiles",
        out rejection);

    context.ExternalExpressions.TryAddRequirement(
        "my.cool.mod",
        "scanner-calibrated",
        ShuttleExternalProfileRequirementSeverity.Warning,
        "MyCoolMod_ScannerCalibrated_Message",
        "Scanner calibration is recommended before launch.",
        "MyCoolMod_ScannerCalibrated_Tooltip",
        false,
        false,
        out rejection);
}
```

Guidance:

- Use a stable owner package id, such as `my.cool.mod`.
- Keep local keys stable.
- If a row is rejected, handle it as a validation failure and leave that row out
  of your assumptions.
- Requirements become copied profile/read snapshot data. They are not broad
  launch rule providers unless a future API explicitly supports that.

## Cargo Read And Query

Cargo read/query is read-only. It returns copied DTO rows and summary values.
It does not expose cargo truth, cargo owners, or live item handles.

Main types:

- `IShuttleExternalCargoReadProvider`
- `ShuttleExternalCargoReadSnapshot`
- `ShuttleExternalCargoItemRowSnapshot`
- `ShuttleExternalCargoSummaryByDefSnapshot`
- `ShuttleExternalCargoQuery`
- `ShuttleExternalCargoQueryResult`

Important truncation fields:

- Snapshot source rows: `SourceRowCount`
- Snapshot returned rows: `ReturnedRowCount`
- Snapshot row truncation: `RowsTruncated`
- Summary completeness: `SummaryByDefComplete`
- Query evaluated rows: `EvaluatedRowCount`
- Query source truncation: `SourceRowsTruncated`
- Query returned rows: `ReturnedRowCount`
- Query result truncation: `ResultRowsTruncated`

If any source or result truncation flag is true, treat rows and counts as
partial over the returned/evaluated data.

Example: query loaded steel by defName.

```csharp
if (!ShuttleExternalSDK.IsFeatureSupported(
        ShuttleExternalSDKFeatureKeys.ExternalCargoQuery))
{
    return;
}

ShuttleExternalCargoQuery query = new ShuttleExternalCargoQuery(
    ShuttleExternalCargoQueryScope.LoadedOnly,
    "Steel",
    null,
    null,
    null,
    false,
    false,
    false,
    true,
    32);

ShuttleExternalCargoQueryResult result;
if (ShuttleExternalSDK.TryQueryCargo(sdkHost, query, out result) &&
    result != null &&
    result.Available)
{
    int steelCount = result.MatchedUnitCount;
    bool partial = result.SourceRowsTruncated || result.ResultRowsTruncated;

    // result.Rows are copied data, not write handles. Re-query before decisions
    // that depend on fresh state.
}
```

Cargo locations include loaded, assigned-to-load, blocked, refrigerated, and
queued/read-only states. P1 cargo transactions mutate only the supported normal
loaded cargo paths described below.

## Cargo Consume

Current supported consume behavior:

- Quote consume from normal loaded cargo.
- Consume normal loaded cargo.
- Optional filters: item defName, category defName, stuff defName, quality.
- Default constructor requests exact count.
- Partial consume is only valid when `AllowPartial == true` and
  `RequireExactCount == false`.
- Queued/assigned cargo is excluded.
- Blocked cargo is excluded.
- Refrigerated cargo is excluded.
- Pawn and corpse cargo are excluded.

Main types:

- `ShuttleExternalCargoTransactionSource`
- `ShuttleExternalCargoConsumeRequest`
- `ShuttleExternalCargoTransactionResult`

Safe pattern:

1. Check feature keys.
2. Read/query cargo if useful.
3. Build a source trace.
4. Quote consume.
5. If quote succeeds, call consume.
6. Handle structured failure.
7. Inspect diagnostics if failures repeat.

Example: consume 5 industrial components from loaded normal cargo.

```csharp
if (!ShuttleExternalSDK.IsFeatureSupported(
        ShuttleExternalSDKFeatureKeys.ExternalCargoConsume) ||
    !ShuttleExternalSDK.IsFeatureSupported(
        ShuttleExternalSDKFeatureKeys.ExternalCargoConsumeQuote))
{
    return;
}

ShuttleExternalCargoTransactionSource source =
    new ShuttleExternalCargoTransactionSource(
        "my.cool.mod",
        "fabricator",
        moduleInstanceId,
        "craft-batch");

ShuttleExternalCargoConsumeRequest request =
    new ShuttleExternalCargoConsumeRequest(
        source,
        "ComponentIndustrial",
        5);

ShuttleExternalCargoTransactionResult quote;
if (!ShuttleExternalSDK.TryQuoteCargoConsume(sdkHost, request, out quote) ||
    quote == null ||
    !quote.Success)
{
    // Inspect quote.FailureReason and quote.Message if quote is not null.
    return;
}

ShuttleExternalCargoTransactionResult consume;
if (!ShuttleExternalSDK.TryConsumeCargo(sdkHost, request, out consume) ||
    consume == null ||
    !consume.Success)
{
    // Cargo may have changed between quote and consume. Handle failure.
    return;
}
```

Avoid calling consume every tick. Use quotes, cooldowns, player intent, or your
own state machine to keep transaction attempts deliberate.

## Cargo Deposit

Current supported deposit behavior:

- Quote deposit into normal loaded cargo.
- Deposit internally-created plain item stacks into normal loaded cargo.
- Exact count is required.
- The target must be an ordinary item `ThingDef` by defName.
- No partial deposit.
- No caller-selected target region.
- No live item handles.
- No stuff support.
- No quality support.
- No hit point, comp state, rottable age, style, faction, or ideology data.
- No pawn, corpse, minified, building, plant, filth, or non-item deposit.

Main type:

- `ShuttleExternalCargoDepositRequest`

Safe pattern:

1. Check feature keys.
2. Build a source trace.
3. Quote deposit.
4. If quote succeeds, call deposit.
5. Handle failure reason.
6. Read cargo snapshot afterward if you need fresh cargo state.

Example: deposit 10 steel.

```csharp
if (!ShuttleExternalSDK.IsFeatureSupported(
        ShuttleExternalSDKFeatureKeys.ExternalCargoDeposit) ||
    !ShuttleExternalSDK.IsFeatureSupported(
        ShuttleExternalSDKFeatureKeys.ExternalCargoDepositQuote))
{
    return;
}

ShuttleExternalCargoTransactionSource source =
    new ShuttleExternalCargoTransactionSource(
        "my.cool.mod",
        "resource-printer",
        moduleInstanceId,
        "print-steel");

ShuttleExternalCargoDepositRequest request =
    new ShuttleExternalCargoDepositRequest(
        source,
        "Steel",
        10);

ShuttleExternalCargoTransactionResult quote;
if (!ShuttleExternalSDK.TryQuoteCargoDeposit(sdkHost, request, out quote) ||
    quote == null ||
    !quote.Success)
{
    return;
}

ShuttleExternalCargoTransactionResult deposit;
if (!ShuttleExternalSDK.TryDepositCargo(sdkHost, request, out deposit) ||
    deposit == null ||
    !deposit.Success)
{
    return;
}
```

`Steel` is a common RimWorld defName. Third-party authors should still handle
missing or modded defs through normal failure handling.

## Cargo Transaction Diagnostics

Diagnostics are read-only Dev/support data. They record SDK transaction
attempts and results; they are not authoritative cargo state.

Main types:

- `IShuttleExternalCargoTransactionDiagnosticsProvider`
- `ShuttleExternalCargoTransactionDiagnosticsSnapshot`
- `ShuttleExternalCargoTransactionDiagnosticRecord`
- `ShuttleExternalCargoTransactionFailureSummary`
- `ShuttleExternalCargoTransactionSourceSummary`

Properties and limits:

- Recent record cap: 128.
- Failure summaries are aggregate counters.
- Source summaries are aggregate counters by owner package id and local
  requester key.
- Diagnostics are non-persistent and may reset across reload/session.
- `RecordsTruncated == true` means old recent rows were dropped.

Example: inspect recent failed records for your owner package id.

```csharp
ShuttleExternalCargoTransactionDiagnosticsSnapshot diagnostics;
if (ShuttleExternalSDK.TryGetCargoTransactionDiagnostics(
        sdkHost,
        out diagnostics) &&
    diagnostics != null &&
    diagnostics.Available)
{
    for (int i = 0; i < diagnostics.RecentRecords.Count; i++)
    {
        ShuttleExternalCargoTransactionDiagnosticRecord record =
            diagnostics.RecentRecords[i];
        if (record.OwnerPackageId == "my.cool.mod" && !record.Success)
        {
            ShuttleExternalCargoTransactionFailureReason reason =
                record.FailureReason;
            string message = record.Message;
        }
    }
}
```

Use diagnostics to identify repeated invalid calls, stale assumptions, or
state-gate failures. For current cargo truth, use cargo read/query instead of
diagnostics.

## Failure Reason Guide

| Failure reason | Usually means | Category | Recommended response |
| --- | --- | --- | --- |
| `InvalidRequest` | The request object or option combination is invalid. | Caller error | Fix request construction before retrying. |
| `InvalidSource` | Missing or invalid owner/requester source trace. | Caller error | Provide stable owner and local requester keys. |
| `InvalidDef` | A defName is missing or not known. | Caller error | Check def availability and mod load order. |
| `InvalidQuantity` | Count or mass limit is out of bounds. | Caller error | Clamp count/mass before calling. |
| `CargoUnavailable` | Normal cargo surface cannot be reached. | Temporary state | Re-read host state; retry later if appropriate. |
| `InsufficientItems` | Matching loaded normal cargo is not enough. | State/data | Query cargo again or wait for cargo changes. |
| `UnsupportedCargoKind` | The matched cargo kind is not supported. | Caller/scope | Use a different SDK surface for that cargo kind. |
| `QueuedCargoExcluded` | Matching cargo is queued/assigned, not loaded normal cargo. | Scope | Wait until loaded or use read-only data only. |
| `BlockedCargoExcluded` | Cargo is blocked by cargo region policy. | Policy | Respect cargo region settings. |
| `RefrigeratedCargoExcluded` | Matching cargo is refrigerated. | Scope | Wait for a future refrigerated transaction API. |
| `PawnCargoExcluded` | Matching cargo is a pawn. | Scope | No pawn cargo transaction support in P1. |
| `CorpseCargoExcluded` | Matching cargo is a corpse. | Scope | No corpse cargo transaction support in P1. |
| `HostUnavailable` | Shuttle host, map, or controller context is unavailable. | Temporary state | Reacquire host and handle despawn/load timing. |
| `RuntimeBlocked` | Critical damage, module removal, or external runtime failure blocks writes. | Temporary state | Pause writes and retry only after state clears. |
| `LaunchTransferActive` | Holder/launch transfer or recovery is active. | Temporary state | Wait for transfer to finish before mutating cargo. |
| `CargoTransferActive` | A local holder/cargo transfer is active. | Temporary state | Wait and retry later. |
| `DepositUnsupported` | Deposit is not available for the requested case. | Scope | Check feature support and request shape. |
| `UnsupportedThingDef` | Deposit target def is not a supported ordinary item. | Caller/scope | Choose a supported plain item def. |
| `UnsupportedStuff` | Stuffed item deposit is not supported. | Scope | Omit stuff data in P1 deposit. |
| `UnsupportedQuality` | Quality item deposit is not supported. | Scope | Omit quality data in P1 deposit. |
| `StackLimitExceeded` | The item def has no supported stack limit. | Caller/scope | Choose a stackable ordinary item. |
| `InsufficientCargoCapacity` | Deposit would exceed normal cargo capacity or max mass. | State/data | Quote again with lower count or more capacity. |
| `CargoFilterRejected` | No active cargo region filter accepts the deposit item. | Policy | Respect cargo filters or ask user to change config. |
| `ThingCreationFailed` | Internal item creation failed. | Internal/def issue | Report diagnostics; avoid repeated spam. |
| `DepositTargetUnavailable` | No normal transporter target can accept planned stack. | Temporary state | Re-read cargo and retry later if sensible. |
| `DepositAddFailed` | Holder rejected a created stack. | Temporary/internal | Inspect diagnostics; retry only after state changes. |
| `CleanupFailed` | Deposit cleanup/rollback could not fully recover. | Internal failure | Pause calls and report logs/diagnostics. |
| `PartialApplyRejected` | Exact transaction postcondition failed and was rejected. | Timing/internal | Re-read state; report if repeated. |
| `RollbackFailed` | Consume rollback could not fully restore taken cargo. | Internal failure | Pause calls and report logs/diagnostics. |
| `InternalError` | Unexpected exception or internal error. | Internal failure | Report with diagnostics and source trace. |

`None` is not a failure reason. Successful results should have
`FailureReason == None`.

## Common Mistakes

- Assuming an SDK read snapshot is live truth. Re-read before decisions that
  depend on current cargo, power, cooldown, or readiness.
- Caching a provider forever without handling host despawn, destruction, reload,
  or optional seam unavailability.
- Treating cargo row ids, source ids, or reference ids as write handles. They
  are copied diagnostics/source-trace identifiers only.
- Trusting cargo query summaries when `RowsTruncated`, `SourceRowsTruncated`,
  `ResultRowsTruncated`, or `SummaryByDefComplete == false` says the result is
  incomplete.
- Calling cargo transactions every tick. Use quotes, player intent, cooldowns,
  or your own state machine.
- Assuming `TryQuoteLaunch` validates a destination. It is destination-
  independent in SDK `1.7.0`.
- Assuming launch rule blockers stop actual launch. They affect SDK Launch Read
  / Quote readiness only in `1.7.0`.
- Harmony-patching into `ShuttleController`, cargo backends, command executors,
  holder manifests, or launch services for normal SDK use.
- Passing or storing live `Thing`, `Pawn`, `Map`, `WorldObject`, `ThingOwner`,
  or `ThingDef` references as if the copied SDK accepted them.
- Treating diagnostics as persistent history. Diagnostics are bounded support
  data and may reset.

## Safety Rules For Third-Party Authors

- Use a namespaced owner package id.
- Use stable local requester keys.
- Feature-check every optional surface.
- Obtain services through `ShuttleExternalSDK` facade methods.
- Avoid relying on live Verse references from the SDK.
- Treat snapshot row ids as copied identifiers, not write handles.
- Treat snapshots and quotes as stale immediately after reading.
- Quote before cargo mutation.
- Handle every failure reason.
- Avoid calling cargo transactions every tick.
- Use diagnostics for support, not gameplay truth.
- Keep SDK calls on the main RimWorld thread.
- Avoid Harmony patches into internal controller/backend/holder state for normal
  SDK integrations.

## Deferred Or Out Of Scope In P1

The following surfaces are outside P1's supported read/write scope:

- Cargo transfer API.
- Durable reserve API.
- Refrigerated mutation.
- Queued/assigned cargo mutation.
- Blocked cargo mutation.
- Live item handles.
- Target region selection.
- Stuff support.
- Quality support.
- Hit points, comp state, rottable age, style, faction, ideology, ingredient,
  or arbitrary item metadata.
- Destination-specific launch quote, launch execution/confirm API, actual
  launch validation participation for rule providers, and launch participant
  hooks.
- Pawn, medical, prisoner, and mech services.
- Declarative UI.
- Combat, shield, and weapon providers.
- Power/load-shedding provider.
- Job/work/reservation provider.
- Assembly placement tags.

If you need one of these surfaces, treat it as future design work and plan a
public API instead of reaching through reflection.

## Minimal Integration Checklist

- [ ] Use a stable owner package id such as `my.cool.mod`.
- [ ] Use stable local requester keys for each system.
- [ ] Check `ShuttleExternalSDK.APIVersion`.
- [ ] Check feature keys before every optional surface.
- [ ] Acquire providers through facade methods.
- [ ] Treat snapshots as stale.
- [ ] Query/read cargo before making cargo decisions.
- [ ] Quote before consume/deposit.
- [ ] Handle failure reasons without log spam.
- [ ] Avoid transaction calls every tick.
- [ ] Use diagnostics for support reports.
- [ ] Keep calls on the main RimWorld thread.
- [ ] Avoid Harmony/reflection access to controller/backend/holders.
- [ ] Document which SDK feature keys your mod requires.

## Versioning Notes

- See `Docs/ExternalSDK_P1_Changelog.md` for the compact SDK P1 release
  timeline.
- `1.7.0` is the first SDK version with read-only launch rule provider rows for
  Launch Read / Quote. These rows are SDK-evaluated only and are not enforced by
  actual launch validation.
- `1.6.0` is the first SDK version with destination-independent launch read/quote.
- `1.5.1` is the first SDK version with cargo transaction diagnostics.
- Minor SDK version bumps can add copied DTO fields, feature keys, or optional
  provider surfaces.
- A high enough version is not a substitute for feature checks. Check feature
  keys.
- Optional provider calls may still return `false` for a particular host or
  runtime state.

## Troubleshooting

Cargo read returns unavailable:

- Reacquire the host and retry when the shuttle is spawned and stable.
- Check `UnavailableReason`.

Query counts look incomplete:

- Check `SourceRowsTruncated`, `RowsTruncated`, `ResultRowsTruncated`, and
  `SummaryByDefComplete`.
- Raise your query row cap only when needed.

Quote succeeds but consume/deposit fails:

- Cargo or shuttle state changed between quote and mutation.
- Re-read cargo and handle the transaction failure reason.

Deposit fails with filter/capacity reasons:

- The cargo bay filters or mass capacity reject the item.
- Ask the player to adjust cargo configuration or reduce count.

Diagnostics show repeated invalid requests:

- Fix caller request construction.
- Fix invalid requests before retrying them.

Diagnostics disappeared:

- Diagnostics are non-persistent and may reset across reload/session.
