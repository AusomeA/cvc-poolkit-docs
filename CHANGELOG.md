# Changelog

## 1.1.0 (2026-09)

New:

- **Auto-tune.** Every Play In Editor or `-game` session run from the editor records, per pool, the peak active
  count, time near the peak, requests, failed requests and recycles, and saves the recording to
  `Saved/PoolKit/AutoTune/*.json`. The Auto-Tune review window (PoolKit Debugger > Auto-Tune..., or
  `PoolKit.AutoTune.Review`) shows the recommended Prewarm Count, Max Inactive and Max Active next to the current
  Project Settings values, with a headroom percentage, and writes the rows you tick into Project Settings. Also
  available from Blueprint (`Start/Stop Auto-Tune Recording`, `Get Recommendations`, `Apply Recommendations`) and the
  console (`PoolKit.AutoTune.Start/Stop/Diff/Apply/Clear`).
- **Widget pooling (UMG).** `Acquire Widget From Pool` / `Release Widget To Pool`: widgets are removed from their
  parent (viewport or panel), animations stopped, timers cleared, and visibility, opacity, render transform, color and
  enabled state restored on release. The Slate widget is kept, so reuse does not rebuild it. Auto-release delay,
  limits and prewarm per widget class.
- **Fire-and-forget sounds and Niagara.** `Play Sound At Location/Attached From Pool` and
  `Spawn Niagara At Location/Attached From Pool` return the component to its pool when the sound or system finishes,
  or when the component it is attached to is destroyed. Niagara support is in its own module, PoolKitNiagara.
- **Component pooling.** `Spawn Decal At Location/Attached From Pool` (with optional fade-out) and
  `Spawn Static Mesh (Attached) From Pool`, one pool per decal material or mesh, plus a generic C++ API for other scene
  components. Limits and prewarm per asset in Project Settings > PoolKit > Asset Configs.
- **PoolKit Debugger** (Tools > Debug > PoolKit Debugger, editor module PoolKitEditor): live rows for every pool of
  every kind during PIE, with active, inactive, peak, reuse %, failed, recycled, double releases, estimated memory,
  a 30-second graph and warnings (undersized, possible leak, double release). Buttons: Prewarm to Peak, Trim,
  Release All, Reset Stats, Start/Stop Recording, Auto-Tune.
- **Multiplayer.** Replicated actors are pooled server-authoritatively on listen and dedicated servers: a small
  replicated state component makes clients hide and deactivate released actors, teleport and reset reused ones and call
  the Poolable events; released actors go dormant. Late joiners do not receive inactive pooled actors. Clients cannot
  release or destroy the server's pooled actors. Client-side cosmetic pooling works as before. See README, Multiplayer.
- The stats overlay, `PoolKit.Dump`, `PoolKit.ReleaseAll` and `PoolKit.Trim` cover every pool kind; the overlay and
  `Get All Pool Infos` show warnings. `PoolKit.Demo <rate> all` adds pooled decals, meshes and damage-number widgets;
  `PoolKit.Demo <rate> replicated` spawns replicated projectiles on a server.
- 27 automation tests (11 in 1.0).

Fixed:

- The stats overlay did not draw on UE 5.4-5.6 (the engine passes no player controller to debug draw delegates
  there). 1.0 showed it only on 5.7.

Compatibility:

- The 1.0 Blueprint nodes and C++ API are unchanged. `FPoolKitPoolStats` has two new fields
  (`TotalInvalidReleases`, `TimeAtPeakSeconds`). `ResetStats(None)` now resets pools of every kind.
  `PoolKit.Trim` and `PoolKit.ReleaseAll` now also act on widget and component pools.
- The plugin now depends on the Niagara plugin (enabled by default in every UE5 project) for the PoolKitNiagara module.

## 1.0.0 (2026-09)

- First release.
