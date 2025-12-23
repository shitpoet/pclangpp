# Drop-in replacement for `clang++` with automatic std headers precompilation

`clang++-autopch` is a small wrapper around `clang++` that automatically precompiles standard C++ headers (`#include <...>`) used in your source files and reuses them across compilations.

The main goal is to **speed up compilation of small C++ programs**, especially in teaching or interactive workflows where the same tiny file is compiled and run many times with similar flags.

The tool:

- scans `.cpp` files for standard headers
- builds a single precompiled header (PCH) for them
- caches the PCH based on compiler flags + headers
- reuses it transparently on subsequent runs

If anything goes wrong, it silently falls back to invoking `clang++` directly.

## Usage

Use `clang++-autopch` exactly like `clang++`:

```bash
clang++-autopch -std=c++26 a.cpp
```

Wildcard "*.cpp" is supported:

```bash
clang++-autopch -std=c++26 *.cpp
```

The wrapper forwards all arguments to `clang++` and preserves stdout, stderr, and exit codes.

### Cache directory

By default, precompiled headers are stored in:

```
~/.cache/clang++-autopch
```

You can override this location using the environment variable:

```bash
CLANGPP_AUTOPCH_DIR=/path/to/cache
```

Each cached entry consists of:

- `<hash>.hpp` — a synthetic header containing `#include <...>`
- `<hash>.pch` — the corresponding precompiled header


## Some limitations

- Only standard headers included with angle brackets (`#include <...>`) are considered.
- Only `.cpp` source files are supported.
- Header order matters; only common prefixes across multiple files are used.
- If standard headers are included in unusual places (e.g. in the middle of a file), they are still detected, but correctness is not guaranteed (for example, `cmath` can introduce conflicts for the later part of file, but not for the earlier)
- The tool is **not** a build system and does not replace Make, Ninja, or CMake.
- Small Python startup overhead is introduced (tens of ms)

## Cleanup

Cached precompiled headers are cleaned up automatically:

- Cleanup runs probabilistically (not on every invocation).
- Entries not used for **14 days** are removed.

You can also delete the entire cache directory manually at any time.


## How it works

1. The wrapper parses the `clang++` command-line arguments.
2. It finds all `.cpp` source files (including `*.cpp` wildcards).
3. Each source file is scanned for `#include <...>` directives.
4. The common set of standard headers is determined and deduplicated.
5. A hash is computed from:
   - the relevant compiler flags
   - the ordered list of standard headers
6. If a matching PCH already exists in the cache, it is reused.
7. Otherwise, a small synthetic header is generated and compiled into a PCH.
8. Finally, `clang++` is executed with the PCH injected automatically.

All of this happens transparently; the user interacts with the tool exactly like with `clang++`.
