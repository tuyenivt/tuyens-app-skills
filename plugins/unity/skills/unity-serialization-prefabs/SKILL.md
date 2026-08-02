---
name: unity-serialization-prefabs
description: Control what Unity's serializer keeps and drops - SerializeField, SerializeReference, prefab overrides, .meta GUID stability, YAML merge conflicts.
metadata:
  category: mobile
  tags: [unity, serialization, prefabs, scriptableobject, meta-files, guid, merge-conflicts]
user-invocable: false
---

# Unity Serialization and Prefabs

> This skill owns **what the serializer persists and how prefab and asset identity is preserved**. Where code lives and what it may depend on belongs to `unity-architecture-patterns`; callback timing belongs to `unity-monobehaviour-lifecycle`; C# language mechanics belong to `csharp-unity-patterns`; save-file format and schema migration belong to `unity-save-persistence`.

## When to Use

- A field is set in the inspector and null at runtime, or resets itself
- Adding a polymorphic or interface-typed field to a serialized class
- Reviewing prefab, prefab variant, scene, or `.meta` changes in a diff
- A merge produced a broken scene, or every reference in a folder went missing

## Rules

- **A field the inspector does not show is a field Unity does not save.** If it is not visible and not `[HideInInspector]`, it is not serialized
- Private fields need `[SerializeField]`; properties are never serialized regardless of accessibility; `static`, `const`, and `readonly` are never serialized
- `Dictionary` does not serialize. Neither do interfaces, abstract types, or polymorphic subclasses - unless the field carries `[SerializeReference]`
- **A `.meta` file is the asset's identity.** Deleting or regenerating one breaks every reference to that asset across every scene and prefab. `.meta` files are committed, always, alongside their asset in the same commit
- Serialized structs and custom classes cannot be null. Unity replaces a null with a default-constructed instance on load
- A prefab override is a divergence from the prefab and must be intentional. Review every override in a diff as a change, not as noise
- Never hand-merge scene or prefab YAML without Unity's merge tool; resolve by choosing one side when the tool is unavailable

## Patterns

### What serializes

| Serializes | Does not serialize |
| --- | --- |
| `public` fields of a serializable type | properties (auto or backed) |
| `private`/`protected` with `[SerializeField]` | `static`, `const`, `readonly` |
| primitives, `string`, `enum`, Unity built-in structs | `Dictionary`, `HashSet`, multidimensional and jagged arrays |
| `UnityEngine.Object` references | interface-typed and abstract fields (without `[SerializeReference]`) |
| `List<T>` and `T[]` of a serializable `T` | `List<List<T>>`, nullable value types |
| plain classes marked `[System.Serializable]` | anything not in a serializable field type |

```csharp
// Bad - none of these persist; all are silently empty at runtime
public Dictionary<string, int> scores;
public int Level { get; set; }
private float speed;

// Good
[SerializeField] private List<ScoreEntry> scores;   // with [System.Serializable] on ScoreEntry
[SerializeField] private int level;
public int Level => level;
```

The failure is silent: no warning, no error, just a value that reverts. For a dictionary, serialize parallel lists or a `List<KeyValuePair>`-shaped serializable struct and rebuild the dictionary in `OnAfterDeserialize`.

`JsonUtility` runs this same serializer, so this table governs what reaches a JSON string too: a `Dictionary` or `int[,]` field is silently absent from the output whether or not the field carries `[SerializeField]`. That consequence is this skill's finding. It has its own further limits (no polymorphism, no top-level array) - confirm against the project's editor version. Whether the resulting file is written durably, versioned, and migrated belongs to `unity-save-persistence`.

### Null is not preserved for serializable classes and structs

```csharp
// Bad - `config` reads as null in code, but loads back as a default instance
[System.Serializable] public class Config { public int rounds; }
[SerializeField] private Config config;    // never null after a domain reload or load
```

A serialized custom class or struct field is always instantiated. Code branching on `config == null` never takes the null path after deserialization. Use an explicit `bool hasConfig` flag, or `[SerializeReference]`, which does preserve null.

### [SerializeReference]

`[SerializeReference]` stores the field by managed reference rather than by value, which buys polymorphism, interface-typed fields, null preservation, and shared references between fields.

```csharp
// Bad - stores the base slice only; subclass data is dropped on reload
[SerializeField] private List<Effect> effects;

// Good - each element keeps its concrete type
[SerializeReference] private List<Effect> effects;
```

Its pitfalls are real and cost data:

- The reference stores the concrete type by assembly, namespace, and class name. Renaming or moving the type breaks every existing asset unless the type carries `[MovedFrom]` (`UnityEngine.Scripting.APIUpdating`), which is the attribute the serializer consults for this - there is no separate managed-reference-specific attribute. Apply it to the class *before* the rename ships; any of assembly, namespace, and class name may change, and a null argument means "unchanged". A rename that reaches an asset without it surfaces as `ManagedReferenceMissingType` and the data is gone. `[FormerlySerializedAs]` does not help here: it renames a *field*, not a stored type.
- Instances are added from code or a custom editor, not by dragging in the inspector as with a `UnityEngine.Object` reference.
- Managed references have their own YAML block, which makes diffs noisier and merges harder.
- Serialization cost and file size exceed plain by-value fields; it is not the default choice.

Where the polymorphism is really "one of several authored configurations", a `ScriptableObject` reference is simpler, diffs better, and is shared rather than copied.

Note that `[SerializeReference]` list *elements* can be null, unlike by-value serialized fields - a null element is the visible symptom of `ManagedReferenceMissingType`, so guard elements individually rather than assuming the list is dense.

### Renaming a serialized field

`[FormerlySerializedAs("oldName")]` (`UnityEngine.Serialization`) is the field-rename counterpart to `[MovedFrom]`. The serializer matches stored data to fields by name, so a rename without it leaves every existing asset with an orphaned entry and the new field at its type default - silent data loss across every prefab and scene at once.

The ordering is what makes it safe, and it is easy to get wrong:

1. Add the attribute in the **same commit** as the rename. A commit that renames without it loses data on the next import.
2. Let the assets reimport, then **force a rewrite** (`EditorUtility.SetDirty` over the affected assets, then `AssetDatabase.SaveAssets()`). Until an asset is re-saved, its file still carries the old key and the attribute is load-bearing.
3. Commit the rewritten assets with their `.meta` files. Nothing here changes a GUID, so a `.meta` diff at this step means something else happened - investigate before merging.
4. Keep the attribute for at least one release after every asset is rewritten. Removing it while any unrewritten asset, branch, or stash remains drops those values.

### Prefab variants, nested prefabs, and overrides

A **nested prefab** is a prefab instance inside another prefab; it keeps its own link to its source. A **prefab variant** inherits from a base prefab and stores only its differences, so a base change propagates to every variant except where the variant overrode it.

An **override** is a per-instance divergence: a changed value, an added component, an added child, or a removed component. Overrides are the review surface, because they are invisible in the scene view and only appear in the Overrides dropdown or the YAML diff.

```
# Bad - a scene instance quietly diverges; the prefab fix never reaches this one
m_Modifications:
  - target: {fileID: ...} propertyPath: moveSpeed value: 12   # prefab says 5

# Good - the value lives on the prefab, no instance modification
```

Rules that keep this tractable: tune on the prefab and apply, do not tune on the instance and forget; put per-instance data (spawn point, level index) in overrides deliberately and nowhere else; treat an unexpected `m_Modifications` entry in a diff as a defect until justified. An added component on an instance cannot be removed by the prefab, and a component the variant added is not removable from the base - so component composition decisions belong on the base.

### .meta files and GUID stability

Every asset and folder has a sibling `.meta` holding a GUID and the importer settings. References are stored as that GUID, not as a path, which is why moving or renaming an asset inside Unity keeps references intact.

- Move and rename assets **from the Unity Project window**, not from Explorer or a shell. The editor moves the `.meta` with the asset; a shell move leaves it orphaned and Unity mints a new GUID, breaking every reference.
- Commit the `.meta` in the same commit as its asset. An asset without its `.meta` gets a fresh GUID on every machine that imports it, so references break for teammates and not for the author - the hardest version of this bug to see.
- Never add `*.meta` to `.gitignore`. Do ignore `Library/`, `Temp/`, `Logs/`, `obj/`, `Build/`, and the generated `.csproj`/`.sln`.
- A `.meta` deleted in a diff, with its asset still present, is a Critical finding.

### Scene and prefab YAML merge

Scenes and prefabs are YAML, but with `fileID` cross-references that a text merge reorders or duplicates into an unopenable file. Prevention beats resolution:

1. Set `Asset Serialization` to Force Text in Project Settings -> Editor (required for any merge or readable diff at all).
2. Register Unity's `UnityYAMLMerge` (`SmartMerge`) as the mergetool for `*.unity` and `*.prefab` in `.gitattributes` plus the git config.
3. Split large scenes so two people rarely edit the same one; move shared content into prefabs, which conflict less.
4. Without the merge tool, take one side whole (`git checkout --ours/--theirs`) and redo the other change by hand. A partially hand-merged scene often opens and is subtly wrong, which is worse than one that fails to open.

### Missing scripts and recovery

A component whose script cannot be resolved shows as "Missing (Mono Script)" and its serialized data is retained in the YAML but unreachable. Causes: the class was renamed or deleted, its `.meta` GUID changed, or the assembly it lives in failed to compile or was moved to a different `.asmdef`.

Recovery in order: fix the compile error first, because every script in a broken assembly reports missing; if the class was renamed, add `[MovedFrom]` or restore the original name and rename through the editor; if a `.meta` GUID changed, restore the old GUID from git history rather than reassigning references by hand. Reassigning the script through the inspector loses the serialized values.

### ISerializationCallbackReceiver

```csharp
// Good - dictionary rebuilt from serializable lists around the serializer
public class Table : ISerializationCallbackReceiver {
    [SerializeField] private List<string> keys = new();
    [SerializeField] private List<int> values = new();
    public Dictionary<string, int> Map = new();
    public void OnBeforeSerialize() { keys.Clear(); values.Clear(); foreach (var kv in Map) { keys.Add(kv.Key); values.Add(kv.Value); } }
    public void OnAfterDeserialize() { Map = new(); for (int i = 0; i < keys.Count; i++) Map[keys[i]] = values[i]; }
}
```

Both callbacks can run on a background thread, so neither may touch the Unity API - no `Debug.Log`, no `GameObject`, no `Resources.Load`. They also run far more often than expected in the editor (every inspector repaint, undo, and reload), so keep them cheap and free of side effects. Throwing inside either can leave the object partially deserialized.

## Output Format

Two modes, chosen by whether the request supplies code to judge or asks for code to be produced.

**Authoring mode** - the request is to write or design something. Emit the code or design, then any `Deferred:` lines. No finding blocks, no severity, no status line: nothing was reviewed, so a not-run line would misdescribe the work.

**Review mode** - source, a diff, or a symptom report was supplied. Emit one block per defect. highest severity first. Where one defect caused another (a `*.meta` ignore rule and the deleted `.meta` it dropped), emit the cause and name the consequence in its `Impact`.

```
### [Severity] {file:line | symbol or type.member, when source was supplied without paths | asset path and YAML key | symptom, when no source was supplied}

- Category: {NotSerialized | NullNotPreserved | SerializeReferenceRisk | PrefabOverride | PrefabVariant | MetaFileGuid | MergeHazard | MissingScript | SerializationCallback | RepoHygiene}
- Evidence: {source | inferred (state what was not seen)}
- Code: {one-line citation, or `not supplied` when the finding is inferred}
- Impact: {what is lost - "value reverts on reload", "every reference to this asset breaks for other clones"}
- Fix: {concrete change}
```

`RepoHygiene` covers `.gitignore` and commit-pairing defects that put asset identity at risk without being a specific asset's defect.

`Severity: {Critical | High | Medium | Low}` - Critical = a `.meta` deleted or GUID-changed for a referenced asset, a hand-merged scene or prefab, or `.meta` files gitignored. High = a field that does not serialize where the surrounding code reads or writes it as if it does, a `[SerializeReference]` type renamed without a persistence attribute, or a prefab override that contradicts the prefab's intent. Medium = a missing script with recoverable data, or `ISerializationCallbackReceiver` touching the Unity API. Low = a serialization nit with no data loss.

Severity that does not fit a listed band: assign the nearest lower band and state why in `Impact`. `Category` takes exactly one value - where a defect fits two, pick the one the `Fix` addresses and name the other in `Impact`; where it fits none, pick the closest and name the real concern in `Impact`.

`Evidence: inferred` is required whenever the file or asset was not read. It bounds the header at High: a Critical-band defect is written High, and `Impact` names the uncapped band. It never raises a block - a Medium defect stays Medium. Among blocks sharing a band, order by what the reader must fix first: root cause before the symptoms it produces. A diff summary naming a path is a source for the path and inferred for its contents; a diff hunk is a source for the lines it shows.

A defect owned by a sibling named in the ownership blockquote is not emitted as a finding. Write those after the findings, one per line, as `Deferred: {defect} -> {owning skill}`, so the workflow routes rather than drops them. Omit entirely when there are none.

In review mode, close with exactly one status line, after any `Deferred:` lines:

| Condition | Line |
| --- | --- |
| One or more findings emitted | none - the findings are the output |
| No findings, and a symptom or report was available to reason from | `No serialization findings.` |
| No source, asset, symptom, or report of any kind was supplied | `Serialization check not run: no source supplied.` |

A symptom-only report (a QA ticket, a verbal description of missing references) is checkable input: emit `Evidence: inferred` findings from it rather than the not-run line.

## Avoid

- Expecting a property, `static`, `readonly`, or bare `private` field to persist
- `Dictionary`, `HashSet`, or a nested `List<List<T>>` as a serialized field
- Null checks on a serialized custom class or struct field
- `[SerializeReference]` on a type that is renamed or moved without a persistence attribute
- Moving, renaming, or deleting assets outside the Unity Project window
- `*.meta` in `.gitignore`, or an asset committed without its `.meta`
- Hand-resolving a `*.unity` or `*.prefab` conflict in a text editor
- Instance overrides used as the place to tune values that belong on the prefab
- Unity API calls inside `OnBeforeSerialize` or `OnAfterDeserialize`
