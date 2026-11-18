---
layout: post
title: "Adding Rust to an Existing CMake C++ Project"
permalink: rust-in-cmake-cpp-project
tags: Rust C++ CMake
---

How to put Rust into a large C/C++ codebase without rewriting it: one cargo workspace, one static library, one CMake target, and the handful of traps you hit along the way.

---

**TL;DR**

- Put every Rust crate that C++ needs behind **one umbrella `staticlib` crate**. Never link two Rust static libraries into one binary.
- Build it with a plain `add_custom_target` that runs `cargo build`, and wrap the result in an `INTERFACE` library. C++ targets then just `target_link_libraries(... rust_bridge)`.
- Use [cxx](https://cxx.rs) when both sides are C++ and Rust. Use [cbindgen](https://github.com/mozilla/cbindgen) when you need a plain C API.
- Add one include root (`<target-dir>/cxxbridge`), declare `BYPRODUCTS` for Ninja, and keep `RUSTFLAGS` in `.cargo/config.toml`, not in CMake.

---

## Context

I work on a Linux system daemon, a large C and C++ codebase built with CMake and shipped as DEB/RPM packages to a dozen distributions. New code kept landing in areas where C++ was a bad deal: parsing untrusted input, talking to D-Bus, walking the filesystem safely, async networking. We wanted to write those parts in Rust while keeping the existing build, packaging and release process.

This post is only about the plumbing: how cargo and CMake live in one build. How we moved real modules over one at a time, and how logging and crash reporting work across the language boundary, will be separate posts.

Everything below comes from a small demo project I built while writing this; the snippets are complete and compile on Linux and macOS.

## The shape

```
project/
├── CMakeLists.txt
├── Cargo.toml              # cargo workspace
├── .cargo/config.toml      # rustflags shared by CMake and by you
├── src/main.cpp            # existing C++ code
└── rust/
    ├── fsutil/             # a Rust crate exposed to C++
    ├── textutil/           # another one
    └── bridge/             # umbrella staticlib + its CMakeLists.txt
```

Feature crates are ordinary Rust libraries that also contain a `#[cxx::bridge]` module. The `bridge` crate contains no logic: it depends on all of them and is built as a `staticlib`. CMake sees exactly one `.a` file.

### Why one umbrella library

Every Rust `staticlib` carries its own copy of the Rust standard library and of every dependency it uses. Link two of them into one executable and you get either duplicate symbol errors or, worse, two silently different copies of the same crate (two global loggers, two `OnceLock`s that should have been one).

One umbrella crate gives you:

- one copy of `std` and of every shared dependency;
- one `cargo build` invocation, so dependencies compile once;
- one place to add a new crate.

If the workspace also produces Rust executables (we have a CLI and a desktop notifier), build them in the same cargo invocation with the same `--target-dir`. Common dependencies are then compiled once for everything.

## cxx or cbindgen?

Both are good; they solve different problems.

| | cxx | cbindgen |
|---|---|---|
| Direction | Rust ⇄ C++, both ways | Rust → C header |
| What you write | a `#[cxx::bridge]` module that declares the interface | `extern "C"` functions with `#[no_mangle]`, `#[repr(C)]` types |
| Strings, vectors | `rust::String`, `rust::Vec`, `rust::Str`, `CxxString`, `CxxVector` | raw pointers + lengths, you manage ownership |
| Errors | `Result<T>` becomes a C++ exception (`rust::Error`) | error codes / out-params, by hand |
| `unsafe` at the boundary | none in your code | every function |
| C consumers | no | yes |

Our consumers were all C++, and we were tired of hand-written ownership rules at boundaries, so we went with **cxx**. I would pick **cbindgen** if the library had to be callable from C, from another language through a C ABI, or shipped as a stable SDK.

## Step by step

### 1. The workspace

`Cargo.toml` at the repository root:

```toml
[workspace]
resolver = "2"
members = ["rust/bridge"]

[profile.release]
debug = true        # keep DWARF for crash symbolication; strip later when packaging
lto = "thin"        # thin LTO keeps debug info usable, fat LTO tends to drop it
panic = "unwind"    # required if you ever want catch_unwind
```

`.cargo/config.toml`:

```toml
[build]
rustflags = ["-C", "force-frame-pointers=yes"]
```

Why a config file and not `RUSTFLAGS=...` in CMake: cargo includes the flags in its fingerprint. If CMake builds with one set of flags and you then run `cargo test` or `cargo clippy` by hand with another, cargo rebuilds the whole dependency tree, both times. With the config file, every invocation agrees.

(Frame pointers are there for crash reports; that is a topic for another post. Drop them if you don't need them.)

### 2. A feature crate

`rust/fsutil/Cargo.toml`:

```toml
[package]
name = "fsutil"
version = "0.1.0"
edition = "2021"

[dependencies]
cxx = "1.0"

[build-dependencies]
cxx-build = "1.0"
```

`rust/fsutil/build.rs` generates the C++ side of the bridge and compiles it into the crate:

```rust
fn main() {
    cxx_build::bridge("src/lib.rs").compile("fsutil");
    println!("cargo:rerun-if-changed=src/lib.rs");
}
```

`rust/fsutil/src/lib.rs`:

```rust
#[cxx::bridge(namespace = "fsutil")]
mod ffi {
    extern "Rust" {
        fn read_file(path: &str) -> Result<String>;
        fn mkdir_p(path: &CxxString) -> bool;
    }
}

fn read_file(path: &str) -> std::io::Result<String> {
    std::fs::read_to_string(path)
}

fn mkdir_p(path: &cxx::CxxString) -> bool {
    path.to_str()
        .map(|p| std::fs::create_dir_all(p).is_ok())
        .unwrap_or(false)
}
```

A second crate, `textutil`, shows shared structs and vectors:

```rust
#[cxx::bridge(namespace = "textutil")]
mod ffi {
    struct Stats {
        lines: usize,
        words: usize,
    }

    extern "Rust" {
        fn stats(text: &str) -> Stats;
        fn top_words(text: &str, n: usize) -> Vec<String>;
    }
}

fn stats(text: &str) -> ffi::Stats {
    ffi::Stats {
        lines: text.lines().count(),
        words: text.split_whitespace().count(),
    }
}

fn top_words(text: &str, n: usize) -> Vec<String> {
    let mut counts = std::collections::HashMap::new();
    for w in text.split_whitespace() {
        *counts.entry(w.to_lowercase()).or_insert(0usize) += 1;
    }
    let mut v: Vec<_> = counts.into_iter().collect();
    v.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)));
    v.into_iter().take(n).map(|(w, _)| w).collect()
}
```

Notice there is no `unsafe` and no manual memory management anywhere: `Stats` is a real struct on both sides, `Vec<String>` arrives in C++ as `rust::Vec<rust::String>`.

### 3. The umbrella crate

`rust/bridge/Cargo.toml`:

```toml
[package]
name = "rust_bridge"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["staticlib"]

[dependencies]
fsutil = { path = "../fsutil" }
textutil = { path = "../textutil" }
```

`rust/bridge/src/lib.rs`:

```rust
// Force the linker to keep every re-exported crate, including its cxx glue.
extern crate fsutil;
extern crate textutil;
```

Yes, `extern crate` in 2021 edition. A dependency that nothing references from the umbrella crate can be dropped, and its exported functions vanish from the archive. Naming the crate here is the cheapest way to keep it.

### 4. CMake

`rust/bridge/CMakeLists.txt`:

```cmake
set(CARGO_TARGET_DIR ${CMAKE_BINARY_DIR}/cargo)

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CARGO_PROFILE_FLAG "")
    set(CARGO_PROFILE_DIR debug)
else()
    set(CARGO_PROFILE_FLAG --release)
    set(CARGO_PROFILE_DIR release)
endif()

set(RUST_BRIDGE_LIB
    ${CARGO_TARGET_DIR}/${CARGO_PROFILE_DIR}/${CMAKE_STATIC_LIBRARY_PREFIX}rust_bridge${CMAKE_STATIC_LIBRARY_SUFFIX})

# Always run cargo; it has its own incremental build and returns fast when nothing changed.
add_custom_target(rust_bridge_cargo
    COMMAND cargo build ${CARGO_PROFILE_FLAG}
            --manifest-path ${CMAKE_SOURCE_DIR}/Cargo.toml
            --target-dir ${CARGO_TARGET_DIR}
            -p rust_bridge
    BYPRODUCTS ${RUST_BRIDGE_LIB}
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    COMMENT "cargo build (rust_bridge)"
    VERBATIM)

add_library(rust_bridge INTERFACE)
add_dependencies(rust_bridge rust_bridge_cargo)

# One root for everything cxx generates: "rust/cxx.h" and "<crate>/src/<file>.rs.h".
target_include_directories(rust_bridge INTERFACE ${CARGO_TARGET_DIR}/cxxbridge)

# System libraries Rust's std needs; get the exact list with:
#   cargo rustc -p rust_bridge -- --print native-static-libs
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(RUST_NATIVE_LIBS pthread dl m)
endif()

target_link_libraries(rust_bridge INTERFACE ${RUST_BRIDGE_LIB} ${RUST_NATIVE_LIBS})
```

Top-level `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.20)
project(demo CXX)
set(CMAKE_CXX_STANDARD 17)

add_subdirectory(rust/bridge)

add_executable(demo src/main.cpp)
target_link_libraries(demo PRIVATE rust_bridge)
```

That is all a C++ target needs to know about Rust: one name in `target_link_libraries`.

### 5. Calling Rust from C++

```cpp
#include <iostream>

#include "rust/cxx.h"           // rust::Error is only declared here
#include "fsutil/src/lib.rs.h"
#include "textutil/src/lib.rs.h"

int main(int argc, char** argv) {
    if (argc < 2) {
        std::cerr << "usage: demo <file>\n";
        return 2;
    }
    try {
        rust::String text = fsutil::read_file(argv[1]);
        textutil::Stats s = textutil::stats(text);
        std::cout << s.lines << " lines, " << s.words << " words\n";
        for (const rust::String& w : textutil::top_words(text, 3))
            std::cout << "  " << std::string(w) << '\n';
    } catch (const rust::Error& e) {
        std::cerr << "error: " << e.what() << '\n';
        return 1;
    }
}
```

```
$ cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
$ cmake --build build
$ ./build/demo notes.txt
2 lines, 8 words
  the
  dog
  end
$ ./build/demo missing.txt
error: No such file or directory (os error 2)
```

The `io::Error` from Rust became a C++ exception with the original message. On an unchanged tree, `cmake --build` finishes in well under a second: cargo runs, sees nothing to do, and Ninja skips the link.

## Traps

These are the ones that cost me real time.

**Generated header names collide.** cxx names the generated header after the source file, not the crate: `src/lib.rs` becomes `lib.rs.h` in every crate. If you add `cxxbridge/fsutil/src` and `cxxbridge/textutil/src` as include directories, `#include "lib.rs.h"` silently picks whichever comes first. Add only the `cxxbridge` root and include `"<crate>/src/<file>.rs.h"`. As a bonus, adding a crate no longer touches CMake at all.

**`rust::Error` is not in the generated header.** A function returning `Result<T>` throws `rust::Error`, but the generated header doesn't declare that class. `#include "rust/cxx.h"` wherever you catch it.

**Panics abort.** A panic that reaches a cxx boundary aborts the process; it is not turned into an exception. Return `Result` for anything that can fail, and treat a panic as a bug, the same way you treat a segfault in C++. If a library you call may panic and you must survive it, wrap the call in `std::panic::catch_unwind`, which also needs `panic = "unwind"` in the profile.

**Ninja wants `BYPRODUCTS`.** Without it, a clean Ninja build fails with "missing and no known rule to make it" for the `.a`, because nothing claims to produce that file. With it, Ninja also relinks C++ binaries when the archive changes, which is what you want.

**Link order and system libraries.** Rust's `std` needs a few system libraries. On Linux that is usually `pthread`, `dl` and `m`. On macOS the C++ driver already links what is needed. Don't guess; ask cargo:

```
$ cargo rustc -p rust_bridge --release -- --print native-static-libs
note: native-static-libs: -lc++ -lSystem -lc -lm
```

Your own crates add to this list when they link C libraries (we have `libdbus` and `libcryptsetup` there), so rerun it when dependencies change.

**macOS deployment target warnings.** On macOS you may see `object file was built for newer 'macOS' version than being linked`. The C++ code that cxx-build compiles uses `MACOSX_DEPLOYMENT_TARGET`, and CMake uses `CMAKE_OSX_DEPLOYMENT_TARGET`. Set both to the same value.

**Debug vs Release.** Map `CMAKE_BUILD_TYPE` to cargo's profile explicitly, as above. Multi-config generators (Visual Studio, Xcode, Ninja Multi-Config) need generator expressions instead; we only ship single-config Linux builds, so I skipped that.

## What about Corrosion?

[Corrosion](https://github.com/corrosion-rs/corrosion) is a CMake module that imports cargo crates as CMake targets, and it handles multi-config generators, cross-compilation targets and cxx integration for you. If you are starting fresh, look at it first. We stayed with a plain custom target because it is twenty lines, does exactly one thing, and nobody on the team has to learn another layer when a build breaks.

## Tests

The Rust side is tested with plain `cargo test` in each crate, without CMake. C++ tests link `rust_bridge` like any other target, so they exercise the real boundary: the conversions, exceptions and ownership rules. In CI we run both, plus `cargo clippy -D warnings`.

## Result

Three years later, every new module in that daemon starts in Rust: device checks, D-Bus clients, filesystem helpers, IPC, a gRPC server. The C++ main loop calls into them through this one library. The build change itself was about forty lines of CMake and has barely changed since. The work was in deciding where to cut the boundaries, which is what the next post is about.
