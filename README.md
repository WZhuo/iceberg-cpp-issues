# iceberg-cpp-issues

Automated daily code analysis pipeline for [iceberg-cpp](https://github.com/apache/iceberg-cpp) — a C++ implementation of Apache Iceberg. Each day, one of 14 modules is compared against the Java reference implementation to catch bugs, inconsistencies, and incomplete features.

## How It Works

```
         10:30 AM daily (launchd)
                │
                ▼
    run_daily_analysis.py
                │
                ├── Reads analysis_prompt.md
                ├── Calls: claude -p --model opus --permission-mode bypassPermissions
                │       ├── Determines today's module via daily_analysis_state.json
                │       ├── Reads all C++ source files in that module
                │       ├── Finds and reads corresponding Java reference files
                │       ├── Analyzes for: logic errors, Java inconsistencies, incomplete features
                │       └── Writes report to daily_reports/YYYY-MM-DD_<module>.md
                ├── Commits & pushes report to origin/main
                └── Logs everything to logs/analysis_<timestamp>.log
```

## Findings Categories

| Category | What It Looks For |
|----------|-------------------|
| **Logic Errors** | Off-by-one, null/dangling pointers, resource leaks, thread safety, uninitialized variables, integer overflow |
| **Java Inconsistencies** | Different defaults, missing validation, different algorithms, different error handling |
| **Incomplete Features** | TODOs, stubs, missing overloads/variants that Java provides |

## Module Rotation (14 modules, ~2 weeks per cycle)

1. **Core/Schema** — Core types, schema, table metadata, snapshots
2. **Arrow** — Apache Arrow integration, S3 I/O
3. **Avro** — Avro reader/writer, schema utilities
4. **Catalog** — Hive, REST, SQL, in-memory catalogs
5. **Data** — Data writers, scan task readers, delete filters
6. **Deletes** — Position delete files, bitmap indexes
7. **Expression** — Expression evaluators, binders, aggregates
8. **Manifest** — Manifest reader/writer, list merging
9. **Metrics** — Scan/commit reporting, counters, timers
10. **Parquet** — Parquet reader/writer, schema utilities
11. **Puffin** — Puffin statistics files
12. **Row** — Arrow array wrappers, partition values
13. **Update** — Snapshot management, table updates
14. **Util** — Base64, decimal, hashing, formatting

## Project Structure

```
iceberg-cpp-issues/
├── README.md                       ← This file
├── analysis_prompt.md              ← Prompt template fed to claude -p
├── run_daily_analysis.py           ← Python orchestration script (launched by launchd)
├── daily_analysis_state.json       ← Tracks rotation index & module definitions
├── daily_reports/                  ← Generated reports (auto-committed)
│   └── YYYY-MM-DD_<module>.md
├── logs/                           ← Analysis run logs (gitignored)
├── memory/                         ← Claude Code persistent memory
│   └── daily-module-analysis.md
├── MEMORY.md                       ← Claude memory index
└── .gitignore
```

## Prerequisites

- macOS with `launchd`
- [Claude Code](https://claude.ai/code) CLI (`claude`)
- Node.js (managed via nvm, currently v25.8.2)
- Python 3.9+
- Git

## Scheduling

The analysis runs via a macOS launchd agent:

```bash
# Check status
launchctl list | grep iceberg

# Stop
launchctl unload ~/Library/LaunchAgents/com.iceberg-cpp.daily-analysis.plist

# Start
launchctl load ~/Library/LaunchAgents/com.iceberg-cpp.daily-analysis.plist
```

**Plist:** `~/Library/LaunchAgents/com.iceberg-cpp.daily-analysis.plist`
**Logs:** `logs/analysis_YYYYMMDD_HHMMSS.log`

## Manual Run

```bash
./run_daily_analysis.py
```

> Note: `run_daily_analysis.py` redirects all output to a log file. Check `logs/` for results.

## Recent Reports

- [2026-06-10 — Core/Schema](daily_reports/2026-06-10_core_schema.md)
- [2026-06-11 — Arrow](daily_reports/2026-06-11_arrow.md)
