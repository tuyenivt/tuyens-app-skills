---
name: task-unity-review-security
description: Unity mobile game security review - rewarded-ad grant integrity, IAP receipt validation, save tampering, secrets in builds, deep links, store privacy.
agent: unity-security-engineer
metadata:
  category: mobile
  tags: [unity, csharp, security, iap, rewarded-ads, tampering, secrets, il2cpp, privacy, workflow]
  type: workflow
user-invocable: true
---

> **Behavioral directive:** Load `Use skill: behavioral-principles` before executing this workflow.

# Unity Security Review

Client security review for a shipped game binary: what the client is trusted to assert, what a player can edit in a save or in memory, what secrets the build carries, what the app accepts at its edges (deep links, remote config, downloaded content), what third-party SDKs collect, and what the store requires. Findings carry a concrete attack, who is harmed, and a remediation that names whether it is server authority or cost-raising.

Security review for Unity projects.

**The client is hostile.** A shipped build runs on a device the player controls: memory is editable, files are readable, traffic is interceptable, and the binary is decompilable. Any value the client asserts can be forged. Anything of real value - an IAP entitlement, a rewarded-ad grant, a leaderboard score - is granted by a server that verified it, not by the client that claims it. The server's own authentication and API design belong to the owning service's plugin.

## When to Use

- A Unity change set adding IAP, rewarded ads, or any client-granted reward
- Player progress, currency, or entitlements moving into a save file
- A third-party SDK integrated or upgraded (ads, analytics, attribution, mediation)
- Deep links, remote config, or downloaded content becoming an input
- Secrets, API keys, or build configuration changing
- Pre-submission privacy and store-compliance pass (ATT, GDPR consent, children's audience)

**Not for:** perf review (`task-unity-review-perf`), general review (`task-unity-review`), server-side auth or API security (the owning service's plugin).

**Depth.** This workflow always runs at `deep` - security has cliff-edge consequences, and a shipped binary cannot be recalled. Every step runs on every invocation; scope by file, not by depth. `deep` is the value written to the report's `depth` field.

## Severity Rubric

| Severity | Definition |
| -------- | ---------- |
| **Critical** | Revenue or an entitlement granted on an unverified client claim (a rewarded-ad reward granted in the client's own ad-closed callback, an IAP item granted without server receipt validation); a live API key, signing key, or store credential compiled into the build or committed; a privacy control absent where it is legally required (no consent gate in the EU, personalized ads in a children's-audience game). Must fix before release; blocks merge. |
| **High** | Tamperable state that affects monetization or other players - an editable save carrying purchased entitlements or leaderboard-bound currency; a deep-link, remote-config, or server-response value driving a grant, unlock, or price with no validation; an SDK initialized before consent is resolved; **a runtime-issued credential in player-readable storage** (`PlayerPrefs`, a plain file) - per-device rather than global, which is what separates it from the Critical shipped-secret clause. Must fix before merge. |
| **Medium** | Tamperable single-player state with no monetization or multiplayer reach; an SDK data-flow gap with consent correctly present; a save integrity check absent where the value is cosmetic. Should fix in this change or the next. |
| **Low** | Cost-raising hardening with no current exploit path - obfuscation posture, an integrity check on non-valuable state, a defensive validation on an input that is already constrained. |

**Combined-finding rule.** When two findings *compose* on the same path into a worse threat than either alone, file one finding at the elevated severity:

- Client-granted reward (High alone in a cosmetic economy) + that currency spendable on IAP-equivalent goods = **Critical**: the purchase funnel is bypassable without the store
- Plaintext editable save (Medium alone) + entitlements stored in it (High) = **Critical**: remove-ads and unlocks are granted by editing a file
- Remote config driving prices (High) + no range clamp on arrival (High) = **Critical**: a compromised or misconfigured config becomes an economy break with no client floor

The enumerated rows above are the rule; where one matches, merge at the elevated severity even though each half is also exploitable alone - that is what "compose" means. File separately instead when the two defects sit on **different paths**, or when one is merely the reach of the other (an exported activity is the reach of the handler it exposes, not a second defect). The rule is not economy-specific: a leaked credential plus a destructive endpoint composes the same way. Where one finding already carries the pair's full impact, say which finding owns it rather than filing the composition twice.

## No-Server Projects

Many casual 2D games ship with no backend. When Step 2 finds no server, **say so once in the Summary** and record every server-authority finding's server half as **accepted exposure** with its reason, rather than emitting fixes the project cannot implement. In that mode the reviewable controls are tamper *detection* (a keyed hash over the save, with the limit stated: the key ships in the build and is extractable), input validation, secret absence, SDK data flow, and store compliance. Do not recommend server validation as an action item to a project with no server - name it as the boundary the exposure sits behind.

**Accepted exposure does not lower severity and does not delete the finding.** Severity states what is at risk; Control type states what can be done about it. A Critical finding whose server half is unimplementable stays Critical, and every client-side remainder it carries - deleting a grant with no precondition, restoring a removed integrity check, ordering a durable write before an acknowledgement - is still an action item and still appears in Next Steps. Only the server half produces no Next Step.

**Server presence is not binary.** Record which of these applies, because it decides what "server authority" can mean:

| Server | Meaning | Effect |
| --- | --- | --- |
| First-party | Team or company owns it | Server-authority findings are actionable; the owning team is named in the finding's Control type, and standalone runs also raise a `[Delegate]` Next Step |
| Third-party BaaS | A backend exists but you cannot add validation to it (Firebase, PlayFab-as-configured) | The console or dashboard is the trust boundary; client-side clamping is the only implementable control. Write `cost-raising only`, and name access control on that console as the upstream fix |
| None | Client-only game | The No-Server rule above |

## Excluded Surfaces

Not review surface: `Library/`, `Temp/`, `obj/`, `Build/`, generated `*.csproj` and `*.sln`, and imported third-party SDK sources under `Assets/Plugins/`. **The SDK's own source is excluded; its integration, configuration, and initialization order are not** - an ads SDK initialized in `Awake` before the consent answer exists is a finding cited at the calling code. Do not raise a finding *on* a `.meta` file; cite the asset it describes.

**Assets are review surface.** A ScriptableObject `.asset` holding an API key, a scene wiring a reward handler directly to a wallet, or a prefab carrying a debug cheat component is a legitimate finding cited at the asset path.

**Test files are review surface for credentials only.** A real token, key, or endpoint credential in a test fixture is committed and therefore disclosed - it is a finding at its shipped-secret severity even though the test assembly never reaches the player build, and the remediation is rotation, not deletion. Raise nothing else against test code.

## Invocation

| Form | Meaning |
|------|---------|
| `/task-unity-review-security` | Review the working-tree change set (unstaged + staged + untracked) |
| `/task-unity-review-security --staged` | Review the staged change set only |

When the tree is clean, `review-precondition-check` falls back to the last commit.

**Never modify the working tree.** Read via `git diff` only; uncommitted changes are the review subject, not an obstacle.

When invoked as a subagent (of `task-unity-review`), the parent supplies the precondition handle, the pre-read diff, the detected project shape, and the excluded-surface list: Step 3 is skipped, no git is re-run, and Step 10 returns findings instead of writing - the parent owns the report.

## Workflow

### Step 1 - Behavioral Principles

Use skill: `behavioral-principles`. Accept the parent's confirmation if invoked as a subagent.

### Step 2 - Stack and Project Shape

Accept the project shape from the parent when invoked as a subagent. Otherwise read `ProjectSettings/ProjectVersion.txt`; if it is absent, stop - this workflow reviews Unity projects only.

Record: engine version from `ProjectSettings/ProjectVersion.txt` (internal form), scripting backend (IL2CPP / Mono), **server presence and kind** (first-party / third-party BaaS / none), **the declared audience** (general 13+ / children under 13 / mixed), monetization surfaces present (IAP, rewarded ads, mediation), save store and format, third-party SDKs from `Packages/manifest.json` and `Assets/Plugins/`, deep-link and remote-config mechanisms, and platform targets.

**Audience is recorded here because it decides severity at Step 9, not after it.** A children's-audience declaration makes personalized ads and identifier collection illegal rather than consent-gated, so a finding graded before the audience is known is graded wrong.

**Engine floor is Unity 6.3 LTS (`6000.3.x`).** Compare numerically. Below the floor, state the mismatch and stop. If `ProjectVersion.txt` is unreadable, say so and proceed with version-independent findings only.

An absent server in a game that sells anything is itself the headline signal - record it and drive the No-Server Projects rule from it.

### Step 3 - Resolve the Change Set

Use skill: `review-precondition-check`. Forward `--staged` if passed. Surface any fail-fast verbatim and stop.

The handle supplies `mode`, `base`, `current_branch`, `reviewable`, `counts`, and `notes`. Carry all of them forward - the report is built from them.

Read the diff content once and reuse:

- `git diff <base>` for the change body
- `git diff --name-status <base>` for the file list

Restrict analysis to the handle's `reviewable` paths; binary and generated paths are excluded and never diffed. Untracked files in `reviewable` do not appear in `git diff` - read them directly and treat their whole content as added lines.

**Skip entirely** when invoked as a subagent and the parent passed the handle plus the pre-read diff.

**No-op exit.** When the change set touches only excluded surfaces, write no report: state `No reviewable change in this change set - <reason>` and stop.

A `.meta`-only change set is not automatically a no-op: read the changes, and proceed when a setting delta is present, cited at the asset. GUID or importer-version churn alone is a no-op.

### Step 4 - Read the Security Surface

Cite real `file:line` or asset path. Open:

- Every purchase, reward, and entitlement grant path - the IAP callback, the ad-closed callback, and whatever writes the wallet or inventory
- Save write and read call sites, plus the save DTO and any integrity check
- `Packages/manifest.json` and `Assets/Plugins/` for SDK presence and versions; the SDK initialization call sites and their ordering relative to consent
- Deep-link and custom-scheme handling, remote-config fetch and use sites, and any downloaded-content path
- `ProjectSettings/` for scripting backend, stripping level, and anything that reads like a credential
- Android manifest and iOS `Info.plist` overrides under `Assets/Plugins/Android` and `Assets/Plugins/iOS` - permissions, exported components, registered URL schemes, ATT usage string
- CI configuration for signing credentials and store API keys
- ScriptableObject `.asset` files added or changed, for embedded keys and endpoints

**Removed controls are findings.** When the diff drops a validation call, an integrity check, a consent gate, or a server round-trip, read the prior revision (`git log -p`) - the blame trail is authoritative. A removal is evidence of insecure design even when each line looks small, and the stated rationale ("we will add it back", "it broke the test build") is not a compensating control.

### Step 5 - Asset Triage

**Triage pass**, not a findings list. One verdict per row (`yes` / `no signal in diff`). Steps 6-9 produce the findings.

| Asset | Real risk | Owning step |
| ----- | --------- | ----------- |
| IAP entitlement | Fake purchase, replayed receipt | 6 |
| Rewarded-ad grant | Client claims a reward it never watched | 6 |
| Currency / progress | Save-file edit, memory edit | 7 |
| Leaderboard score | Forged submission | 6 |
| API keys and secrets | Extraction from the binary | 7 |
| Untrusted external input | Crafted input driving a grant or navigation - deep link, remote config, downloaded content, or a server response | 8 |
| SDK data collection | Collection before consent, or beyond the declaration | 9 |
| Store privacy posture | ATT, GDPR, children's audience | 9 |

The right control depends on who is harmed. A single-player 2048 high score does not need what a rewarded-ad grant needs - say so rather than filing both at the same tier.

### Step 6 - Client Authority: Ads, IAP, and Scores

Use skill: `unity-security-patterns` for the canonical grant-boundary patterns.

**The rewarded-ad grant is the casual-game exploit.** The client tells itself the ad finished, and the reward lands.

- [ ] **A rewarded-ad reward is granted by the network's server-side verification callback**, not by the client's ad-closed event. The client event is a UI hint that triggers a refresh of state it does not compute
- [ ] **IAP entitlements are granted on a receipt the server validated with Apple or Google.** Local receipt validation raises the bar against casual tampering, but the validating code and its keys ship inside the binary the attacker controls
- [ ] **A transaction is not acknowledged or consumed until the entitlement is durably recorded** - acknowledging first is how a player pays and receives nothing
- [ ] **Leaderboard submissions are validated or simulated server-side** where placement has value
- [ ] **No grant path is reachable from a debug, cheat, or test component left in a shipped scene or prefab**

```csharp
// Bad - the reward is granted by client code the attacker controls
void OnAdClosed() { wallet.Add(500); }

// Good - the network calls the server; the client refreshes state it does not compute
void OnAdClosed() => StartCoroutine(RefreshWalletFromServer());
```

**IAP and ads SDK APIs move faster than the engine.** Name the flow and the integrity boundary; check the package's current surface rather than asserting a call signature.

### Step 7 - Save Tampering and Secrets in the Build

Use skill: `unity-save-persistence` when the finding touches save format or storage path.

- [ ] **Monetized or multiplayer-visible state is server-authoritative**, not save-resident. A save is a file on a device the player owns
- [ ] **Where a server is absent, an integrity check is a detection control with its limit stated** - a keyed hash over the payload catches a hand-edited value, and the key ships in the build and is extractable. Never present it as protection
- [ ] **`PlayerPrefs` holds nothing security-relevant.** It is a registry key, a plist, or shared preferences - editable without root, and never a store for tokens, entitlements, or currency
- [ ] **No secret in code, in a ScriptableObject, in `Resources`, in a bundled asset, or in `PlayerPrefs`.** Any of these is a *published* secret: the fix is to move the capability behind a server and rotate the key, never to hide it better
- [ ] **IL2CPP is cost, not protection.** It makes decompilation meaningfully harder than Mono's near-source IL; strings, resources, and asset bundles stay readable and IL2CPP metadata tooling exists. A finding whose proposed fix is "we use IL2CPP" is not fixed
- [ ] **Tokens that must exist on-device sit behind a platform keychain plugin**, with the rooted-device caveat stated; the durable fix is a short-lived server-issued token, not a long-lived embedded key
- [ ] **Signing credentials, keystores, and store API keys live in CI secrets** - never in the repo, never in `ProjectSettings`

### Step 8 - Untrusted External Input

Everything crossing into the process from outside is attacker-controllable: deep-link and custom-scheme parameters, remote-config values, downloaded content and Addressables catalogs, notification payloads, and clipboard contents.

- [ ] **Deep-link parameters are parsed defensively and validated against what exists and what the player has unlocked** before they drive navigation, a grant, or a purchase
- [ ] **A custom URL scheme is not an authentication channel** - any installed app can register the same scheme. Only verified app links and universal links carry an ownership claim
- [ ] **Remote-config values driving currency, pricing, unlocks, or difficulty are validated and range-clamped on arrival.** A compromised or misconfigured config otherwise becomes an economy bug with no floor
- [ ] **Downloaded content and remote catalogs are treated as untrusted** - served over TLS, size-bounded, and never used to select code paths by name
- [ ] **The transport is not a trust boundary.** Every endpoint is TLS and no cleartext exception is configured, but a user-installed CA on a device the player owns makes any response attacker-authored. Certificate pinning raises the cost of that and is `cost-raising only` - it never substitutes for validating the response
- [ ] **Server responses are parsed defensively and validated field by field before they touch live state** - including a first-party server's. Ownership of the endpoint is not a trust boundary: the transport terminates on a device the player controls, so a proxy with a user-installed CA makes the response attacker-authored. A response driving progress, currency, or unlocks with no validation is High, the same as an unvalidated deep link
- [ ] **Conflict resolution does not key on the device clock.** Last-write-wins on a client-supplied timestamp is a write primitive: the player sets the date forward and their save wins permanently. The server assigns the timestamp, or the client sends a monotonic counter the server compares
- [ ] **Android components are exported only when they must be**; a new permission arriving through an SDK's manifest is justified or removed

```csharp
// Bad - a crafted link drives arbitrary navigation and grants
var level = int.Parse(deepLinkParams["level"]);
LoadLevel(level);

// Good - validated against what exists and what the player unlocked
if (int.TryParse(deepLinkParams["level"], out var level)
    && progression.IsUnlocked(level)) LoadLevel(level);
```

### Step 9 - SDK Data Flow and Store Compliance

- [ ] **Each ads, analytics, attribution, and mediation SDK has a known collection profile** - what it collects, whether it needs consent first, and whether it initializes before the consent answer exists. An SDK that phones home in `Awake` before the dialog is answered is a compliance defect regardless of the dialog's correctness
- [ ] **Consent gates SDK initialization, not just reporting.** Order is startup -> consent resolved -> SDK init -> events
- [ ] **iOS ATT is requested before tracking that reaches the advertising identifier**, with the usage string present
- [ ] **GDPR consent is gated in the EU**, and the decision is revocable and honored
- [ ] **A children's-audience game restricts personalized ads outright** and changes which SDK mode is legal - decided before integration, not after. The ads posture and the analytics posture are answered by the same audience declaration
- [ ] **The store data-collection declaration matches what the app actually sends.** A field added in code without updating the declaration is a compliance defect, not only a privacy one
- [ ] **No PII in what leaves the device** - names, emails, precise location, raw device identifiers, or user-authored text. Player identifiers are opaque and app-scoped

This step owns whether the collection is legal and whether the SDK respects the gate.

### Step 10 - Write Report

Standalone only. A subagent run returns the `## Asset Triage` table, the `## Findings` sections, and `## Reviewed, Not Filed` - nothing else. No frontmatter, no Summary block, no Recommendations, no Next Steps: the parent owns those and cannot merge two of them. Where the no-server or third-party-BaaS boundary applies, state it once inside the affected findings' Control type rather than in a Summary the subagent does not write.

Write the assembled report to `review-security-<branch>.md` in the current working directory, overwriting the file if it already exists.

Derive `<branch>` from the handle's `current_branch`, sanitized for a filename: replace `/` and any character outside `[A-Za-z0-9_-]` with `-`, collapse consecutive `-` into one, strip leading and trailing `-`.

The file is this YAML frontmatter followed by the report body (raw Markdown, unfenced):

```yaml
---
branch: <branch>
scope_mode: working-tree | staged-only | last-commit
files: <N>
scope: core-only | +perf | +sec | full
depth: standard | deep
generated_at: <ISO 8601 UTC timestamp>
---
```

Field sources: `branch` = the handle's `current_branch` (unsanitized), `scope_mode` = the handle's `mode`, `files` = the handle's `counts.reviewable`, `scope` = `+sec`, `depth` = `deep` (this workflow always runs full depth), `generated_at` = the current UTC time in ISO 8601.

After writing, print exactly one confirmation line:

```
Report written to review-security-<branch>.md (<N> files, scope: <scope>)
```

## Output Format

The fence below delimits the template for display only - it is not part of the report. Emit the report body as raw Markdown so headings, tables, and lists render; never wrap the whole report in a code fence.

```markdown
## Unity Security Review Summary

**Stack Detected:** Unity <marketing name> (<internal version>) / IL2CPP | Mono
**Server:** first-party at <endpoint owner> | third-party BaaS (<vendor>) - no server-side validation available to this team | **none - client-only game**
**Audience:** general (13+) | **children (under 13)** | mixed
**Monetization:** IAP | rewarded ads | mediation | present but untouched in diff | none
**Save Store:** <path and format> | integrity check at file:line | none
**SDKs in Diff:** <list> | none
**App Edges:** deep links | remote config | downloaded content | none in diff
**Platform Targets:** <list>
**Overall Posture:** Clean | Issues Found - [Critical/High/Medium/Low count]

[2-3 sentence assessment naming the client-specific risks: a reward granted in the client's ad callback, an entitlement in an editable save, an API key in a ScriptableObject, an SDK collecting before consent.]

[When **Server: none**, one sentence here: "This project has no server, so no client-side control can make a grant authoritative. Server-authority findings below are recorded as accepted exposure, not as action items."]

Summary field whose observed state matches no listed value: write the closest value followed by ` - <what was actually observed>` rather than forcing a wrong one or omitting the line. Every field is always present.

## Asset Triage

| Asset                      | Verdict                 |
| -------------------------- | ----------------------- |
| IAP entitlement            | yes / no signal in diff |
| Rewarded-ad grant          | ...                     |
| Currency / progress        | ...                     |
| Leaderboard score          | ...                     |
| API keys and secrets       | ...                     |
| Untrusted external input   | ...                     |
| SDK data collection        | ...                     |
| Store privacy posture      | ...                     |

## Findings

### Critical

- **Location:** [file:line or asset path; a finding whose defects compose across several files lists each, marking the one where the fix starts]
- **Issue:** [the defect in Unity terms: "`OnAdClosed` adds 500 coins directly, so any player who calls the handler or edits the callback grants themselves the reward without watching"]
- **Attack:** [what an attacker does, concretely: "edits `coins` in `save.json` and relaunches", "invokes the reward handler through a modified build", "reads the key out of the ScriptableObject in the asset bundle"]
- **Impact:** [who is harmed and how - revenue loss, economy break, other players' rankings, data exposure, store rejection]
- **Regression of:** [the commit that added the control this diff removes, and what it was added to fix - written only when the diff removes a control; omit the line otherwise]
- **Severity rationale:** [tier] per rubric - [which clause applies]
- **Control type:** server authority | cost-raising only | accepted exposure (`<reason>` - no server, or a backend this team cannot add validation to) - where a finding's fixes span two, write both as `<primary> + <secondary>` and say which fix is which
- **Fix:** [concrete C#, asset, or configuration change with code]

### High / Medium / Low

[Same structure]

_Omit severity sections with no findings. If all are omitted: "No security issues found."_

## Reviewed, Not Filed

One line per reviewable path that produced no finding, with why. A near-clean change set is the common case, and without this section the report cannot be distinguished from one where the file was never opened. Omit only when every reviewable path produced a finding.

## Recommendations

[Prioritized hardening not tied to a single finding]

## Next Steps

Each tagged `[Implement]` or `[Delegate]`. Order: Must > Recommend.
Severity maps to intent: Critical / High -> [Must]; Medium / Low -> [Recommend].
Every finding carries exactly one label: `[Must]` or `[Recommend]`. No other label is written.
A finding also produces a `[Delegate]` entry whenever the control that closes it sits outside this codebase - `server authority` naming what the backend must enforce, or a third-party BaaS naming the console access control that is the upstream fix. The client fix alone does not close either. Where Control type is `accepted exposure`, the server half produces no Next Step, and every client-side remainder the finding names still produces one - a finding never leaves Next Steps empty-handed because its server half is unimplementable.

1. **[Implement]** [Must] file:line - [one-line action]
2. **[Delegate]** [Recommend] [scope: server contract] - [one-line action]

_Omit if no issues found._
```

## Rules

- Validate at the app's edges: deep-link parameters, remote-config values, downloaded content, notification payloads, and every server response
- A secret that reached a shipped build is rotated, not hidden - IL2CPP, stripping, and obfuscation are cost, never remediation for an embedded credential
- Never widen the app's edge (a new scheme, exported component, or config key driving a grant) without recording why it is safe
- Authority over anything of value is the server's; a client-side check is a UX affordance and is never cited as the control
- State which control type each fix is - server authority, cost-raising, or accepted exposure - so the reader is never told a hash is protection

## Self-Check

**Verifiable from the diff (must check):**

- [ ] Step 1: `behavioral-principles` loaded (or accepted from parent)
- [ ] Step 2: stack confirmed Unity; engine version checked numerically against the `6000.3.x` floor; scripting backend, **server presence and kind**, **declared audience**, monetization surfaces, save store, SDKs, edges, and platform targets recorded before any finding is graded
- [ ] Step 3: `review-precondition-check` ran (or parent-supplied handle and diff reused); `git diff <base>` read once and reused; untracked files read directly as added lines; analysis restricted to the handle's `reviewable` paths; no-op exit taken on an excluded-only change set
- [ ] Step 4: security surface read directly (grant paths, save call sites, SDK init order, edges, `ProjectSettings/`, platform overrides, CI, changed `.asset` files); prior revision consulted wherever a control was removed
- [ ] Step 5: asset triage produced one verdict per row; triage verdicts not duplicated as standalone findings
- [ ] Step 6: `unity-security-patterns` consulted; ad-grant boundary, IAP receipt validation, transaction acknowledgement ordering, leaderboard authority, and shipped debug grant paths checked
- [ ] Step 7: `unity-save-persistence` consulted where relevant; save authority, integrity-check limits stated, `PlayerPrefs` misuse, no shipped secret, IL2CPP framed as cost, CI credential placement
- [ ] Step 8: every edge in the diff triaged as untrusted input - deep links, remote config, downloaded content, exported components and permissions
- [ ] Step 9: SDK collection profile, consent gating init not just reporting, ATT, GDPR, children's-audience posture, store declaration match, and PII checked
- [ ] Step 10: standalone: report written to `review-security-<branch>.md` with the sanitized branch name and complete frontmatter (`branch`, `scope_mode`, `files`, `scope`, `depth`, `generated_at`); confirmation line printed; subagent: findings returned to parent, no file written
- [ ] Excluded surfaces raised no findings; SDK *integration and init order* still reviewed; `.meta` defects cited at the asset; scenes, prefabs, and `.asset` files treated as reviewable
- [ ] Severity rubric applied consistently; combined-finding rule applied where two defects compose on one path
- [ ] Every finding carries a concrete attack, an impact, and a Control type; a removed control also carries `Regression of:`
- [ ] No-server rule honored: stated once in the Summary, server-authority findings reclassified as accepted exposure, no unimplementable action items
- [ ] Next Steps tagged `[Implement]` / `[Delegate]`, ordered Must > Recommend (omitted only when no issues)

**Requires repo or device access:**

- [ ] Merged Android manifest inspected for permissions and components contributed by SDK plugins
- [ ] Release build configuration confirmed (scripting backend, stripping level, absence of debug grant paths) rather than inferred from the diff

## Avoid

- State-changing git from this workflow (checkout/merge/pull/rebase/fetch/stash) - uncommitted work is the review subject and must not be disturbed
- Writing a report when invoked as a subagent - the parent owns it
- Writing a report at all when the change set touches only excluded surfaces
- Recommending server validation to a project with no server, instead of recording accepted exposure
- Reporting without a concrete attack ("input not validated" vs "a crafted link sets `level=99` and skips the paid unlock")
- Skipping an asset-triage row - state `no signal in diff` rather than leaving it blank
- Raising findings against `Library/`, `Temp/`, `obj/`, `Build/`, generated `*.csproj` / `*.sln`, or third-party SDK sources - while still reviewing their integration and init order
- Raising a finding on a `.meta` file rather than the asset it describes
- Granting currency, items, or entitlements on a client-side callback
- Treating a client-side rewarded-ad completion event as proof of a view
- Local-only IAP receipt validation for anything of value
- Any secret, API key, keystore, or signing key shipped inside a build or committed to the repo
- `PlayerPrefs` for tokens, entitlements, or currency
- Presenting IL2CPP, stripping, obfuscation, or a hash check as protection rather than cost
- Deep-link, remote-config, or downloaded values used without validation and range clamping
- An ads or analytics SDK initialized before consent is resolved
- Personalized ads in a children's-audience game
- Filing a tamperable cosmetic single-player value at the same tier as a monetized one
- Reviewing the server's authentication or API contract here - it belongs to the owning service
