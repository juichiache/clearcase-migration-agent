# Skill: `build_record_ingester`

**Phase:** Planning Agent
**Mode:** Read-only

---

## Purpose

Reads scripted YAML build records produced by the Python extraction script and produces a normalized, queryable in-memory representation. This is the primary data source for the Planning Agent. Without build records, the agent cannot make package assignments.

---

## Inputs

| Input | Type | Required | Description |
|---|---|---|---|
| `viewPath` | string | Yes | Absolute path to mounted ClearCase view |
| `release` | string | Yes | Release identifier (e.g., `R2024.1`) |
| `product` | string | No | Limit to a specific product; default: all products with YAML records |
| `buildRecordDir` | string | No | Override directory to search for YAML files; default: `{viewPath}/build_records/{release}/` |

---

## Outputs

Produces an in-memory `BuildRecordSet` object (not persisted to disk; consumed by subsequent planning skills):

```typescript
interface BuildRecordSet {
  release: string;
  products: ProductRecord[];
  ingestionWarnings: IngestionWarning[];
}

interface ProductRecord {
  product: string;
  sourceFiles: SourceFileRecord[];
  libraryFileSets: LibraryFileSet[];
}

interface SourceFileRecord {
  filePath: string;           // Relative to ClearCase view root
  macroCombinations: MacroCombo[];
}

interface MacroCombo {
  macros: Record<string, string>;   // { FEATURE_X: "1", PLATFORM: "win32" }
  includedHeaders: string[];        // Paths relative to view root
}

interface LibraryFileSet {
  libraryName: string;
  sourceFiles: string[];    // Paths relative to view root
}

interface IngestionWarning {
  filePath: string;
  message: string;
  severity: "info" | "warning" | "error";
}
```

---

## Behavior

### 1. Discover Build Record Files

Search `buildRecordDir` for YAML files matching the pattern `*.build-record.yaml` (or whatever pattern the extraction script produces — see `open-questions.md §2.1`).

> [OPEN] §2.1 — The exact YAML schema is unknown. The field names and structure below are assumed. Update when the real schema is provided.

### 2. Parse Each YAML File

For each YAML file, parse the following assumed structure:

```yaml
product: libcore
release: R2024.1
source_files:
  - path: src/core/allocator.cpp
    macro_combinations:
      - macros:
          FEATURE_THREADS: "1"
          PLATFORM: "linux"
        includes:
          - include/core/allocator.h
          - include/platform/linux/thread.h
      - macros:
          FEATURE_THREADS: "0"
          PLATFORM: "win32"
        includes:
          - include/core/allocator.h
          - include/platform/win32/compat.h
library_file_sets:
  - name: libcore
    source_files:
      - src/core/allocator.cpp
      - src/core/memory.cpp
```

### 3. Normalize Paths

All file paths are normalized to be relative to the ClearCase view root. Absolute paths from the YAML are converted using `viewPath` as the base.

### 4. Validate Required Fields

For each source file record, validate:
- `filePath` is non-empty and the file exists in the view
- At least one `macroCombination` is present
- Each macro combination has at least zero (empty is valid) `includedHeaders`

### 5. Emit Warnings

For each validation failure, emit an `IngestionWarning` with severity:
- `error` — file path does not exist in view (will cause downstream issues)
- `warning` — unexpected field names (possible schema version mismatch)
- `info` — no macro combinations for a file (will be treated as a single combination with empty macro set)

### 6. Deduplicate

If the same file appears in multiple YAML files (e.g., a shared file appearing in both `libcore` and `libplatform`), retain all records and flag with an `info` warning. The `existing_package_loader` will handle deduplication via the forkable file policy.

### 7. Log Invocation

Append to `RunLogEvent.jsonl`:
```json
{
  "timestamp": "...",
  "runId": "...",
  "skill": "build_record_ingester",
  "phase": "complete",
  "inputs": { "release": "R2024.1", "fileCount": 42 },
  "outputs": { "sourceFileCount": 1847, "warningCount": 3 },
  "durationMs": 4200
}
```

---

## Error Handling

| Error Condition | Behavior |
|---|---|
| `buildRecordDir` does not exist | Halt with error: "No build records found at {path}. Confirm the view is mounted and the extraction script has been run." |
| Zero YAML files found | Halt with error and list products that have no records |
| YAML parse error | Skip the file, emit `error`-severity warning, continue |
| All files produce `error` warnings | Halt after processing all files; report summary |
| `viewPath` is not accessible | Halt immediately with error |

---

## Worked Example

**Input:** ClearCase view at `/vobs`, release `R2024.1`, product `libcore`

**YAML file found:** `/vobs/build_records/R2024.1/libcore.build-record.yaml`

```yaml
product: libcore
release: R2024.1
source_files:
  - path: src/core/allocator.cpp
    macro_combinations:
      - macros: { THREADS: "1" }
        includes: [include/core/allocator.h, include/thread.h]
      - macros: { THREADS: "0" }
        includes: [include/core/allocator.h]
library_file_sets:
  - name: libcore
    source_files: [src/core/allocator.cpp, src/core/memory.cpp]
```

**Output (in-memory `BuildRecordSet`):**
```json
{
  "release": "R2024.1",
  "products": [{
    "product": "libcore",
    "sourceFiles": [{
      "filePath": "src/core/allocator.cpp",
      "macroCombinations": [
        { "macros": {"THREADS": "1"}, "includedHeaders": ["include/core/allocator.h", "include/thread.h"] },
        { "macros": {"THREADS": "0"}, "includedHeaders": ["include/core/allocator.h"] }
      ]
    }],
    "libraryFileSets": [{
      "libraryName": "libcore",
      "sourceFiles": ["src/core/allocator.cpp", "src/core/memory.cpp"]
    }]
  }],
  "ingestionWarnings": []
}
```

**`RunLogEvent.jsonl` entry:**
```json
{"timestamp":"2024-12-15T14:30:22Z","runId":"plan-20241215T143022Z","skill":"build_record_ingester","phase":"complete","inputs":{"release":"R2024.1","product":"libcore","fileCount":1},"outputs":{"sourceFileCount":1,"warningCount":0},"durationMs":312}
```
