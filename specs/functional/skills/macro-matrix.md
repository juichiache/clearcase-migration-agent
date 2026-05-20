# Skill: `macro_matrix`

**Phase:** Planning Agent
**Mode:** Read-only

---

## Purpose

Analyzes all per-file macro combinations across a package and produces the minimal set of Conan options that correctly covers all observed combinations. This is the primary time-saving feature of the planning phase: deriving Conan options from observed build evidence instead of manually analyzing macros.

**Critical rule:** The output of this skill is the ONLY authorized source of Conan options. No option may appear in a generated `conanfile.py` unless it is listed in this skill's output for that package.

---

## Inputs

| Input | Type | Required | Description |
|---|---|---|---|
| `buildRecordSet` | `BuildRecordSet` | Yes | Output of `build_record_ingester` |
| `packageAssignment` | Draft `PackageAssignment` | Yes | Current (partial) package-to-file mapping; used to group files by package |
| `minimizationStrategy` | string | No | How to minimize options: `"greedy"` (default) or `"exact"` |

---

## Outputs

Produces `MacroMatrix.json` (written to run directory after schema validation):

```typescript
interface MacroMatrix {
  runId: string;
  packages: PackageMacroMatrix[];
}

interface PackageMacroMatrix {
  packageName: string;
  combinations: ObservedCombination[];
  conanOptions: ConanOption[];
  coverageNote?: string;  // e.g., "3 combinations collapsed to 2 options via minimization"
}

interface ObservedCombination {
  id: string;              // e.g., "combo-0"
  macros: Record<string, string>;   // { FEATURE_THREADS: "1", PLATFORM: "linux" }
  sourceFiles: string[];   // Which files in the package use this combination
  mappedToOption: string;  // conanOptionName this combo maps to
}

interface ConanOption {
  name: string;            // e.g., "threads"
  type: "bool" | "enum";
  values: string[];        // For bool: ["True", "False"]; for enum: observed values
  defaultValue: string;
  macroMapping: Record<string, string>;  // Which macros this option controls
  description: string;
}
```

---

## Behavior

### 1. Collect Per-Package Macro Combinations

For each package P in the draft `PackageAssignment`:
  - For each file F assigned to P:
    - For each macro combination C in F's build record:
      - Add C to the set of observed combinations for P

This produces a set of observed `{macroName → value}` maps per package.

### 2. Deduplicate Combinations

Two combinations are identical if they have the same set of macro name/value pairs. Deduplicate and merge the `sourceFiles` lists.

### 3. Identify Independent Macro Axes

Group macros by independence: if macro A's value is always the same regardless of macro B's value (and vice versa), they are on independent axes and can become separate Conan options.

**Example:**
- Combo 1: `{THREADS: "1", PLATFORM: "linux"}`
- Combo 2: `{THREADS: "0", PLATFORM: "linux"}`
- Combo 3: `{THREADS: "1", PLATFORM: "win32"}`
- Combo 4: `{THREADS: "0", PLATFORM: "win32"}`

`THREADS` and `PLATFORM` are independent → two separate options: `threads` (bool) and `platform` (enum: linux, win32).

### 4. Minimize Options

Apply the minimization strategy to reduce the number of distinct option values:

**Greedy (default):** Identify macros that always appear together and collapse them into a single option.

**Exact:** Enumerate all possible combinations and find the minimum cover. More accurate but slower for large macro sets.

### 5. Derive Conan Option Definitions

For each independent macro axis, produce a `ConanOption`:
- `name`: derived from the macro name, lowercased and snake_cased (e.g., `FEATURE_THREADS` → `feature_threads`)
- `type`: `"bool"` if values are `{"0", "1"}` or `{"ON", "OFF"}` or `{"true", "false"}`; otherwise `"enum"`
- `values`: for bool: `["True", "False"]`; for enum: all observed values
- `defaultValue`: the most common value across all observed combinations
- `macroMapping`: the macro name(s) this option controls

### 6. Write Output

Validate against `MacroMatrix.schema.json`. Write to `artifacts/runs/{runId}/MacroMatrix.json`.

---

## Error Handling

| Error Condition | Behavior |
|---|---|
| Package has zero files with build records | Emit warning: "Package {name} has no build records; Conan options cannot be derived." |
| All files in package have one macro combo | Valid: one or zero options needed; emit a `coverageNote` |
| Macro with more than 10 distinct values | Emit warning: "Macro {name} has {N} distinct values; consider whether all values are intentional." |
| Minimization fails to produce a valid cover | Fall back to enumerating all combos as distinct options; emit warning |

---

## Worked Example

**Package:** `libcore`
**Files and their macro combinations:**

| File | Macros |
|---|---|
| `src/core/allocator.cpp` | `{THREADS: "1", DEBUG: "0"}` |
| `src/core/allocator.cpp` | `{THREADS: "0", DEBUG: "0"}` |
| `src/core/allocator.cpp` | `{THREADS: "1", DEBUG: "1"}` |
| `src/core/memory.cpp` | `{THREADS: "1", DEBUG: "0"}` |
| `src/core/memory.cpp` | `{THREADS: "0", DEBUG: "0"}` |

**Step 3 — Independence check:** `THREADS` and `DEBUG` appear in independent combinations → two axes.

**Step 5 — Options derived:**
- `threads`: bool (`True`/`False`), default `True`, maps to `THREADS`
- `debug`: bool (`True`/`False`), default `False`, maps to `DEBUG`

**`MacroMatrix.json` output (libcore entry):**
```json
{
  "packageName": "libcore",
  "combinations": [
    { "id": "combo-0", "macros": {"THREADS": "1", "DEBUG": "0"}, "sourceFiles": ["src/core/allocator.cpp", "src/core/memory.cpp"], "mappedToOption": "threads=True,debug=False" },
    { "id": "combo-1", "macros": {"THREADS": "0", "DEBUG": "0"}, "sourceFiles": ["src/core/allocator.cpp", "src/core/memory.cpp"], "mappedToOption": "threads=False,debug=False" },
    { "id": "combo-2", "macros": {"THREADS": "1", "DEBUG": "1"}, "sourceFiles": ["src/core/allocator.cpp"], "mappedToOption": "threads=True,debug=True" }
  ],
  "conanOptions": [
    {
      "name": "threads",
      "type": "bool",
      "values": ["True", "False"],
      "defaultValue": "True",
      "macroMapping": { "THREADS": "1 when True, 0 when False" },
      "description": "Enable multi-threading support (THREADS macro)"
    },
    {
      "name": "debug",
      "type": "bool",
      "values": ["True", "False"],
      "defaultValue": "False",
      "macroMapping": { "DEBUG": "1 when True, 0 when False" },
      "description": "Enable debug mode (DEBUG macro)"
    }
  ],
  "coverageNote": "3 distinct combinations covered by 2 independent options (2^2 = 4 combinations; 1 not observed: threads=False,debug=True)"
}
```

**Generated `conanfile.py` options block (derived from this output):**
```python
options = {
    "threads": [True, False],
    "debug": [True, False],
}
default_options = {
    "threads": True,
    "debug": False,
}
```
