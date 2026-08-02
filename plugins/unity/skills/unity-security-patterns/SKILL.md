---
name: unity-security-patterns
description: Harden Unity mobile games - save tampering, IAP and rewarded-ad grant integrity, secrets in builds, SDK data flow, deep links, store privacy.
metadata:
  category: mobile
  tags: [unity, security, iap, ads, tampering, secrets, il2cpp, privacy, gdpr, deep-links]
user-invocable: false
---

# Unity Security Patterns

> This skill owns **what an attacker or a hostile client can do**. Save file format and migration belong to `unity-save-persistence`; stripping and signing mechanics belong to `unity-build-release`.

## When to Use

- Adding IAP, rewarded ads, or any client-granted reward
- Storing player progress, currency, or entitlements
- Integrating a third-party SDK (ads, analytics, attribution)
- Handling deep links, remote config, or downloaded content
- Preparing a store submission with a privacy declaration

## Rules

- **The client is hostile.** A shipped build runs on a device the player controls: memory is editable, files are readable, traffic is interceptable, and the binary is decompilable. Any value the client asserts can be forged
- **Anything of real value is validated server-side.** IAP receipts, rewarded-ad completions, leaderboard scores, and entitlements are granted by a server that verified them, not by the client that claims them
- No secret ships in a build. Not in code, not in a ScriptableObject, not in `Resources`, not in `PlayerPrefs`. IL2CPP raises the cost of extraction; it does not prevent it
- Obfuscation and integrity checks raise attacker cost. They are not a substitute for server authority - state which one a given control is
- All external input is untrusted: deep-link parameters, remote config values, downloaded content, and clipboard. Validate type, range, and origin before use
- Third-party SDKs are a data-flow decision, not just a dependency. Know what each one collects before it ships
- A single-player game with no server has no authoritative validation available - say so explicitly and scope the controls to tamper *detection*, rather than implying protection it cannot have. A managed backend (Firebase, PlayFab, Unity Gaming Services) counts as a server: cloud functions and server-to-server callbacks provide authority without self-hosting

## Patterns

### What actually needs protecting

| Asset | Real risk | Control |
| --- | --- | --- |
| IAP entitlement | Fake purchase, replayed receipt | Server-side receipt validation with the store |
| Rewarded-ad grant | Client claims a reward it never watched | Server-to-server ad callback, not a client call |
| Currency / progress | Save-file edit, memory edit | Server authority when monetized; integrity check when not |
| Leaderboard score | Forged submission | Server validation or server-side simulation |
| Cosmetic-only progress | Low - the player cheats themselves | Integrity check at most; do not over-engineer |
| API keys / secrets | Extraction from the binary | Do not ship them; proxy through a server |

The right control depends on who is harmed. A single-player 2048 high score does not need what a rewarded-ad grant needs.

### Rewarded ads: the grant boundary

The most common monetization exploit in casual games - the client tells itself the ad finished.

```csharp
// Bad - the reward is granted by client code an attacker controls
void OnAdClosed() { wallet.Add(500); }

// Good - the ad network calls the server; the client refreshes state it does not compute
void OnAdClosed() => StartCoroutine(RefreshWalletFromServer());
```

Ad networks provide a **server-side verification callback** (server-to-server) for exactly this. Wire it and treat the client-side completion event as a UI hint only. Where a game genuinely has no server, say so and accept the exposure explicitly rather than pretending the client call is a control.

### IAP receipt validation

```csharp
// Bad - grants the item because the client-side purchase handler ran
public void OnPurchaseComplete(Product p) { inventory.Grant(p.definition.id); }

// Good - the receipt goes to a server that validates it with Apple/Google, then grants
public void OnPurchaseComplete(Product p) => api.ValidateAndGrant(p.receipt);
```

Local receipt validation exists and raises the bar against casual tampering, but the validating code and its keys ship inside the same binary the attacker controls. Server-side validation against the store is the only control that holds. Restore-purchases flow is a correctness concern rather than a security one.

### Save integrity without a server

For an unmonetized single-player game, the goal is detecting casual edits, not defeating a determined attacker:

```csharp
// Bad - plaintext JSON with the value an editor searches for first
{"coins": 5000}

// Good - payload plus a keyed hash; a hand-edited value fails verification
{"data": "...", "sig": "<HMAC over data with a build-embedded key>"}
```

The key ships in the build and is therefore extractable - this raises cost, it does not prevent tampering. State that limit when recommending it. Never use this pattern as the basis for granting anything another player's experience depends on.

### Secrets and the build

`PlayerPrefs` is not secure storage: it is a plist or registry entry, readable and editable without root. Nothing sensitive goes in it. There is no secure keystore equivalent shipped with Unity, so tokens that must exist on-device belong behind a platform keychain plugin, with the same caveat that a rooted device defeats it.

The durable fix is architectural: the client holds a short-lived token issued by a server, not a long-lived key that unlocks a third-party service.

### IL2CPP and reverse engineering

IL2CPP compiles managed code to native, which makes decompilation meaningfully harder than Mono's near-source IL. It does not make it impossible - strings, resources, and asset bundles remain readable, and tooling exists for IL2CPP metadata. Treat it as raising cost, and never as an argument for shipping a secret.

### Untrusted external input

```csharp
// Bad - a crafted link drives arbitrary navigation and grants
var level = int.Parse(deepLinkParams["level"]);
LoadLevel(level);

// Good - validated against what actually exists and what the player has unlocked
if (int.TryParse(deepLinkParams["level"], out var level)
    && progression.IsUnlocked(level)) LoadLevel(level);
```

Same discipline for remote config: a value driving currency, pricing, or unlocks is validated and range-clamped on arrival, because a compromised or misconfigured config becomes an economy bug.

### Third-party SDK data flow

Each ads, analytics, or attribution SDK collects device and behavioural data on its own schedule. Before shipping, know for each: what it collects, whether it needs consent first, and whether it initializes before the consent answer exists. An SDK that phones home in `Awake` before the consent dialog has been answered is a compliance defect regardless of the dialog's correctness.

Store requirements that touch code: ATT prompt before tracking on iOS, GDPR/consent gating in the EU, and children's-audience rules (COPPA and equivalents) which restrict personalized ads outright and change which SDK mode is legal. A game plausibly appealing to children needs this decided before integration, not after.

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line: nothing was reviewed, so a not-run line would misdescribe the work.

**Review mode** - source, a diff, or a symptom report was supplied. Emit one block per finding.

```
### [Severity] {file:line | symbol or type.member, when source was supplied without paths | asset path | symptom, when no source was supplied}

- Category: {ClientAuthority | IAPValidation | AdGrant | SaveTampering | Secret | SDKDataFlow | UntrustedInput | Privacy | Obfuscation}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Attack: {what an attacker does, concretely - "edits coins in the save file", "calls the reward handler without watching"}
- Impact: {who is harmed and how - revenue loss, economy break, data exposure, store rejection}
- Fix: {concrete change, naming whether it is server authority or cost-raising}
```

`Severity: {Critical | High | Medium | Low}` - Critical = revenue or entitlement granted on unverified client claims, a shipped secret, or a privacy control that is absent where legally required. High = tamperable state that affects other players or monetization. Medium = tamperable single-player state, or an SDK data-flow gap with consent present. Low = a cost-raising control worth adding with no current exploit path.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`; where it fits none, pick the closest and name the real concern in `Impact`.

`Evidence: inferred` is required whenever the source was not read. It bounds the header at High: a Critical-band defect is written High, and `Impact` names the uncapped band. It never raises a block - a Medium defect stays Medium. Among blocks sharing a band, order by what the reader must fix first: root cause before the symptoms it produces.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. In authoring mode the same line routes a design decision the sibling owns (`Deferred: save file atomic write -> unity-save-persistence`). Omit entirely when there are none.

When the project has no server and no managed backend, say so once in the report and reclassify server-authority findings as accepted exposure with the reason, rather than emitting fixes the project cannot implement.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No security findings.` |
| No source, diff, symptom, or report of any kind was supplied | `Security check not run: no source supplied.` |

A symptom-only report (a QA ticket, a verbal description) is checkable input: emit `Evidence: inferred` findings from it rather than the not-run line.

## Avoid

- Granting currency, items, or entitlements on a client-side callback
- Treating a client-side rewarded-ad completion event as proof of a view
- Local-only IAP receipt validation for anything of value
- Any secret, API key, or signing key shipped inside a build
- `PlayerPrefs` for tokens, entitlements, or anything security-relevant
- Presenting obfuscation, IL2CPP, or a hash check as protection rather than cost
- Deep-link, remote-config, or downloaded values used without validation
- An ads or analytics SDK initialized before consent is resolved
- Personalized ads in a children's-audience game
- Recommending server validation without noting the project has no server
