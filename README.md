# mlc — Memory Leak Check

A header-only C++17 memory leak checker that tracks allocations made while a recording scope is active. Leaks are reported as a pointer / size map, making it easy to pinpoint allocations that were never freed.

## Features

- Header-only — just `#include "mlc.h"`
- Scope-based recording: `issueRecord()` starts a recording scope, and nested scopes merge unfreed allocations into their parent
- Thread-local records, so multiple threads can record independently
- Reports leaked pointers and allocation sizes through a callback
- `checkMemoryLeak()` returns `true` when no leaks are found
- Small CMake interface target (`mlc::mlc`)

## How it works

The header replaces the global `operator new` / `operator delete`. While a `ScopeMemoryLeakCheck` record is active on the current thread, every allocation is inserted into that record. `checkMemoryLeak()` diffs the set of recorded allocations against the set that is still live and reports the remaining ones. When a record is destroyed, its unfreed allocations are merged into the parent record instead of being lost.

## Usage

### CMake

```cmake
add_subdirectory(mlc)
target_link_libraries(${PROJECT_NAME} PRIVATE mlc::mlc)
```

### Code

```cpp
#include "mlc.h"

using ohtoai::mlc::ScopeMemoryLeakCheck;

int main() {
    auto record = ScopeMemoryLeakCheck::issueRecord();

    new int;  // leaked

    bool clean = record->checkMemoryLeak([](const auto& leaks) {
        for (const auto& [ptr, size] : leaks) {
            std::cout << "leak: " << ptr << " (" << size << " bytes)\n";
        }
    });

    std::cout << (clean ? "no leaks" : "leaks found") << "\n";
}
```

See [`test/main.cpp`](test/main.cpp) for a longer example covering nested records, arrays, and manual cleanup.

## Building the demo

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
./build/mlc_demo        # Linux / macOS
.\build\Debug\mlc_demo.exe   # Windows (Visual Studio generator)
```

## Notes

- Only allocations made through the replaced `operator new` are tracked; direct `malloc` / platform allocators are not.
- The global `operator new` / `operator delete` definitions live in the header. Include it in a single translation unit per binary (the one that should provide the replacement), rather than including it in every source file.
- The checker is intended for debugging and tests; enabling it globally can add noticeable overhead.

## Project layout

```text
include/mlc.h    header-only implementation
test/main.cpp    usage example / demo
CMakeLists.txt   interface target and demo executable
```

## License

[MIT](LICENSE.txt)
