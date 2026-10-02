# PoolKit: Actor, Widget and FX Pooling with Auto-Tune for Unreal Engine

PoolKit lets you reuse actors, UMG widgets, sounds, Niagara effects, decals and static mesh components instead of
creating and destroying them, from Blueprint or C++. Replace **Spawn Actor from Class** with **Spawn Actor From Pool**,
and **Destroy Actor** with **Release Actor To Pool**. That's all the setup there is.

- No manager actor to place, no base class to inherit, no interface required.
- Pooled objects are reset automatically, so a reused actor behaves like a freshly spawned one.
- **Auto-tune**: play your game, and PoolKit recommends Prewarm Count and limits from what it recorded, shows them next
  to your current settings, and writes the ones you pick into Project Settings.
- **PoolKit Debugger** editor window: live counts, 30-second graphs and warnings for every pool during Play In Editor.
- **Beyond actors**: widget pools (damage numbers, hit markers, list items), fire-and-forget sounds and Niagara effects,
  decals and static mesh components.
- **Multiplayer**: replicated actors are pooled server-authoritatively on listen and dedicated servers.
- Prewarm, limits, auto-release, live stats overlay, console commands, a built-in benchmark and 27 automation tests.
- No content, so it works in any project.

Supported engine versions: 5.4, 5.5, 5.6, 5.7, 5.8. Tested on Windows (Win64). On 5.8 (5.8.3) the plugin builds and the 27 automation tests pass; the multi-process multiplayer test (server plus two clients) was run on 5.4-5.7 only.

**Get PoolKit:** [itch.io](https://barbed-wire-glove-games.itch.io/poolkit) (source plugin, $17.99). The Fab listing is in review and not live yet; this page will link it once it is approved.

---

## Contents

1. [Installation](#1-installation)
2. [Quick start (Blueprint)](#2-quick-start-blueprint)
3. [Quick start (C++)](#3-quick-start-c)
4. [How pooling works in PoolKit](#4-how-pooling-works-in-poolkit)
5. [The PoolKit Poolable interface](#5-the-poolkit-poolable-interface)
6. [Widget pooling (UMG)](#6-widget-pooling-umg)
7. [Sounds, Niagara, decals and static meshes](#7-sounds-niagara-decals-and-static-meshes)
8. [Multiplayer](#8-multiplayer)
9. [PoolKit Debugger](#9-poolkit-debugger)
10. [Auto-tune](#10-auto-tune)
11. [Project Settings](#11-project-settings)
12. [Blueprint node reference](#12-blueprint-node-reference)
13. [C++ API reference](#13-c-api-reference)
14. [Stats overlay, console commands and benchmark](#14-stats-overlay-console-commands-and-benchmark)
15. [Demo without any content](#15-demo-without-any-content)
16. [Automation tests](#16-automation-tests)
17. [Limitations and FAQ](#17-limitations-and-faq)

---

## 1. Installation

**From Fab (once the listing is approved; it is in review today):** add PoolKit to your library, install it to your engine
from the Epic Games Launcher, then enable it in **Edit > Plugins** (search "PoolKit") and restart the editor. Fab
ships code plugins as source, so the project must be able to build C++ (see the itch.io steps below).

**From itch.io:** download the zip for your engine version and unzip it so the `PoolKit` folder sits in
`<YourProject>/Plugins/`. It's source code, so the project must be a C++ project with Visual Studio (or Rider) set up
to build it. A Blueprint-only project becomes one when you add any C++ class (**Tools > New C++ Class**). Open the
project and let the editor build the plugin.

**Manual (project plugin):** copy the `PoolKit` folder into `<YourProject>/Plugins/` and restart the editor.
C++ projects rebuild it automatically; Blueprint-only projects must first become C++ projects (Tools > New C++ Class) so the plugin can be built.

PoolKit has three modules:

| Module | Type | Contents |
|---|---|---|
| `PoolKit` | Runtime | Actor, widget and component pools, sounds, decals, meshes, multiplayer, auto-tune, overlay, console commands |
| `PoolKitNiagara` | Runtime | Niagara pooling (`Spawn Niagara ... From Pool`). The only module that depends on Niagara. |
| `PoolKitEditor` | Editor | PoolKit Debugger window and Auto-Tune review window. Not loaded in packaged games. |

**C++ projects** that call PoolKit from code add the modules they use to their `.Build.cs`:

```csharp
PublicDependencyModuleNames.Add("PoolKit");
PublicDependencyModuleNames.Add("PoolKitNiagara"); // only if you call the Niagara functions from C++
```

## 2. Quick start (Blueprint)

1. Where you spawn a projectile, effect or pickup, use **Spawn Actor From Pool** (category *PoolKit*).
   Pick the class; the return value is already typed to that class, just like Spawn Actor.
2. Where that actor would be destroyed, call **Release Actor To Pool** (Target defaults to *Self*).
   To keep the timed life of Set Life Span, either set **Auto Release Delay** on the spawn node or call
   **Release Actor To Pool After**.
3. Optional: add the **PoolKit Poolable** interface to the actor (Class Settings > Interfaces) and implement
   **On Acquired From Pool** for per-use setup that used to live in BeginPlay (reset health, pick a random
   mesh, and so on).
4. For effects: **Play Sound At Location From Pool**, **Spawn Niagara At Location From Pool** and
   **Spawn Decal At Location From Pool** are fire-and-forget; the component goes back to its pool by itself.
5. For UI: **Acquire Widget From Pool**, then Add to Viewport (or add it to a panel); **Release Widget To Pool** when done.

Press Play, open **Tools > Debug > PoolKit Debugger** (or type `PoolKit.ShowStats 1` in the console) to watch the
pools work. After playing, click **Auto-Tune...** in the debugger to size your pools from what you just played.

## 3. Quick start (C++)

```cpp
#include "PoolKitSubsystem.h"

void AMyWeapon::Fire()
{
	UPoolKitSubsystem* Pool = GetWorld()->GetSubsystem<UPoolKitSubsystem>();

	// Typed acquire; the projectile returns to the pool automatically after 3 seconds.
	AMyProjectile* Projectile = Pool->AcquireActor<AMyProjectile>(ProjectileClass, MuzzleTransform, this, GetInstigator(), 3.f);
}

void AMyProjectile::OnHit(/* ... */)
{
	if (UPoolKitSubsystem* Pool = UPoolKitSubsystem::Get(this))
	{
		Pool->ReleaseActor(this); // instead of Destroy()
	}
}
```

Implement the interface in C++ like this:

```cpp
#include "PoolKitPoolable.h"

UCLASS()
class AMyProjectile : public AActor, public IPoolKitPoolable
{
	GENERATED_BODY()

public:
	virtual void OnAcquiredFromPool_Implementation() override { Health = MaxHealth; }
	virtual void OnReleasedToPool_Implementation() override { Target = nullptr; }
};
```

## 4. How pooling works in PoolKit

- There is one pool per **exact actor class** in each **game or PIE world**, created the first time you use
  the class. Pools live in `UPoolKitSubsystem`, a world subsystem, so they are cleaned up when the world
  unloads. Editor (non-play) worlds have no pools.
- **Spawn Actor From Pool / AcquireActor** takes the most recently released actor of that class. If there is
  none, it spawns a new one (construction script and BeginPlay run as usual). Either way,
  `OnAcquiredFromPool` is called afterwards.
- **Release Actor To Pool / ReleaseActor** calls `OnReleasedToPool`, then deactivates the actor and keeps it
  in the world, hidden, for reuse. If the pool already holds *Max Inactive* actors, the actor is destroyed
  instead.
- If a pooled actor is destroyed some other way (Destroy Actor, level cleanup), PoolKit notices through
  `OnDestroyed` and forgets it. The pool stays consistent.
- Widget pools are keyed by widget class; component pools by component class and asset (the sound, Niagara system,
  decal material or mesh). They follow the same rules: limits, recycling, prewarm, stats.

### Lifecycle of a pooled actor

| Moment | What runs |
|---|---|
| First acquire (new actor) | Construction script, BeginPlay, then OnAcquiredFromPool |
| Prewarm (new actor) | Construction script, BeginPlay (actor spawned hidden, without collision), then it goes straight into the pool. OnReleasedToPool is **not** called. |
| Every later acquire | Automatic reset (below), then OnAcquiredFromPool. BeginPlay does **not** run again. |
| Release | OnReleasedToPool, then automatic deactivation (below) |
| Pool full on release / trim | EndPlay and Destroyed, as with Destroy Actor |

### Automatic deactivation on release

In this order:
1. Timers and latent actions (for example **Delay** nodes) owned by the actor and its components are
   cleared. *(Setting: Clear Timers On Release)*
2. The actor is detached from its parent actor. *(Setting: Detach On Release)*
3. Movement components get `StopMovementImmediately`; simulating physics bodies get zero linear and angular velocity.
4. The actor is hidden, and its collision and tick are disabled.
5. Every active component is deactivated (FX components with `DeactivateImmediate`) and component tick is
   disabled. *(Setting: Reset Components On Reuse)*
6. Owner and Instigator are cleared.

### Automatic reset on acquire

1. Owner and Instigator are set from the call.
2. The actor is teleported to the spawn transform (physics state reset). As with SpawnActor, the transform is
   combined with the root component's default relative transform.
3. Hidden, collision enabled and actor tick are restored to the **class defaults**.
4. Component tick is restored to each component's *Start with Tick Enabled*, and every component with
   **Auto Activate** is activated again with reset. Audio, particle, Niagara and movement components restart
   the way they do on spawn. *(Setting: Reset Components On Reuse)*
5. Active **Projectile Movement** components get their initial velocity again: the default Velocity
   direction scaled to Initial Speed, in local space if *Initial Velocity in Local Space* is set, exactly as
   on spawn. *(Setting: Restart Projectile Movement On Acquire)*

Anything else that your BeginPlay sets up per use (variables, health, materials, AI) belongs in
**On Acquired From Pool**.

## 5. The PoolKit Poolable interface

Optional. Implement it on the actor or on **any component** of the pooled actor. Both are called. Pooled widgets and
pooled components (for example your own component class used with the generic component pool) can implement it too.

| Event | When |
|---|---|
| `On Acquired From Pool` | After every acquire, including the first. Transform, owner and instigator are already set. |
| `On Released To Pool` | At the start of every release, while the object is still visible and active. |

On network clients, both events also run on the client's copy of a replicated pooled actor (see Multiplayer).

Global notifications are also available as the subsystem's `OnActorAcquired` / `OnActorReleased` events
(Blueprint: *Get PoolKitSubsystem > Bind Event to On Actor Acquired*).

## 6. Widget pooling (UMG)

For widgets you create and remove often: damage numbers, hit markers, kill-feed lines, notifications, and list items in
scroll boxes you fill yourself. (List View and Tile View already recycle their entry widgets.)

```
Acquire Widget From Pool (Widget Class, Auto Release Delay, [Owning Player]) -> widget, not in any parent
    ... Add to Viewport / Add Child to a panel, set text, play an animation ...
Release Widget To Pool (Widget)   or let Auto Release Delay do it
```

On release PoolKit:
1. calls `On Released To Pool` (if the widget implements PoolKit Poolable),
2. stops all its animations and clears its timers and latent actions,
3. removes it from its parent, whether that is the viewport or a panel,
4. restores Visibility, Render Opacity, Render Transform (and pivot), Color and Opacity and Is Enabled from the widget
   class defaults.

Everything inside the widget (texts, images, child widget states) is yours to reset in `On Acquired From Pool`.
PoolKit keeps each pooled widget's Slate widget alive, so re-adding it to the viewport does not rebuild it.

- **Owning player**: None uses the first local player, like Create Widget with a world context. A widget reused
  for another local player gets `SetOwningPlayer`.
- **Dedicated servers** create no widgets: Acquire Widget From Pool returns None there.
- **Limits and prewarm** per widget class: Project Settings > PoolKit > Widgets. Prewarmed widgets are created at
  world begin play with the first local player as owner.
- **Double release** is rejected and counted (debugger warning).

## 7. Sounds, Niagara, decals and static meshes

These pools hold components, one pool per asset: per sound, per Niagara system, per decal material, per static mesh.
Components are registered in the world without an owner actor, so they survive the actor they were attached to.

| Node | Pool key | Returns to the pool when |
|---|---|---|
| **Play Sound At Location From Pool** / **Play Sound Attached From Pool** | Sound (Wave, Cue, MetaSound Source) | the sound finishes, Auto Release Delay passes, the attach parent is destroyed, or you call Release Component To Pool |
| **Spawn Niagara At Location From Pool** / **Spawn Niagara Attached From Pool** (PoolKitNiagara module) | Niagara System | the system finishes (On System Finished), Auto Release Delay passes, the attach parent is destroyed, or you release it |
| **Spawn Decal At Location From Pool** / **Spawn Decal Attached From Pool** | Decal material | Life Span passes (optional fade-out over the last Fade Out Duration seconds), the attach parent is destroyed, or you release it |
| **Spawn Static Mesh From Pool** / **Spawn Static Mesh Attached From Pool** | Static mesh | Life Span passes, the attach parent is destroyed, or you release it |

- **Looping** sounds and Niagara systems never finish on their own: set Auto Release Delay or release them yourself.
- **Parameters**: the nodes return the component, so you can set Niagara user parameters or sound parameters right after
  the call. They persist on the pooled component; set the ones you use on every spawn.
- **Decals**: use Max Active with Recycle Oldest When Full (Asset Configs) to cap bullet holes; the oldest decal is
  reused. PoolKit clears the decal's own life-span timer, so the engine never destroys a pooled decal.
- **Static meshes**: collision is off and override materials are cleared on release. Use actor pooling for meshes that
  need collision or physics.
- **On release** components are stopped (sound Stop, FX Deactivate Immediate), detached, hidden, their tick and timers
  are cleared, decal fades reset and mesh override materials removed.
- **Limits and prewarm** per asset: Project Settings > PoolKit > Asset Configs; defaults per kind in the same section.
- **C++**: `UPoolKitSubsystem::AcquireComponent(FPoolKitComponentRequest)` pools any other `USceneComponent` class, and
  `RegisterComponentHandler` adds new component kinds (this is how PoolKitNiagara plugs in).
- **Niagara is optional for your code**: the core `PoolKit` module does not depend on Niagara. PoolKit declares the
  Niagara plugin as a dependency because of the PoolKitNiagara module, so enabling PoolKit keeps Niagara enabled even if
  your .uproject turns it off (tested: PoolKit loads and all tests pass in such a project). Niagara ships with the engine
  and is on by default in every UE5 project.

## 8. Multiplayer

PoolKit pools **replicated** actors on listen and dedicated servers. The server owns the pool; clients follow.

What happens (Project Settings > PoolKit > Multiplayer > Replicate Pool State, on by default):

1. **Acquire on the server.** When the server spawns a pooled actor whose class replicates, PoolKit adds a small
   replicated component, *PoolKit Net State*, before the first replication. It holds the pool state (active or
   inactive), a use counter and the acquire transform.
2. **Release on the server.** The actor is hidden (hidden replicates as usual), the state becomes inactive, and the
   actor is put to sleep (net dormancy *Dormant All*, setting *Use Dormancy For Inactive Actors*) after the change is
   sent, so inactive pooled actors cost no bandwidth. Clients that receive the release call `On Released To Pool` on
   their copy and deactivate it locally: collision, tick, components, movement and timers off.
3. **Reuse on the server.** The actor wakes up, and clients receive the new state and visibility in the same update.
   Clients teleport their copy to the acquire transform (for actors that don't replicate movement; actors that
   replicate movement use their replicated location), reset it like a fresh spawn (collision, tick, Auto Activate
   components, Projectile Movement restart) and call `On Acquired From Pool`. If a client missed a release (released
   and reused in one update), it still runs the release and acquire steps in order.
4. **Late joiners** never receive inactive pooled actors: hidden actors without collision are not network-relevant.
   Actors in use replicate to them normally; if the server reuses a dormant actor later, the late joiner gets it then.
5. **Clients never destroy the server's pooled actors.** `Release Actor To Pool` on a client's copy logs a warning and
   returns false (even with Destroy If Not Pooled).
6. **Client-side cosmetic pooling** is unchanged: actors spawned from pool on a client are local, as in 1.0.

**Listen servers running far above 200 fps.** On a listen server the engine updates at most NetClientTicksPerSecond
client connections per second (200 by default, doubled on LAN; [/Script/Engine.Engine] in DefaultEngine.ini). When a
listen server runs uncapped far above that with several clients (for example a headless test server), the first
replication of an actor to a client can be delayed by seconds, because actors that need a new channel on a skipped
connection wait for their next update slot. That affects any actor, not only pooled ones; for PoolKit it shows up for
late joiners and for reused actors that were dormant. Cap the server frame rate (	.MaxFPS 60, as a rendering listen
server usually is) or raise NetClientTicksPerSecond. Dedicated servers are not throttled this way.
`Get Pooled Actor State` on a client returns the state replicated by the server. `PoolKit.Dump` on a client lists
the replicated pooled actors it can see (state, hidden, collision, location).

### What is and isn't supported

| Case | Supported | Notes |
|---|---|---|
| Replicated actors pooled on a listen server | Yes | Tested with a server and two clients (one joining late) as separate processes on one PC over localhost, on UE 5.4, 5.5, 5.6 and 5.7 (40/40 checks each) |
| Replicated actors pooled on a dedicated server | Yes | Tested the same way on UE 5.4, 5.5, 5.6 and 5.7 |
| Actors that replicate movement (projectiles) | Yes | Location comes from replicated movement |
| Actors that don't replicate movement | Yes | Reuse teleports the client copy to the acquire transform |
| Late-joining clients | Yes | Inactive pooled actors are not sent to them |
| Trimming or Max Inactive overflow on the server | Yes | Destroyed actors are destroyed on clients too |
| Client-side cosmetic pooling (local actors, effects, widgets) | Yes | Unchanged from 1.0 |
| Acquire/release on a client for a replicated class | No | Spawns a local, non-replicated actor (client-side pooling). Pool replicated actors on the server only. |
| Releasing the server's actor from a client | No (by design) | Refused with a warning; send an RPC to the server instead |
| Replicated widgets or components (sounds, Niagara, decals) | No | They are local to each machine; play them on each client (for example from a multicast or OnRep) |
| Your own replicated properties on pooled actors | Yes, your job | They replicate as usual; reset them on the server in On Acquired From Pool |
| Client-side prediction of pooled projectiles | No | Not provided; the server's actor is authoritative |
| Replication Graph, Iris | Not tested | The mechanism uses standard dormancy, relevancy and a replicated subobject, but only the default replication system was tested |
| Seamless travel | Pools belong to the world | Each map starts with empty pools, as in 1.0 |

## 9. PoolKit Debugger

**Tools > Debug > PoolKit Debugger** (or console `PoolKit.OpenDebugger`) opens an editor tab. During Play In Editor
it shows every pool of every kind in the selected play world (choose the server or a client when you play with several
players):

| Column | Meaning |
|---|---|
| Kind, Pool | Actor, Widget, Sound, Niagara, Decal, Mesh or Component; class or asset name (tooltip: full path) |
| Active, Inactive, Peak | Objects in use, waiting, and the highest number in use at once (tooltip: time near the peak) |
| Reuse | Share of requests served without creating an object |
| Failed, Recycled | Requests that failed at Max Active, and requests that recycled the oldest object |
| 2x Rel. | Releases of objects that were already in the pool (double release) |
| Est. Memory | Rough object memory of all instances: UObject sizes of the actor/widget/component and its parts, plus Niagara's own estimate. Shared assets (meshes, textures) are not included. |
| Active, last 30 s | Graph of the active count, sampled every 0.25 s |
| Warnings | **Undersized** (requests failed, or more than 5 % recycled, at Max Active), **Leak?** (the active count only grew by 20+ during the last 20 s), **Double release** |

Buttons act on the selected rows, or on every row if none is selected: **Prewarm to Peak**, **Trim**, **Release All**,
**Reset Stats**, **Start/Stop Recording** (auto-tune) and **Auto-Tune...**. Select a row for its limits, counters and
the full warning text. The leak and recycle thresholds are in Project Settings > PoolKit > Diagnostics.

The same data is available at runtime from Blueprint or C++ with **Get All Pool Infos** (`FPoolKitPoolInfo`).

## 10. Auto-tune

Auto-tune sizes your pools from real play:

1. **Record.** Every Play In Editor session and every `-game` session started from the editor is recorded
   automatically (Project Settings > PoolKit > Auto-Tune > Record Every Play Session). Packaged builds record only when
   you call **Start Auto-Tune Recording** / `PoolKit.AutoTune.Start`. Per pool it records the peak active count, the
   time spent near the peak (within 10 %), requests, failed requests and recycle events. When the world ends (or on
   **Stop Auto-Tune Recording**), the recording is saved as JSON in `Saved/PoolKit/AutoTune/` (the newest 20 are kept).
2. **Review.** Click **Auto-Tune...** in the PoolKit Debugger (or run `PoolKit.AutoTune.Review`). The window lists every
   recorded pool with its current Project Settings values and the recommended ones side by side (changes highlighted),
   the peak, time at peak, requests, failures, recycles and a note. Change the headroom and the values update.
3. **Apply.** Tick the rows you want and click **Apply Selected to Project Settings**. PoolKit adds or updates entries
   in Class Configs, Widget Configs and Asset Configs and saves `Config/DefaultGame.ini`. New pools use the values the
   next time you press Play.

How the values are computed (highest peak over all saved recordings, H = headroom, default 25 %):

| Setting | Recommendation |
|---|---|
| Prewarm Count | ceil(peak x (1 + H)), at most Max Active |
| Max Inactive | same as Prewarm Count |
| Max Active, pool had a cap and requests failed | raised: ceil(cap x (1 + H)), at least cap + 1. Play again to check, because the real demand was hidden by the cap. |
| Max Active, pool had a cap and recycled | kept as configured (a recycling cap is a design choice) |
| Max Active, unlimited pool | kept unlimited, unless *Auto-Tune Cap Unlimited Pools* is on (then peak + headroom) |
| Recycle Oldest When Full | kept as configured |

Notes flag spikes (near the peak for under 0.5 s) and pools with few requests. Generic C++ component pools have no
settings entry and are not listed.

Console: `PoolKit.AutoTune.Start`, `.Stop`, `.Diff [Headroom]` (log), `.Apply [Headroom]`, `.Clear`, `.Review` (editor).
Blueprint: **Start Auto-Tune Recording**, **Stop Auto-Tune Recording**, **Is Auto-Tune Recording**, **Get Recommendations**,
**Apply Recommendations**, **Load Recordings**, **Delete Recordings**, **Describe Recommendation**.
In a packaged build, Apply Recommendations only changes the settings in memory.

## 11. Project Settings

**Project Settings > Plugins > PoolKit** (saved to `Config/DefaultGame.ini`):

| Setting | Default | Meaning |
|---|---|---|
| Class Configs | empty | Per actor class: *Actor Class*, *Prewarm Count*, *Limits* |
| Default Limits | 0 / 0 / off | Limits for actor classes without an entry. 0 = unlimited. |
| Widget Configs | empty | Per widget class: *Widget Class*, *Prewarm Count*, *Limits* |
| Default Widget Limits | 0 / 0 / off | Limits for widget classes without an entry |
| Asset Configs | empty | Per sound, Niagara system, decal material or static mesh: *Asset*, *Prewarm Count*, *Limits* |
| Default Sound / Niagara / Decal / Static Mesh / Component Limits | 0 / 0 / off | Limits for assets without an entry, per kind |
| Prewarm On World Begin Play | on | Prewarm every config entry when a game world begins play |
| Spread Begin Play Prewarm Over Frames | on | Spawn actors a few per frame instead of all in the first frame |
| Prewarm Actors Per Frame | 8 | Budget for any actor prewarm that is spread over frames |
| Reset Components On Reuse | on | Deactivate components on release, re-activate Auto Activate components on acquire |
| Clear Timers On Release | on | Clear timers and latent actions of the actor and its components on release |
| Restart Projectile Movement On Acquire | on | Restart Projectile Movement with its initial speed on acquire |
| Detach On Release | on | Detach the actor from its parent on release |
| Replicate Pool State | on | Server-authoritative pooling of replicated actors (see Multiplayer) |
| Use Dormancy For Inactive Actors | on | Released replicated actors stop replicating until reused |
| Record Every Play Session | on | Auto-tune recording of PIE and `-game` sessions run from the editor |
| Auto-Tune Headroom Percent | 25 | Capacity added on top of the recorded peak |
| Auto-Tune Cap Unlimited Pools | off | Also recommend a Max Active for pools that are unlimited now |
| Max Saved Recordings | 20 | Recordings kept in Saved/PoolKit/AutoTune |
| Warn On Misuse | on | Log a warning for double releases or releasing objects PoolKit did not create |
| Leak Warning Min Growth / Window Seconds | 20 / 20 | Leak warning thresholds |
| Recycle Warning Ratio | 0.05 | Undersized warning when more than this share of requests recycled |

**Limits** (`FPoolKitPoolLimits`):

| Field | Meaning |
|---|---|
| Max Inactive | Most objects kept waiting in the pool. Extra released objects are destroyed. 0 = unlimited. |
| Max Active | Most objects of the pool in use at once. 0 = unlimited. |
| Recycle Oldest When Full | When Max Active is reached: on = release and reuse the oldest active object (bullets, decals, shell casings); off = the request returns None. |

Classes and assets in the config arrays are loaded synchronously at world begin play when their Prewarm Count is above 0.

## 12. Blueprint node reference

All nodes are in the **PoolKit** category.

**Actors**

| Node | Description |
|---|---|
| **Spawn Actor From Pool** (Class, Spawn Transform, Auto Release Delay, *Owner*, *Instigator*) returns Actor | Typed output. Auto Release Delay > 0 returns the actor to the pool after that many seconds. Returns None for invalid or abstract classes, outside game worlds, or when Max Active is reached with recycling off. |
| **Release Actor To Pool** (Actor, Destroy If Not Pooled = true) returns bool | Returns the actor to its pool. With *Destroy If Not Pooled*, actors that PoolKit did not create are destroyed, so the node is always safe to use in place of Destroy Actor (except a client's copy of a server's pooled actor, which is never destroyed). |
| **Release Actor To Pool After** (Actor, Delay) returns bool | Schedules (or re-schedules) an automatic release. Delay <= 0 cancels a scheduled release. |
| **Prewarm Pool** (Class, Count, Spread Over Frames) returns int | Makes sure the pool holds at least *Count* actors (active plus inactive). Returns the number spawned, or queued when spreading. |
| **Get Pooled Actor State** (Actor) returns EPoolKitActorState | *Not Pooled*, *Active* or *Inactive*. |
| **Is Pooled Actor** (Actor) returns bool | True if PoolKit created the actor. |
| **Get Pool Stats** (Class) returns FPoolKitPoolStats | Counters of one actor pool. |
| **To String (PoolKit Pool Stats)** | One-line text for Print String. |

**Widgets**

| Node | Description |
|---|---|
| **Acquire Widget From Pool** (Widget Class, Auto Release Delay, *Owning Player*) returns widget | Typed output; the widget has no parent. None on dedicated servers or at Max Active without recycling. |
| **Release Widget To Pool** (Widget, Remove If Not Pooled = true) returns bool | Removes it from its parent and resets it (see section 6). |
| **Release Widget To Pool After** (Widget, Delay) | Scheduled release; Delay <= 0 cancels. |
| **Prewarm Widget Pool** (Widget Class, Count) returns int | Creates widgets in advance. |

**Effects and components**

| Node | Description |
|---|---|
| **Play Sound At Location From Pool** (Sound, Location, *Rotation, Volume, Pitch, Start Time, Attenuation, Concurrency, Auto Release Delay*) returns Audio Component | Fire-and-forget sound. |
| **Play Sound Attached From Pool** (Sound, Attach To Component, Attach Point Name, Location, Rotation, Location Type, ...) | Attached version; released if the parent is destroyed. |
| **Spawn Niagara At Location From Pool** (System, Location, Rotation, *Scale, Auto Release Delay*) returns Niagara Component | Fire-and-forget Niagara (PoolKitNiagara module). |
| **Spawn Niagara Attached From Pool** (System, Attach To Component, Attach Point Name, Location, Rotation, Location Type, *Auto Release Delay*) | Attached version. |
| **Prewarm Niagara Pool** (System, Count) returns int | Creates Niagara components in advance. |
| **Spawn Decal At Location From Pool** (Decal Material, Decal Size, Location, Rotation, Life Span, Fade Out Duration) returns Decal Component | Pooled decal. |
| **Spawn Decal Attached From Pool** (..., Attach To Component, Attach Point Name, Location, Rotation, Location Type, Life Span, Fade Out Duration) | Attached version. |
| **Spawn Static Mesh From Pool** (Mesh, Transform, Life Span, *Override Material*) returns Static Mesh Component | Pooled mesh component, collision off. |
| **Spawn Static Mesh Attached From Pool** (Mesh, Attach To Component, Attach Point Name, Relative Transform, Life Span, *Override Material*) | Attached version. |
| **Release Component To Pool** (Component, Destroy If Not Pooled = true) returns bool | Returns any pooled component. |
| **Release Component To Pool After** (Component, Delay) | Scheduled release. |

**Any kind and debugging**

| Node | Description |
|---|---|
| **Get Pooled Object State** (Object) | State of a pooled actor, widget or component. |
| **Get All Pool Infos** returns array of FPoolKitPoolInfo | Kind, name, pooled class or asset, stats, estimated memory, warnings, warning text and 30 s history of every pool. |
| Auto-tune nodes | See section 10. |

More functions are on the subsystem node (**Get PoolKitSubsystem**): `Release All Active`, `Trim Pool`,
`Set Pool Limits`, `Get Pool Limits`, `Get All Pool Stats`, `Reset Stats`, `Set Auto Release`, `Acquire Widget`,
`Release Widget`, `Set Widget Auto Release`, `Prewarm Widgets`, `Release Component`, `Set Component Auto Release`,
`Get Object State`, `Prewarm Pool Of Kind`, `Release All Of Kind`, `Trim Pool Of Kind`, `Release Everything`,
`Trim Everything`, and the `On Actor Acquired` / `On Actor Released` events.

**FPoolKitPoolStats fields:** Actor Class (actor pools only), Num Active, Num Inactive, Peak Active, Total Spawned,
Total Acquires, Total Reuses, Total Releases, Total Destroyed, Total Recycled, Total Failed Acquires, Pending Prewarm,
Reuse Ratio (0..1), Limits, Total Invalid Releases (double releases), Time At Peak Seconds (near the peak, within 10 %).

## 13. C++ API reference

`#include "PoolKitSubsystem.h"`: `UPoolKitSubsystem : UWorldSubsystem`

```cpp
static UPoolKitSubsystem* Get(const UObject* WorldContextObject);   // null outside game/PIE worlds

// Actors (unchanged since 1.0)
AActor* AcquireActor(TSubclassOf<AActor> ActorClass, const FTransform& SpawnTransform,
                     AActor* Owner = nullptr, APawn* Instigator = nullptr, float AutoReleaseDelay = 0.f);
template <typename T>
T*      AcquireActor(TSubclassOf<T> ActorClass, const FTransform& SpawnTransform,
                     AActor* Owner = nullptr, APawn* Instigator = nullptr, float AutoReleaseDelay = 0.f);
bool    ReleaseActor(AActor* Actor);                       // false if not pooled or already released
bool    SetAutoRelease(AActor* Actor, float Delay);        // Delay <= 0 cancels
int32   Prewarm(TSubclassOf<AActor> ActorClass, int32 Count, bool bSpreadOverFrames = false);
int32   ReleaseAllActive(TSubclassOf<AActor> ActorClass);  // nullptr = every actor class
int32   TrimPool(TSubclassOf<AActor> ActorClass, int32 KeepInactive = 0); // nullptr = every actor pool
void    SetPoolLimits(TSubclassOf<AActor> ActorClass, const FPoolKitPoolLimits& Limits);
FPoolKitPoolLimits GetPoolLimits(TSubclassOf<AActor> ActorClass) const;
EPoolKitActorState GetActorState(const AActor* Actor) const;
FPoolKitPoolStats  GetPoolStats(TSubclassOf<AActor> ActorClass) const;
TArray<FPoolKitPoolStats> GetAllPoolStats() const;         // actor pools
void    ResetStats(TSubclassOf<AActor> ActorClass);         // nullptr = every pool of every kind
FPoolKitActorEvent OnActorAcquired, OnActorReleased;

// Widgets
UUserWidget* AcquireWidget(TSubclassOf<UUserWidget> WidgetClass, APlayerController* OwningPlayer = nullptr, float AutoReleaseDelay = 0.f);
template <typename T> T* AcquireWidget(TSubclassOf<T> WidgetClass, ...);
bool    ReleaseWidget(UUserWidget* Widget);
bool    SetWidgetAutoRelease(UUserWidget* Widget, float Delay);
int32   PrewarmWidgets(TSubclassOf<UUserWidget> WidgetClass, int32 Count);

// Components
USceneComponent* AcquireComponent(const FPoolKitComponentRequest& Request);  // Kind, ComponentClass, Asset, Transform,
template <typename T> T* AcquireComponent(const FPoolKitComponentRequest&);  // AttachParent, socket, LocationType,
bool    ReleaseComponent(UActorComponent* Component);                        // AutoReleaseDelay, ReleaseWhenInactive
bool    SetComponentAutoRelease(UActorComponent* Component, float Delay);
int32   PrewarmComponents(EPoolKitPoolKind Kind, UObject* Asset, int32 Count, UClass* ComponentClass = nullptr);
void    NotifyComponentFinished(UActorComponent* Component);
static void RegisterComponentHandler(const FPoolKitComponentHandler& Handler);  // add your own component kinds
static void UnregisterComponentHandler(EPoolKitPoolKind Kind);

// Any kind
EPoolKitActorState GetObjectState(const UObject* Object) const;
TArray<FPoolKitPoolInfo> GetAllPoolInfos() const;
int32   PrewarmPoolOfKind(EPoolKitPoolKind Kind, UObject* PooledObject, int32 Count);
int32   ReleaseAllOfKind(EPoolKitPoolKind Kind, UObject* PooledObject);   // PooledObject null = every pool of Kind
int32   TrimPoolOfKind(EPoolKitPoolKind Kind, UObject* PooledObject, int32 KeepInactive = 0);
int32   ReleaseEverything();
int32   TrimEverything(int32 KeepInactive = 0);

// Auto-tune recording
void    StartAutoTuneRecording();
FString StopAutoTuneRecording(bool bSaveToDisk = true);
bool    IsAutoTuneRecording() const;
FPoolKitAutoTuneSession GetAutoTuneSession() const;
```

Other headers: `PoolKitBlueprintLibrary.h` (static nodes), `PoolKitAutoTune.h` (`UPoolKitAutoTune`: recordings,
`ComputeRecommendations`, `ApplyRecommendations`), `PoolKitNiagaraLibrary.h` (PoolKitNiagara module),
`PoolKitNetStateComponent.h` (replicated state), `PoolKitPoolable.h` (interface), `PoolKitSettings.h`
(`UPoolKitSettings`, read with `GetDefault<UPoolKitSettings>()`), `PoolKitTypes.h` (structs and enums),
`Tests/PoolKitTestWorld.h` (automation test helpers). Log category: `LogPoolKit`.

**Re-entrancy:** you can acquire, release or destroy objects from inside `OnAcquiredFromPool`,
`OnReleasedToPool`, BeginPlay or overlap events. PoolKit re-validates its state after every call into user code.

## 14. Stats overlay, console commands and benchmark

Console commands are available in Editor, Development and Debug builds (not Shipping). Run them during Play.

| Command | Description |
|---|---|
| `PoolKit.ShowStats 1` / `0` | Overlay per pool of every kind: kind, active, inactive, peak, spawned, reuse %, recycled, failed, warnings (SIZE, LEAK, 2xREL). Rows turn yellow/orange when the reuse ratio is low and red when requests fail or a warning is raised. |
| `PoolKit.Dump` | Writes all pools, counters, limits, memory estimates and warnings to the log, plus the replicated pooled actors this machine sees. |
| `PoolKit.Prewarm <Class> <Count> [spread]` | Prewarms an actor class. Class may be a short name (`BP_Bullet_C`, `PoolKitDemoProjectile`), a class path (`/Game/BP_Bullet.BP_Bullet_C`) or a Blueprint asset path (`/Game/BP_Bullet`). |
| `PoolKit.Trim [KeepInactive]` | Destroys inactive objects in every pool of every kind, keeping at most *KeepInactive* per pool. |
| `PoolKit.ReleaseAll` | Releases every active pooled object of every kind. |
| `PoolKit.Demo [SpawnsPerSecond] [nopool\|replicated\|all]` / `PoolKit.Demo off` | Toggles a projectile fountain in front of the camera (see below). |
| `PoolKit.Benchmark [Count] [Class]` | Times `SpawnActor` + `Destroy` against `AcquireActor` + `ReleaseActor` for *Count* actors (default 1000, default class PoolKitDemoProjectile). |
| `PoolKit.AutoTune.Start` / `.Stop` / `.Diff [Headroom]` / `.Apply [Headroom]` / `.Clear` | Auto-tune recording and recommendations (section 10). |
| `PoolKit.AutoTune.Review`, `PoolKit.OpenDebugger` | Editor only: open the review window or the debugger tab. |

About the benchmark: it measures game-thread time of the calls themselves on your machine. The cost of
garbage-collecting destroyed actors comes later and isn't included. The one-time prewarm cost is shown
separately. Results depend on the class: actors with many components or heavy BeginPlay logic gain the
most from pooling. The actors are placed far above the level. Afterwards the pool is trimmed back to its
previous size, and if the class had no pool activity before, its counters are reset so the overlay isn't skewed.

## 15. Demo without any content

PoolKit ships no assets. The demo uses only engine content:

- **PoolKit Demo Projectile**: a small bouncing sphere with Projectile Movement. It implements
  PoolKit Poolable, gets a new random color on every acquire (so reuse is visible), and counts its acquires,
  releases and BeginPlay calls. With *Impact Effects* on, its first bounce spawns a pooled decal, a pooled mesh marker
  and a pooled damage-number widget.
- **PoolKit Demo Replicated Projectile**: the same, replicated (with movement), for multiplayer tests.
- **PoolKit Demo Damage Number**: a C++ widget (no Widget Blueprint) that floats up and fades.
- **PoolKit Demo Spawner**: a fountain actor. Settings: Projectile Class, Spawns Per Second, Projectile
  Lifetime, Cone Half Angle, **Use Pool** (off = plain Spawn Actor + Set Life Span, for comparison) and Impact Effects.

To try it: open any level with a floor (for example the default *Basic* map), press Play, and run
`PoolKit.Demo 200`. The fountain is placed on the floor about 10 m in front of the camera. You can also drag
*PoolKit Demo Spawner* from the Place Actors panel into the level. Variants:

- `PoolKit.Demo 90 all`: adds pooled decals, mesh markers and damage numbers on impact (four pool kinds at once;
  open the PoolKit Debugger to watch them). The spawner enables an invisible ground at its base in this mode, so the
  projectiles bounce even on levels whose floor has no collision.
- `PoolKit.Demo 200 nopool`: plain spawning, for comparison with `stat unit`.
- `PoolKit.Demo 30 replicated`: run on a listen server (or the server of a PIE session with several players) to see
  replicated pooled projectiles on the clients.

## 16. Automation tests

PoolKit includes 27 automation tests. Open **Tools > Session Frontend > Automation** (called Test Automation in some
versions), filter for `PoolKit` and run them. They cover actors (acquire/release/reuse, projectile restart, misuse,
limits, prewarm, auto-release, destroyed actors, release-all/trim, demo, console), widgets (reuse and reset, limits from
settings, auto-release), sounds, decals (recycling, fade), static meshes (attach, parent destroyed), generic components,
Niagara (finish, reuse, attach, prewarm and recycling from settings), auto-tune (recording, JSON round trip,
recommendations, applying to settings, prewarm from applied settings), warnings, pool infos for all kinds, multiplayer
state (a listen-server test world publishing state and dormancy; a client copy applying replicated states) and the
editor windows. From the command line:

```
UnrealEditor-Cmd.exe YourProject.uproject -ExecCmds="Automation RunTests PoolKit" -TestExit="Automation Test Queue Empty" -unattended -nullrhi
```

The Niagara tests load a template system that ships with the Niagara plugin; the sound tests use an engine sound.

## 17. Limitations and FAQ

**Is it replicated / multiplayer ready?** Yes for replicated actors pooled on the server (section 8, with its table of
what is and isn't supported). Widget and component pools are local to each machine.

**My actor doesn't move after reuse.** The root component must be *Movable*. Static actors can't be
teleported to the new spawn transform.

**Something I set in BeginPlay isn't reset.** BeginPlay runs only when the actor is first created. Move
per-use setup into *On Acquired From Pool*. Components without *Auto Activate* aren't re-activated
automatically; activate them there too.

**Does it pool plain UObjects?** No. PoolKit pools actors, user widgets and scene components.

**Are subclasses pooled together?** No. Each exact class has its own pool, so `BP_Bullet` and `BP_BigBullet`
never hand out each other's actors.

**What about level changes?** Pools belong to the world. When the map changes, the old world and its pooled
objects are destroyed and the new world starts with empty pools, which get prewarmed again from Project
Settings if configured.

**Does it work with World Partition / level streaming?** Pooled actors are spawned into the persistent
level, so streaming a sublevel out doesn't remove them.

**Where do inactive actors go?** They stay where they were released, hidden, with collision, tick and
components off. They aren't rendered and don't collide.

**Does auto-tune change anything by itself?** No. It only records. Settings change when you click Apply (or call
Apply Recommendations / `PoolKit.AutoTune.Apply`).

**Can I turn off recording?** Project Settings > PoolKit > Auto-Tune > Record Every Play Session.

---

Support: open an issue on this repository, or comment on the itch.io page: https://barbed-wire-glove-games.itch.io/poolkit
