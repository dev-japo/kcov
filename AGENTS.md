# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Project Overview

**kcov** is a code coverage analysis tool for compiled programs (C/C++), Python scripts, and Bash scripts. It works on FreeBSD, Linux, and macOS without requiring special compiler flags by leveraging DWARF debugging information for compiled languages.

### Key Features
- Multi-language support: C/C++, Python, Bash/shell scripts
- Multiple output formats: HTML, Cobertura XML, JSON, Coveralls, Codecov, SonarQube
- CI/CD integration: Travis CI, GitHub Actions, Jenkins, GitLab CI
- Merge mode: Combine coverage from multiple test runs
- System-wide instrumentation mode for full-system coverage
- No special compiler flags needed (uses DWARF debug info)

### Core Technologies
- **Language**: C++ (C++11 standard)
- **Build System**: CMake (minimum version 3.12)
- **Supported Architectures**: x86, x86_64, ARM, AArch64, PowerPC, RISC-V, LoongArch, SPARC64, s390, s390x
- **Key Dependencies**:
  - libstdc++ (C++ standard library)
  - libcurl (HTTP client for uploading coverage)
  - elfutils/libdw (DWARF debugging information parsing)
  - zlib (compression)
  - binutils/libbfd (optional, for binary file format handling)
  - libiberty (optional, for demangling)

## Architecture

kcov follows a modular architecture with clear separation of concerns:

### Core Components

1. **FileParser** (`src/parsers/`)
   - Reads binaries and extracts debugging information
   - Maps file/line numbers to memory addresses
   - Implementations: ELF parser (dwarf.cc), Mach-O parser (macho-parser.cc)
   - Runtime parsers for Python and Bash

2. **Engine** (`src/engines/`)
   - Low-level execution control (breakpoints, signals, process management)
   - Implementations:
     - `ptrace.cc` - Linux/FreeBSD ptrace-based engine
     - `bash-engine.cc` - Bash script coverage via PS4/DEBUG traps
     - `python-engine.cc` - Python coverage via sys.settrace
     - `kernel-engine.cc` - Kernel module coverage
     - `system-mode-engine.cc` - Full-system instrumentation

3. **Collector** (`src/collector.cc`)
   - Orchestrates execution control
   - Sets up breakpoints based on FileParser data
   - Reports hits to Reporter

4. **Reporter** (`src/reporter.cc`)
   - Tracks which lines have been executed
   - Maintains coverage statistics
   - Persists data between runs

5. **OutputHandler** (`src/output-handler.cc`)
   - Coordinates multiple writers
   - Manages output directory structure
   - Handles periodic output updates

6. **Writer** (`src/writers/`)
   - Generates coverage reports in various formats
   - Implementations: HTML, Cobertura, JSON, Coveralls, Codecov, SonarQube

7. **Filter** (`src/filter.cc`)
   - Include/exclude path and pattern filtering
   - Line and region exclusion support

### Data Flow
```
Binary/Script → FileParser → Collector ← Engine
                                ↓
                            Reporter → OutputHandler → Writers
                                ↓
                            Filter (applied throughout)
```

## Building and Running

### Build Instructions

**Prerequisites** (Ubuntu/Debian):
```bash
apt-get install binutils-dev build-essential cmake libssl-dev \
  libcurl4-openssl-dev libelf-dev libstdc++-12-dev zlib1g-dev \
  libdw-dev libiberty-dev
```

**Standard Build**:
```bash
mkdir build
cd build
cmake ..
make
make install
```

**Build with Custom Options**:
```bash
cmake \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr/local \
  -DCMAKE_C_FLAGS="-O3" \
  ..
make -j$(nproc)
```

**macOS Build**:
```bash
brew install zlib bash cmake pkgconfig dwarfutils openssl ninja
cmake -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DOPENSSL_ROOT_DIR=$(brew --prefix openssl) \
  ..
ninja
# Sign the binary for debugging support
codesign -s - --entitlements osx-entitlements.xml -f ./src/Release/kcov
```

### Running Tests

Tests are located in `tests/` directory and built via `tests/CMakeLists.txt`:
```bash
cd build
make
# Run individual test binaries
./tests/main-tests
./tests/fork
```

### Basic Usage

```bash
# Basic coverage collection
kcov /path/to/output ./executable [args...]

# With filtering
kcov --include-path=/src --exclude-pattern=/usr/include \
  /path/to/output ./executable

# Merge multiple runs
kcov --merge /output /run1 /run2 /run3

# Collect only (no report generation)
kcov --collect-only /output ./executable

# Report only (from collected data)
kcov --report-only /output ./executable
```

## Development Conventions

### Code Style
- C++11 standard (`-std=c++0x`)
- Header files in `src/include/`
- Implementation files in `src/`
- Use of interfaces (pure virtual classes) for extensibility
- Naming: Classes use PascalCase with 'I' prefix for interfaces (e.g., `IEngine`, `IFileParser`)

### Project Structure
```
src/
├── include/          # Public headers (interfaces)
├── engines/          # Execution engines
├── parsers/          # Binary/script parsers
├── writers/          # Output format writers
├── system-mode/      # Full-system instrumentation
└── *.cc              # Core implementation files

tests/                # Test programs and scripts
doc/                  # Documentation
data/                 # HTML/CSS/JS resources for reports
cmake/                # CMake modules
```

### Key Design Patterns
- **Strategy Pattern**: Multiple engine implementations (ptrace, bash, python)
- **Factory Pattern**: Engine creation via `IEngineFactory`
- **Observer Pattern**: Configuration listeners for dynamic updates
- **Singleton Pattern**: Configuration instance
- **Template Method**: Writer base class with format-specific implementations

### Adding New Features

**Adding a new output format**:
1. Create new writer class in `src/writers/` inheriting from `IWriter`
2. Implement `onStartup()`, `onStop()`, and `write()` methods
3. Register writer in `main.cc` or `output-handler.cc`

**Adding a new engine**:
1. Create new engine in `src/engines/` implementing `IEngine` interface
2. Implement required methods: `start()`, `continueExecution()`, `kill()`
3. Register in `IEngineFactory` for automatic selection

### Testing
- Unit tests use custom framework (see `tests/unit-tests/`)
- Integration tests are standalone executables in `tests/`
- Test coverage includes: fork handling, signals, shared libraries, multi-threading
- Python tests require Python 3.8+

### Contribution Guidelines
- Prefix commit messages with "Issue #xxx" when fixing reported issues
- Submit pull requests via GitHub
- Refer to `CONTRIBUTING.md` for the patch acceptance process
- Review `doc/design.txt` for architecture understanding

### Debugging
- Use `--debug=<level>` flag (max 31) for verbose output
- Debug mask controls different subsystems
- `--verify` option checks breakpoint setup (requires libbfd)

### Platform-Specific Notes
- **Linux**: Uses ptrace for process control
- **FreeBSD**: Uses ptrace with FreeBSD-specific syscalls
- **macOS**: Uses LLDB/Mach APIs, requires code signing for debugging
- Bash coverage uses PS4 trap or DEBUG trap methods
- Python coverage uses `sys.settrace()` API

### Configuration System
- Centralized in `configuration.cc`
- Key-value store with string, int, and string vector types
- Listener pattern for configuration changes
- Command-line parsing via getopt_long
- Supports `--configure=key=value` for runtime configuration

### Memory Management
- Manual memory management with explicit `delete` calls
- Cleanup handled in `do_cleanup()` function
- Signal handlers forward signals to traced processes

## Important Files

- `src/main.cc` - Entry point and mode selection
- `src/configuration.cc` - Configuration parsing and management
- `src/collector.cc` - Core coverage collection logic
- `src/engines/ptrace.cc` - Linux/FreeBSD process tracing
- `src/parsers/elf-parser.cc` - ELF binary parsing
- `doc/design.txt` - Architecture documentation
- `CMakeLists.txt` - Build configuration
- `INSTALL.md` - Detailed build instructions
