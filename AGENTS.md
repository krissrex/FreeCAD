# FreeCAD Agent Guidelines

This file provides guidance for AI agents working on the FreeCAD codebase.

## Build Commands

FreeCAD uses **CMake** with **Ninja** and **pixi** for dependency management:

```bash
# Initialize dependencies and submodules
pixi run initialize

# Configure (debug is default) - generates Ninja build files
pixi run configure
pixi run configure-debug
pixi run configure-release

# Build using Ninja (via cmake --build or directly)
pixi run build
cmake --build build/debug
ninja -C build/debug

# Install
pixi run install

# Run all tests
pixi run test
ctest --test-dir build/debug

# Run a single test (example)
ctest --test-dir build/debug -R Base_Vector3D
./build/debug/bin/Tests --gtest_filter=Vector3DTest.*

# Run FreeCAD
pixi run freecad
```

Windows-specific:
```bash
pixi run configure-debug -DCMAKE_GENERATOR_PLATFORM= -DCMAKE_GENERATOR_TOOLSET=
pixi run freecad-debug  # Uses .pixi/envs/default/Library/bin/FreeCAD.exe
```

## Code Style & Formatting

### C++

- **Formatter**: clang-format (configured in `.clang-format`)
  - Based on LLVM style
  - 4 spaces indentation (no tabs)
  - 100 character line limit
  - Braces on new lines for functions/classes
- **Linter**: clang-tidy (configured in `.clang-tidy`)
  - Run manually: `clang-tidy -p build/debug <file.cpp>`

**Naming Conventions:**
- Classes: `PascalCase` (e.g., `PartFeature`, `BaseClass`)
- Methods: `camelCase` (e.g., `getTypeId()`)
- Variables: `camelCase` (e.g., `classTypeId`)
- Macros: `UPPER_CASE` with underscores (e.g., `TYPESYSTEM_HEADER`)
- Namespaces: lowercase (e.g., `namespace Part`)
- Member variables: no specific prefix/suffix convention

### Python

- **Formatter**: black (100 character line length)
- **Linter**: pylint (configured in `.pylintrc`)

**Naming Conventions:**
- Functions: `snake_case`
- Classes: `PascalCase`
- Variables: `snake_case`
- Constants: `UPPER_CASE`

## Imports

### C++
- Sort includes: Not enforced (SortIncludes: Never)
- Group includes logically (system, library, project)
- Use `#include <...>` for system/library headers, `#include "..."` for project headers

### Python
- Follow PEP 8 import ordering
- Use absolute imports where possible
- Avoid wildcard imports

## License Headers

All files MUST include the SPDX license identifier:
```cpp
// SPDX-License-Identifier: LGPL-2.1-or-later
```

Or for Python:
```python
# SPDX-License-Identifier: LGPL-2.1-or-later
```

## Error Handling

- C++: Use exceptions for exceptional cases, return values for expected errors
- Python: Use exceptions appropriately, check with pylint for bare except clauses
- Prefer specific exception types over generic `Exception`

## Testing

- Tests are in `tests/` directory
- C++ tests use Google Test
- Enable tests with `ENABLE_DEVELOPER_TESTS=ON` (enabled by default in conda presets)
- Run specific tests with CTest `-R` flag or gtest_filter

## Project Structure

- `src/` - Main source code
  - `Base/` - Core base classes and utilities
  - `App/` - Application layer
  - `Gui/` - GUI layer
  - `Mod/` - Workbench modules (Part, Draft, FEM, etc.)
- `tests/` - Test suites
- `data/` - Data files
- `cMake/` - CMake modules

## Contribution Guidelines

See [CONTRIBUTING.md](CONTRIBUTING.md):
- PRs must be minimal and solve exactly one problem
- All commits must compile cleanly
- Code must pass style checks
- Avoid breaking Python API used by extensions
- Write clear commit messages following git best practices

## PRs and Commits
1. Commit message **should** include prefix with module name, e.g. `Part: `
2. Commit message **should** use imperative mode - just like git does, e.g. `Part: Add fillet feature`
    <details>
    A properly formed Git commit subject line should always be able to complete the following sentence:

    If applied, this commit will **your commit message**,

    e.g. If applied, this commit will add fillet feature.
    </details>
3. If changeset is big and spans across multiple modules it **should** be split into one commit per module.
4. Every commit in the PR **must** compile and run properly. `git bisect` workflow does not work if some commits are not possible to compile.
5. Commit message **can** contain additional information after one blank line.
6. Every PR **should** have description, even if it seems obvious.


## Basic Code Rules
1. (C++ only) New code **must** be formatted with clang-format tool or in a way that is compatible with clang-format result if file is excluded from auto formatting.
1. (Python only) Code **must** follow the PEP 8 standard and formatted with Black.
3. Main execution path **should** be the least indented one, i.e. conditions should cover specific cases.
4. Early-Exit **should** be preferred to prune unwanted execution branches fast.
7. Global state (global variables, static fields, singletons) **should be** avoided.
8. (C++ only) `enum class` **should** be preferred over normal enum whenever possible.
9. (C++ only) C++ libraries/structures **should** be preferred over C ones. i.e. `std::array/vector` should be used instead of C array.
10. (C++ only) C++ standard library **should** be utilized if possible, especially the `<algorithms>` one.
    <details>
        <summary>Example #1</summary>

        Consider following code:
        ```c++
            std::vector<int> vertices;

            std::set<int> vertexSet;
            for (auto &s : face.getSubShapes(TopAbs_VERTEX)) {
                int idx = shape.findShape(s) - 1;
                if (idx >= 0 && vertexSet.insert(idx).second) {
                    vertices.push_back(idx);
                }
            }
        ```

        You can rewrite it in a following way:
        ```c++
            std::vector<int> vertices;

            std::set<int> vertexSet;
            for (auto &vertex : face.getSubShapes(TopAbs_VERTEX)) {
                int vertexId = shape.findShape(vertex);

                if (vertexId > 0) {
                    vertexSet.insert(vertexId);
                }
            }


            std::copy(vertexSet.begin(), vertexSet.end(), std::back_inserter(vertices));
        ```

        This way you split the responsibility of computing unique set with result preparation.
    </details>
11. (C++ only) Return values **should** be preferred over out arguments.
    a. For methods that can fail `std::optional` should be used
    b. For methods that return multiple values it may be better to either provide dedicated struct for result or use `std::tuple` with expression binding.
13. (C++ only) If dealing with pointers `auto` **must** be suffixed with `*`: `auto*`.
14. (C++ only) If dealing with references `auto` **must** be suffixed with `&`: `auto&`
15. (C++ only) `auto` **should** be used if:
    a. the right-hand side makes clear which type it is, e.g. with `static_cast`, or `new`
    b. the type is long or verbose (iterators, ranges, ...)
    c. it reduces redundancy in code (repeated type)
15. (C++ only) `auto` **should not** be used if:
    a. dealing with primitives (the type is not always clear, int vs unsigned etc)
    b. the type is not clear from context, method name or template arguments
16. If expression is not obvious - it **should** be given a name by using variable.
    <details>
        <summary>Example #1</summary>
        TODO: Find some good example
    </details>
17. (C++ only) Code **should not** be put in anonymous namespace.
18. All members **must** be initialized.
    <details>
        <summary>Rationale</summary>
        Not initialized members can easily cause undefined behaviors that are really hard to find.
    </details>
19. If possible, functions / methods **should** not use more than 3 indentation levels.
    <details>
        <summary>Rationale</summary>
        Main path of code should be the least indented, exceptional cases should be handled as
        early-exit if statements. Code that is highly indented is harder to process and often
        has large cognitive load - if something is not possible to write within 3 indentation
        levels it is a good signal that it might require splitting into smaller methods.
    </details>
20. (C++ only) dedicated casts should be preferred over `dynamic_cast` due to performance
    a. QObjects should use `qobject_cast`
    b. FreeCAD types (deriving from `BaseClass`) should use `freecad_cast`

## Design / Architecture
1. Each class / method / function **should** be doing only one thing and should do it well.
2. Classes that do provide business logic **should** be stateless. This helps with reusability.
3. Functions / Methods **should** be pure i.e. their result should only depend on arguments (and object state in case of method). This helps with reusability, predictability and testing.
4. Immutable data **should** be used whenever possible, (For C++ use `const`)
    <details>
        <summary>Rationale</summary>
        It is much easier to reason about code that deals with data that does not change.

        Using const modifier ensures that the object will stay unmodified and compiler will make sure that it is the case.
    </details>
5. (C++ only) If possible `constexpr` **should** be used for storing constant data.
5. Long methods **should** be split into smaller, better described ones.
6. (C++ only) Defining new macros **must** be avoided, unless absolutely necessary.
7. Integers **must not** be used to express anything other than numbers. For enumerations enums **must** be used.
8. Code **should** be written in a way that it expresses intent, i.e. what should be done, rather than just how it is done.
    <details>
        <summary>Example #1</summary>
        Consider this code:
        ```c++
            void setOverlayMode(OverlayMode mode)
            {
                // ... some code ...

                QDockWidget *dock = nullptr;

                for (auto w = qApp->widgetAt(QCursor::pos()); w; w = w->parentWidget()) {
                    dock = qobject_cast<QDockWidget*>(w);
                    if (dock) {
                        break;
                    }
                    auto tabWidget = qobject_cast<OverlayTabWidget*>(w);
                    if (tabWidget) {
                        dock = tabWidget->currentDockWidget();
                        if (dock) {
                            break;
                        }
                    }
                }

                if (!dock) {
                    for (auto w = qApp->focusWidget(); w; w = w->parentWidget()) {
                        dock = qobject_cast<QDockWidget*>(w);
                        if (dock) {
                            break;
                        }
                    }
                }

                // some more code ...

                toggleOverlay(dock, m);
            }
        ```

        It is hard to understand what is the job of the for loop inside `if (!dock)` statement.
        We can refactor it to a new `QWidget* findClosestDockWidget()` method for it to look like this:

        ```c++
            void setOverlayMode(OverlayMode mode)
            {
                // ... some code ...

                QDockWidget *dock = findClosestDockWidget();

                // ... some more code ...

                toggleOverlay(dock, m);
            }
        ```
        The findClosestDockWidget() could either be implemented as private method or an inner function using lambdas.

        ```c++
        auto findClosestDockWidget = []() { ... }
        ```

        That way reading through code of `setOverlayMode` we don't need to care about the details of finding the closest dock widget.
    </details>
10. Boolean arguments **must** be avoided. Use enumerations instead - enum with 2 values is absolutely fine.
    For python boolean arguments are ok, but they **must** forced to be keyword ones.
    <details>
        <summary>example #1 (c++)</summary>
        consider following example:

        ```c++
        mapper.populate(false, it.key(), it.value());
        ```

        it is impossible to understand what false means without consulting the documentation or at least the method signature.
        instead the enum should be used:

        ```c++
        mapper.populate(mappingstatus::modified, it.key(), it.value());
        ```

        now the intent is clear
    </details>
11. (C++ only) Code **should** prefer uniform initialization `{ }` (also called brace initialization). Prevents narrowing.
12. (C++ only) Class member variables **should** be initialized at declaration, not in constructors
13. Magic numbers or other literals **must** be avoided.


## Naming Things

Naming described in that section of the document refers mostly to newer parts of code. Some modules use different naming conventions,
while checking the code Reviewer should take into account surrounding code and first of all ensure consistency within the closest scope.

1. Code symbols (classes, structs, methods, functions, variables...) **must** have names that are meaningful and grammatically correct.
2. Variables **should not** be named using abbreviations and/or 1 letter names. Iterator variables or math related ones like `i` or `u` are obviously not covered by this rule.
3. Names **must not** use the hungarian notation.
4. (C++ only) Classes/Structs/Enums **must** be written in `PascalCase`, underscores are allowed but should be avoided.
5. (C++ only) Class/Struct members **should** be written in `camelCase`, underscores are allowed but should be avoided.
6. (C++ only) Global functions **should** be written in `camelCase`, underscores are allowed but should be avoided.
7. (C++ only) Enum cases **should** use `PascalCase`
9. (C++ only) Enums should use singular form (i.e. `enum ObjectState` instead of `enum ObjectStates`)
8. (C++ only) Local variables **should** use `camelCase`

### Naming Conventions
1. Coin nodes **should** be prefixed with `So`, e.g. `SoTransformDragger`. Reimplementation of existing Coin nodes **should** use `SoFC` prefix, e.g. `SoFCTransform`.
2. Features in Part and Part Design workbenches **should** be prefixed with `Feature`, e.g. `FeatureHole`.
3. View Providers corresponding to features / other objects **should** be prefixed `ViewProvider`, e.g. `ViewProviderHole`
4. Task Views (content for task panels) **should** be prefixed with `Task`, e.g. `TaskTransform`
5. Task Dialog for Task View **should** be additionally suffixed with `Dialog`, e.g. `TaskTransformDialog`
6. Preference Pages **should** be prefixed with `DlgSettings`, e.g. `DlgSettingsLights`.
7. Commands **should** be prefixed with `Cmd` and Workbench name e.g. `CmdPartDesignCreateSketch`


## Commenting the code
1. Good naming **must** be preferred over commenting the code.
2. Comments that describe what code does **should** be avoided, instead comments **should** explain intended result.
3. All edge-cases in code **must** be described with comment describing when such edge-case can occur and why it is handled in certain way.
4. All "Hacks" **must** be described with how the hack works, why it is applied and when it no longer will be needed.
5. Commented code **must** be contain additional information on why it was commented out and when it is safe to remove it.
