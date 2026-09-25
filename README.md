# PoolKit: Zero-Setup Actor Pooling for Unreal Engine

PoolKit lets you reuse actors instead of spawning and destroying them, from Blueprint or C++.
Replace **Spawn Actor from Class** with **Spawn Actor From Pool**, and **Destroy Actor** with
**Release Actor To Pool**. That's all the setup there is.

- No manager actor to place, no base class to inherit, no interface required.
- Pooled actors are reset automatically, so a reused actor behaves like a freshly spawned one.
- Optional prewarm, pool limits, auto-release, live stats overlay, console commands and a built-in benchmark.
- Runtime C++ module with no dependencies beyond the engine. No content, so it works in any project.

Supported engine versions: 5.4, 5.5, 5.6, 5.7. Tested on Windows (Win64).

---

## Contents

1. [Installation](#1-installation)
2. [Quick start (Blueprint)](#2-quick-start-blueprint)
3. [Quick start (C++)](#3-quick-start-c)
4. [How pooling works in PoolKit](#4-how-pooling-works-in-poolkit)
5. [The PoolKit Poolable interface](#5-the-poolkit-poolable-interface)
6. [Project Settings](#6-project-settings)
7. [Blueprint node reference](#7-blueprint-node-reference)
8. [C++ API reference](#8-c-api-reference)
9. [Stats overlay, console commands and benchmark](#9-stats-overlay-console-commands-and-benchmark)
10. [Demo without any content](#10-demo-without-any-content)
11. [Automation tests](#11-automation-tests)
12. [Limitations and FAQ](#12-limitations-and-faq)

---

## 1. Installation

**From Fab:** install PoolKit to your engine from the Epic Games Launcher (Fab library), then enable it in
**Edit > Plugins** (search "PoolKit") and restart the editor.

**Manual (project plugin):** copy the `PoolKit` folder into `<YourProject>/Plugins/` and restart the editor.
C++ projects rebuild it automatically; Blueprint-only projects need the prebuilt binaries from Fab.

**C++ projects** that call PoolKit from code add the module to their `.Build.cs`:

```csharp
PublicDependencyModuleNames.Add("PoolKit");
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

Press Play, open the console (`~`) and type `PoolKit.ShowStats 1` to watch the pool work.

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

Optional. Implement it on the actor or on **any component** of the pooled actor. Both are called.

| Event | When |
|---|---|
| `On Acquired From Pool` | After every acquire, including the first. Transform, owner and instigator are already set. |
| `On Released To Pool` | At the start of every release, while the actor is still visible and active. |

Global notifications are also available as the subsystem's `OnActorAcquired` / `OnActorReleased` events
(Blueprint: *Get PoolKitSubsystem > Bind Event to On Actor Acquired*).

## 6. Project Settings

**Project Settings > Plugins > PoolKit** (saved to `Config/DefaultGame.ini`):

| Setting | Default | Meaning |
|---|---|---|
| Class Configs | empty | Per-class entries: *Actor Class*, *Prewarm Count*, *Limits* |
| Default Limits | 0 / 0 / off | Limits for classes without an entry. 0 = unlimited. |
| Prewarm On World Begin Play | on | Prewarm the Class Configs when a game world begins play |
| Spread Begin Play Prewarm Over Frames | on | Spawn those a few per frame instead of all in the first frame |
| Prewarm Actors Per Frame | 8 | Budget for any prewarm that is spread over frames |
| Reset Components On Reuse | on | Deactivate components on release, re-activate Auto Activate components on acquire |
| Clear Timers On Release | on | Clear timers and latent actions of the actor and its components on release |
| Restart Projectile Movement On Acquire | on | Restart Projectile Movement with its initial speed on acquire |
| Detach On Release | on | Detach the actor from its parent on release |
| Warn On Misuse | on | Log a warning for double releases or releasing actors PoolKit did not create |

**Limits** (`FPoolKitPoolLimits`):

| Field | Meaning |
|---|---|
| Max Inactive | Most actors kept waiting in the pool. Extra released actors are destroyed. 0 = unlimited. |
| Max Active | Most actors of the class in use at once. 0 = unlimited. |
| Recycle Oldest When Full | When Max Active is reached: on = release and reuse the oldest active actor (bullets, decals, shell casings); off = the request returns None. |

Classes in Class Configs are loaded synchronously at world begin play when their Prewarm Count is above 0.

## 7. Blueprint node reference

All nodes are in the **PoolKit** category.

| Node | Description |
|---|---|
| **Spawn Actor From Pool** (Class, Spawn Transform, Auto Release Delay, *Owner*, *Instigator*) returns Actor | Typed output. Auto Release Delay > 0 returns the actor to the pool after that many seconds. Returns None for invalid or abstract classes, outside game worlds, or when Max Active is reached with recycling off. |
| **Release Actor To Pool** (Actor, Destroy If Not Pooled = true) returns bool | Returns the actor to its pool. With *Destroy If Not Pooled*, actors that PoolKit did not create are destroyed, so the node is always safe to use in place of Destroy Actor. Returns true if it was an active pooled actor and got released (if the pool was already at Max Inactive, it was destroyed instead). |
| **Release Actor To Pool After** (Actor, Delay) returns bool | Schedules (or re-schedules) an automatic release. Delay <= 0 cancels a scheduled release. |
| **Prewarm Pool** (Class, Count, Spread Over Frames) returns int | Makes sure the pool holds at least *Count* actors (active plus inactive). Returns the number spawned, or queued when spreading. |
| **Get Pooled Actor State** (Actor) returns EPoolKitActorState | *Not Pooled*, *Active* or *Inactive*. |
| **Is Pooled Actor** (Actor) returns bool | True if PoolKit created the actor. |
| **Get Pool Stats** (Class) returns FPoolKitPoolStats | Counters of one pool (see below). |
| **To String (PoolKit Pool Stats)** | One-line text for Print String. Converts automatically when you connect stats to a string pin. |

More functions are on the subsystem node (**Get PoolKitSubsystem**): `Release All Active`, `Trim Pool`,
`Set Pool Limits`, `Get Pool Limits`, `Get All Pool Stats`, `Reset Stats` (one class or all), `Set Auto Release`, and the
`On Actor Acquired` / `On Actor Released` events.

**FPoolKitPoolStats fields:** Actor Class, Num Active, Num Inactive, Peak Active, Total Spawned, Total Acquires,
Total Reuses, Total Releases, Total Destroyed, Total Recycled, Total Failed Acquires, Pending Prewarm,
Reuse Ratio (0..1), Limits.

## 8. C++ API reference

`#include "PoolKitSubsystem.h"`: `UPoolKitSubsystem : UWorldSubsystem`

```cpp
static UPoolKitSubsystem* Get(const UObject* WorldContextObject);   // null outside game/PIE worlds

AActor* AcquireActor(TSubclassOf<AActor> ActorClass, const FTransform& SpawnTransform,
                     AActor* Owner = nullptr, APawn* Instigator = nullptr, float AutoReleaseDelay = 0.f);
template <typename T>
T*      AcquireActor(TSubclassOf<T> ActorClass, const FTransform& SpawnTransform,
                     AActor* Owner = nullptr, APawn* Instigator = nullptr, float AutoReleaseDelay = 0.f);

bool    ReleaseActor(AActor* Actor);                       // false if not pooled or already released
bool    SetAutoRelease(AActor* Actor, float Delay);        // Delay <= 0 cancels
int32   Prewarm(TSubclassOf<AActor> ActorClass, int32 Count, bool bSpreadOverFrames = false);
int32   ReleaseAllActive(TSubclassOf<AActor> ActorClass);  // nullptr = every class
int32   TrimPool(TSubclassOf<AActor> ActorClass, int32 KeepInactive = 0); // nullptr = every pool
void    SetPoolLimits(TSubclassOf<AActor> ActorClass, const FPoolKitPoolLimits& Limits);
FPoolKitPoolLimits GetPoolLimits(TSubclassOf<AActor> ActorClass) const;
EPoolKitActorState GetActorState(const AActor* Actor) const;
FPoolKitPoolStats  GetPoolStats(TSubclassOf<AActor> ActorClass) const;
TArray<FPoolKitPoolStats> GetAllPoolStats() const;
void    ResetStats(TSubclassOf<AActor> ActorClass);   // nullptr = every pool

FPoolKitActorEvent OnActorAcquired;   // dynamic multicast, (AActor* Actor)
FPoolKitActorEvent OnActorReleased;
```

Other headers: `PoolKitBlueprintLibrary.h` (static nodes), `PoolKitPoolable.h` (interface),
`PoolKitSettings.h` (`UPoolKitSettings`, read with `GetDefault<UPoolKitSettings>()`), `PoolKitTypes.h` (structs and enum).
Log category: `LogPoolKit`.

**Re-entrancy:** you can acquire, release or destroy actors from inside `OnAcquiredFromPool`,
`OnReleasedToPool`, BeginPlay or overlap events. PoolKit re-validates its state after every call into user code.

## 9. Stats overlay, console commands and benchmark

Console commands are available in Editor, Development and Debug builds (not Shipping). Run them during Play.

| Command | Description |
|---|---|
| `PoolKit.ShowStats 1` / `0` | Overlay per pool: active, inactive, peak, spawned, reuse %, recycled, failed. Rows turn yellow/orange when the reuse ratio is low (pool too small) and red when acquires fail (Max Active reached). |
| `PoolKit.Dump` | Writes all pools, counters and limits to the log. |
| `PoolKit.Prewarm <Class> <Count> [spread]` | Prewarms a class. Class may be a short name (`BP_Bullet_C`, `PoolKitDemoProjectile`), a class path (`/Game/BP_Bullet.BP_Bullet_C`) or a Blueprint asset path (`/Game/BP_Bullet`). |
| `PoolKit.Trim [KeepInactive]` | Destroys inactive actors in every pool, keeping at most *KeepInactive* per pool. |
| `PoolKit.ReleaseAll` | Releases every active pooled actor. |
| `PoolKit.Demo [SpawnsPerSecond] [nopool]` / `PoolKit.Demo off` | Toggles a projectile fountain in front of the camera (see below). |
| `PoolKit.Benchmark [Count] [Class]` | Times `SpawnActor` + `Destroy` against `AcquireActor` + `ReleaseActor` for *Count* actors (default 1000, default class PoolKitDemoProjectile), and prints the results on screen and in the log. |

About the benchmark: it measures game-thread time of the calls themselves on your machine. The cost of
garbage-collecting destroyed actors comes later and isn't included. The one-time prewarm cost is shown
separately. Results depend on the class: actors with many components or heavy BeginPlay logic gain the
most from pooling. The actors are placed far above the level. Afterwards the pool is trimmed back to its
previous size, and if the class had no pool activity before, its counters are reset so the overlay isn't skewed.

## 10. Demo without any content

PoolKit ships no assets. The demo uses only engine basic shapes:

- **PoolKit Demo Projectile**: a small bouncing sphere with Projectile Movement. It implements
  PoolKit Poolable, gets a new random color on every acquire (so reuse is visible), and counts its acquires,
  releases and BeginPlay calls. It bounces off the level but ignores other projectiles and pawns.
- **PoolKit Demo Spawner**: a fountain actor. Settings: Projectile Class, Spawns Per Second, Projectile
  Lifetime, Cone Half Angle, and **Use Pool** (off = plain Spawn Actor + Set Life Span, for comparison).

To try it: open any level with a floor (for example the default *Basic* map), press Play, and run
`PoolKit.Demo 200`. The fountain is placed on the floor about 10 m in front of the camera. You can also drag *PoolKit Demo Spawner* from the Place Actors panel into the level.
Run `stat unit` next to `PoolKit.ShowStats 1`, or compare with `PoolKit.Demo 200 nopool`.

## 11. Automation tests

PoolKit includes automation tests. Open **Tools > Session Frontend > Automation** (called Test Automation in some
versions), filter for `PoolKit` and run them. They cover acquire/release/reuse, projectile restart, misuse
(double release, foreign actors, null class), Max Inactive, Max Active with and without recycling,
immediate and spread prewarm, auto-release and timer clearing, destroyed pooled actors, and release-all/trim.
From the command line:

```
UnrealEditor-Cmd.exe YourProject.uproject -ExecCmds="Automation RunTests PoolKit" -TestExit="Automation Test Queue Empty" -unattended -nullrhi
```

## 12. Limitations and FAQ

**Is it replicated / multiplayer ready?** No. Pools are local to each world and PoolKit doesn't replicate
anything. It works for client-side cosmetic actors (impact effects, shell casings, decals) and for
server-only actors. Pooling replicated actors isn't supported or tested.

**My actor doesn't move after reuse.** The root component must be *Movable*. Static actors can't be
teleported to the new spawn transform.

**Something I set in BeginPlay isn't reset.** BeginPlay runs only when the actor is first created. Move
per-use setup into *On Acquired From Pool*. Components without *Auto Activate* aren't re-activated
automatically; activate them there too.

**Does it pool widgets, components or plain UObjects?** No. PoolKit pools actors (any AActor subclass, C++ or Blueprint).

**Are subclasses pooled together?** No. Each exact class has its own pool, so `BP_Bullet` and `BP_BigBullet`
never hand out each other's actors.

**What about level changes?** Pools belong to the world. When the map changes, the old world and its pooled
actors are destroyed and the new world starts with empty pools, which get prewarmed again from Project
Settings if configured.

**Does it work with World Partition / level streaming?** Pooled actors are spawned into the persistent
level, so streaming a sublevel out doesn't remove them.

**Where do inactive actors go?** They stay where they were released, hidden, with collision, tick and
components off. They aren't rendered and don't collide.

---

Support: see the Fab listing page for the support contact.
