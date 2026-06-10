## Daily Iceberg-CPP Module Analysis

You are performing the daily code analysis for the iceberg-cpp project. This task analyzes ONE module per day, rotating through all 14 modules.

### Step 1: Determine Today's Module

Read the state file at `/Users/wz/Code/iceberg-cpp-issues/daily_analysis_state.json`.

Compute today's index: `(last_analyzed_index + 1) % 14`. Get the module at that index. Read the `cpp_glob`, `java_base`, and `name` fields.

### Step 2: Read C++ Source Files

Use Glob to find all files matching `cpp_glob` under `/Users/wz/Code/iceberg-cpp/`. Then read every `.cc` and `.h` file in the module. Read them ALL — do not skip any.

### Step 3: Read Corresponding Java Reference Files

For each C++ source file you read, derive the corresponding Java file:
- Take the C++ filename relative to `cpp_path` (e.g., `src/iceberg/avro/avro_reader.cc` → `avro_reader.cc`)
- Convert to Java naming: remove `.cc`/`.h`, convert snake_case to PascalCase (e.g., `avro_reader` → `AvroReader.java`)
- Look under `java_base` (absolute path: `/Users/wz/Code/iceberg-cpp/` + `java_base`)

Read up to 3 most relevant Java files for comparison. If no Java counterpart exists, note that.

**IMPORTANT GUIDANCE FOR FINDING JAVA FILES:** The Java reference is a LARGE repository. Don't try to list all files recursively — it will time out. Instead:
- Derive the expected file name from the C++ file (snake_case → PascalCase + .java)
- Search with `find ... -name "ExpectedName.java"` for specific files
- If the exact name doesn't exist, search for the key class/interface name in the Java source tree
- Focus on reading 2-3 most relevant Java files, not the entire Java module
- If a Java file search takes more than a few seconds, stop and note "Java reference not found" — don't spend excessive time searching

### Step 4: Analyze for Issues

For each C++ file, analyze for these categories:

**Category A: Logic Errors (bugs)**
- Off-by-one errors in loops
- Null pointer / dangling reference risks (raw pointers, unchecked optionals)
- Incorrect conditionals (wrong operators, missing branches)
- Resource leaks (missing cleanup in error paths)
- Integer overflow/underflow in size calculations
- Thread safety issues in shared state
- Incorrect move/copy semantics
- Uninitialized variables
- Buffer overflows in string/array operations

**Category B: Inconsistencies with Java Implementation**
- Different default values than Java
- Missing validation that Java performs
- Different error handling behavior (e.g., Java throws, C++ silently returns)
- Different algorithm or data structure choices that produce different results
- Missing edge case handling that Java covers
- Different serialization/deserialization logic

**Category C: Incomplete or Unfinished Features**
- TODO/FIXME/HACK comments
- Functions that are declared but return "not implemented" or empty results
- Stub implementations that lack real logic
- Missing overloads or variants that Java provides
- Partial implementations of a spec (e.g., only handling some types)

### Step 5: Write Report

Write the report to `/Users/wz/Code/iceberg-cpp-issues/daily_reports/YYYY-MM-DD_<module_name>.md` (replace YYYY-MM-DD with today's date, module_name with the sanitized module name — lowercase, underscores for spaces/slashes).

Report format:
```markdown
# Daily Analysis Report: {module_name}
**Date:** YYYY-MM-DD
**Files Analyzed:** N C++ files, M Java reference files

## Summary
- Logic Errors: X found
- Java Inconsistencies: Y found
- Incomplete Features: Z found

## Findings

### Logic Errors
#### {file_name}:{line} — {short_title}
- **Severity:** high | medium | low
- **Description:** {what the bug is}
- **Fix:** {suggested fix}

### Java Inconsistencies
#### {cpp_file}:{line} vs {java_file}:{line} — {short_title}
- **Severity:** high | medium | low
- **C++ behavior:** {what C++ does}
- **Java behavior:** {what Java does}
- **Impact:** {why this matters}
- **Fix:** {suggested fix}

### Incomplete Features
#### {file_name}:{line} — {short_title}
- **Description:** {what is missing or incomplete}
- **Java reference:** {what Java provides}
- **Priority:** high | medium | low

## Recommendations
1. {actionable recommendation}
2. ...
```

### Step 6: Update State

Update the state file: set `last_analyzed_index` to today's index, and add a `last_analyzed_at` field with today's date (ISO format).

---

**IMPORTANT:** Be thorough. Read EVERY file in the module. Compare with Java reference files. Don't just skim — look at the actual logic, data flow, error handling, and edge cases. Flag anything suspicious even if you're not 100% sure. Prefer flagging uncertain issues over missing real problems.
