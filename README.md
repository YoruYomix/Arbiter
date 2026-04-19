# Arbiter

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![.NET](https://img.shields.io/badge/.NET-Standard%202.0+-blue.svg)](#requirements)

A generic **arbiter pattern** library for C# that solves the universal problem of *"choosing one from many candidates."*

Works everywhere .NET runs — Unity, Godot (.NET), ASP.NET, console apps, and more.

## Table of Contents

- [Why Arbiter?](#why-arbiter)
- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Core Concepts](#core-concepts)
  - [Strategy Pipeline](#strategy-pipeline)
  - [Rules and Scorers](#rules-and-scorers)
  - [Context](#context)
  - [History](#history)
- [Creating an Arbiter](#creating-an-arbiter)
- [Resolving Candidates](#resolving-candidates)
- [Runtime Modifications](#runtime-modifications)
  - [OverrideStrategy](#overridestrategy)
  - [Override](#override)
  - [Insert](#insert)
  - [Suppress](#suppress)
  - [Weight Competition](#weight-competition)
- [Debugging](#debugging)
  - [Snapshots](#snapshots)
  - [Subscribe](#subscribe)
  - [YAML Output](#yaml-output)
- [Advanced Usage](#advanced-usage)
  - [Strategy Grouping via Interfaces](#strategy-grouping-via-interfaces)
  - [Immunity via Weights](#immunity-via-weights)
- [DI Integration](#di-integration)
- [Examples](#examples)
- [Design Principles](#design-principles)
- [Thread Safety](#thread-safety)
- [License](#license)

## Why Arbiter?

Games and applications constantly face the same class of problem: *multiple candidates, pick one.*

- **Souls-like lock-on**: Multiple enemies in range — who gets targeted?
- **RPG interaction**: An NPC, a chest, and a door overlap — what does the F key activate?
- **Auto-battle**: A healer has several wounded allies — who gets healed first?
- **3D action**: Wall-climb, ledge-drop, and ladder are all available — which takes priority?
- **TCG**: Several triggered effects can fire simultaneously — which one resolves?
- **Resource management**: Under memory pressure — which loaded asset gets unloaded first?

The typical solution is `if-else` chains, hardcoded behavior tree leaves, or state machine transition guards. Every new condition forces changes to existing branches, and the combinatorial space explodes.

**Arbiter replaces all of that with a strategy pipeline.**

## Features

- **Strategy pipeline** — compose rule-based filtering and score-based ranking in any order
- **Runtime modifications** — override, suppress, or insert strategies/rules at runtime via `IDisposable` handles
- **Weight competition** — deterministic priority resolution when multiple modifications target the same slot
- **Type-driven identification** — no magic strings; strategies and rules are identified by C# types
- **Full diagnostic tools** — snapshots, YAML serialization, subscribe-to-changes, evaluation history
- **Zero dependencies** — pure C# BCL only
- **AOT safe** — compatible with IL2CPP, NativeAOT, and other AOT compilers
- **Cross-platform** — Unity, Godot (.NET), ASP.NET, console apps, or any .NET runtime

## Requirements

- .NET Standard 2.0+ (or any compatible runtime)
- No external dependencies

## Installation

### NuGet

```bash
dotnet add package Arbiter
```

### Unity (via UPM)

Add the following to your `Packages/manifest.json`:

```json
{
  "dependencies": {
    "com.yoruyomix.arbiter": "https://github.com/YoruYomix/Arbiter.git"
  }
}
```

### Manual

Copy the source files into your project. Arbiter has zero dependencies — it's just C#.

## Quick Start

```csharp
// 1. Define your candidate type
public interface IEnemy
{
    float HpRatio { get; }
    bool IsHidden { get; }
    int Row { get; }
}

// 2. Write rules and scorers
public class ExcludeHidden : IRule<IEnemy>
{
    public IReadOnlyList<IEnemy> Apply(
        IReadOnlyList<IEnemy> candidates,
        IArbiterContext context,
        ArbiterHistory history)
    {
        var visible = candidates.Where(e => !e.IsHidden).ToList();
        history.Append($"ExcludeHidden: {candidates.Count} → {visible.Count}");
        return visible.Count > 0 ? visible : candidates;
    }
}

public class HpRatioScorer : IScorer<IEnemy>
{
    public float Score(IEnemy candidate, IArbiterContext ctx, ArbiterHistory history)
        => 1f - candidate.HpRatio; // lower HP = higher score
}

// 3. Define strategy tags (empty structs)
public struct BasicRules { }
public struct BasicScoring { }

// 4. Create the arbiter
var arbiter = Arbiter<IEnemy>.Create("archer_targeting",
    new RuleStrategy<BasicRules>(new ExcludeHidden(), new BackRowFilter()),
    new ScoringStrategy<BasicScoring>(
        (new HpRatioScorer(), 0.5f),
        (new ThreatScorer(), 0.5f)
    )
);

// 5. Resolve
var result = arbiter.Resolve(enemies, context);
if (result.Count > 0)
    result[0].TakeDamage(damage);

// 6. Dispose when done
arbiter.Dispose();
```

## Core Concepts

### Strategy Pipeline

```
Candidates → Strategy[0] → Strategy[1] → ... → Strategy[N] → Final Selection
```

An arbiter executes its strategies in order. Each strategy receives the previous strategy's output as input. The arbiter doesn't care whether a strategy filters or scores — it just passes results forward.

Two strategy types can be freely mixed in the pipeline:

- **`RuleStrategy<TTag>`** — applies rules sequentially to filter candidates
- **`ScoringStrategy<TTag>`** — scores candidates with weighted scorers, keeping only the highest-scored

```csharp
var arbiter = Arbiter<IEnemy>.Create(
    new RuleStrategy<FilterTag>(rule1, rule2, rule3),
    new ScoringStrategy<ScoreTag>(
        (scorer1, 0.5f),
        (scorer2, 0.3f),
        (scorer3, 0.2f)
    ),
    new RuleStrategy<ValidationTag>(finalCheck)
);
```

### Rules and Scorers

Users implement two interfaces:

```csharp
public interface IRule<T>
{
    IReadOnlyList<T> Apply(
        IReadOnlyList<T> candidates,
        IArbiterContext context,
        ArbiterHistory history);
}

public interface IScorer<T>
{
    float Score(T candidate, IArbiterContext context, ArbiterHistory history);
}
```

**Rules** filter out candidates that don't meet conditions. **Scorers** assign a score to each candidate.

Within a `RuleStrategy`, rules execute in order — each rule's output feeds into the next. Within a `ScoringStrategy`, all scorers run independently and their scores are combined via blend ratios (weighted sum). The candidate(s) with the highest combined score are kept.

Blend ratios represent each scorer's relative contribution. They don't need to sum to 1.0 — since all candidates receive the same scale, only the relative ratios between scorers matter. `(0.5, 0.5)` and `(1.0, 1.0)` produce identical rankings.

### Context

`IArbiterContext` carries situational information into rules and scorers:

```csharp
public interface IArbiterContext { }

public class BattleContext : IArbiterContext
{
    public int CurrentTurn { get; set; }
    public bool IsBossPhase { get; set; }
}
```

Rules and scorers use pattern matching to access the concrete context type they care about. If a rule receives an irrelevant context type, it simply passes candidates through unchanged.

```csharp
public class BossPhaseFilter : IRule<IEnemy>
{
    public IReadOnlyList<IEnemy> Apply(
        IReadOnlyList<IEnemy> candidates,
        IArbiterContext ctx,
        ArbiterHistory history)
    {
        if (ctx is BattleContext battle && battle.IsBossPhase)
        {
            var bosses = candidates.Where(e => e.IsBoss).ToList();
            return bosses.Count > 0 ? bosses : candidates;
        }
        return candidates;
    }
}
```

### History

`ArbiterHistory` is a per-`Resolve` writer that lets rules and scorers record evaluation context. It's created fresh for each call and included in the result.

```csharp
history.Append($"ExcludeHidden: {candidates.Count} → {filtered.Count}");
```

Recording is optional — rules that don't need tracing can ignore the `history` parameter entirely. The accumulated entries are available via `result.History` (newline-joined string) or `result.HistoryEntries` (individual entries).

## Creating an Arbiter

Use the static factory `Arbiter<T>.Create(...)`:

```csharp
// Auto-generated id ("Arbiter_0", "Arbiter_1", ...)
var arbiter = Arbiter<IEnemy>.Create(
    new RuleStrategy<BasicRules>(new ExcludeHidden())
);

// Explicit id
var arbiter = Arbiter<IEnemy>.Create("archer_targeting",
    new RuleStrategy<BasicRules>(new ExcludeHidden()),
    new ScoringStrategy<BasicScoring>((new HpRatioScorer(), 1.0f))
);
```

Rules and strategies can carry **weights** that protect them from runtime overrides:

```csharp
var arbiter = Arbiter<IEnemy>.Create("boss_targeting",
    new RuleStrategy<BossRules>(
        (new ForcedTauntTarget(), weight: 50), // high weight = resistant to overrides
        new ExcludeHidden()                     // default weight 0
    )
);
```

Every created arbiter registers itself in a global registry and is queryable via `Arbiter.GetSnapshots()`. Arbiters implement `IDisposable` — call `Dispose()` to unregister and clean up.

## Resolving Candidates

Two resolution methods are available:

```csharp
// Lightweight — suitable for hot paths (per-frame, per-tick)
ArbiterResult<T> result = arbiter.Resolve(candidates, context);

// Diagnostic — includes a full state snapshot
ArbiterSnapshotResult<T> result = arbiter.ResolveWithSnapshot(candidates, context);
```

Both return `Candidates` (always non-null; empty list if all filtered out) and `History`. The snapshot variant additionally captures the arbiter's configuration at the moment of evaluation.

```csharp
var result = arbiter.Resolve(enemies, context);

// Iterate directly — no need to unwrap Candidates
foreach (var target in result)
    target.TakeDamage(damage);

// Check history on failure
if (result.Count == 0)
    Debug.LogWarning($"Targeting failed:\n{result.History}");
```

## Runtime Modifications

All runtime modifications return `IDisposable` handles. Dispose the handle to revert the change. Modifications target strategies and rules by type, and compete via weights when multiple modifications target the same slot.

### OverrideStrategy

Replace an entire strategy temporarily:

```csharp
var charm = arbiter.OverrideStrategy<BasicRules>(
    new RuleStrategy<CharmRules>(new AllyFilter()),
    weight: 50
);

// Revert
charm.Dispose();
```

### Override

Replace a specific rule within a strategy:

```csharp
var upgraded = arbiter.Override<BasicRules, ExcludeHidden>(
    new ImprovedExcludeHidden(), weight: 10);

upgraded.Dispose();
```

For scorers, you can also override the blend ratio:

```csharp
var boosted = arbiter.Override<BasicScoring, HpRatioScorer>(
    new RemainingHpScorer(), blend: 0.8f, weight: 10);
```

### Insert

Add rules at specific positions:

```csharp
// At the front
var taunt = arbiter.InsertFirst<BasicRules>(new ForcedTauntTarget());

// At the end
var extra = arbiter.InsertLast<BasicRules>(new ExtraFilter());

// Before/after a specific rule
var before = arbiter.InsertBefore<BasicRules, ExcludeHidden>(new ShieldFilter());
var after = arbiter.InsertAfter<BasicRules, ExcludeHidden>(new RangeFilter());

// Across all strategies
var global = arbiter.InsertFirstAll(new ForcedTauntTarget());

taunt.Dispose();
```

### Suppress

Temporarily disable a specific rule or scorer:

```csharp
var truesight = arbiter.Suppress<BasicRules, ExcludeHidden>(weight: 10);

truesight.Dispose(); // rule resumes
```

### Weight Competition

When multiple modifications target the same slot, the highest weight wins. Equal weights favor the later modification. Handles remain alive in a priority queue — disposing the active handle promotes the next in line.

```
Queue: [Charm w:20 active] [Confuse w:10 waiting] [Original w:0 waiting]

Charm.Dispose()   → [Confuse w:10 active] [Original w:0 waiting]
Confuse.Dispose() → [Original w:0 active]
```

**Override vs Suppress interaction**: Suppress must beat `max(original weight, active override weight)` to succeed. On ties, Suppress wins — disabling is considered a stronger intent than replacing.

### All-Target Variants

Every modification API has an `...All` variant that applies to all matching strategies:

| Method | Purpose |
|---|---|
| `OverrideStrategy<T>` | Replace one strategy |
| `Override<TStrategy, TRule>` | Replace one rule in one strategy |
| `Suppress<TStrategy, TRule>` | Disable one rule in one strategy |
| `InsertFirst<T>` / `InsertLast<T>` | Insert at edges of one strategy |
| `InsertBefore<TS, TR>` / `InsertAfter<TS, TR>` | Insert relative to a rule |
| `OverrideStrategyAll` | Replace all strategies |
| `OverrideAll<TRule>` | Replace a rule across all strategies |
| `SuppressAll<TRule>` | Disable a rule across all strategies |
| `InsertFirstAll` / `InsertLastAll` | Insert across all strategies |

All return `IDisposable`.

## Debugging

### Snapshots

Capture the arbiter's state as an immutable snapshot:

```csharp
// Single arbiter
ArbiterSnapshot snap = arbiter.GetSnapshot();

// All registered arbiters
ArbiterSnapshot[] all = Arbiter.GetSnapshots();
```

Snapshots are readonly structs — safe to pass across threads after creation.

### Subscribe

React to state changes in real time:

```csharp
var subscription = arbiter.Subscribe(snap =>
{
    Debug.Log($"[Changed] {snap.Id}: {snap.Strategies.Length} strategies");
});

// Fires immediately with current state, then on every modification
subscription.Dispose(); // stop listening
```

The callback fires synchronously on the calling thread. No snapshot is allocated when there are zero subscribers.

### YAML Output

```csharp
Debug.Log(arbiter.GetSnapshot().ToYaml());
```

```yaml
id: archer_targeting
candidateType: IEnemy
strategies:
  - type: BasicRules
    overridden: false
    activeWeight: 0
    rules:
      - type: ForcedTauntTarget
        state: active
        origin: Inserted
        activeWeight: 30
      - type: ImprovedExcludeHidden
        state: active
        origin: Overridden
        originalType: ExcludeHidden
        activeWeight: 20
      - type: BackRowFilter
        state: active
        origin: OriginalCreate
        activeWeight: 0
  - type: BasicScoring
    overridden: false
    activeWeight: 0
    rules:
      - type: HpRatioScorer
        state: active
        origin: OriginalCreate
        activeWeight: 0
        blend: 0.5
      - type: ThreatScorer
        state: active
        origin: OriginalCreate
        activeWeight: 0
        blend: 0.5
```

Use `ToFullYaml()` to include pending handles (waiting overrides, suppresses) for detailed diagnostics.

## Advanced Usage

### Strategy Grouping via Interfaces

Use C# interfaces on strategy tag types to group them. Runtime modifications targeting an interface apply to all matching strategies:

```csharp
public interface IBasicRules { }

public struct ArcherBasicRules : IBasicRules { }
public struct TankBasicRules : IBasicRules { }
public struct MageBasicRules : IBasicRules { }

// Insert into all basic rule strategies at once
var taunt = arbiter.InsertFirst<IBasicRules>(new ForcedTauntTarget());

// Insert into archer only
var buff = arbiter.InsertFirst<ArcherBasicRules>(new ExcludeHidden());
```

Multiple interface inheritance lets a single strategy belong to several groups:

```csharp
public struct ArcherBasicRules : IBasicRules, IBackLineRules, IPhysicalRules { }
```

### Immunity via Weights

High creation weights make strategies or rules resistant to runtime overrides:

```csharp
// Minion: default weight 0 — easily overridden
var minionArbiter = Arbiter<IEnemy>.Create("minion",
    new RuleStrategy<MinionRules>(new FrontRowFilter())
);

// Boss: weight 100 — immune to low-weight overrides
var bossArbiter = Arbiter<IEnemy>.Create("boss",
    new RuleStrategy<BossRules>(new FrontRowFilter(), weight: 100)
);

// Charm (weight 50): overrides minion, blocked by boss
var charm = arbiter.OverrideStrategy<IBasicRules>(
    new RuleStrategy<CharmRules>(new AllyFilter()), weight: 50);
```

Rule-level weights provide fine-grained protection:

```csharp
new RuleStrategy<BossRules>(
    new FrontRowFilter(),                      // weight 0, freely overridable
    (new ForceAvoidInvincible(), weight: 100)   // protected
)
```

## DI Integration

Arbiter integrates with DI containers via factory registration. Since it implements `IDisposable`, most containers handle cleanup automatically.

### Microsoft.Extensions.DependencyInjection

```csharp
services.AddScoped<Arbiter<IEnemy>>(sp =>
    Arbiter<IEnemy>.Create("archer_targeting",
        new RuleStrategy<BasicRules>(new ExcludeHidden()),
        new ScoringStrategy<BasicScoring>((new HpRatioScorer(), 1.0f))
    )
);
```

When you need multiple arbiters for the same candidate type, use thin wrapper types or keyed services (.NET 8+):

```csharp
public sealed class ArcherTargetArbiter : IDisposable
{
    public Arbiter<IEnemy> Inner { get; }
    public ArcherTargetArbiter(Arbiter<IEnemy> inner) => Inner = inner;
    public void Dispose() => Inner.Dispose();
}
```

### Unity (VContainer)

```csharp
builder.Register<Arbiter<IEnemy>>(_ =>
    Arbiter<IEnemy>.Create("archer_targeting", ...),
    Lifetime.Scoped
);
```

### Zenject

Register via `IDisposable` bindings for automatic cleanup.

## Examples

### Souls-like Lock-On

```csharp
var lockOn = Arbiter<IEnemy>.Create("lock_on",
    new RuleStrategy<LockOnRules>(
        new ExcludeOutOfSight(),
        new ExcludeOutOfRange()
    ),
    new ScoringStrategy<LockOnScoring>(
        (new CameraDirectionScorer(), 0.4f),
        (new DistanceScorer(), 0.3f),
        (new HpRatioScorer(), 0.3f)
    )
);
```

### Auto-Battle Targeting

```csharp
var archer = Arbiter<IEnemy>.Create("archer_targeting",
    new RuleStrategy<ArcherRules>(
        new ForcedTauntTarget(),
        new ExcludeHidden(),
        new ExcludeInvincible(),
        new BackRowFilter()
    ),
    new ScoringStrategy<ArcherScoring>(
        (new HpRatioScorer(), 0.5f),
        (new DebuffCountScorer(), 0.3f),
        (new ThreatScorer(), 0.2f)
    )
);
```

### Context Action Selection (3D Action)

```csharp
var contextAction = Arbiter<IContextAction>.Create("context_action",
    new RuleStrategy<ContextRules>(
        new ExcludeUnavailable(),
        new ExcludeOnCooldown()
    ),
    new ScoringStrategy<ContextScoring>(
        (new DistanceScorer(), 0.4f),
        (new AngleScorer(), 0.3f),
        (new PriorityScorer(), 0.3f)
    )
);
```

### Asset Unload Priority

```csharp
var assetArbiter = Arbiter<LoadedAsset>.Create("asset_unload",
    new RuleStrategy<AssetUnloadRules>(
        new ExcludeCurrentSceneAssets(),
        new ExcludePreloadedAssets()
    ),
    new ScoringStrategy<AssetUnloadScoring>(
        (new LastAccessTimeScorer(), 0.4f),
        (new MemorySizeScorer(), 0.3f),
        (new ReferenceCountScorer(), 0.3f)
    )
);
```

## Design Principles

- **Single concept** — an arbiter executes a strategy array in order. That's it.
- **Two strategies, one pipeline** — rule strategies and scoring strategies mix freely.
- **Types are identifiers** — no string keys; strategy tags are C# types.
- **Composition over complexity** — build from small, reusable rules and scorers.
- **IDisposable + weights** — the only mechanism for all runtime modifications.
- **Zero dependencies** — pure C# BCL; no third-party packages.
- **AOT safe** — works with IL2CPP, NativeAOT, and other AOT compilers.
- **Single-threaded by design** — no locking overhead for the common case.

## Thread Safety

Arbiter assumes single-threaded access. All creation, resolution, runtime modification, and disposal should happen on the main thread.

**Not safe across threads:**
- Concurrent `Resolve` calls on the same arbiter
- `Resolve` on one thread while modifying on another
- Concurrent `Create` or `Dispose` calls

**Safe:**
- All API calls from a single thread
- Reading returned snapshots and results from any thread (they are immutable after creation)
- Calling `ToYaml()` / `ToFullYaml()` / `ToString()` on snapshots and results from any thread

**Recommended pattern**: perform all arbiter operations on the main thread. If another thread needs arbiter state, capture a snapshot on the main thread first, then pass it across.

## License

[MIT](LICENSE)
