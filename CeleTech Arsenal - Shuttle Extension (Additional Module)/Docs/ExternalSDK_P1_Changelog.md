# External SDK P1 Changelog

Compact handoff notes for External SDK `1.7.0`.

## SDK 1.7.0 Summary

External SDK P1 is stable for third-party authoring around feature discovery,
copied read snapshots, profile expression rows, cargo read/query, bounded
normal-cargo quote/consume/deposit transactions, cargo transaction diagnostics,
destination-independent launch read/quote, and advisory launch rule providers.

Public SDK DTOs do not expose live Verse/RimWorld objects. Use feature keys and
optional providers. Treat all snapshots and quotes as stale after reading.

## Version Timeline

| Version | Additions |
| --- | --- |
| `1.2.0` | Profile expression capability, metric, requirement, and source trace rows. |
| `1.3.0` | Cargo read and cargo query surfaces. |
| `1.3.1` | Cargo row truncation/completeness semantics. |
| `1.4.0` | Cargo quote/consume for normal loaded cargo. |
| `1.5.0` | Cargo quote/deposit for internally-created plain item stacks into normal loaded cargo. |
| `1.5.1` | Cargo transaction diagnostics. |
| `1.6.0` | Destination-independent Launch Read / Quote. |
| `1.7.0` | Launch Rule Provider advisory read-snapshot layer and rule diagnostics. |

## Supported In 1.7.0

- SDK version, compatibility, and feature discovery.
- Integration health snapshots.
- Broad copied host/read snapshots.
- Profile expression capability, metric, requirement, and contribution source
  rows.
- Cargo read and cargo query, including truncation/completeness flags.
- Cargo quote/consume for normal loaded cargo.
- Cargo quote/deposit for internally-created plain item stacks into normal
  loaded cargo.
- Cargo transaction diagnostics for bounded support data.
- Destination-independent launch read/quote.
- Advisory launch rule provider rows in SDK Launch Read / Quote diagnostics.

## Explicit Non-Goals

- Live Verse/RimWorld object access through public SDK DTOs.
- Direct controller, cargo backend, holder, transporter, or launch service
  mutation.
- Cargo transfer API.
- Durable cargo reserve API.
- Refrigerated, queued, assigned, or blocked cargo mutation.
- Stuffed, quality, partial, pawn, corpse, or live item deposit.
- Destination-specific launch quote.
- Launch targeter, confirmation, execution, arrival, participant, or transfer
  hooks.
- Actual launch validator participation for external rule providers.
- Declarative UI, pawn/medical/prisoner/mech services, combat/shield/weapon
  providers, power/load-shedding providers, job/work/reservation providers, or
  assembly placement tags.

## Safety And Migration Notes

- Public feature keys describe implemented surfaces; prefer feature checks over
  version checks.
- Minor versions may add copied DTO fields or optional provider surfaces.
- Optional provider calls can return unavailable for a specific host or runtime
  state even when the feature key exists.
- Rule provider blockers affect SDK readiness only in `1.7.0`; actual launch
  validation remains authoritative.
- Cargo transaction diagnostics are bounded support data, not persistent
  gameplay history.
