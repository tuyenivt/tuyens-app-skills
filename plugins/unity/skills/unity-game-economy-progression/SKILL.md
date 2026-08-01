---
name: unity-game-economy-progression
description: Model idle and tower-defense economies - offline progress, clock-tamper limits, big-number overflow, prestige loops, data-driven balance.
metadata:
  category: mobile
  tags: [unity, idle, economy, progression, offline-progress, bignumber, balance, scriptableobject, clock]
user-invocable: false
---

# Unity Game Economy and Progression

> This skill owns **currency, progression, and time-based accrual math**. The `IClock` seam and the engine-free rules boundary belong to `unity-architecture-patterns`; board and turn resolution belongs to `unity-2d-gameplay-patterns`; storing and migrating the save belongs to `unity-save-persistence`; server-side receipt validation and anti-tamper enforcement belong to `unity-security-patterns`.

## When to Use

- Adding offline progress, idle accrual, or timed rewards
- Balancing currency sources against sinks, or a prestige/soft-reset loop
- Numbers exceed what `long` or `double` represents accurately
- Scaling tower-defense waves, costs, or difficulty
- Reviewing economy code for tamper resistance or overflow

## Rules

- **Wall-clock time is untrusted input.** Any grant computed from `DateTime.Now` or `DateTime.UtcNow` is a value the player controls by changing the device clock. Either validate the elapsed span server-side, or clamp it and accept the residual exploit deliberately
- Elapsed time comes from a substitutable `IClock`, never from a direct `DateTime` or `Time` read inside economy math. Offline math is untestable otherwise
- Monotonic time (`Time.realtimeSinceStartup`, `Stopwatch`) is tamper-resistant but does not survive process death; wall-clock survives but is forgeable. Every offline-progress design picks a combination and states which failure it accepts
- Accrual is computed as a **closed-form function of elapsed time**, not by simulating skipped ticks. A one-week absence must not run a week of steps
- Currency totals that can exceed `double`'s exact-integer range use a big-number representation. Silent precision loss is the failure, not overflow to a wrong sign
- Balance numbers - costs, rates, curve coefficients, wave tables - live in ScriptableObjects or imported data, never as literals in gameplay code
- Every currency has an enumerated set of sources and sinks. A currency with no sink inflates; a sink with no source is dead content

## Patterns

### The clock boundary

The exploit is one line long: set the device clock forward a year, collect a year of idle income, set it back.

| Time source | Survives app kill | Tamper-resistant | Use for |
| --- | --- | --- | --- |
| `DateTime.UtcNow` | yes | no | offline elapsed, when clamped or server-checked |
| `Time.realtimeSinceStartup` | no (resets) | yes | in-session timers, cooldowns while running |
| `Stopwatch` / monotonic OS tick | no | yes | in-session accrual, background-duration checks |
| Server timestamp | yes | yes | authoritative grants, anything monetised |

```csharp
// Bad - the device clock is the authority; forward-setting mints currency
var elapsed = DateTime.UtcNow - save.LastSeenUtc;
Grant(rate * elapsed.TotalSeconds);

// Good - bounded, and backward jumps yield nothing instead of negative or huge values
var raw = _clock.UtcNow - save.LastSeenUtc;
var elapsed = raw < TimeSpan.Zero ? TimeSpan.Zero : (raw > MaxOffline ? MaxOffline : raw);
```

Three defences, in increasing strength:

1. **Clamp** to a maximum offline window (a design number anyway - most idle games cap at 2-12 hours). Costs nothing, bounds the exploit to one cap per app launch
2. **Detect backwards movement**: a `LastSeenUtc` in the future, or a wall-clock delta far exceeding the monotonic delta measured while running, marks the save suspicious. Record the flag, degrade the grant, do not silently continue
3. **Server timestamp** for anything with real-money consequence. The server is the only authority a modified client cannot forge; enforcement design belongs to `unity-security-patterns`

Store `LastSeenUtc` in UTC. Local time makes a timezone flight indistinguishable from tampering, and a DST transition mints or destroys an hour.

Write `LastSeenUtc` on pause and focus loss, not only on quit - mobile processes are killed without a quit callback.

### Offline accrual as a closed form

```csharp
// Bad - a week offline runs 604,800 iterations on the main thread at resume
for (var t = 0; t < elapsedSeconds; t++) sim.Step(1f);

// Good - constant time regardless of absence length
var earned = ratePerSecond * elapsedSeconds;
```

Non-linear accrual still has a closed form. Compound growth over elapsed time `t` at rate `r` is `initial * Math.Pow(1 + r, t)`, not a loop. Where a genuinely non-analytic system (a TD wave simulation) must advance, run a **capped** number of coarse steps and state the approximation in the design, rather than an unbounded catch-up.

Offline earnings are conventionally reduced against online earnings (a fraction of the online rate). Put that fraction in the balance data, not in the accrual code.

### Big numbers

`double` represents integers exactly only up to 2^53 (about 9.0e15); `long` overflows at about 9.2e18. Idle games with exponential growth cross both, and the `double` failure is worse because it is silent: additions of small values to a large total simply stop having an effect.

```csharp
// Bad - past 2^53, adding 1 gold to the total changes nothing at all
double gold = 9.1e15;
gold += 1; // gold is unchanged

// Good - mantissa and exponent kept separate and normalised
public readonly struct BigDouble { public readonly double Mantissa; public readonly int Exponent; }
```

Choose deliberately:

| Range needed | Representation |
| --- | --- |
| below ~9e15 | `long`, exact, checked for overflow |
| unbounded exponential growth | mantissa-plus-exponent struct (a `BigDouble`-style type) |
| exact arbitrary precision | `System.Numerics.BigInteger` - correct but allocating; wrong for per-frame math |

Requirements for a mantissa/exponent type, all of which bite in practice: normalise after every operation, define comparison and equality (mantissa comparison after exponent comparison), define a display format (scientific, engineering, or named tiers - K/M/B/T then AA/AB), and make it serialisable in a form that round-trips exactly through the save (`unity-save-persistence`). Write it as a `readonly struct` in the rules assembly so it stays engine-free and allocation-free.

Adding a value more than ~17 orders of magnitude below the total is a no-op in any mantissa representation. That is correct behaviour, not a bug, but the UI must not show a "+1" that never lands.

### Currency, sources, and sinks

Every currency gets an explicit table before any of it is implemented:

| Currency | Sources | Sinks | Resets on prestige |
| --- | --- | --- | --- |
| Soft (coins) | idle accrual, level clear, ad reward | upgrades, retries, cosmetics | yes |
| Hard (gems) | IAP, rare milestones | premium upgrades, skips | no |
| Prestige (stars) | prestige conversion | permanent multipliers | no |

Two rules this table enforces: hard currency earned and hard currency purchased should be tracked separately (refunds, analytics, and regional pricing all need the split), and a currency with no sink is inflation the player will notice within a session.

Grant and spend go through one guarded path:

```csharp
// Bad - negative balances, silent underflow, no audit trail
wallet.Coins -= cost;

// Good - the caller must handle failure, and every mutation has a reason attached
public bool TrySpend(CurrencyId id, BigDouble cost, string reason);
```

The `reason` string feeds analytics and makes economy bugs diagnosable.

### Prestige and soft reset

A prestige loop trades current progress for a permanent multiplier. The design points that break implementations:

- **Conversion is a curve, not a ratio.** `stars = floor(k * sqrt(lifetimeEarned / threshold))` and similar sublinear forms keep late resets meaningful without runaway. Put `k` and the threshold in balance data
- **The reset must be an explicit list of what is cleared and what persists**, held as data. An implicit "clear everything except these fields" reset drifts every time a field is added, and the resulting bug destroys player progress
- Preview the exact gain before confirming; an irreversible reset with a surprise result is the top complaint in the genre
- Reset is a save-schema event. Version it and make it idempotent, so an interrupted reset does not double-apply (`unity-save-persistence`)

### Balance as data

```csharp
// Bad - a rebalance is a code change, a rebuild, and a store release
private const float UpgradeCost = 25f * 1.15f;

// Good - authored, tunable, and shippable without a client build
[CreateAssetMenu] public sealed class EconomyConfig : ScriptableObject {
    [SerializeField] private AnimationCurve costCurve;
    public float CostAt(int level) => costCurve.Evaluate(level);
}
```

Format by use:

| Data | Format |
| --- | --- |
| A handful of tunables, designer-edited in the editor | ScriptableObject with serialized fields |
| Wave tables, level tables, hundreds of rows | CSV or JSON imported into a ScriptableObject at build time, validated on import |
| Values that must change after release | remote config, with the shipped ScriptableObject as the fallback |

Validate on import, not at first use: a wave table with a missing column should fail the import, not produce a null-reference on wave 40 in production. Curve shapes are worth checking too - a cost curve that dips is an infinite-money exploit, and it is cheap to assert monotonicity at import.

Never mutate the ScriptableObject at runtime to hold current progress; that edit persists in the editor and diverges from the build (`unity-architecture-patterns`).

### Difficulty, cost, and wave curves

Three shapes cover nearly all of it:

| Shape | Formula | Fits |
| --- | --- | --- |
| Linear | `base + k * n` | early tutorial pacing, small counts |
| Geometric | `base * r^n` (typically `r` in 1.07-1.15) | upgrade costs, idle income tiers |
| Sublinear | `base * n^p`, `p < 1` | prestige conversion, catch-up bonuses |

Geometric costs against geometric income is the standard idle balance: keep the income ratio slightly above the cost ratio and the player advances; invert it and progress stalls. Express both in the same balance asset so the relationship is visible rather than emergent.

For tower-defense waves, scale count, health, and reward on separate curves. One shared multiplier makes a wave that is simultaneously unbeatable and unrewarding, and a reward curve that outpaces cost growth trivialises the run. Verify by simulating the curves headlessly in an EditMode test - the rules layer is engine-free, so a hundred waves run in milliseconds.

### Testing economy math

Because accrual takes `IClock` and randomness takes `IRandom` (`unity-architecture-patterns`), the whole economy is testable without Play mode:

```csharp
// Advance a fake clock by a week; assert the grant equals the clamp, not the raw elapsed
var clock = new FakeClock(start); clock.Advance(TimeSpan.FromDays(7));
Assert.AreEqual(MaxOffline.TotalSeconds * rate, Compute(save, clock));
```

Cases worth having: clock moved backwards, clock moved forward past the clamp, elapsed exactly zero, a total near the representation's precision ceiling, and prestige applied twice from the same pre-reset save.

## Output Format

When reviewing, emit one block per finding:

```
### [Severity] {file:line | asset path | symptom, when no source was supplied}

- Category: {ClockTrust | OfflineAccrual | NumericPrecision | CurrencyFlow | PrestigeReset | BalanceHardcoding | CurveShape | TimeInjection}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation - code, config value, or curve setting; or `not supplied` when the finding is inferred}
- Impact: {what it allows or breaks - "clock change mints unlimited currency", "totals stop increasing past 9e15"}
- Fix: {concrete change}
```

`Severity: {Critical | High | Medium | Low}` - Critical = an unbounded currency grant from player-controllable input, or progress loss on reset or migration. High = silent precision loss, an offline catch-up that hangs at resume, or a monetised grant with no server check. Medium = balance hardcoded in code, a curve flaw affecting pacing, or an untestable time read. Low = a naming, structure, or documentation nit in economy data.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`; where it fits none, pick the closest and name the real concern in `Impact`.

`Evidence: inferred` is required whenever the source was not read, and caps the block at High - write the capped severity in the header, not the uncapped one, and name the uncapped band in `Impact`.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. Omit entirely when there are none.

Close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No economy findings.` |
| No source, diff, symptom, or report of any kind was supplied | `Economy check not run: no source supplied.` |

A symptom-only report (a QA ticket, a verbal description) is checkable input: emit `Evidence: inferred` findings from it rather than the not-run line.

## Avoid

- `DateTime.Now` or `DateTime.UtcNow` read directly inside economy math
- Local time stored as the last-seen timestamp
- Offline elapsed time used unclamped and unchecked
- Backward clock movement producing a negative or absolute elapsed span
- Simulating skipped ticks to catch up an absence
- `double` or `long` totals in a game with exponential growth
- A mantissa/exponent type without normalisation, comparison, and exact save round-trip
- Direct arithmetic on a currency field instead of a guarded `TrySpend`
- Prestige reset written as "clear everything except" rather than an explicit data-driven list
- Balance constants as literals in gameplay code
- Data tables validated at first use rather than at import
- ScriptableObject balance assets mutated at runtime
- Wave count, health, and reward driven by one shared multiplier
