# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build System and Development Commands

### Essential Build Commands
```bash
# Build extension (release)
make

# Build extension (debug) - recommended for development
make debug

# Build with VPTR sanitizer disabled (for DuckDB 1.4.0 compatibility)
make DISABLE_VPTR_SANITIZER=1

# Build DuckDB shell with extension loaded
make shell          # Creates build/debug/duckdb binary

# Clean build artifacts
make clean

# Update Rust/C++ bindings (after modifying Rust code)
make rust_binding_headers
```

### Testing Commands
```bash
# Run all SQL logic tests
make test           # Release mode
make test_debug     # Debug mode

# Run specific memory/performance tests
./test/memory/test_algorithm_benchmark.sh [binary_path] [operations_count]
./build/debug/duckdb -unsigned < test/memory/test_algorithm_memory_comparison.sql

# Manual testing with built shell
./build/debug/duckdb -unsigned
# Then: LOAD crypto; SELECT crypto_hash('blake3', 'test');
```

### Memory Leak Testing
```bash
# Quick memory leak test (1K operations)
./build/debug/duckdb -unsigned < test/memory/test_memory_leak.sql

# Comprehensive benchmark with detailed metrics
./test/memory/test_algorithm_benchmark.sh ./build/debug/duckdb 10000

# Large-scale validation (1M operations)
./test/memory/test_algorithm_benchmark.sh ./build/debug/duckdb 1000000
```

## Architecture Overview

This is a **hybrid Rust/C++ DuckDB extension** that provides cryptographic hash functions. The key architectural components:

### Core Components
- **`src/crypto_extension.cpp`**: Main C++ extension interface, registers functions with DuckDB
- **`duckdb_crypto_rust/src/lib.rs`**: Rust implementation of cryptographic algorithms
- **`src/include/rust.h`**: Auto-generated C++ bindings (via cbindgen)
- **Memory Management**: Critical `FreeResultCString()` function prevents memory leaks at Rust/C++ boundary

### Build Integration
- **Corrosion**: Integrates Rust builds into CMake/DuckDB build system
- **cbindgen**: Generates C++ headers from Rust code
- **Static Linking**: Rust library becomes `libcrypto_extension.a`

### Extension Interface
```cpp
// Two main functions exposed to DuckDB:
crypto_hash(algorithm: VARCHAR, data: VARCHAR) -> VARCHAR
crypto_hmac(algorithm: VARCHAR, key: VARCHAR, data: VARCHAR) -> VARCHAR
```

### Supported Algorithms
- **Blake family**: `blake2b-512`, `blake3` (memory leak fix applied)
- **SHA family**: `sha1`, `sha2-224/256/384/512`, `sha3-224/256/384/512`
- **Others**: `md4`, `md5`, `keccak224/256/384/512`

## Critical Memory Management

### Memory Leak Fix (IMPORTANT)
The extension previously had a memory leak at the Rust/C++ boundary. **Fixed in `src/crypto_extension.cpp:19-26`**:

```cpp
static void FreeResultCString(ResultCString &result) {
    if (result.tag == ResultCString::Tag::Ok && result.ok._0 != nullptr) {
        duckdb_free(result.ok._0);  // Free successful hash result
    } else if (result.tag == ResultCString::Tag::Err && result.err._0 != nullptr) {
        duckdb_free(result.err._0); // Free error message
    }
}
```

**Critical**: Always call `FreeResultCString()` after `StringVector::AddString()` in both success and error paths.

### Memory Validation
- Extension handles 1M+ operations with stable ~415MB memory usage
- No memory growth between operations (linear scaling only)
- Memory leak tests validate both Blake and other algorithms

## Development Notes

### DuckDB Version Compatibility
- Built for **DuckDB 1.4.0**
- Telemetry function disabled for compatibility (`src/crypto_extension.cpp:101`)
- Use `-unsigned` flag when loading extension during development

### File Organization
- **`test/sql/`**: SQLLogicTest format tests for DuckDB test runner
- **`test/memory/`**: Comprehensive memory leak and performance validation suite
- **`build/debug/extension/crypto/`**: Final extension artifacts

### Extension Loading
```sql
-- For development/unsigned extensions:
LOAD crypto;

-- For production (community extensions):
INSTALL crypto FROM community;
LOAD crypto;
```

The extension is memory-safe, performance-validated, and ready for production use with proper build procedures.