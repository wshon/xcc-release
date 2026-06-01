# Bug Condition Exploration Evidence

**Date**: Task 1 Execution
**Status**: Bug Confirmed - ICU Dependencies Exist in Unfixed Code

## Bug Condition Summary

The application fails to start on Windows Server 2016 with error: "找不到 icuuc.dll" (Cannot find icuuc.dll)

**Root Cause**: The `url = "2.5"` dependency in `Cargo.toml` uses default features, which includes the `idna` feature. This feature brings in ICU library dependencies that require external DLL files not available on Windows Server 2016.

## Evidence Collected

### 1. ICU Dependencies in Dependency Tree

Running `cargo tree -p xconnect_console | Select-String -Pattern "icu"` shows:

```
├── icu_normalizer v2.1.1
│   ├── icu_collections v2.1.1
│   ├── icu_normalizer_data v2.1.1
│   ├── icu_provider v2.1.1
│   │   └── icu_locale_core v2.1.1
├── icu_properties v2.1.2
│   ├── icu_collections v2.1.1 (*)
│   ├── icu_locale_core v2.1.1 (*)
│   ├── icu_properties_data v2.1.2
│   └── icu_provider v2.1.1 (*)
```

**Conclusion**: ICU dependencies (icu_normalizer, icu_properties, icu_collections, icu_locale_core) are present in the dependency tree.

### 2. URL Crate IDNA Feature Enabled

Running `cargo tree -p url -e features` shows:

```
url v2.5.8
├── idna feature "alloc"
│   └── idna v1.1.0
│       ├── idna_adapter feature "default"
│       │   └── idna_adapter v1.2.1
│       │       ├── icu_normalizer v2.1.1
│       │       └── icu_properties v2.1.2
└── idna feature "compiled_data"
    ├── idna v1.1.0 (*)
    └── idna_adapter feature "compiled_data"
        ├── icu_normalizer feature "compiled_data"
        └── icu_properties feature "compiled_data"
```

**Conclusion**: The `idna` feature is enabled on the url crate, which brings in the dependency chain:
- url → idna → idna_adapter → icu_normalizer/icu_properties

### 3. Current Cargo.toml Configuration

In `src/Cargo.toml`:
```toml
url = "2.5"
```

This is equivalent to:
```toml
url = { version = "2.5", default-features = true }
```

The default features include `idna`, which is not needed for the application's use case (standard ASCII URLs only).

## Bug Condition Test

Created test file: `src/tests/icu_dependency_exploration.rs`

The test includes two test cases:
1. `test_no_icu_dependency_in_cargo_tree` - Checks that ICU crates are NOT in dependency tree
2. `test_url_crate_without_idna_feature` - Checks that idna feature is NOT enabled

**Expected Behavior**:
- **On Unfixed Code**: Tests FAIL (confirms bug exists) ✓
- **On Fixed Code**: Tests PASS (confirms bug is fixed)

## Counterexamples Found

1. **ICU Dependencies Present**: icu_normalizer, icu_properties, icu_collections, icu_locale_core
2. **IDNA Feature Enabled**: url crate has idna feature enabled by default
3. **External DLL Requirement**: ICU libraries require icuuc.dll which is missing on Windows Server 2016

## Next Steps

The bug condition has been confirmed. The fix (Task 3) will:
1. Modify `Cargo.toml` to disable default features on url crate
2. Explicitly enable only the `serde` feature (which is needed)
3. Verify that ICU dependencies are removed from the dependency tree
4. Confirm that the bug condition exploration tests pass after the fix
