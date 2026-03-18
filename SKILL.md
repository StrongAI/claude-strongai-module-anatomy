---
name: module-anatomy
description: >-
  Use when reading, understanding, navigating, or explaining StrongAI module
  structure. Triggers on: "what is this module", "how is X organized", reading
  cppm files, understanding partition vs standalone, traits directory structure,
  module naming conventions, export blocks, GMF blocks, umbrella modules, "what
  does declarations.hpp do", module interface units, 〇 namespace,
  header-to-module mapping, cppm anatomy, "where is the implementation".
---

# StrongAI Module Anatomy

## Overview

A StrongAI module is a git submodule directory containing `.cppm` files (C++23 module interface units), a `traits/` hierarchy of header files, a `Makefile`, and optionally a `modules.modulemap`. Every module follows the same anatomical structure regardless of what it implements.

## When to Use

- Reading or navigating any module directory
- Understanding what a `.cppm` file does
- Figuring out what each `traits/` file is responsible for
- Tracing how a type is exported from a module
- Understanding partition vs standalone submodule distinction

**NOT for**: build failures (use `module-build-test`), PID/Aspect type system design (use `pid-aspect-system`).

## Module Directory Layout

Using `aspect` as the canonical example:

```
aspect/
  strongai.aspect.cppm                     # Umbrella module (re-exports all partitions)
  strongai.aspect.identifiers.cppm         # Standalone submodule (dot in PCM name)
  strongai.aspect.declarations.cppm        # Partition (dash in PCM name)
  strongai.aspect.errors.cppm
  strongai.aspect.is.cppm
  strongai.aspect.set.cppm
  strongai.aspect.get.cppm
  strongai.aspect.implementation.cppm
  ...
  traits/
    identifiers.hpp                        # ID tag structs
    declarations.hpp                       # Forward declarations, using aliases
    declarations/                          # Subdirectory for large declaration sets
      interface.hpp
    definition.hpp                         # Definition<> template specializations
    dependencies.hpp                       # Implementation dependencies
    root.hpp                               # RootAspect alias — the module's public type entry
    partitions/                            # Headers that map 1:1 to cppm partitions
      declarations.hpp
      identifiers.hpp
  abstract/                                # CRTP base classes (optional)
  implementation.hpp                       # Concrete implementation
  Makefile
  modules.modulemap                        # Clang header module declarations (optional)
  test/
    Makefile
    test_runner.cpp or tests.cpp
```

## CPPM File Anatomy

Every `.cppm` file has up to four blocks:

```cpp
// 1. Global Module Fragment (GMF) — system headers needed before module context
module;
#include <cstdint>

// 2. Module declaration — what this file defines
export module strongai.aspect:declarations;    // partition
// OR: export module strongai.aspect;          // umbrella
// OR: export module strongai.aspect.identifiers;  // standalone

// 3. Imports — other modules/partitions this depends on
import std;
import strongai.identifiers;
import :pid;                                   // partition import (same module)

// 4. Export block — the actual exported declarations
export {
#include "traits/declarations.hpp"
}
```

**GMF rules**: Only `#include` of system/third-party headers. Required when the exported headers use types from `<cstdint>`, `<errno.h>`, etc. that must be visible before the module context.

**Umbrella pattern**: The umbrella `.cppm` re-exports all partitions via `export import :name`:

```cpp
export module strongai.aspect;
import std;
import strongai.type.detect;
import strongai.pid;

export import strongai.aspect.identifiers;     // standalone submodule (dot)
export import :declarations;                    // partition (colon)
export import :errors;
export import :is;
export import :implementation;
```

## Naming System

| Concept        | Format                                        | Example                           |
| -------------- | --------------------------------------------- | --------------------------------- |
| Package Name   | `StrongAI•Type•Pack` (bullet separator)       | `StrongAI•Aspect`                 |
| Module Name    | `strongai.type.pack` (dot separator)          | `strongai.aspect`                 |
| Directory Path | `type/pack` (slash separator)                 | `aspect`                          |
| Partition PCM  | `strongai.aspect-declarations.pcm` (**dash**) | Dash = partition of parent module |
| Standalone PCM | `strongai.aspect.identifiers.pcm` (**dot**)   | Dot = independent module          |
| Header Module  | `strongai__type__pack` (double underscore)    | Used in `modules.modulemap`       |

**Critical distinction**: Dash (`-`) in PCM name = partition. Dot (`.`) = standalone submodule. This determines whether `import :name` or `import strongai.module.name` is used.

## traits/ File Roles

| File               | Purpose                                    | Contains                                                    |
| ------------------ | ------------------------------------------ | ----------------------------------------------------------- |
| `identifiers.hpp`  | Compile-time identity tags                 | Empty `struct ID`, nested ID types, marker types            |
| `declarations.hpp` | Forward declarations and type aliases      | `template<typename...> struct Definition;`, `using` aliases |
| `pid.hpp`          | PID system participation                   | `PID::Define` specializations for inheritance               |
| `definition.hpp`   | Behavior specializations                   | `template<> struct Definition<TypeCategory, ID, ...>`       |
| `root.hpp`         | Public type entry point                    | `using RootAspect = Aspect::Root<PID, ID>;` + type aliases  |
| `dependencies.hpp` | Implementation prerequisites               | Imports/includes needed by implementation code              |
| `partitions/*.hpp` | Headers mapped 1:1 to cppm partition files | Content exported by each partition                          |

**Reading order**: identifiers → declarations → pid → definition → root. Each file depends on the ones before it.

## The `〇` Namespace

The root namespace `::〇::` (Unicode circle U+3007) is the framework's internal namespace. All core types (Stack, Joint, Context, Aspect, TypeAspect, etc.) live here. This is **intentional** — not an encoding error.

When reading code:
- `〇::Aspect<...>` = the internal Aspect template
- `::StrongAI::Aspect::Root<...>` = the public entry point that wraps `〇::Aspect`
- `〇::Stack<...>` = the compile-time type stack used by the PID system

## Common Mistakes

1. **Confusing partition vs standalone**: `strongai.aspect-declarations` (dash) is a partition imported via `:declarations`. `strongai.aspect.identifiers` (dot) is a standalone module imported via `strongai.aspect.identifiers`.

2. **Looking for logic in declarations.hpp**: It only has forward declarations and using aliases. The actual template bodies are in `definition.hpp` and `implementation.hpp`.

3. **Missing the umbrella**: The umbrella `.cppm` (same name as the module) is what consumers import. It re-exports everything. Individual partitions are internal.

4. **GMF confusion**: The `module;` line starts the GMF. Only raw `#include` of system headers goes here. Never module imports. The `export module` line ends the GMF.

5. **traits/partitions/ vs traits/**: Files in `traits/partitions/` map 1:1 to cppm files and contain the actual exported content. Files directly in `traits/` are the canonical headers that may be included by multiple partitions.
