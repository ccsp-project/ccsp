# lib_ccsp_base: Rationale, Design, and API Guide

**Version:** 1.0-draft  
**Status:** Design Document  
**License:** CC-BY-SA-4.0 (this document), MIT (code)

---

## Table of Contents

- [Part I: Rationale](#part-i-rationale)
  - [Why This Library Exists](#why-this-library-exists)
  - [The Problem With The Status Quo](#the-problem-with-the-status-quo)
  - [What We Are Not](#what-we-are-not)
  - [The Failure Mode We Are Preventing](#the-failure-mode-we-are-preventing)
  - [Licensing](#licensing)
- [Part II: Design](#part-ii-design)
  - [Principles](#principles)
  - [The Platform Table](#the-platform-table)
  - [The Shim Philosophy](#the-shim-philosophy)
  - [Integer Types](#integer-types)
  - [String Model](#string-model)
  - [Memory Model](#memory-model)
  - [Error Model](#error-model)
  - [Threading Model](#threading-model)
  - [The Dependency Rule](#the-dependency-rule)
  - [What Is Not In lib_ccsp_base](#what-is-not-in-lib_ccsp_base)
- [Part III: API Guide Table of Contents](#part-iii-api-guide-table-of-contents)

---

# Part I: Rationale

## Why This Library Exists

Every serious C program needs the same small set of things before it can do anything useful:

- Integer types whose sizes are known and stable across platforms
- A way to represent strings that does not require null termination
- A memory allocator interface that can be replaced for testing or embedded use
- Linked lists and hash tables that do not allocate separately for every node
- A monotonic clock that does not jump when NTP adjusts the system time
- Cryptographically secure random bytes
- Atomic operations for lock-free reference counting and once-initialization
- Structured logging that produces machine-readable output
- A consistent error type that carries location information

Every C program that needs these things currently either reimplements them from scratch, pulls in a large framework library that also brings an object system and an event loop and an HTTP client and a bookmark file parser, or uses POSIX interfaces directly in ways that are subtly wrong on at least one platform in the matrix.

lib_ccsp_base provides exactly these things and nothing else.

## The Problem With The Status Quo

### GLib

GLib is the most widely deployed C utility library. It gets the data structures mostly right. Then it adds GObject (a runtime type system in C that requires macro soup to use and generates code impossible to audit), GMainLoop (a reimplementation of select-based event dispatch that is not compatible with any other event loop), GIOChannel (a buffered I/O abstraction that is the third I/O abstraction in the same library), GThread (a threading abstraction that was bolted on after the fact), GRegex, GKeyFile, GBookmarkFile, GDateTime, GDBus, GNetwork, GSettings, GApplication, GResource, and GDBus again in case you missed it the first time.

GLib is not a foundation library. It is an operating system that runs inside your process. Because GTK depends on it, everything in the Linux desktop stack depends on a library that contains a bookmark file parser. This is not a design. It is an accumulation.

The specific technical failures that matter for us:

- GObject cannot be audited meaningfully. The macro-generated code obscures control flow and ownership in ways that make security review impractical.
- GMainLoop is not compatible with lib_ccsp_event, kqueue, epoll, or any other event loop. Applications that need both (everything that uses GTK and also does network I/O) run two event loops in one process.
- GHashTable allocates a separate heap node for every entry. This is cache-hostile and means every insert is a malloc.
- GLib's error type (GError) is coupled to GObject's memory management model and cannot be used without pulling in GObject.
- There is no deletion condition on anything. Features added in 1996 are present in 2024 unchanged because everything depends on them.

### Rolling Your Own

The alternative to GLib is rolling your own. This is what most projects do. The result is:

- A linked list implementation that uses separate allocation for nodes (cache-hostile, allocation-failure paths everywhere).
- A hash table that uses chaining with separate allocation (same problems).
- `typedef int bool` that conflicts with the next library's `typedef int bool`.
- `#define TRUE 1` that conflicts with the next library's `#define TRUE 1`.
- Integer types defined as `typedef unsigned int uint32` that are wrong on platforms where `unsigned int` is not 32 bits.
- A logging system that writes to stderr with fprintf and provides no structured output.
- Error handling through global errno with no location information.
- No monotonic clock; everything uses `gettimeofday()` which jumps.
- No secure random; everything uses `rand()` seeded with `time()` or reads `/dev/urandom` directly without handling the blocking-before-seeded case.

These are not exotic failure modes. They are the default outcome of C development without a foundation library. The CVE corpus documents the consequences.

## What We Are Not

**We are not GLib.** We do not have an object system. We do not have an event loop. We do not have an I/O abstraction. We do not have a networking layer. We do not have application lifecycle management. We do not have a configuration system tied to any specific backend. We do not have a bookmark file parser.

If you need an event loop, use libevent — it is battle-tested and correct. We wrap it. If you need TLS, we wrap a real TLS implementation. We use what is good and wrap it behind stable APIs. We do not rewrite working code for the sake of rewriting it.

**We are not libc.** We do not wrap what is in standard C17. We assume you have a modern compiler on a modern machine. We wrap the allocator through an interface so it can be replaced for testing and embedded use. We do not reimplement the standard library.

**We think stdio is broken.** `FILE *`, `printf`, `gets`, `scanf` — the entire `<stdio.h>` model is fundamentally wrong. We do not wrap it. We replace it with a better buffered I/O abstraction.

**We are not a portability layer for the whole Unix API.** We provide portability for the specific things where platform divergence causes bugs in application code: integer types, atomic operations, monotonic time, secure random. We do not abstract sockets, signals, processes, or file I/O.

**We are not a framework.** We do not have an opinion about how you structure your application. We do not require you to write your code in a particular style. We provide tools. You use them or you don't.

## Opinions

Forty years of CVEs have earned us these opinions:

| Broken | Why | What we do instead |
|--------|-----|-------------------|
| Null-terminated strings | Buffer overflows, no O(1) length, embedded nulls impossible | Pointer + out-of-band length (`ccsp_str_t`) |
| `errno` | Global state, thread hazards, no location info | `ccsp_err_t` with code, domain, message, file, line |
| `<stdio.h>` | `gets`, format strings, buffering semantics, error handling | Buffered I/O abstraction |
| `select()` | `FD_SETSIZE` is a trap, bit-array semantics | Wrap libevent |
| `gethostbyname()` | Global state, no IPv6, no DNSSEC | Async DNS with modern resolver |
| `rand()` / `random()` | Not random, global state, dangerous near security code | `ccsp_random_bytes()` from OS entropy |
| `gettimeofday()` | NTP jumps, not monotonic, useless for intervals | `ccsp_monotonic_ns()` |
| `getopt()` | No types, no env fallback, help drifts from code | Declarative CLI with source tracking |
| Signals | Not broken, but nearly impossible to use correctly | Minimal surface, self-pipe trick |
| Text encodings | "Current locale" is chaos | UTF-8 or raw bytes, nothing else |
| IPv4-first | It is 2026 | IPv6 native, IPv4 as fallback |
| CA-only TLS validation | CAs are a cartel, frequently compromised | DANE (TLSA) wherever possible, CA as fallback |
| ASCII-only interfaces | It is 2026 | UTF-8 everywhere, your terminal can handle it |
| 32-bit assumptions | It is 2026 | Assume 64-bit, try not to break 32-bit, try not to break 16-bit at lower levels |
| `setjmp`/`longjmp` | Skips cleanup, violates resource management | Explicit error returns with `ccsp_err_t` |
| `alloca` / VLAs | Stack overflow traps | Heap via allocator interface |
| Floating point for money/time | Floats lie | Integers. Nanoseconds for time, cents for money. |
| Time zones | Chaos | UTC internally, local time only at display boundary |
| Ownership semantics | Implicit ownership causes use-after-free | Always explicit, every API documents who frees what |
| Sensitive memory | Lingers after free, gets swapped, shows up in dumps | `ccsp_secure_free()` zeros before release |
| Blocking I/O | Blocks the world, hard to timeout, hard to cancel | Non-blocking default, blocking is opt-in |
| `goto` for cleanup | — | Actually correct in C. Use it. |
| Autotools | m4 macros, generated shell scripts, inscrutable errors | Pretend it doesn't exist. Use CMake or plain Makefiles. |

We do not have opinions about tabs vs spaces, brace style, editor choice, build system, or application architecture. Those are your problems.

## The Failure Mode We Are Preventing

The CVE corpus -- our public database of classified C systems vulnerabilities -- shows that 87% of security vulnerabilities in C systems software fall into 12 patterns. Three of those patterns are directly prevented by using lib_ccsp_base correctly:

**Pattern: implicit integer size assumption.**
The code assumes `int` is 32 bits, or `long` is 64 bits, or `size_t` is the same size as a pointer. On some platform in the portability matrix it isn't. The resulting truncation or overflow is a vulnerability. `ccsp_base.h` with `uint32_t`, `int64_t`, and friends eliminates this pattern. The fix is `#include <stdint.h>` (or our shim before C99 is available on your platform) and never writing `int` when you mean `uint32_t`.

**Pattern: non-reentrant global state.**
The code calls `gethostbyname()`, `strtok()`, `gmtime()`, `rand()`, or any other function with hidden global state from a context where concurrent calls are possible. lib_ccsp_base's equivalents for the things it covers -- the monotonic clock, the secure RNG, the hash table, the allocator -- have no hidden global state. Everything that needs state takes it as an explicit parameter.

**Pattern: ignored error return.**
The code calls a function that can fail, ignores the return value, and uses the output as if the call succeeded. lib_ccsp_base functions that can fail return a typed error code and take a `ccsp_err_t *` output parameter. The error carries the call site location (`__FILE__`, `__LINE__`) and a human-readable message. Static analysis with `warn_unused_result` on every fallible function catches ignoring the return value at compile time.

The remaining nine patterns are prevented by libraries that build on lib_ccsp_base (lib_ccsp_bio, lib_ccsp_connect, lib_ccsp_lineproto, lib_ccsp_ocl). They are documented in the CVE corpus and in those libraries' rationale documents.

## Licensing

Documentation: CC-BY-SA-4.0. Code: MIT.

---

# Part II: Design

## Principles

### 1. Never own what someone else should own

Every line of code in lib_ccsp_base is a line we have to audit, maintain, test, and defend in a CVE. If a well-maintained widely-deployed library or standard owns a problem, our job is to have a clean interface to it and stay out of its way.

This is why we use the platform's allocator rather than writing our own. This is why we call `clock_gettime(CLOCK_MONOTONIC)` rather than implementing a monotonic clock. This is why every shim has a deletion condition: the shim exists until the standard or platform provides the right thing, and then the shim is deleted.

### 2. Caller provides storage; library never allocates hidden

Every function that writes output takes a caller-provided buffer or pointer. The library does not call malloc internally except for opaque object allocation (hash tables, error contexts) where the caller explicitly requests object creation and explicitly destroys it.

This has three consequences:

- Allocation failure paths are in the caller's control, not scattered through library internals.
- The library has no hidden memory that needs to be freed. Object lifetime is explicit.
- Embedded targets with custom allocators or static memory pools can use lib_ccsp_base without modification.

### 3. Explicit is better than implicit

Every piece of state that a function needs is passed as a parameter. No global variables. No thread-local state except where the platform requires it (errno). No "current allocator" global. The platform table is passed at initialization and stored in the context; it is not a global.

This makes functions testable in isolation. A test creates a mock allocator, passes it to `ccsp_hash_create()`, and verifies that allocation failures are handled correctly. Without this principle the test has to intercept malloc, which is fragile and platform-specific.

### 4. One ifdef location per platform concern

Platform-specific ifdefs belong in one place and one place only. The integer type shims are in `ccsp_base.h`. The platform capability detection is in `ccsp_platform.h`. The atomic operation shims are in `ccsp_atomic.h`. The endianness utilities are in `ccsp_endian.h`.

No other file in lib_ccsp_base contains `#ifdef __linux__` or `#ifdef _WIN32` or `#ifdef __APPLE__`. If a platform concern requires handling it goes in the appropriate shim header. If no appropriate shim header exists one is created.

This discipline makes platform bugs findable. When something breaks on AIX you look in `ccsp_platform.h`, `ccsp_atomic.h`, and the relevant backend file. You do not grep the entire codebase for `_AIX`.

### 5. Every shim has a deletion condition

Every piece of code that exists to work around a platform limitation or missing standard has a comment at the top:

```c
/*
 * TEMPORARY SHIM -- delete when [condition]
 * See: https://tracker/issues/NNN
 * Last reviewed: [date]
 */
```

The tracker issue is always open. It is reviewed on a schedule. The shim never becomes permanent by neglect. You must actively decide to keep it.

When the condition is met -- C99 is available on all minimum target platforms, `getrandom(2)` is stable, `clock_gettime` is in POSIX -- the shim is deleted. The deletion is a merge, not a deprecation. The old code is gone.

### 6. Test vectors from day one, machine-readable

Every component has a machine-readable test vector file. Known-answer tests against published standards (NIST, IETF, Unicode Consortium) are in the test suite and run on every build. A component that fails a known-answer test does not ship.

The test vectors are version-controlled alongside the code. Adding a platform means the new platform must pass all existing test vectors before the port is accepted.

### 7. UTF-8 is the only text encoding

Inside lib_ccsp_base and every library that depends on it, text is UTF-8 or it is bytes. There is no third option. Functions that accept text accept `const uint8_t *` with an explicit length. They validate UTF-8 strictly: no overlong encodings, no surrogates, no codepoints above U+10FFFF.

If you have text in a different encoding it is bytes until you convert it to UTF-8 using lib_ccsp_charconv. The conversion happens at the boundary. Inside lib_ccsp_base there is nothing to configure, nothing to detect, and nothing to get wrong.

This decision was made once and applies everywhere. It is not revisited.

## The Platform Table

The platform table is the one mechanism by which lib_ccsp_base adapts to its environment. It is passed at initialization and stored in the base context. It contains function pointers for the operations that vary by platform or deployment context.

```c
typedef struct {
    /* memory */
    void *(*malloc)(size_t);
    void *(*realloc)(void *, size_t);
    void  (*free)(void *);

    /* time */
    uint64_t (*monotonic_ns)(void);

    /* random */
    int (*getrandom)(uint8_t *, size_t);

    /* threading -- NULL for single-threaded builds */
    int  (*mutex_init)(void **);
    int  (*mutex_lock)(void *);
    int  (*mutex_unlock)(void *);
    void (*mutex_destroy)(void *);

    /* logging backend */
    void (*log_emit)(void *ctx,
                     int level,
                     const char *file,
                     int line,
                     const char *msg,
                     const char **keys,
                     const char **vals,
                     size_t kv_count);
    void *log_ctx;

} ccsp_platform_t;
```

`ccsp_platform_default()` returns a platform table that uses the system allocator, `clock_gettime(CLOCK_MONOTONIC)` or its platform equivalent, `getrandom(2)` or its platform equivalent, pthreads mutexes, and a stderr log backend.

A test replaces `malloc`/`free` with a tracking allocator to verify allocation failure handling. An embedded target replaces `malloc`/`free` with its pool allocator and `getrandom` with its hardware RNG. The library does not change. The platform table changes.

## The Shim Philosophy

Shims exist in four categories:

**Type shims:** Define standard types before the standard provides them. `ccsp_base.h` defines `uint8_t`, `uint16_t`, `uint32_t`, `uint64_t`, `int8_t`, `int16_t`, `int32_t`, `int64_t`, `bool`, `true`, `false`. When C99 is available on all minimum target platforms the shim is deleted and replaced with `#include <stdint.h>` and `#include <stdbool.h>`.

**Syscall shims:** Wrap platform-specific system calls behind a portable interface. `ccsp_random.c` implements `ccsp_getrandom()` using `getrandom(2)` on Linux, `getentropy(2)` on OpenBSD, `arc4random_buf()` on FreeBSD and macOS, `BCryptGenRandom()` on Windows, and `/dev/urandom` with blocking-until-seeded detection everywhere else. When `stdc_secure_random()` is standardized (proposed for C2x) the shim is deleted.

**Behavior shims:** Normalize platform-specific behavior differences. `ccsp_signal.c` wraps `sigaction()`, `sigvec()`, and `signal()` behind a single interface with consistent restart semantics. When all minimum target platforms provide `sigaction()` with `SA_RESTART` the shim is deleted.

**Feature shims:** Implement missing features that are not yet standard. `ccsp_atomic.h` provides atomic operations using GCC `__sync_*` builtins, MSVC `Interlocked*` functions, or fallback mutex-based implementations. When C11 `<stdatomic.h>` is available on all minimum target platforms the shim is deleted and `<stdatomic.h>` is used directly.

Every shim is in a file named `ccsp_<thing>_shim.c` or `ccsp_<thing>_shim.h`. The naming makes them findable. The tracker issue makes them temporary. The deletion condition makes the temporariness enforceable.

## Integer Types

Use `ccsp_base.h` integer types everywhere. Never use `int` when you mean `uint32_t`. Never use `long` when you mean `int64_t`. Never use `char` when you mean `uint8_t` for byte data.

The rule has one exception: loop counters and array indices that are bounded by small constants can use `int`. Everything else is explicitly sized.

The reason is not pedantry. The reason is the portability matrix. The matrix includes platforms where `int` is 16 bits, where `long` is 32 bits on a 64-bit OS, and where pointer-sized arithmetic requires `uintptr_t`. Code that uses implicit sizes is code that has latent bugs on some platform in the matrix.

The test suite includes a platform conformance check that validates integer sizes on every new platform. If your platform has `sizeof(int) != 4` the conformance check tells you before the first application bug does.

## String Model

All strings in lib_ccsp_base are length-prefixed byte arrays. The canonical type is:

```c
typedef struct {
    const uint8_t  *data;   /* pointer to bytes, NOT null-terminated */
    size_t          len;    /* length in bytes, NOT codepoints */
} ccsp_str_t;
```

Text strings (as opposed to binary byte arrays) are UTF-8. The `len` field is always byte length. Codepoint count requires `ccsp_utf8_codepoint_count()` which is O(n) and documented as such.

Null termination is available at API boundaries where C convention requires it. `ccsp_str_cstr()` returns a null-terminated copy (caller frees). `CCSP_STR_LIT("hello")` creates a `ccsp_str_t` from a string literal using `sizeof` to get the length without calling `strlen`.

String literals in lib_ccsp_base source use `CCSP_STR_LIT`. `strlen` is not called in lib_ccsp_base except in `ccsp_str_from_cstr()` which converts a null-terminated string to a `ccsp_str_t`.

## Memory Model

### Allocation

All allocation goes through the platform table. The library calls `platform->malloc`, `platform->realloc`, and `platform->free`. Never `malloc`, `realloc`, or `free` directly.

Functions that allocate return NULL and set `ccsp_err_t` on allocation failure. They do not abort. They do not call a registered failure handler. The caller handles the failure.

### Zeroing

Memory containing secrets is zeroed before free. `ccsp_free_secure()` zeros the memory before calling `platform->free`. The zeroing is not optimized away: we use `memset_s()` if available, a volatile write loop otherwise, and we have a test that verifies the memory was actually zeroed (by catching the optimizer removing it).

Memory that does not contain secrets uses `platform->free` directly.

### Ownership

Ownership is explicit in function signatures and documentation. Every function that returns a pointer documents who owns it. Functions that return pointers into existing buffers (like `ccsp_hash_get()`) document that the pointer is valid only as long as the hash table is not modified. Functions that return allocated memory document that the caller must free it.

We do not use reference counting in lib_ccsp_base. Reference-counted objects belong in the libraries built on top of lib_ccsp_base. lib_ccsp_base objects have single owners.

## Error Model

```c
typedef struct {
    int32_t     code;       /* library-specific error code, 0 = success */
    const char *domain;     /* string literal identifying the library */
    const char *message;    /* human-readable description */
    const char *file;       /* __FILE__ at error site */
    int32_t     line;       /* __LINE__ at error site */
} ccsp_err_t;
```

Rules:

- Every function that can fail takes `ccsp_err_t *err` as its last parameter.
- `err` may be NULL if the caller does not need error details (for simple predicates).
- On success the function returns the documented success value (0, non-NULL pointer, etc.) and does not modify `*err`.
- On failure the function returns the documented failure value (negative int, NULL pointer, etc.) and sets `*err`.
- `ccsp_err_t` values are stack-allocated by the caller. The library never heap-allocates an error.
- `message` points to a string literal or a static buffer. It is never heap-allocated. It is valid for the lifetime of the process.
- `file` and `line` identify where the error was detected, not where the function was called from. Use `ccsp_err_wrap()` to add call-site context when propagating errors up the call stack.

The `__FILE__`/`__LINE__` in the error is from `CCSP_ERR()`, the macro used to set errors:

```c
#define CCSP_ERR(err, code_, domain_, message_)         \
    ((err) ? ((err)->code    = (code_),                  \
              (err)->domain  = (domain_),                \
              (err)->message = (message_),               \
              (err)->file    = __FILE__,                 \
              (err)->line    = __LINE__, (code_)) : (code_))
```

The macro returns the error code so it can be used in a return statement:

```c
if (len > MAX_LEN)
    return CCSP_ERR(err, CCSP_ERR_TOO_LARGE, "ccsp.base",
                    "input exceeds maximum length");
```

## Threading Model

lib_ccsp_base functions are thread-safe if and only if documented as thread-safe. Most functions are thread-safe because they have no global state. Functions that operate on a single object (hash table, string, buffer) are not thread-safe for concurrent operations on the same object unless documented otherwise.

The threading primitives (mutexes, atomic operations, once-initialization) are in lib_ccsp_base and are used by the libraries built on top of it. lib_ccsp_base itself uses them only for the initialization of the once-initialized global state (the signal pipe, if present).

The platform table's mutex functions may be NULL for single-threaded builds. Functions that use mutexes check for NULL and skip locking in single-threaded mode.

## The Dependency Rule

lib_ccsp_base has no dependencies except:

- The C standard library (libc)
- The platform table (provided by the caller at initialization)
- Optionally, platform-specific system libraries (libpthread for the mutex shim, nothing else)

lib_ccsp_base does not depend on lib_ccsp_event. It does not depend on libssl or any crypto library. It does not depend on GLib. It does not depend on any library that has not existed since at least 1990.

This is the dependency rule: **lib_ccsp_base's dependencies fit on one line.**

Libraries built on lib_ccsp_base (lib_ccsp_event, lib_ccsp_bio, lib_ccsp_connect, lib_ccsp_ocl) depend on lib_ccsp_base. lib_ccsp_base does not depend on them. The dependency graph is a DAG with lib_ccsp_base at the root. No cycles. No surprises.

## What Is Not In lib_ccsp_base

The following things are explicitly not in lib_ccsp_base and never will be:

**An event loop.** Use libevent. We wrap it.

**Networking and TLS.** Use existing implementations. We wrap them.

**Cryptography.** Use existing implementations. We wrap them.

**Regular expressions.** Use PCRE2 or similar. We wrap it.

**JSON or TOML parsing.** Separate libraries, built on lib_ccsp_base.

**An object system.** C has structs. Structs are sufficient. If you need inheritance, use function pointers and explicit vtables.

**A plugin or module system.** The platform provides dynamic linking. We do not build a plugin architecture on top of it.

**Application lifecycle management.** Your application manages its own lifecycle. We provide tools. We do not impose a structure.

lib_ccsp_base stays small. Everything else is either wrapped from good existing code or built as a separate library on top of lib_ccsp_base.

---

# Part III: API Guide Table of Contents

## Volume 1: Foundations

### Chapter 1: Getting Started

- 1.1 Building lib_ccsp_base
  - 1.1.1 Requirements
  - 1.1.2 CMake configuration
  - 1.1.3 Platform detection output
  - 1.1.4 Verifying the build with the conformance suite
- 1.2 The Platform Table
  - 1.2.1 `ccsp_platform_default()` — the standard configuration
  - 1.2.2 Replacing the allocator
  - 1.2.3 Replacing the log backend
  - 1.2.4 Single-threaded builds
  - 1.2.5 Embedded targets with custom memory
- 1.3 The Base Context
  - 1.3.1 `ccsp_ctx_create()` — creating a context
  - 1.3.2 `ccsp_ctx_destroy()` — cleaning up
  - 1.3.3 Context vs global state — why there is no global state
- 1.4 Error Handling
  - 1.4.1 `ccsp_err_t` — the error type
  - 1.4.2 `CCSP_ERR()` — setting an error
  - 1.4.3 `ccsp_err_wrap()` — adding call-site context
  - 1.4.4 `ccsp_err_clear()` — resetting an error
  - 1.4.5 Passing NULL for err — when you don't need details
  - 1.4.6 Error codes — the complete list
  - 1.4.7 Pattern: propagating errors up the call stack

### Chapter 2: Types and Portability

- 2.1 Integer Types
  - 2.1.1 The type table — `uint8_t` through `uint64_t` and signed equivalents
  - 2.1.2 `uintptr_t` and `ptrdiff_t` — pointer-sized arithmetic
  - 2.1.3 `size_t` — the right type for sizes and counts
  - 2.1.4 `bool`, `true`, `false` — from `<stdbool.h>` or the shim
  - 2.1.5 What to do when the compiler doesn't support C99 types
  - 2.1.6 Platform conformance — what the test suite checks
- 2.2 Byte Order
  - 2.2.1 Why your integer casts are wrong on SPARC
  - 2.2.2 `ccsp_read_be16()`, `ccsp_read_be32()`, `ccsp_read_be64()` — big-endian reads
  - 2.2.3 `ccsp_write_be16()`, `ccsp_write_be32()`, `ccsp_write_be64()` — big-endian writes
  - 2.2.4 `ccsp_read_le*()`, `ccsp_write_le*()` — little-endian variants
  - 2.2.5 Why these use byte-by-byte access rather than casts
  - 2.2.6 `CCSP_BIG_ENDIAN`, `CCSP_LITTLE_ENDIAN` — compile-time detection
  - 2.2.7 `ccsp_bswap16()`, `ccsp_bswap32()`, `ccsp_bswap64()` — byte swapping
- 2.3 Platform Detection
  - 2.3.1 OS detection macros — `CCSP_OS_LINUX`, `CCSP_OS_FREEBSD`, etc.
  - 2.3.2 Architecture detection — `CCSP_ARCH_X86`, `CCSP_ARCH_SPARC`, etc.
  - 2.3.3 `CCSP_STRICT_ALIGN` — platforms that bus-error on unaligned access
  - 2.3.4 Feature detection — `CCSP_HAS_CLOCK_GETTIME`, `CCSP_HAS_GETRANDOM`, etc.
  - 2.3.5 Adding a new platform — the checklist

### Chapter 3: UTF-8 and Strings

- 3.1 The UTF-8 Commitment
  - 3.1.1 Why UTF-8 and nothing else
  - 3.1.2 What this means for your code
  - 3.1.3 Handling legacy encodings — lib_ccsp_charconv at the boundary
- 3.2 `ccsp_str_t` — the string type
  - 3.2.1 Structure — `data` and `len`
  - 3.2.2 `CCSP_STR_LIT()` — string literals without `strlen`
  - 3.2.3 `ccsp_str_from_cstr()` — from null-terminated
  - 3.2.4 `ccsp_str_cstr()` — to null-terminated (allocates)
  - 3.2.5 `ccsp_str_eq()` — equality comparison
  - 3.2.6 `ccsp_str_dup()` — copying (allocates)
  - 3.2.7 `ccsp_str_free()` — freeing a dup'd string
- 3.3 UTF-8 Validation and Iteration
  - 3.3.1 `ccsp_utf8_validate()` — strict validation with error position
  - 3.3.2 What "strict" means — overlong encodings, surrogates, out-of-range
  - 3.3.3 `ccsp_utf8_decode()` — decode one codepoint, advance pointer
  - 3.3.4 `ccsp_utf8_encode()` — encode one codepoint to bytes
  - 3.3.5 `ccsp_utf8_next()` — advance without decoding
  - 3.3.6 `ccsp_utf8_prev()` — retreat without decoding
  - 3.3.7 `ccsp_utf8_charlen()` — byte length of next codepoint
  - 3.3.8 `ccsp_utf8_codepoint_count()` — count codepoints, O(n), use carefully
- 3.4 Pattern: Processing UTF-8 Text
  - 3.4.1 Iterating codepoints
  - 3.4.2 Splitting on ASCII delimiters safely
  - 3.4.3 Comparing strings case-insensitively (ASCII fast path)
  - 3.4.4 What not to do — the strlen/strcat antipatterns

## Volume 2: Data Structures

### Chapter 4: Memory and Allocation

- 4.1 The Allocator Interface
  - 4.1.1 `ccsp_malloc()`, `ccsp_realloc()`, `ccsp_free()` — platform table wrappers
  - 4.1.2 `ccsp_calloc()` — zeroing allocation
  - 4.1.3 `ccsp_free_secure()` — zeroing free for secrets
  - 4.1.4 `ccsp_strdup()` — string duplication
  - 4.1.5 Allocation failure — what the library does and what you must do
- 4.2 The Slab Allocator
  - 4.2.1 When to use slab allocation
  - 4.2.2 `ccsp_slab_t` — the slab type
  - 4.2.3 `ccsp_slab_create()` — creating a slab for a fixed object size
  - 4.2.4 `ccsp_slab_alloc()` — allocating from the slab
  - 4.2.5 `ccsp_slab_free()` — returning to the slab
  - 4.2.6 `ccsp_slab_destroy()` — freeing the entire slab
  - 4.2.7 Thread safety — the slab is not thread-safe, use one per thread
- 4.3 Memory Safety Patterns
  - 4.3.1 Checking for integer overflow before allocation
  - 4.3.2 `ccsp_size_add()`, `ccsp_size_mul()` — overflow-checked arithmetic
  - 4.3.3 The `ccsp_free_secure()` pattern for secrets
  - 4.3.4 Valgrind and AddressSanitizer annotations

### Chapter 5: Intrusive Linked Lists

- 5.1 Why Intrusive Lists
  - 5.1.1 The allocation problem with non-intrusive lists
  - 5.1.2 Cache performance — keeping data and links together
  - 5.1.3 The `container_of` pattern
- 5.2 Doubly-Linked Lists
  - 5.2.1 `ccsp_list_node_t` — the embedded node
  - 5.2.2 `ccsp_list_t` — the list head with sentinel
  - 5.2.3 `ccsp_list_init()` — initializing a list
  - 5.2.4 `ccsp_list_insert_head()`, `ccsp_list_insert_tail()` — O(1) insertion
  - 5.2.5 `ccsp_list_insert_before()`, `ccsp_list_insert_after()` — relative insertion
  - 5.2.6 `ccsp_list_remove()` — O(1) removal
  - 5.2.7 `ccsp_list_empty()` — emptiness check
  - 5.2.8 `ccsp_list_count()` — O(n) count (use sparingly)
  - 5.2.9 `CCSP_LIST_FOREACH()` — iteration macro
  - 5.2.10 `CCSP_LIST_FOREACH_SAFE()` — iteration safe for removal
  - 5.2.11 `CCSP_CONTAINER_OF()` — getting the containing struct
- 5.3 Singly-Linked Lists
  - 5.3.1 When to use singly vs doubly linked
  - 5.3.2 `ccsp_slist_node_t` and `ccsp_slist_t`
  - 5.3.3 Operations — insert, remove_head, foreach
- 5.4 Pattern: Embedding Lists in Application Structs

### Chapter 6: Hash Tables

- 6.1 Design Decisions
  - 6.1.1 Open addressing with linear probing — why not chaining
  - 6.1.2 Insertion-order preservation — the parallel order array
  - 6.1.3 Typed keys — why not `void *`
  - 6.1.4 Load factor and resize policy
- 6.2 Creating and Destroying
  - 6.2.1 `ccsp_hash_create()` — key type, initial capacity, options
  - 6.2.2 `ccsp_hash_destroy()` — freeing the table and interned keys
  - 6.2.3 `ccsp_hash_opts_t` — load factor, custom hash function
- 6.3 Operations
  - 6.3.1 `ccsp_hash_set()` — insert or update
  - 6.3.2 `ccsp_hash_get()` — lookup, returns NULL if absent
  - 6.3.3 `ccsp_hash_delete()` — remove, returns whether found
  - 6.3.4 `ccsp_hash_count()` — O(1) entry count
  - 6.3.5 `ccsp_hash_clear()` — remove all entries, keep allocation
- 6.4 Key Types
  - 6.4.1 `CCSP_HASH_KEY_STRING` — intern copy of null-terminated string
  - 6.4.2 `CCSP_HASH_KEY_BYTES` — intern copy of length-prefixed bytes
  - 6.4.3 `CCSP_HASH_KEY_UINT64` — integer key, no copy
  - 6.4.4 `CCSP_HASH_KEY_PTR` — pointer identity, no copy
  - 6.4.5 Key storage — how the intern pool works
- 6.5 Iteration
  - 6.5.1 `ccsp_hash_each()` — in insertion order, always
  - 6.5.2 `ccsp_hash_iter_t` — explicit iterator for interleaved operations
  - 6.5.3 Modifying during iteration — what is and is not safe
- 6.6 Thread Safety
  - 6.6.1 Hash tables are not thread-safe
  - 6.6.2 Read-only sharing after construction — safe
  - 6.6.3 Wrapping with a mutex — the pattern

### Chapter 7: Growable Buffers

- 7.1 `ccsp_buf_t` — the byte buffer
  - 7.1.1 Structure — data, len, capacity
  - 7.1.2 `ccsp_buf_init()` — initial allocation
  - 7.1.3 `ccsp_buf_destroy()` — freeing
  - 7.1.4 `ccsp_buf_append()` — appending bytes
  - 7.1.5 `ccsp_buf_append_str()` — appending a `ccsp_str_t`
  - 7.1.6 `ccsp_buf_reserve()` — ensuring capacity without committing
  - 7.1.7 `ccsp_buf_commit()` — committing reserved space
  - 7.1.8 `ccsp_buf_truncate()` — reducing length
  - 7.1.9 `ccsp_buf_clear()` — resetting to zero length
  - 7.1.10 `ccsp_buf_steal()` — taking ownership of the internal buffer
- 7.2 The Ring Buffer
  - 7.2.1 When to use ring buffers vs growable buffers
  - 7.2.2 `ccsp_ring_t` — caller-provided storage
  - 7.2.3 `ccsp_ring_init()` — initializing with a fixed-size backing array
  - 7.2.4 `ccsp_ring_readable()`, `ccsp_ring_writable()` — space queries
  - 7.2.5 `ccsp_ring_peek()`, `ccsp_ring_consume()` — zero-copy read
  - 7.2.6 `ccsp_ring_reserve()`, `ccsp_ring_commit()` — zero-copy write
  - 7.2.7 Wraparound — how the ring handles the boundary

## Volume 3: Platform Services

### Chapter 8: Time

- 8.1 The Monotonic Clock
  - 8.1.1 Why `time()` and `gettimeofday()` are wrong for intervals
  - 8.1.2 `ccsp_monotonic_ns()` — nanoseconds since unspecified epoch
  - 8.1.3 Why unsigned wraparound is defined and correct for elapsed time
  - 8.1.4 `ccsp_elapsed_ns()` — safe elapsed time computation
  - 8.1.5 `ccsp_elapsed_ms()` — milliseconds convenience wrapper
  - 8.1.6 Platform implementations — `CLOCK_MONOTONIC`, `mach_absolute_time()`, `QueryPerformanceCounter()`
- 8.2 Wall Clock
  - 8.2.1 `ccsp_wallclock_us()` — microseconds since Unix epoch
  - 8.2.2 When wall clock is appropriate vs monotonic
  - 8.2.3 The NTP adjustment problem — never use wall clock for intervals
- 8.3 Sleeping and Waiting
  - 8.3.1 `ccsp_sleep_ns()` — sleep with nanosecond precision
  - 8.3.2 Handling EINTR — automatic retry
  - 8.3.3 Why you should not sleep in production code — use lib_ccsp_event timers

### Chapter 9: Secure Random

- 9.1 The Entropy Problem
  - 9.1.1 Why `rand()` is not acceptable for anything security-adjacent
  - 9.1.2 Why `time()` as a seed is not acceptable
  - 9.1.3 What "cryptographically secure" means in practice
  - 9.1.4 The blocking-before-seeded problem and how we handle it
- 9.2 The API
  - 9.2.1 `ccsp_random_bytes()` — fill a buffer with secure random bytes
  - 9.2.2 `ccsp_random_uint32()` — a random uint32_t
  - 9.2.3 `ccsp_random_uint64()` — a random uint64_t
  - 9.2.4 `ccsp_random_range()` — uniform random in [0, n) without modulo bias
  - 9.2.5 Error handling — what happens if the platform has no entropy source
- 9.3 Platform Implementations
  - 9.3.1 Linux — `getrandom(2)` with `GRND_NONBLOCK` detection
  - 9.3.2 OpenBSD — `getentropy(2)`
  - 9.3.3 FreeBSD/macOS — `arc4random_buf()`
  - 9.3.4 Other POSIX — `/dev/urandom` with seeding detection
  - 9.3.5 The shim deletion condition — `stdc_secure_random()` when standardized

### Chapter 10: Atomic Operations

- 10.1 When You Need Atomics
  - 10.1.1 Reference counting without a mutex
  - 10.1.2 Once-initialization — `ccsp_once_t`
  - 10.1.3 Lock-free flags and state
  - 10.1.4 What atomics cannot do — they are not a replacement for mutexes
- 10.2 The API
  - 10.2.1 `ccsp_atomic_t` — the atomic type
  - 10.2.2 `ccsp_atomic_load()` — read atomically
  - 10.2.3 `ccsp_atomic_store()` — write atomically
  - 10.2.4 `ccsp_atomic_exchange()` — swap and return old value
  - 10.2.5 `ccsp_atomic_compare_exchange()` — CAS operation
  - 10.2.6 `ccsp_atomic_fetch_add()` — atomic increment, returns old value
  - 10.2.7 Sequential consistency — why we do not expose memory orders
- 10.3 `ccsp_once_t` — one-time initialization
  - 10.3.1 `CCSP_ONCE_INIT` — static initialization
  - 10.3.2 `ccsp_once_run()` — run a function exactly once
  - 10.3.3 Thread safety guarantees
- 10.4 Platform Implementations
  - 10.4.1 C11 `<stdatomic.h>` — preferred
  - 10.4.2 GCC `__sync_*` builtins — fallback
  - 10.4.3 MSVC `Interlocked*` — Windows fallback
  - 10.4.4 Mutex-based fallback — for platforms with nothing else
  - 10.4.5 The shim deletion condition — C11 everywhere

## Volume 4: Observability

### Chapter 11: Structured Logging

- 11.1 Why Structured Logging
  - 11.1.1 Logs are data, not text
  - 11.1.2 The machine-readable requirement
  - 11.1.3 Why `printf` to stderr is not a logging system
- 11.2 Log Levels
  - 11.2.1 `CCSP_LOG_DEBUG` — development only, not for production builds
  - 11.2.2 `CCSP_LOG_INFO` — normal operational events
  - 11.2.3 `CCSP_LOG_WARN` — unexpected but recoverable
  - 11.2.4 `CCSP_LOG_ERROR` — failure requiring attention
  - 11.2.5 `CCSP_LOG_FATAL` — unrecoverable, process will exit
  - 11.2.6 Compile-time level filtering
- 11.3 The Logging API
  - 11.3.1 `CCSP_LOG()` — the primary logging macro
  - 11.3.2 Key-value pairs — typed logging fields
  - 11.3.3 `ccsp_log_set_backend()` — installing a custom backend
  - 11.3.4 The default backend — stderr with ISO 8601 timestamps
  - 11.3.5 The null backend — discarding all log output
- 11.4 Log Backends
  - 11.4.1 The backend interface — `ccsp_log_fn`
  - 11.4.2 Writing a JSON lines backend
  - 11.4.3 Writing a syslog backend
  - 11.4.4 Writing a test-capture backend
- 11.5 Pattern: Structured Log Events
  - 11.5.1 Naming log keys consistently
  - 11.5.2 The connection_id pattern — correlating log lines
  - 11.5.3 Logging errors with full context

### Chapter 12: Diagnostics and Testing

- 12.1 The `ccsp-check` Integration
  - 12.1.1 What ccsp-check checks in lib_ccsp_base usage
  - 12.1.2 Running ccsp-check against your code
  - 12.1.3 Interpreting ccsp-check output
- 12.2 AddressSanitizer and Valgrind
  - 12.2.1 Annotations for the sanitizers
  - 12.2.2 `ccsp_free_secure()` and the sanitizers — false positives and how to suppress them
  - 12.2.3 Custom allocator and the sanitizers — tracking allocations
- 12.3 Testing Patterns
  - 12.3.1 Mock platform tables for unit testing
  - 12.3.2 Testing allocation failure paths
  - 12.3.3 Testing error propagation
  - 12.3.4 The conformance suite — running it on a new platform

## Volume 5: Reference

### Chapter 13: Complete API Reference

- 13.1 `ccsp_platform.h` — platform detection macros
- 13.2 `ccsp_base.h` — integer types and boolean
- 13.3 `ccsp_err.h` — error type and macros
- 13.4 `ccsp_alloc.h` — allocation functions
- 13.5 `ccsp_str.h` — string type and operations
- 13.6 `ccsp_utf8.h` — UTF-8 validation and iteration
- 13.7 `ccsp_endian.h` — byte order utilities
- 13.8 `ccsp_list.h` — intrusive doubly-linked list
- 13.9 `ccsp_slist.h` — intrusive singly-linked list
- 13.10 `ccsp_hash.h` — hash table
- 13.11 `ccsp_buf.h` — growable buffer
- 13.12 `ccsp_ring.h` — ring buffer
- 13.13 `ccsp_slab.h` — slab allocator
- 13.14 `ccsp_time.h` — monotonic clock and wall clock
- 13.15 `ccsp_random.h` — secure random
- 13.16 `ccsp_atomic.h` — atomic operations
- 13.17 `ccsp_once.h` — one-time initialization
- 13.18 `ccsp_log.h` — structured logging
- 13.19 `ccsp_size.h` — overflow-checked size arithmetic
- 13.20 `ccsp_ctx.h` — base context

### Chapter 14: Error Code Reference

- 14.1 Core errors — `CCSP_ERR_*`
- 14.2 Allocation errors — `CCSP_ERR_NOMEM`, `CCSP_ERR_OVERFLOW`
- 14.3 Argument errors — `CCSP_ERR_NULL`, `CCSP_ERR_RANGE`, `CCSP_ERR_TOO_LARGE`
- 14.4 Encoding errors — `CCSP_ERR_UTF8_INVALID`, `CCSP_ERR_UTF8_OVERLONG`
- 14.5 Platform errors — `CCSP_ERR_NO_ENTROPY`, `CCSP_ERR_NO_MONOTONIC`

### Chapter 15: Platform Port Guide

- 15.1 The platform checklist
- 15.2 Required: integer type sizes
- 15.3 Required: byte order
- 15.4 Required: alignment requirements
- 15.5 Required: monotonic time source
- 15.6 Required: secure entropy source
- 15.7 Optional: native atomic operations
- 15.8 Optional: native once-initialization
- 15.9 Running the conformance suite
- 15.10 Submitting a port

### Appendix A: The CVE Corpus Patterns Relevant to lib_ccsp_base

- A.1 Pattern: implicit integer size assumption — CVEs and the fix
- A.2 Pattern: non-reentrant global state — CVEs and the fix
- A.3 Pattern: ignored error return — CVEs and the fix
- A.4 Pattern: uninitialized memory use — CVEs and the fix
- A.5 The ccsp-check rules that catch these at build time

### Appendix B: Shim Status and Deletion Conditions

- B.1 `ccsp_base.h` integer shim — deletion condition: C99 on all targets
- B.2 `ccsp_atomic.h` atomic shim — deletion condition: C11 on all targets
- B.3 `ccsp_random.c` entropy shim — deletion condition: `stdc_secure_random()` standardized
- B.4 `ccsp_time.c` monotonic shim — deletion condition: `stdc_monotonic_now()` standardized
- B.5 `ccsp_signal.c` signal shim — deletion condition: `sigaction()` on all targets
- B.6 Shim review schedule and tracker links

### Appendix C: Design Decisions Not Taken

- C.1 Why not a reference-counted string type
- C.2 Why not a custom allocator with pools
- C.3 Why not an object system
- C.4 Why not integrate lib_ccsp_event into lib_ccsp_base
- C.5 Why not support encodings other than UTF-8
- C.6 Why not use C++ for any part of this

### Appendix D: Changelog

- D.1 Version 1.0
- D.2 Planned: version 1.1 (shim deletions pending C99 universal availability)

---

*lib_ccsp_base is maintained by the CCSP Project.*

*Source:* `https://github.com/ccsp-project`
*Issues:* `https://github.com/ccsp-project/ccsp/issues`
*Discussions:* `https://github.com/ccsp-project/ccsp/discussions`
