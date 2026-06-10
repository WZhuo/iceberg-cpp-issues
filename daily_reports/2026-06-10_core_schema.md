# Daily Analysis Report: Core/Schema
**Date:** 2026-06-10
**Files Analyzed:** 82 C++ files (all .cc and .h under src/iceberg/), 3 Java reference files

## Summary
- Logic Errors: 4 found
- Java Inconsistencies: 4 found
- Incomplete Features: 10 found

## Findings

### Logic Errors

#### transform_function.cc:134 — YearTransform constructed with wrong TransformType
- **Severity:** high
- **Description:** `YearTransform::YearTransform(std::shared_ptr<Type> const& source_type)` passes `TransformType::kTruncate` to the `TransformFunction` base class instead of `TransformType::kYear`. This means `YearTransform::transform_type()` will return `kTruncate` instead of `kYear`, which affects any code path that dispatches based on `transform_type()` (e.g., `Transform::Project()`, `Transform::ProjectStrict()`, `Transform::SatisfiesOrderOf()`).
- **Fix:** Change line 134 from `TransformFunction(TransformType::kTruncate, source_type)` to `TransformFunction(TransformType::kYear, source_type)`.

#### statistics_file.cc:53-58 — ToString(PartitionStatisticsFile) missing closing bracket
- **Severity:** low
- **Description:** The `ToString(const PartitionStatisticsFile&)` function starts its string representation with `PartitionStatisticsFile[` but returns without appending the closing `"]"`. The line `SS_FMT(repr, "PartitionStatisticsFile[snapshot_id={} path={}]", ...)` lacks the trailing bracket.
- **Fix:** Change the format string to `"PartitionStatisticsFile[snapshot_id={} path={}]"` (add `]` before closing quote).

#### partition_summary_internal.h:42 — Dangling reference risk in PartitionFieldStats
- **Severity:** medium
- **Description:** `PartitionFieldStats` stores `const std::shared_ptr<Type>& type_{nullptr}` — a reference to a `shared_ptr` owned by the caller's `SchemaField`. If the caller's `SchemaField` goes out of scope or its type is replaced, this reference dangles. The `PartitionSummary` constructor passes `field.type()` (which returns a `const shared_ptr<Type>&`) to `PartitionFieldStats`. While this works if the `SchemaField` outlives the `PartitionSummary`, the design is fragile and a future refactoring could easily break it.
- **Fix:** Store `std::shared_ptr<Type>` by value (copy) instead of by reference.

#### name_mapping.h:91-92, 131-132 — Thread-unsafe lazy initialization
- **Severity:** medium
- **Description:** `MappedFields::LazyNameToId()` and `LazyIdToField()` use `mutable` members with lazy initialization patterns that are not thread-safe. If multiple threads access the same `const MappedFields` object simultaneously, the "check-then-init" pattern can race. Similarly, `NameMapping::LazyFieldsById()` and `LazyFieldsByName()` have the same issue. The documentation doesn't note any thread-safety guarantees.
- **Fix:** Either document that `MappedFields` / `NameMapping` are not thread-safe for concurrent lazy initialization, or add a mutex/atomic to protect the lazy init path.

### Java Inconsistencies

#### table_metadata.h:80-83 vs Schema.java:62-68 — Missing type entries in kMinFormatVersions
- **Severity:** medium
- **C++ behavior:** `TableMetadata::kMinFormatVersions` contains only `kTimestampNs → 3`, `kTimestampTzNs → 3`, and `kUnknown → 3`.
- **Java behavior:** Java's `Schema.MIN_FORMAT_VERSIONS` additionally includes `VARIANT → 3`, `GEOMETRY → 3`, and `GEOGRAPHY → 3`.
- **Impact:** If/when C++ adds Variant/Geometry/Geography type support, `Schema::Validate()` won't enforce the correct minimum format version for these types. This could allow v2 metadata to reference types that require v3.
- **Fix:** When Variant/Geometry/Geography types are implemented in C++, update `kMinFormatVersions` to match Java.

#### transform_function.cc:255 vs Java VoidTransform — VoidTransform::Transform behavior differs
- **Severity:** medium
- **C++ behavior:** `VoidTransform::Transform(const Literal& literal)` returns `literal.IsNull() ? literal : Literal::Null(literal.type())`. The non-null case wraps the returned null literal with the input's type.
- **Java behavior:** Java's `VoidTransform.apply()` always returns null for both null and non-null inputs, without preserving the type.
- **Impact:** If downstream code relies on the VoidTransform result type being consistent with Java, the C++ behavior may differ. The C++ version preserves type information that Java discards.
- **Fix:** Consider returning a plain null literal without type preservation, or document the difference explicitly.

#### schema.cc:54-55,63-64 vs Java Types.java — Unknown type in Map key is rejected, Java allows
- **Severity:** low
- **C++ behavior:** `ValidateFieldNullability()` at line 64 explicitly checks `key.type()->type_id() != TypeId::kUnknown` and returns an error for unknown map keys.
- **Java behavior:** Java's `Types.MapType` does not have this explicit check for unknown type keys.
- **Impact:** The C++ implementation is stricter than Java. This could cause validation failures in C++ that would pass in Java, although unknown type map keys are extremely rare in practice.
- **Fix:** Either relax the C++ check to match Java, or align both implementations — the C++ stricter validation may actually be better.

#### Snapshot::Equals — Missing manifest_list and summary comparison
- **Severity:** low
- **C++ behavior:** `Snapshot::Equals()` compares `snapshot_id`, `parent_snapshot_id`, `sequence_number`, `timestamp_ms`, and `schema_id`, but NOT `manifest_list` or `summary`.
- **Java behavior:** Java's `Snapshot` is an interface; `BaseSnapshot.equals()` compares `manifestList` and `summary` as well.
- **Impact:** Two Snapshots with different manifest lists or summaries but identical metadata fields would be considered equal in C++ but not in Java. This could lead to incorrect deduplication or equality checks.
- **Fix:** Add `manifest_list` and `summary` comparisons to `Snapshot::Equals()`.

### Incomplete Features

#### table_metadata.cc:1650-1651 — RemoveSnapshots by shared_ptr vector throws "not implemented"
- **Description:** `TableMetadataBuilder::RemoveSnapshots(const std::vector<std::shared_ptr<Snapshot>>&)` throws `IcebergError("... not implemented")` instead of converting snapshot pointers to IDs and calling the ID-based overload.
- **Java reference:** Java's `TableMetadata.Builder.removeSnapshots()` accepts `List<Snapshot>`.
- **Priority:** medium

#### table_metadata.cc:1659-1660 — SuppressHistoricalSnapshots throws "not implemented"
- **Description:** `TableMetadataBuilder::SuppressHistoricalSnapshots()` throws `IcebergError("... not implemented")`. This feature removes metadata for historical snapshots that are not referenced by any ref.
- **Java reference:** Java's `TableMetadata.Builder.suppressHistoricalSnapshots()` is fully implemented.
- **Priority:** medium

#### table_metadata.cc:1704-1711 — AddEncryptionKey and RemoveEncryptionKey throw "not implemented"
- **Description:** Both `AddEncryptionKey()` and `RemoveEncryptionKey()` throw `IcebergError("... not implemented")`. Encryption key management is not yet implemented.
- **Java reference:** Java supports encryption keys in `TableMetadata`.
- **Priority:** low

#### file_io.h:155-157 — FileIO::DeleteFile returns NotImplemented
- **Description:** `FileIO::DeleteFile()` returns `NotImplemented("DeleteFile not implemented")`. Concrete FileIO implementations must override this, but the default is not useful.
- **Java reference:** Java's `FileIO.deleteFile()` has a proper default implementation.
- **Priority:** low

#### json_serde.cc:312 — TODO for default values in StructType serialization
- **Description:** Comment `// TODO(gangwu): add default values` in `StructType` JSON serialization. Iceberg v3 supports default values for fields.
- **Java reference:** Java's `StructType` JSON serialization includes default value handling.
- **Priority:** medium

#### json_serde.cc:708-710 — Commented-out snapshot summary field validation
- **Description:** The validation of snapshot summary fields against `kValidSnapshotSummaryFields` is entirely commented out, allowing any key-value pairs into snapshot summaries without validation.
- **Java reference:** Java validates summary fields more strictly.
- **Priority:** medium

#### file_writer.h:67 — TODO for adding table properties with write format prefixes
- **Description:** `// TODO(gangwu): add table properties with write.avro|parquet|orc.*` — format-specific writer configuration from table properties is not yet implemented.
- **Java reference:** Java wires table properties through to writer configuration.
- **Priority:** low

#### delete_file_index.h:249 — TODO for lazy iterator to avoid large memory allocation
- **Description:** `// TODO(gangwu): use lazy iterator to avoid large memory allocation` in `ReferencedDeleteFiles()` — the current implementation may materialize large delete file lists in memory.
- **Java reference:** Java uses lazy/streaming approaches for delete file iteration.
- **Priority:** low

#### location_provider.cc:197 — TODO for custom location provider implementation
- **Description:** `// TODO(xxx): create location provider specified by "write.location-provider.impl"` — the extensible location provider mechanism is not yet implemented. Only the default and object-store providers are available.
- **Java reference:** Java supports custom location providers via the `write.location-provider.impl` property.
- **Priority:** low

#### metadata_columns.h:123 — TODO for building partition columns from table schema
- **Description:** `// TODO(gangwu): add functions to build partition columns from a table schema` — partition metadata columns are not yet dynamically constructed from schemas.
- **Java reference:** Java's `MetadataColumns` fully supports building partition columns from schema.
- **Priority:** medium

## Recommendations

1. **CRITICAL: Fix YearTransform constructor** (`transform_function.cc:134`). The wrong `TransformType::kTruncate` instead of `kYear` is a high-severity bug that affects all code paths dispatching on transform type (Project, ProjectStrict, SatisfiesOrderOf). This is a copy-paste error that should be fixed immediately.

2. **Fix PartitionStatisticsFile::ToString formatting** — add the missing closing bracket.

3. **Address thread-safety of lazy initialization** in `NameMapping` and `MappedFields`. Either document the lack of thread-safety or add synchronization. Given that Iceberg tables are typically loaded from a single thread, documentation may suffice.

4. **Implement the stubbed-out builder methods** — `RemoveSnapshots` (by shared_ptr), `SuppressHistoricalSnapshots`, `AddEncryptionKey`, `RemoveEncryptionKey`. The `RemoveSnapshots` shared_ptr overload is particularly important as it can be trivially implemented by extracting IDs from the shared pointers.

5. **Update `kMinFormatVersions`** to include Variant, Geometry, and Geography entries when those types are added to C++, to match Java behavior.

6. **Consider aligning `VoidTransform::Transform`** behavior with Java — the type preservation on null result in C++ may cause subtle inconsistencies.

7. **Restore snapshot summary field validation** in `json_serde.cc` — the commented-out validation weakens data integrity checks compared to Java.

8. **Consider storing `shared_ptr<Type>` by value** in `PartitionFieldStats` instead of by reference to prevent potential dangling reference bugs.
