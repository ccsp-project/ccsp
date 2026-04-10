# The CCSP Stack: Unified Design, Rationale, and Library Guide

**Namespace:** `lib_ccsp_*` / `ccsp_` symbol prefix  
**Version:** 1.0-draft  
**C Standard:** C17 baseline, C23 where available  
**Status:** Master Design Document  
**License:** CC-BY-SA-4.0 (this document), MIT (code)

---

## A Note on Naming

All libraries use the `lib_ccsp_` prefix (Correct C Systems Programming).
All public symbols use `ccsp_`. Headers are `ccsp_*.h`. The prefix is
long enough to be globally unique and short enough to tolerate in code.

Yes, we called it "correct." Forty years of buffer overflows, format string
vulnerabilities, use-after-frees, and billion-dollar CVEs have earned the
word. The industry has exposed the control group. We are the treatment.

This is the only meaningful naming convention in C and everyone ignores
it until they have their third symbol collision of the day.

---

## A Note on the Baseline

This stack targets **C17** as the minimum. Not C99. Not C11. C17.

GCC 8+ (2018), Clang 6+ (2018), MSVC 2019+ all ship C17. If your
platform cannot compile C17, your platform is not supported. This is
not negotiable and it is not a problem in 2026.

Consequences:
- `<stdint.h>`, `<stdbool.h>`, `<stdatomic.h>`, `<threads.h>` are
  standard library. We use them directly. No shims.
- `uint8_t`, `uint32_t`, `bool`, `true`, `false`, `_Atomic`,
  `_Static_assert` are just the language. No wrappers.
- `static_assert()` (C11 macro) is available.
- `[[nodiscard]]` (C23) used where available, falls back to
  `__attribute__((warn_unused_result))` on older compilers.

The shim table that was 14 entries long in previous versions of this
document is now 2 entries: io_uring (Linux only, optional) and
Vulkan (optional GPU backend). Both are opt-in, not defaults.

---

## Table of Contents

- [Part I: Why This Exists](#part-i-why-this-exists)
- [Part II: Architectural Principles](#part-ii-architectural-principles)
- [Part III: Dependency Architecture](#part-iii-dependency-architecture)
- [Part IV: Library Catalog](#part-iv-library-catalog)
- [Part V: Cross-Cutting Concerns](#part-v-cross-cutting-concerns)
- [Part VI: API Guide Table of Contents](#part-vi-api-guide-table-of-contents)
- [Part VII: Roadmap](#part-vii-roadmap)
- [Appendices](#appendices)

---

# Part I: Why This Exists

## The Problem Statement

The CVE corpus maintained alongside this stack classifies C systems
vulnerabilities by root cause. Twelve patterns cover 87% of findings.
Mean time introduction→disclosure: 5.8 years. Every pattern has a
structural fix in this stack -- not "less likely to occur" but "not
expressible in the API."

| Pattern | CVEs | Library That Prevents It |
|---------|------|--------------------------|
| Hand-rolled line reader, fixed buffer | 47 | lib_ccsp_lineproto |
| Caller-supplied length trusted | 31 | lib_ccsp_bio, lib_ccsp_ocl |
| gethostbyname nonreentrant global | 28 | lib_ccsp_dns |
| Integer overflow before allocation | 24 | lib_ccsp_base |
| SO_ERROR miss in nonblocking connect | 19 | lib_ccsp_connect |
| select() FD_SETSIZE unchecked | 17 | lib_ccsp_event (libevent) |
| Nonce/IV reuse in symmetric encryption | 15 | lib_ccsp_ocl |
| sprintf without bounds | 22 | lib_ccsp_base |
| ASN.1 ANY type parser confusion | 12 | lib_ccsp_asn1 (dispatch tables, not open-ended ANY) |
| Implicit integer size assumption | 34 | lib_ccsp_base (now rare with stdint) |
| Non-constant-time secret comparison | 11 | lib_ccsp_ocl |
| Non-reentrant global in crypto path | 9 | lib_ccsp_ocl |

The last two decades of security research on C systems code produced
this list. It has not changed materially since 2015. The patterns are
solved. The solutions are in these libraries.

## What Has Changed Since The Last Major Audit

Several things that were uncertain or contested in 2010--2015 are now
settled:

**TLS 1.0 and 1.1 are dead.** RFC 8996 (2021). Nothing in this stack
mentions them except the migration appendix.

**Ed25519 and X25519 are first-class.** They were experimental a decade
ago. They are now the right default for new key generation. NIST P-256
is still supported for interop. RSA-2048 is still supported for legacy
interop. Neither is the default for new systems.

**QUIC and HTTP/3 exist.** lib_ccsp_connect and lib_ccsp_http acknowledge
them. Full QUIC implementation is a roadmap item, not a 1.0 deliverable.
The abstractions are designed to accommodate it.

**Wayland has won on Linux desktop.** X11 still runs via XWayland.
lib_ccsp_draw_wayland is the primary Linux display backend.
lib_ccsp_draw_x11 is the legacy/embedded backend.

**io_uring exists** and is the fastest I/O interface on Linux for the
right workloads. lib_ccsp_event supports it as an optional backend.
It is not the default because it requires kernel 5.1+ and the syscall
surface is large.

**DNS-over-HTTPS and DNS-over-TLS are deployed.** lib_ccsp_dns supports
both as resolver transports. Plain UDP port 53 is still the default
because that is what resolver infrastructure actually provides, but
DoH and DoT are first-class options.

**DANE is still not widely deployed despite being correct.** We support
it. We document why it is correct. We do not pretend adoption is higher
than it is.

## What We Are Not Replacing

**libevent.** It is battle-tested, widely deployed, has good kqueue
and epoll implementations, and handles the edge cases that matter.
lib_ccsp_event wraps libevent's `event_base` behind a stable API.
This is the right call. Writing a new event loop in 2026 is hubris.

**FreeType.** Still the right font rasterizer. We wrap it cleanly.
The wrapper hides FreeType from the widget system.

**HarfBuzz.** Still the right text shaper. Same pattern.

**libpng, libjpeg-turbo, libwebp.** Image codecs are not our problem.
We define clean interfaces to them.

**OpenSSL or its forks as an optional backend.** lib_ccsp_ocl has a
native implementation and an OpenSSL-backend mode. The native
implementation is the default for new deployments. The OpenSSL backend
exists for FIPS 140-3 validated contexts where the customer's compliance
program requires an OpenSSL FIPS module.

## The Context Compression Argument

The CVE corpus is the security argument for this stack. There is a
second argument that is equally important: **this stack is a context
compression system for AI-assisted development.**

Every time an AI coding agent starts a systems program from scratch,
it burns context reasoning about infrastructure that is not the program:

- Which allocator strategy to use and how to handle OOM
- How to build an async event loop without the kqueue edge cases
- How to do DNS resolution without gethostbyname and its global state
- How to parse command-line options with environment variable fallback,
  config file integration, and source tracking
- How to do buffered I/O without the OpenSSL BIO design mistakes
- How to validate TLS certificates correctly per RFC 5280
- How to parse line protocols without the fixed-buffer CVE patterns
- How to represent color without blending in gamma-encoded space
- How to lay out widgets without O(n²) layout thrashing
- How to handle integer overflow before allocation
- How to structure error propagation with call-site context
- How to zero secret memory in a way the compiler cannot elide

None of this is the program. It is infrastructure. It is solved. It
is in this stack. And because it is solved once, correctly, with a
stable API, an agent reasoning about a new program does not need to
reason about any of it. It includes five headers and reasons about
the business logic.

The specification for a TLS-enabled JSON API server with async DNS,
connection pooling, structured logging, command-line configuration,
and correct error handling used to require the agent to hold all of
that infrastructure in context simultaneously. With this stack, the
specification is: implement the request handlers. The infrastructure
is a one-line include, not a specification item.

Context is not free. An agent that burns half its context window
reasoning about correct TLS certificate validation has half as much
context for the actual program. An agent that cannot fit the full
program in context makes worse decisions about the program.
Infrastructure that is correct by construction, documented with stable
contracts, and available as a library header is infrastructure the
agent does not need to reason about. It already reasoned about it once,
when it read this document.

The concrete form of the argument: a sophisticated systems program
written against this stack has a specification that is roughly as long
as its business logic. The same program written without this stack has
a specification that includes all the infrastructure decisions, which
is typically three to five times longer. The agent that uses this stack
reasons about a strictly smaller problem. It makes fewer mistakes. It
produces better code. Not because the agent is smarter -- because the
problem is actually smaller.

The same argument applies to human developers, but less acutely. Humans
have persistent memory across sessions, can build expertise in a
codebase over time, and tolerate context switching reasonably well.
Agents have none of these properties. They start fresh every session.
Correct, stable, documented infrastructure is worth proportionally more
to an agent than to a human, for the same reason a well-specified API
is worth more to a new hire than to someone who has worked in the
codebase for five years.

The design goal, stated plainly: **a competent AI coding agent with
this stack in context should be able to one-shot sophisticated systems
programs that would otherwise require multiple sessions of
infrastructure reasoning.** Every design decision in this stack -- the
stable ABI, the explicit API contracts, the documented error model,
the INVARIANTS block in the meta-prompt, the one-location-per-concern
rules, the cross-cutting logging key names, the consistent ownership
semantics -- serves this goal as directly as it serves the security
goal.

They are the same goal. Infrastructure that is correct, auditable, and
well-specified is infrastructure that compresses well into a prompt.
Infrastructure that is ad hoc, inconsistent, and poorly documented is
infrastructure the agent has to re-derive every time, at context cost,
with error probability proportional to the derivation complexity.

This is not a secondary use case for the stack. It is a primary design
constraint. When two designs are otherwise equal, the one that is easier
to describe precisely in a prompt wins.

## Licensing

Documentation: CC-BY-SA-4.0. Code: MIT.

---

# Part II: Architectural Principles

## Principle 1: Separation of Concerns Is a Security Property

Audit scope is bounded at library boundaries. The record layer of TLS
cannot see the certificate parser. The DNS resolver does not know what
TLS is. Heartbleed (2014) was a record layer bug enabled by the absence
of this separation. It would be structurally impossible in lib_ccsp_tls.

## Principle 2: Use What Already Works

libevent wraps epoll/kqueue correctly. FreeType rasterizes fonts
correctly. HarfBuzz shapes text correctly. Our job is clean interfaces
to these, not replacing them. Every line of code we do not write is a
line we do not have to audit.

## Principle 3: Caller Provides Storage or Explicitly Requests Allocation

Operation functions take caller-provided buffers. Object creation via
explicit `_create()` / `_destroy()` pairs. No hidden allocation in
operation calls. A grep for direct `malloc` in library source should
find zero matches outside of `ccsp_platform_default.c`.

## Principle 4: Explicit Over Implicit

Every piece of state a function needs is a parameter. No global
variables (the one exception: libevent's signal handling, which is
documented). No thread-local state except `errno`. No ambient
authority. Functions that have no hidden dependencies can be tested in
isolation, called from any thread, and audited without understanding
the entire library.

## Principle 5: UTF-8 Is the Only Text Encoding

Inside this stack, text is UTF-8 or it is bytes. Strict validation:
overlong encodings, surrogates, and codepoints above U+10FFFF are all
rejected. Legacy encodings are converted at the boundary by
lib_ccsp_charconv. This is not configurable.

## Principle 6: One Platform Ifdef Location Per Concern

Platform ifdefs belong in one file per concern. `ccsp_platform.h` for
OS and architecture detection. No other file contains `#ifdef __linux__`
or `#ifdef _WIN32`. When something breaks on a new platform, you look
in `ccsp_platform.h` and the relevant backend file. Nowhere else.

## Principle 7: Audit Scope Is Bounded By Design

An auditor reviewing a TLS application can scope precisely to each
library involved. The audit is compositional: if each component is
correct and the interfaces are correctly specified, the composition
is correct. This is not possible with OpenSSL.

## Principle 8: Running Code Controls the Standard

Every standards position from this project comes with a reference
implementation and a conformance test suite. Not a position paper.
Running code. The test suite becomes the conformance mechanism.
This is the IETF's founding principle and it is correct.

## Principle 9: C17 Is the Baseline

No shims for things C17 provides. `uint8_t` is not `ccsp_uint8_t`.
`bool` is not `ccsp_bool_t`. `atomic_int` is `atomic_int`. We use the
language.

## Principle 10: We Are Not Fixing C

This stack does not add an object system, a custom memory allocator, a
garbage collector, a runtime type system, reference counting
infrastructure, a plugin framework, or any other facility whose purpose
is to turn C into a different language.

C has well-understood semantics, a stable ABI, near-universal platform
support, and a 50-year track record of systems programming. It also has
manual memory management, no generics, and no built-in safety
guarantees. These are not bugs to be patched with a sufficiently clever
library. They are properties of the language. If those properties are
wrong for your problem, other languages exist.

Java, C++, C#, Rust, Go, and Zig are right there. You know where to
find them. Each solves different subsets of the problems C does not
solve. None of them requires you to use C. This stack does not compete
with them. It does C things in C, correctly, without pretending to be
something else.

GLib tried to fix C and produced an object system that requires a
code-generation step, a marshaling layer, and a 400-page manual. Qt
tried to fix C++ and produced moc. These are not failures of
implementation. They are failures of premise. The premise -- that you
can add high-level language features to C via a library -- is wrong.
You get the complexity without the safety guarantees, and you get a
codebase that is harder to audit than either C or the language you
were trying to emulate.

The corollary: when a contributor proposes adding reference counting
to lib_ccsp_base, or a vtable dispatch system to lib_ccsp_widget that
works like a class hierarchy, or a plugin loader with dlopen and a
symbol convention, the answer is no. Not because these things are
impossible to implement correctly in C. Because implementing them
correctly in C produces code that is harder to audit than the C++ you
were avoiding, and worse than the Rust you should be using if you
actually need those guarantees.

We use C. We use it well. We stop there.

---

# Part III: Dependency Architecture

## Library Registry

| Library | Symbol Prefix | Header | Layer |
|---------|--------------|--------|-------|
| lib_ccsp_base | `ccsp_` | `ccsp_base.h` | 0 |
| lib_ccsp_event | `ccsp_ev_` | `ccsp_event.h` | 1 |
| lib_ccsp_mpi | `ccsp_mpi_` | `ccsp_mpi.h` | 1 |
| lib_ccsp_regex | `ccsp_rx_` | `ccsp_regex.h` | 1 |
| lib_ccsp_charconv | `ccsp_conv_` | `ccsp_charconv.h` | 1 |
| lib_ccsp_bio | `ccsp_bio_` | `ccsp_bio.h` | 2 |
| lib_ccsp_dns | `ccsp_dns_` | `ccsp_dns.h` | 2 |
| lib_ccsp_mdns | `ccsp_mdns_` | `ccsp_mdns.h` | 2 |
| lib_ccsp_connect | `ccsp_conn_` | `ccsp_connect.h` | 2 |
| lib_ccsp_lineproto | `ccsp_lp_` | `ccsp_lineproto.h` | 3 |
| lib_ccsp_dgram | `ccsp_dgram_` | `ccsp_dgram.h` | 3 |
| lib_ccsp_dtls | `ccsp_dtls_` | `ccsp_dtls.h` | 3 |
| lib_ccsp_coap | `ccsp_coap_` | `ccsp_coap.h` | 3 (roadmap) |
| lib_ccsp_http | `ccsp_http_` | `ccsp_http.h` | 3 |
| lib_ccsp_smtp | `ccsp_smtp_` | `ccsp_smtp.h` | 3 |
| lib_ccsp_imap | `ccsp_imap_` | `ccsp_imap.h` | 3 |
| lib_ccsp_asn1 | `ccsp_asn1_` | `ccsp_asn1.h` | 4 |
| lib_ccsp_cert | `ccsp_cert_` | `ccsp_cert.h` | 4 |
| lib_ccsp_pki | `ccsp_pki_` | `ccsp_pki.h` | 4 |
| lib_ccsp_ocl | `ccsp_ocl_` | `ccsp_ocl.h` | 4 |
| lib_ccsp_tls | `ccsp_tls_` | `ccsp_tls.h` | 4 |
| lib_ccsp_data | `ccsp_data_` | `ccsp_data.h` | 5 |
| lib_ccsp_toml | `ccsp_toml_` | `ccsp_toml.h` | 5 |
| lib_ccsp_json | `ccsp_json_` | `ccsp_json.h` | 5 |
| lib_ccsp_cbor | `ccsp_cbor_` | `ccsp_cbor.h` | 5 |
| lib_ccsp_cas | `ccsp_cas_` | `ccsp_cas.h` | 5 |
| lib_ccsp_cli | `ccsp_cli_` | `ccsp_cli.h` | 5 |
| lib_ccsp_draw | `ccsp_draw_` | `ccsp_draw.h` | 6 |
| lib_ccsp_draw_wayland | `ccsp_dway_` | `ccsp_draw_wayland.h` | 6 |
| lib_ccsp_draw_x11 | `ccsp_dx11_` | `ccsp_draw_x11.h` | 6 |
| lib_ccsp_draw_fb | `ccsp_dfb_` | `ccsp_draw_fb.h` | 6 |
| lib_ccsp_draw_img | `ccsp_dimg_` | `ccsp_draw_img.h` | 6 |
| lib_ccsp_draw_gpu | `ccsp_dgpu_` | `ccsp_draw_gpu.h` | 6 |
| lib_ccsp_font | `ccsp_font_` | `ccsp_font.h` | 6 |
| lib_ccsp_text | `ccsp_text_` | `ccsp_text.h` | 6 |
| lib_ccsp_widget | `ccsp_wid_` | `ccsp_widget.h` | 6 |
| lib_ccsp_anim | `ccsp_anim_` | `ccsp_anim.h` | 6 |
| lib_ccsp_input | `ccsp_input_` | `ccsp_input.h` | 6 |
| lib_ccsp_a11y | `ccsp_a11y_` | `ccsp_a11y.h` | 6 |
| lib_ccsp_style | `ccsp_style_` | `ccsp_style.h` | 6 |
| lib_ccsp_app | `ccsp_app_` | `ccsp_app.h` | 6 |
| lib_ccsp_test | `ccsp_test_` | `ccsp_test.h` | 7 |
| lib_ccsp_fuzz | `ccsp_fuzz_` | `ccsp_fuzz.h` | 7 |

Tools: `ccsp-check` (static analysis), `ccsp-abi` (ABI checker),
`ccsp-corpus` (CVE corpus query).

## Dependency Graph

```
LAYER 0: FOUNDATION
lib_ccsp_base
  ccsp_err.h       -- error type, CCSP_ERR() macro, [[nodiscard]]
  ccsp_alloc.h     -- allocator interface, platform table
  ccsp_str.h       -- ccsp_str_t {uint8_t*,size_t}, UTF-8 ops
  ccsp_list.h      -- intrusive doubly-linked list
  ccsp_hash.h      -- ordered open-addressing hash table
  ccsp_buf.h       -- growable byte buffer
  ccsp_ring.h      -- fixed ring buffer, zero-copy peek/commit
  ccsp_slab.h      -- slab allocator
  ccsp_log.h       -- structured logging, key-value pairs
  ccsp_size.h      -- overflow-checked size arithmetic
  depends: libc only. No shims for stdint/stdbool/stdatomic.
  These are C17. They are just there.

LAYER 1: CORE SERVICES
  lib_ccsp_event   wraps libevent event_base_*
                  backends: epoll (Linux), kqueue (BSD/macOS),
                            IOCP (Windows), io_uring (Linux, opt-in)
  lib_ccsp_mpi     arbitrary precision, variable-time and constant-time
  lib_ccsp_regex   Thompson NFA default, backtracking opt-in
  lib_ccsp_charconv encoding → UTF-8 conversion at boundaries

LAYER 2: I/O AND NETWORKING
  lib_ccsp_bio     buffered I/O, reader/writer separation, filters
                  (not OpenSSL BIO -- separate reader and writer types)
  lib_ccsp_dns     async resolver: UDP/TCP/DoH/DoT transports
                  full record types, DNSSEC, DANE, SRV
  lib_ccsp_mdns    RFC 6762/6763 multicast DNS, DNS-SD
  lib_ccsp_connect stream connection: happy eyeballs, SRV, DANE,
                  SO_ERROR check, QUIC-aware API surface

LAYER 3: PROTOCOLS
  lib_ccsp_lineproto  declarative line protocol state machine
  lib_ccsp_dgram      datagram (UDP) application server and client
                     opcode dispatch, rate limiting, amplification protection
                     multicast support, correlation ID retransmit for clients
  lib_ccsp_dtls       DTLS 1.2 + 1.3 over lib_ccsp_dgram
                     cookie exchange, epoch management, same ciphers as TLS
  lib_ccsp_coap       CoAP RFC 7252 (roadmap) -- REST over UDP/DTLS
                     observe, block-wise transfer, Matter device protocol
  lib_ccsp_http       HTTP/1.1, HTTP/2 (HTTP/3 on roadmap)
  lib_ccsp_smtp       RFC 5321, DKIM, DANE MX
  lib_ccsp_imap       RFC 3501, IMAP4rev2 (RFC 9051)

LAYER 4: CRYPTOGRAPHY
  lib_ccsp_asn1    BER/DER/CER, schema-driven, ANY DEFINED BY with
                  dispatch tables, bare ANY captured as opaque bytes
  lib_ccsp_cert    certparse / certval / ca-nav
  lib_ccsp_pki     key generation, CSR, signing, PKCS#12
  lib_ccsp_ocl     primitives + schemes + AEAD
                  native impl OR OpenSSL backend (FIPS 140-3 contexts)
  lib_ccsp_tls     TLS 1.2 + 1.3 only. 1.0/1.1 not implemented.

LAYER 5: DATA AND CLI
  lib_ccsp_data    shared ccsp_data_value_t (integer=int64_t, not double)
  lib_ccsp_toml    TOML 1.0 parser
  lib_ccsp_json    JSON RFC 8259, DOM + streaming
  lib_ccsp_cbor    CBOR RFC 8949, DOM + streaming, maps to ccsp_data_value_t
                  no schema compiler needed, IETF binary format
                  COSE integration roadmap
  lib_ccsp_cas     content-addressable storage (Git object model)
  lib_ccsp_cli     declarative option parsing, source tracking

LAYER 6: GRAPHICS AND UI
  lib_ccsp_draw    backend-agnostic 2D drawing, float coords
  lib_ccsp_draw_wayland  PRIMARY Linux display backend
                        libwayland-client, xdg-shell, xdg-decoration
                        wl_shm (software) + wlr-layer-shell support
  lib_ccsp_draw_x11      LEGACY Linux display / embedded
                        still used via XWayland on Wayland compositors
  lib_ccsp_draw_fb       Linux framebuffer, embedded targets
  lib_ccsp_draw_img      software rasterizer → PNG/PPM/raw/PDF
                        reference implementation, visual regression
  lib_ccsp_draw_gpu      Vulkan (primary) + OpenGL ES 3.x (fallback)
  lib_ccsp_font    FreeType 2 wrapper, glyph atlas
  lib_ccsp_text    HarfBuzz shaping, UAX#9 bidi, UAX#14 line break
  lib_ccsp_widget  Flutter layout model: constraints↓ sizes↑
  lib_ccsp_anim    animation engine, curves, spring physics
  lib_ccsp_input   input normalization: Wayland, evdev, X11, Win32
  lib_ccsp_a11y    AT-SPI2 (Linux), NSAccessibility (macOS), UIA (Win)
  lib_ccsp_style   design tokens, theming, not CSS
  lib_ccsp_app     window management, system menus, platform integration

LAYER 7: TOOLING (not runtime)
  lib_ccsp_test    test runner, mock allocator, mock DNS, mock TLS pair,
                  property-based testing, visual regression
  lib_ccsp_fuzz    fuzz targets for every parser, libFuzzer + AFL + OSS-Fuzz
  ccsp-check       static analysis against CVE corpus
  ccsp-corpus      CVE corpus query tool
```

## The Dependency Rule

A library may depend only on libraries in lower or equal layers.
Enforced by the build system at configure time, not by code review.

---

# Part IV: Library Catalog

## Layer 0: Foundation

### lib_ccsp_base

The foundation. Every other library depends on it.

**C17 consequences.** `uint8_t` is `uint8_t`. `bool` is `bool`. We do
not invent `ccsp_uint8_t` or `ccsp_bool_t`. Wrapping standard types in
project-specific typedefs is a 1990s habit that makes code harder to
read and grep. The standard types are the types.

**What lib_ccsp_base actually provides:**

`ccsp_err.h` -- `ccsp_err_t` carrying code, domain string literal,
human-readable message, `__FILE__`, `__LINE__`. `CCSP_ERR()` macro
captures call site. `[[nodiscard]]` on all fallible functions (C23,
with `__attribute__((warn_unused_result))` fallback).

`ccsp_alloc.h` -- `ccsp_malloc()`, `ccsp_realloc()`, `ccsp_free()`,
`ccsp_calloc()`, `ccsp_free_secure()`. The platform table is the only
path to heap allocation in the entire stack. No library calls `malloc`
directly. One grep, zero results. `ccsp_free_secure()` uses
`explicit_bzero()` (POSIX.1-2008) or `memset_s()` (C11 Annex K) or a
compiler-barrier memset. Not a platform shim -- all three are available
on any modern platform.

`ccsp_str.h` -- `ccsp_str_t` is `{const uint8_t *data; size_t len;}`.
Length in bytes, not codepoints. Codepoint count is O(n). Byte count
is O(1). We use byte count. `CCSP_STR_LIT("hello")` with sizeof. Strict
UTF-8 ops via `ccsp_utf8_*`.

`ccsp_list.h` -- intrusive doubly-linked list. `ccsp_list_node_t`
embedded in containing struct. `CCSP_CONTAINER_OF()`. Sentinel. O(1)
insert/remove. `CCSP_LIST_FOREACH_SAFE()` for safe removal during
iteration.

`ccsp_hash.h` -- ordered open-addressing hash table. Linear probing.
Parallel insertion-order array. Key types: string (interned),
bytes (interned), uint64, pointer. Load factor 75%. Iteration is always
insertion order.

`ccsp_buf.h` -- growable byte buffer. `ccsp_buf_t`.
`ccsp_buf_append()`, `ccsp_buf_reserve()`, `ccsp_buf_commit()`,
`ccsp_buf_steal()`. Doubles on growth.

`ccsp_ring.h` -- fixed-capacity ring buffer. Caller-provided storage.
Zero-copy peek/reserve/commit interface. Two-pointer-range return for
wraparound. This is what the TLS record layer writes into without
copying.

`ccsp_log.h` -- structured logging. Level + domain + message + key-value
pairs. Compile-time level filtering. Per-context backend (default
stderr, null, capture for tests). Key names are standardized across
the entire stack (see Part V).

`ccsp_size.h` -- `ccsp_size_add()`, `ccsp_size_mul()`. Overflow-checked.
Returns `false` and sets error on overflow. Used anywhere you compute
an allocation size from external data.

`ccsp_atomic.h` -- thin wrappers over C11 `<stdatomic.h>`. Exists not
as a shim but as a policy layer: sequential consistency by default,
documented exceptions only. `ccsp_once_t` over `once_flag`.

**Dependencies:** libc only.

---

## Layer 1: Core Services

### lib_ccsp_event

**What it is:** libevent, wrapped behind a stable API.

**Why wrap instead of use directly:** The libevent API has evolved
through multiple incompatible versions and has some rough edges.
`ccsp_ev_base_t` is always explicit, never global. The callback
signatures are consistent. Error handling uses `ccsp_err_t`. The
distinction between persistent and one-shot events is in the type,
not a flag.

**Backends:**
- `ccsp_ev_backend_epoll` -- Linux 2.5.44+, the default on Linux
- `ccsp_ev_backend_kqueue` -- FreeBSD, macOS, OpenBSD, NetBSD
- `ccsp_ev_backend_iocp` -- Windows I/O Completion Ports
- `ccsp_ev_backend_io_uring` -- Linux 5.1+, opt-in. Faster for high
  connection counts. Larger kernel attack surface. Not default.
- `ccsp_ev_backend_poll` -- fallback when nothing better is available.
  No fd limit. No FD_SETSIZE.
- `ccsp_ev_backend_select` -- last resort. FD_SETSIZE checked, error
  if exceeded. Never on Linux or BSD in production.

**What is not in lib_ccsp_event:** DNS, HTTP, buffer management. Those
are other libraries. lib_ccsp_event does I/O dispatch, timers, and
signals.

**io_uring specifics:** When enabled, uses the fixed-file and registered
buffer features for reduced syscall overhead. Batch submission. The
backend is optional because `io_uring_setup()` requires elevated
privileges in some container environments and the syscall surface is
a significant kernel attack surface.

**Dependencies:** lib_ccsp_base, libevent.

### lib_ccsp_mpi

Arbitrary precision integers. Two namespaces: `ccsp_mpi_*`
(variable-time, public data) and `ccsp_mpi_ct_*` (constant-time, secret
data). Validated against GMP test vectors. Constant-time operations
have timing variance tests in CI using dudect methodology.

**Dependencies:** lib_ccsp_base.

### lib_ccsp_regex

Thompson NFA by default (O(n), no ReDoS). Backtracking opt-in with
compile warning. Named captures first-class. UTF-8 mode. Compile errors
include position and human-readable message, not error codes.

**Dependencies:** lib_ccsp_base.

### lib_ccsp_charconv

Encoding conversion to/from UTF-8. Used at boundaries only. ICU is an
option for applications needing Unicode normalization, collation, and
locale-aware operations. lib_ccsp_charconv handles conversion. ICU
handles everything else. We do not embed ICU.

**Dependencies:** lib_ccsp_base.

---

## Layer 2: I/O and Networking

### lib_ccsp_bio

Buffered I/O. The thing OpenSSL's BIO attempted and got wrong.

`ccsp_bio_reader_t` and `ccsp_bio_writer_t` are different types because
reading and writing are different operations. TLS is not a filter. It
takes a reader and a writer as inputs. The zero-copy path is
ring_reserve → write directly → ring_commit.

**In 2026:** `ccsp_bio_reader_from_fd_nb()` and
`ccsp_bio_writer_to_fd_nb()` use io_uring submit/complete when the
io_uring backend is active. The caller does not know.

**Dependencies:** lib_ccsp_base, lib_ccsp_event.

### lib_ccsp_dns

Async DNS resolver. The replacement for everything that calls
`getaddrinfo()` synchronously from a thread that should not be blocking.

`getaddrinfo()` is synchronous and is still being called from event
loops in production code in 2026. This is why this library exists.

**Transport options:**
- UDP port 53: default, what resolver infrastructure provides
- TCP port 53: automatic fallback on truncation, explicit option
- DNS-over-TLS (DoT, RFC 7858): port 853, full certificate validation
- DNS-over-HTTPS (DoH, RFC 8484): via lib_ccsp_http, for environments
  where UDP/TCP 53 is filtered or monitored

**Record types:** A, AAAA, NS, CNAME, SOA, PTR, MX, TXT, SRV, NAPTR,
DS, RRSIG, NSEC, NSEC3, DNSKEY, TLSA, CAA, HTTPS, SVCB. The HTTPS and
SVCB record types (RFC 9460) are first-class: they carry Alt-Svc and
ALPN information for lib_ccsp_connect.

**DNSSEC validation:** Trust anchors in config. Every validatable
response is validated. Authenticated denial of existence (NSEC/NSEC3).
Failure modes are explicit in the response struct.

**DANE:** TLSA record resolution and caching alongside A/AAAA.
lib_ccsp_connect uses these automatically.

**Cancellation:** After `ccsp_dns_query_cancel()` returns, the callback
will never fire. Guaranteed.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio,
lib_ccsp_tls (for DoT), lib_ccsp_http (for DoH).

### lib_ccsp_mdns

RFC 6762/6763. Multicast DNS and DNS-SD. Used on local networks.
`.local` resolution is automatic in lib_ccsp_dns -- the resolver routes
`.local` queries to the mDNS subsystem without caller configuration.

In 2026, mDNS matters for embedded, IoT, and local service discovery.
It is less important on cloud deployments where SRV and DANE are more
relevant. Both are supported.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio, lib_ccsp_dns.

### lib_ccsp_connect

Stream connection with every correctness property that hand-rolled
connection code misses.

**The SO_ERROR miss** is still in production code in 2026. After a
nonblocking connect completes (writable), you must check `SO_ERROR`.
Writable does not mean connected. lib_ccsp_connect always checks it.

**Happy eyeballs (RFC 8305, the 2017 update):** IPv6 first, 250ms
stagger (updated from RFC 6555's 50ms recommendation), IPv4 in parallel
if IPv6 has not connected, whichever connects first wins.

**QUIC-aware surface:** `ccsp_conn_config_t` has `transport` field:
`CCSP_CONN_TRANSPORT_TCP` (default), `CCSP_CONN_TRANSPORT_QUIC` (roadmap,
API surface present). Applications that want QUIC use the same API.
When QUIC is available, lib_ccsp_connect provides it. When it is not,
it falls back to TCP with a log event.

**SRV + HTTPS/SVCB:** RFC 9460 HTTPS records provide ALPN and Alt-Svc
information. lib_ccsp_connect uses them to negotiate HTTP/2 and HTTP/3
without additional round trips.

**DANE enforcement:** TLSA from lib_ccsp_dns. Usage 3 (domain-issued
cert, no CA) is supported. When DNSSEC validates TLSA, CA validation is
bypassed. The CA system is a transitional infrastructure.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio, lib_ccsp_dns,
lib_ccsp_tls.

---

## Layer 3: Protocols

### lib_ccsp_lineproto

Declarative state machine engine for line-based protocols. SMTP, IMAP,
POP3, FTP, Redis, memcached, Gopher, IRC. They all share the same
structure. lib_ccsp_lineproto provides it once.

**The 47 CVEs.** Every one of them is structurally impossible in
lib_ccsp_lineproto because the application never touches a buffer.
You write handlers. The library reads lines, parses verbs, enforces
state, dispatches. Zero buffer management in application code.

Schema: command table (verb, arg type, allowed states, callback) +
state table (id, timeout, greeting, error) + protocol descriptor
(max line len, CRLF policy, pipeline, STARTTLS).

The same schema is used by client and server. A test creates both in
the same event loop without a network socket.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio,
lib_ccsp_connect.

### lib_ccsp_http

HTTP/1.1 (RFC 9110--9112) and HTTP/2 (RFC 9113). HTTP/3 is on the
roadmap and the API surface accommodates it.

**HTTP/1.1:** Built on lib_ccsp_lineproto. Headers are line protocol.
Body is data mode. Chunked transfer encoding is a lib_ccsp_bio filter.
Trailers. Expect: 100-continue. Keep-alive managed by connection pool.

**HTTP/2:** Binary framing, HPACK, stream multiplexing, flow control.
Separate implementation from HTTP/1.1 -- they share the API, not the
code. Server push supported but disabled by default (it was a mistake
and is removed in HTTP/3).

**HTTP/3 (roadmap):** QUIC transport. When lib_ccsp_connect provides a
QUIC stream, lib_ccsp_http uses it transparently. QPACK for header
compression. The API is the same.

**Bodies are streams.** A response body is a `ccsp_bio_reader_t`. You
read it incrementally. 10GB files work. Nothing is buffered unless you
explicitly request it.

**WebSocket:** `ccsp_http_upgrade_websocket()`. After upgrade, the
connection is a bio reader/writer pair with WebSocket framing.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio,
lib_ccsp_connect, lib_ccsp_lineproto, lib_ccsp_tls.

### lib_ccsp_smtp

RFC 5321 SMTP + RFC 5322 messages + DKIM + DANE MX. The sendmail CVE
history applied to lib_ccsp_lineproto.

Includes SMTP AUTH (PLAIN, XOAUTH2), PIPELINING (RFC 2920), SIZE
(RFC 1870), 8BITMIME, SMTPUTF8 (RFC 6531), DANE enforcement for
outbound MX connections, DKIM signing and verification.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio,
lib_ccsp_connect, lib_ccsp_lineproto, lib_ccsp_dns, lib_ccsp_ocl.

### lib_ccsp_imap

IMAP4rev1 (RFC 3501) and IMAP4rev2 (RFC 9051). Client and server.

IMAP4rev2 differences matter: MOVE (RFC 6851) is mandatory, not an
extension. STATUS=SIZE. LIST-STATUS. NAMESPACE is mandatory.
RFC 9051 is the correct baseline in 2026.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio,
lib_ccsp_connect, lib_ccsp_lineproto, lib_ccsp_ocl.

### lib_ccsp_dgram

Datagram application server and client framework. The UDP equivalent
of lib_ccsp_lineproto.

**Why this exists.** A large class of important protocols is
datagram-based: NTP, DNS (server side -- lib_ccsp_dns is client-only),
RADIUS, SNMP, Syslog UDP transport, TFTP, CoAP, SIP, and any custom
binary protocol over UDP. The CVE record for hand-rolled UDP servers
includes buffer overflows on fixed-size datagram buffers, missing
source address validation, and amplification vulnerabilities. The same
structural approach that lib_ccsp_lineproto applies to TCP line
protocols applies here.

**The schema:**
```c
typedef struct {
    uint8_t          opcode_offset;          /* byte offset of opcode field */
    uint8_t          opcode_size;            /* 1, 2, or 4 bytes */
    bool             opcode_network_order;   /* big-endian opcode */
    size_t           max_dgram_size;         /* drop datagrams larger than this */
    ccsp_dgram_cmd_t *handlers;              /* NULL-terminated handler table */
    /* rate limiting */
    uint32_t         max_per_second_per_src;
    uint32_t         max_per_second_global;
    /* amplification protection */
    size_t           max_response_multiple; /* response bytes <= N * request bytes */
    /* callbacks */
    void           (*on_bind)(ccsp_dgram_server_t *, void *ud);
    void           (*on_rate_limited)(const ccsp_sockaddr_t *, void *ud);
    void           (*on_amplification_blocked)(const ccsp_sockaddr_t *, void *ud);
} ccsp_dgram_proto_t;
```

**`ccsp_dgram_cmd_t`:** opcode value, handler callback. The handler
receives the opcode, the raw datagram bytes and length, the source
address, and a response builder. It calls `ccsp_dgram_respond()` zero
or more times. Zero responses is valid (rate limiting, invalid state,
intentional drop).

**The amplification protection field is not optional.** DNS
amplification, NTP monlist (CVE-2013-5211), SSDP reflection -- all
share one pattern: small UDP request from spoofed source, large
response sent to victim. `max_response_multiple` is enforced by the
library before calling the handler. A handler that tries to respond
with more bytes than `N * request_size` gets its response silently
truncated and a log event at WARN. N=1 means responses can be no
larger than requests -- appropriate for any protocol that does not
have a legitimate reason to amplify.

**Source address validation:** The library tracks source addresses for
rate limiting. The handler always receives the validated source address
as a `ccsp_sockaddr_t`, never raw bytes it must parse itself.

**UDP server:**
```c
ccsp_dgram_server_t *ccsp_dgram_server_create(ccsp_ev_base_t *,
                                              const ccsp_dgram_proto_t *,
                                              const char *bind_addr,
                                              uint16_t port,
                                              ccsp_err_t *);
void ccsp_dgram_server_destroy(ccsp_dgram_server_t *);
```

**UDP client:** `ccsp_dgram_client_t` for request-response protocols.
Sends a datagram, waits for a response matching a correlation ID
field, retransmits with configurable backoff, calls callback with
result or timeout. The correlation ID field offset and size are in
the protocol descriptor.

**IPv4 and IPv6 simultaneously:** The server binds both where the
platform supports dual-stack sockets, with explicit fallback to two
sockets. The caller does not manage this.

**Multicast:** `ccsp_dgram_server_join_multicast()` for protocols that
use multicast (mDNS uses this internally, SSDP, PTP). Source-specific
multicast (SSM) supported where available.

**Known protocol schemas shipped:** NTP client/server schema, Syslog
UDP receiver schema, TFTP schema, CoAP schema (see lib_ccsp_coap below
for the full CoAP implementation). Application authors use these as
starting points or references.

**Dependencies:** lib_ccsp_base, lib_ccsp_event.

### lib_ccsp_dtls

DTLS 1.2 (RFC 6347) and DTLS 1.3 (RFC 9147) over lib_ccsp_dgram.
The same relationship to lib_ccsp_dgram as lib_ccsp_tls is to lib_ccsp_bio.

**Why DTLS matters.** CoAP mandates DTLS for secure operation.
RADIUS/DTLS (RFC 7360) is the modern RADIUS transport. Any
lib_ccsp_dgram application that needs confidentiality and
authentication over UDP needs DTLS.

**The handshake problem.** DTLS has to solve TLS's handshake over
an unreliable transport. The handshake messages are fragmented and
retransmitted. lib_ccsp_dtls handles this. The application sees the
same post-handshake read/write interface regardless of transport.

**Connection state.** UDP is connectionless but DTLS tracks connection
state per (local addr, remote addr) pair. lib_ccsp_dtls maintains a
connection table. New source addresses trigger a handshake. Known
source addresses reuse established session state.

**Cookie exchange (RFC 6347 §4.2.1):** DTLS HelloVerifyRequest
prevents amplification during the handshake itself. lib_ccsp_dtls
implements this. A ClientHello without a valid cookie gets a
HelloVerifyRequest response, not a full ServerHello. This limits
handshake amplification to 1:1.

**Cipher suites:** Same list as lib_ccsp_tls. TLS 1.3 cipher suites
apply to DTLS 1.3. No CBC. No RC4. No RSA key exchange. No DTLS 1.0
(the DTLS equivalent of TLS 1.1, deprecated by RFC 9147).

**DTLS 1.3 epoch management:** DTLS 1.3 uses epochs to handle
key updates cleanly over UDP where reordering means old-epoch packets
can arrive after a key update. lib_ccsp_dtls retains the previous
epoch's keys briefly after a key update, then discards them.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_dgram,
lib_ccsp_ocl, lib_ccsp_cert.

### lib_ccsp_coap (Roadmap, post-1.0)

CoAP (Constrained Application Protocol, RFC 7252) built on
lib_ccsp_dgram and lib_ccsp_dtls. REST over UDP for constrained
devices.

CoAP is the protocol Matter devices speak for application-layer
operations. It is GET/PUT/POST/DELETE with URIs and content types,
over UDP, with optional reliability (confirmable messages with
retransmission), block-wise transfer for large payloads (RFC 7959),
and observe (RFC 7641) for push notifications.

lib_ccsp_coap provides client and server. The server maps URI paths
to handler callbacks, the same pattern as lib_ccsp_http. The client
uses lib_ccsp_dns for URI resolution and lib_ccsp_dtls for `coaps://`
URIs. Block-wise transfer is transparent to handlers.

This is a roadmap item and not a 1.0 deliverable because CoAP's
full feature set (observe, block-wise, group communication via
multicast) is substantial. The lib_ccsp_dgram and lib_ccsp_dtls
foundations make it straightforward to add later.

**Dependencies (when implemented):** lib_ccsp_base, lib_ccsp_event,
lib_ccsp_dgram, lib_ccsp_dtls, lib_ccsp_dns.

---

## Layer 4: Cryptography and Security

### lib_ccsp_asn1

BER/DER/CER codec. Schema-driven. Strict DER by default. Max nesting
64. All strings normalized to UTF-8.

**The `ANY` situation.** We implement `ANY DEFINED BY`. We do not
implement bare `ANY`. The distinction is not pedantic:

`ANY DEFINED BY field` means the type is determined at runtime by
another field already parsed in the same structure -- typically an OID.
The dispatch table is explicit: you enumerate the known OIDs at compile
time, map each to a typed parser, reject unknown OIDs as errors. This
is implementable correctly and the ambiguity is bounded.

Bare `ANY` means "whatever bytes, figure it out from context not
present in the schema." That is an incomplete specification. When an
IETF draft contains bare `ANY`, the correct working group response is:
"Write the type. If you cannot write the type, you have not finished
the design." We still say this. It does not always work.

In practice the stack must handle X.509 `AlgorithmIdentifier.parameters`
(`ANY DEFINED BY algorithm`), PKCS#8 private key material (`ANY DEFINED
BY algorithm`), CMS `contentType`-gated `ANY`, and similar constructs
in deployed PKI formats. We eat these with dispatch tables.

Bare `ANY` fields encountered in real data are captured as opaque
`ccsp_asn1_any_t` -- raw tag, length, and DER bytes -- and returned to
the caller. The caller gets the bytes and the tag. The caller decides
what to do. We log a warning. We do not silently succeed. The opaque
capture is explicitly not a "parsed" value; it is an evidence bag.

`ccsp_asn1_any_t`:
```c
typedef struct {
    uint8_t         tag;
    const uint8_t  *der;      /* points into original input */
    size_t          der_len;
    bool            constructed;
} ccsp_asn1_any_t;
```

The dispatch table for `ANY DEFINED BY`:
```c
typedef struct {
    ccsp_asn1_oid_t          oid;
    ccsp_asn1_parse_fn       parse;   /* typed parser for this OID */
    const char             *name;    /* human-readable, for logging */
} ccsp_asn1_oid_dispatch_t;
```

`ccsp_asn1_decode_any_defined_by()` takes the discriminating OID and
a dispatch table. Returns `CCSP_ASN1_ERR_UNKNOWN_OID` for OIDs not in
the table. The error is surfaced, not swallowed. Unknown OIDs in
AlgorithmIdentifier mean an algorithm we do not support, which is a
meaningful error, not a decode-and-silently-ignore situation.

**Dependencies:** lib_ccsp_base.

### lib_ccsp_cert

Three modules:

**certparse:** DER bytes → immutable `ccsp_cert_t*` (reference-counted).
All strings UTF-8. Subject/issuer as typed attribute arrays. Full SAN
access. AIA, CDP, DANE TLSA, CAA extensions parsed.

**certval:** RFC 5280 path validation. Implemented from the formal
specification. Explicit `ccsp_time_t at_time` parameter -- never calls
`time()` internally, because a function that tests time-sensitive
behavior and calls `time()` internally cannot be tested. Name
constraints, policy OIDs, EKU, all of it.

**ca-nav:** AIA chasing, OCSP, CRL. Fetch via callback (lib_ccsp_http
in production, mock in tests). Stapled OCSP from TLS handshake
accepted. Result caching with TTL.

**Dependencies:** lib_ccsp_base, lib_ccsp_asn1, lib_ccsp_ocl.

### lib_ccsp_pki

Key generation, CSR, certificate signing, PKCS#12.

**Key generation defaults in 2026:**
- Asymmetric: Ed25519 (signatures), X25519 (key agreement). These are
  the correct defaults. Fast, small, no parameter choices to get wrong.
- Interop fallback: ECDSA P-256, ECDSA P-384.
- Legacy interop only: RSA-2048, RSA-3072.
- RSA-1024: not supported. Not configurable. Gone.

Self-signed certificates for DANE usage 3 are explicitly supported.
No CA required. The chain of trust is DNSSEC.

**Dependencies:** lib_ccsp_base, lib_ccsp_asn1, lib_ccsp_cert, lib_ccsp_ocl,
lib_ccsp_mpi.

### lib_ccsp_ocl

Cryptographic primitives, schemes, and AEAD.

**Two backend modes:**

1. **Native implementation** (default): Clean C17 implementation with
   hardware acceleration (AES-NI on x86/x86_64, ARMv8 crypto on
   AArch64). The right choice for new deployments.

2. **OpenSSL backend** (opt-in): For deployments where compliance
   programs mandate a FIPS 140-3 validated OpenSSL FIPS module.
   Same API. OpenSSL is behind the curtain. The caller cannot tell.

**Default algorithms in 2026:**

Symmetric AEAD: ChaCha20-Poly1305 (preferred for software), AES-256-GCM
(preferred with AES-NI). Both through the same `ccsp_ocl_aead_*` API.

Asymmetric: Ed25519, Ed448, X25519, X448, ECDSA P-256/P-384,
ECDH P-256/P-384. RSA in schemes layer (PSS for signatures,
OAEP for encryption) for legacy interop.

Hash: SHA-256, SHA-384, SHA-512, SHA3-256, SHA3-512, BLAKE3.

KDF: HKDF, PBKDF2, Argon2id (preferred for passwords), scrypt
(for applications requiring it).

**Not supported:** MD5 (except as a HMAC-MD5 shim for legacy RADIUS/NTP
interop, clearly labeled), SHA-1 (except for HMAC-SHA1 in the same
narrow category), 3DES, RC4, AES-ECB, unauthenticated CBC. These are
not "disabled by default" -- they are absent. Write to us if you need
them and tell us why.

**The AEAD interface structural nonce guarantee:** `ccsp_ocl_aead_seal()`
generates the nonce internally from a per-context counter plus a random
base. The application cannot pass a nonce. Nonce reuse is not possible
through the high-level API. The raw ChaCha20 and AES primitives exist
in the primitives layer for protocol implementation (TLS record layer)
where nonce management is the protocol's problem and is handled by
lib_ccsp_tls.

**FIPS 140-3:** The module boundary, self-tests, and zeroization
procedures are explicit in the architecture for the native
implementation. For the OpenSSL backend, FIPS status comes from
OpenSSL's FIPS module certificate.

**Dependencies:** lib_ccsp_base, lib_ccsp_mpi, lib_ccsp_asn1.

### lib_ccsp_tls

TLS 1.2 and TLS 1.3 only. TLS 1.0 and 1.1 are not implemented. They
are not configurable. RFC 8996 deprecated them in 2021.

**Supported cipher suites:**

TLS 1.3 (negotiation is mandatory, no configuration):
- `TLS_AES_256_GCM_SHA384`
- `TLS_CHACHA20_POLY1305_SHA256`
- `TLS_AES_128_GCM_SHA256`

TLS 1.2 (for servers that must maintain compatibility):
- `ECDHE-ECDSA-AES256-GCM-SHA384`
- `ECDHE-RSA-AES256-GCM-SHA384`
- `ECDHE-ECDSA-CHACHA20-POLY1305`
- `ECDHE-RSA-CHACHA20-POLY1305`

Not present: anything with CBC (LUCKY13 class), anything with RC4,
anything without PFS, anything with RSA key exchange.

**The Heartbleed structural fix:** Record layer and handshake layer
have separate buffer management. The record layer has no access to
user-supplied handshake fields. The vulnerability class is not possible.

**SNI:** Required. A TLS client config without `server_name` is a
validation error before any network operation. Encrypted Client Hello
(ECH, RFC draft-ietf-tls-esni) is supported when negotiated.

**No renegotiation:** A renegotiation request closes the connection
with a log event at WARN level.

**QUIC transport:** lib_ccsp_tls exports the TLS record protection
functions (`seal`, `open`) needed by a QUIC implementation. When
lib_ccsp_connect gains QUIC support, it uses these.

**Post-quantum:** CRYSTALS-Kyber / ML-KEM hybrid key exchange (X25519Kyber768)
supported as a compile-time option. The NIST PQC standards (FIPS 203,
204, 205) are final as of 2024. Hybrid modes are the correct deployment
strategy: classical security is maintained if Kyber is broken.

**Dependencies:** lib_ccsp_base, lib_ccsp_ocl, lib_ccsp_cert, lib_ccsp_bio.

---

## Layer 5: Data and CLI

### lib_ccsp_data

Shared value type. `ccsp_data_value_t`: null, bool, integer (`int64_t`,
not `double`), float (`double`), string (UTF-8), bytes, array, table.

Table is `ccsp_hash_t` -- ordered, insertion order always preserved.
TOML round-trips with the same key order. JSON objects round-trip with
the same key order. This is not accidental.

Integer is `int64_t`, not `double`. JavaScript made this mistake. JSON
inherited it. We correct it: a number without a decimal point or
exponent is an integer and is stored as `int64_t`. The distinction is
observable via the type tag. Applications that need precision get it.

**Dependencies:** lib_ccsp_base.

### lib_ccsp_toml

TOML 1.0 parser. The reference implementation of the spec version
published alongside this stack. Passes `toml-test`. Parse errors
include line and column, not just an error code.

**Dependencies:** lib_ccsp_base, lib_ccsp_data.

### lib_ccsp_json

JSON RFC 8259. DOM and streaming. Security limits (depth 64, string 1MB,
document 16MB) all configurable. UTF-8 validation: invalid sequences are
parse errors. Numbers: `strtoll` for integers, `strtod` for floats.
Passes JSONTestSuite mandatory cases.

**Dependencies:** lib_ccsp_base, lib_ccsp_data, lib_ccsp_bio.

### lib_ccsp_cbor

CBOR (Concise Binary Object Representation, RFC 8949) encoder and
decoder. Binary serialization that maps cleanly onto `ccsp_data_value_t`.

**Why CBOR and not protobuf.** Protobuf requires a schema compiler
(`protoc`) and generated C code. The schema compiler is not a small
tool. CBOR requires neither: it is self-describing, like JSON, but
binary. A CBOR decoder needs no schema. A CBOR encoder needs no
schema. The encoded data carries its own type tags.

CBOR is the IETF-blessed binary format. It is used in: COSE (CBOR
Object Signing and Encryption, RFC 9052), CWT (CBOR Web Tokens,
RFC 8392), EDHOC (RFC 9528 -- the lightweight key exchange for
constrained devices), CoAP payloads, FIDO2/WebAuthn authenticator
data, and the CBOR diagnostic notation is human-readable. The IETF
keeps standardizing protocols that use it. We are not picking an
underdog.

Protobuf is better for high-volume service-to-service RPC where you
control both ends, the schema is stable, and the generated code's
performance matters. If you need protobuf, use `protoc` and the
generated C, which interoperates with the rest of this stack via byte
buffer interfaces. We do not own the protobuf schema compiler and we
are not going to.

**Type mapping to `ccsp_data_value_t`:**

| CBOR major type | ccsp_data_value_t |
|----------------|-----------------|
| 0: unsigned int | CCSP_DATA_INTEGER (uint fits int64_t) |
| 1: negative int | CCSP_DATA_INTEGER |
| 2: byte string | CCSP_DATA_BYTES |
| 3: text string | CCSP_DATA_STRING (UTF-8 validated) |
| 4: array | CCSP_DATA_ARRAY |
| 5: map | CCSP_DATA_TABLE |
| 6: tag | preserved as metadata on the value |
| 7: float | CCSP_DATA_FLOAT |
| 7: true/false | CCSP_DATA_BOOL |
| 7: null | CCSP_DATA_NULL |

CBOR integers above INT64_MAX or below INT64_MIN (bignum tags 2 and 3)
are returned as `CCSP_DATA_BYTES` with a tag indicating bignum. The
application decides whether to pass them to lib_ccsp_mpi.

CBOR maps with non-string keys are supported in decode (they produce
a `CCSP_DATA_TABLE` with string-coerced keys and a warning log) but
not in encode (lib_ccsp_data tables have string keys). Applications
needing non-string keyed maps use the streaming encoder directly.

**Streaming encoder:**
```c
ccsp_cbor_enc_t *ccsp_cbor_enc_create(ccsp_bio_writer_t *, ccsp_err_t *);
void ccsp_cbor_enc_uint(ccsp_cbor_enc_t *, uint64_t);
void ccsp_cbor_enc_int(ccsp_cbor_enc_t *, int64_t);
void ccsp_cbor_enc_bytes(ccsp_cbor_enc_t *, const uint8_t *, size_t);
void ccsp_cbor_enc_str(ccsp_cbor_enc_t *, ccsp_str_t);
void ccsp_cbor_enc_array_begin(ccsp_cbor_enc_t *, size_t count);
void ccsp_cbor_enc_map_begin(ccsp_cbor_enc_t *, size_t count);
void ccsp_cbor_enc_tag(ccsp_cbor_enc_t *, uint64_t tag);
void ccsp_cbor_enc_value(ccsp_cbor_enc_t *, const ccsp_data_value_t *);
ccsp_err_t ccsp_cbor_enc_finish(ccsp_cbor_enc_t *);
```

Indefinite-length arrays and maps are supported in decode (common in
streaming CBOR) but the encoder always uses definite lengths because
definite-length CBOR is smaller and faster to parse.

**DOM decoder:** `ccsp_cbor_parse()` -- bytes in, `ccsp_data_value_t *`
out. Security limits: max nesting depth 64, max item count per
array/map 65536, max byte/text string 16MB.

**Streaming decoder:** Same event-based interface as lib_ccsp_json's
streaming parser. Feed bytes incrementally, receive typed events.
For CoAP payloads and constrained devices that cannot buffer the
full message.

**CBOR sequences (RFC 8742):** Multiple CBOR items concatenated
without a wrapper. `ccsp_cbor_parse_sequence()` returns them as an
array. The streaming decoder handles sequences naturally.

**COSE integration (roadmap):** CBOR Object Signing and Encryption
uses lib_ccsp_ocl for the cryptographic operations and lib_ccsp_cbor
for the CBOR structure. COSE_Sign1, COSE_Encrypt0, COSE_Mac0 are
the common single-recipient structures. Full COSE support is a
post-1.0 deliverable.

**Dependencies:** lib_ccsp_base, lib_ccsp_data, lib_ccsp_bio.

Content-addressable storage. Git object model (blob, tree, commit) as
a library. SHA-256 addressed. Hash verified on read. Backends:
filesystem, in-memory (testing), remote (delegate). Useful for build
systems, package management, config management, distributed sync.

**Dependencies:** lib_ccsp_base, lib_ccsp_ocl.

### lib_ccsp_cli

Declarative option parsing. Schema-driven. Every option has a name,
type, default, environment variable, config file key, help text, and
an application variable pointer the parser writes into.

**Types:** BOOL, STRING, INT, UINT, FLOAT, SIZE (k/m/g suffixes),
DURATION (ms/s/m/h stored as microseconds), PATH, ENUM, LIST, COUNT.

**Source priority:** command line → environment variable → TOML config
file → default. Source tracking reports which source set each option.
`ccsp_cli_print_effective_config()` for debugging.

**Help is generated from the schema.** It cannot drift from the actual
options. This is the main feature.

**Dependencies:** lib_ccsp_base, lib_ccsp_toml.

---

## Layer 6: Graphics and UI

### lib_ccsp_draw

Backend-agnostic 2D drawing. Twelve required operations in the backend
vtable. Float coordinates throughout. Text pre-decomposed by lib_ccsp_text
before the backend sees it. Frame lifecycle: begin/end/present.

**No display-specific calls above lib_ccsp_draw.** lib_ccsp_widget does
not know what Wayland is. When a new display protocol appears, one
backend changes.

**Color model.** Three layers, each correct for its purpose:

*Storage: linear light floats.*
`ccsp_draw_color_t` is four `float` channels: `r`, `g`, `b`, `a`.
Linear light (not gamma-encoded). Range [0.0, 1.0] for SDR; above 1.0
valid for HDR and wide-gamut (Display P3, Rec. 2020). This is the
correct representation for compositing. Porter-Duff alpha compositing
is defined in linear light. GPU hardware expects linear light when you
declare the swapchain format as sRGB and let the driver apply the
transfer function at scan-out. Math done in gamma-encoded space is
wrong: blending two sRGB colors by averaging their byte values produces
a result that is too dark. Storage is linear light. This is not
negotiable.

*Manipulation: Oklab / OKLCh.*
When you want to do something to a color -- lighten it, darken it,
mix two colors, rotate the hue, build a gradient -- linear light is
the wrong space. Linear light is perceptually non-uniform: equal steps
in linear RGB do not look equal. Gradients through linear RGB go muddy
in the middle. Hue rotation in HSL is wrong. Lightening by scaling RGB
channels is wrong.

Oklab (Björn Ottosson, 2020) is a perceptually uniform color space
with good hue linearity, simple math (no lookup tables, no CIECAM
complexity), and the same uniform-lightness property as Lab but
without Lab's hue shift problems. OKLCh is its cylindrical form:
Lightness, Chroma, hue angle. It is what CSS Color Level 4 uses for
`oklch()`. It is what Figma uses for gradient interpolation. It is
the right space for color manipulation in 2026.

The manipulation API works in OKLCh or Oklab and returns
`ccsp_draw_color_t` (linear light) for use in paint operations:

```c
/* construct from perceptual coordinates */
ccsp_draw_color_t ccsp_draw_color_from_oklch(float L, float C, float h, float a);
ccsp_draw_color_t ccsp_draw_color_from_oklab(float L, float a_, float b_, float alpha);

/* round-trip: linear light → OKLCh → linear light */
ccsp_draw_oklch_t ccsp_draw_color_to_oklch(ccsp_draw_color_t c);

/* perceptually correct interpolation (through Oklab) */
ccsp_draw_color_t ccsp_draw_color_mix(ccsp_draw_color_t a, ccsp_draw_color_t b, float t);

/* lightness manipulation in OKLCh L axis */
ccsp_draw_color_t ccsp_draw_color_lighten(ccsp_draw_color_t c, float amount);
ccsp_draw_color_t ccsp_draw_color_darken(ccsp_draw_color_t c, float amount);

/* chroma manipulation */
ccsp_draw_color_t ccsp_draw_color_saturate(ccsp_draw_color_t c, float amount);
ccsp_draw_color_t ccsp_draw_color_desaturate(ccsp_draw_color_t c, float amount);

/* hue rotation in OKLCh, no gamut shift */
ccsp_draw_color_t ccsp_draw_color_rotate_hue(ccsp_draw_color_t c, float degrees);

/* alpha */
ccsp_draw_color_t ccsp_draw_color_with_alpha(ccsp_draw_color_t c, float a);
```

`ccsp_draw_color_mix()` interpolates through Oklab. No dark muddy
middle. No hue shift. Gradients built with this look correct. This is
the function `ccsp_draw_paint_t` linear gradients use internally.

`ccsp_draw_color_lighten()` and `ccsp_draw_color_darken()` move along
the L axis of OKLCh. The result is the same hue, same chroma,
perceptually lighter or darker by the specified amount. Not "multiply
RGB by 1.2." Not "add 0.2 to all channels." Move L.

*Input convenience: sRGB.*
Designers think in sRGB hex codes. Tools output sRGB. CSS is sRGB.
The input convenience functions accept sRGB and convert to linear light:

```c
/* applies sRGB transfer function (not divide-by-255) */
ccsp_draw_color_t ccsp_draw_color_from_srgb8(uint8_t r, uint8_t g,
                                            uint8_t b, uint8_t a);
/* same, from packed 0xRRGGBBAA */
ccsp_draw_color_t ccsp_draw_color_from_hex(uint32_t rgba);

/* convenience constant: CSS named colors in linear light */
extern const ccsp_draw_color_t CCSP_DRAW_COLOR_WHITE;
extern const ccsp_draw_color_t CCSP_DRAW_COLOR_BLACK;
extern const ccsp_draw_color_t CCSP_DRAW_COLOR_TRANSPARENT;
```

`ccsp_draw_color_from_srgb8()` applies the actual sRGB piecewise
transfer function, not `channel / 255.0f`. The difference matters:
`from_srgb8(128, 128, 128)` produces linear ~0.216, not 0.502.
If you use 0.502 in compositing you get wrong blending results. We do
not give you the wrong version.

*Backend output.*
Backends that write sRGB bytes (wl_shm software path, framebuffer)
apply the inverse sRGB transfer function at output time.
Backends writing to GPU surfaces declare the swapchain format as sRGB
(`VK_FORMAT_B8G8R8A8_SRGB`) and let the hardware apply the transfer
function at scan-out. Wide-gamut displays get a Display P3 swapchain
format when available. None of this is the caller's problem. The
caller works in linear light, manipulates in OKLCh, and the backend
handles the last-mile conversion.

The framebuffer pixel formats (`RGB565`, `ARGB8888`, etc.) are
hardware wire formats. They live in lib_ccsp_draw_fb's pixel format
conversion layer. `ccsp_draw_color_t` does not know about wire formats.
The wire format layer does not know about color science.

**Dependencies:** lib_ccsp_base, lib_ccsp_event.

### lib_ccsp_draw_wayland

**The primary Linux desktop display backend.**

**Why Wayland is now primary:** It has been the default on Fedora since
Fedora 25 (2016), Ubuntu since 22.04 (2022), and GNOME since 3.36.
XWayland provides X11 compatibility. In 2026, writing new code to
target X11 directly is targeting the compatibility layer, not the
native protocol.

**Implementation:**

libwayland-client for the Wayland protocol. `xdg-shell` for window
management (title, maximize, minimize, resize, fullscreen). `xdg-decoration`
for server-side window decorations. `wp-viewporter` for high-DPI
logical/physical coordinate mapping. `wl-shm` (shared memory) for the
software rasterizer path. `wl_dmabuf` for zero-copy GPU texture sharing.

**Compositor-specific extensions (where necessary):**
`wlr-layer-shell` for panels and overlays (wlroots-based compositors).
`zwlr-foreign-toplevel-management` for taskbar integration.

**Input handling:** `wl_seat` (pointer, keyboard, touch). `wl_data_device`
for clipboard and drag-and-drop. `zwp-text-input-v3` for IME/text
input protocol (replaces X11's XIM, actually works on Wayland).

**High-DPI:** `wl_surface.set_buffer_scale()` and `wl_output.scale`.
Fractional scaling via `wp-fractional-scale-v1`. Float coordinates in
lib_ccsp_widget map cleanly to Wayland's logical coordinate model.

**GPU path:** EGL for OpenGL ES 3.x context. `wl_egl_window` surface.
Vulkan via `VK_KHR_wayland_surface`.

**Software path:** `wl_shm` buffer. Double-buffered. Damage tracking:
`wl_surface_damage_buffer()` for partial updates.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_draw,
libwayland-client.

### lib_ccsp_draw_x11

Legacy and embedded Linux display backend. Used directly on embedded
systems without a Wayland compositor, or via XWayland on Wayland
desktops. XLib + XRender (for antialiased compositing). XShm for
local display. Damage extension for partial repaints.

Not deprecated. Embedded Linux with a framebuffer compositor or custom
display manager is a real deployment target and X11 is often what it
runs. This backend exists for those targets.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_draw.

### lib_ccsp_draw_fb

Linux framebuffer (`/dev/fb0`) or custom embedded target. Software
rasterizer. `flush()` callback for DMA, SPI, or MMIO register updates.
Double-buffered. Pixel format conversion (RGB565, ARGB8888, etc.).
No windowing system required.

**Dependencies:** lib_ccsp_base, lib_ccsp_draw.

### lib_ccsp_draw_img

Software rasterizer to memory buffer. PNG, PPM, raw RGBA, PDF output.
The reference implementation: when GPU output differs from this,
this is correct. Used for visual regression testing and documentation
generation (all examples rendered from actual code).

`ccsp_draw_img_diff()` for pixel comparison. Used by lib_ccsp_test for
`ccsp_test_visual_assert()`.

**Dependencies:** lib_ccsp_base, lib_ccsp_draw, libpng (optional).

### lib_ccsp_draw_gpu

Vulkan primary, OpenGL ES 3.x fallback.

**Why Vulkan:** Vulkan is available on Linux (Mesa), Windows, macOS
(via MoltenVK), Android, and embedded (Raspberry Pi 4+ via Mesa).
In 2026, OpenGL 2.1 is a museum piece. Vulkan's explicit command
buffer model maps directly to the lib_ccsp_draw frame lifecycle
(begin / end / present).

**Why OpenGL ES 3.x fallback:** Some embedded targets (RPi 3,
older Android) have GLES3 but not Vulkan. GLES3 gives us compute
shaders, uniform buffer objects, VAOs, and instanced rendering -- the
things that make GPU rendering not terrible.

**Implementation:**

Vulkan path: command buffer per frame, one submission at frame_end.
Pipeline state objects cached by draw mode. Vertex buffer updated
per frame for dynamic geometry. Glyph atlas as a Vulkan image with
descriptor set. Path tessellation to triangle fans via libtess2
(the tessellator from Skia, MIT licensed, ~2000 lines).

GLES3 path: VAO, VBO, UBO. Same glyph atlas approach. Same
tessellator. Different submit mechanism.

**No path to OpenGL 2.1:** It is gone. If your GPU cannot do Vulkan
or GLES3, use lib_ccsp_draw_fb.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_draw,
Vulkan headers or GLES3 headers.

### lib_ccsp_font

FreeType 2 wrapper. Font loading, glyph rasterization, metrics,
kerning, glyph atlas management. FreeType is the right font rasterizer.
We wrap it so the rest of the stack does not know FreeType exists.

Variable fonts (OpenType Font Variations) supported via FreeType's
variation axis API. In 2026, variable fonts are common and not
supporting them means not supporting the fonts shipped with modern
operating systems.

**Dependencies:** lib_ccsp_base, FreeType2.

### lib_ccsp_text

HarfBuzz for shaping. UAX#9 for bidirectional text. UAX#14 for line
breaking. Paragraph layout. Hit testing. Selection rectangles.

HarfBuzz is the right text shaper. It handles complex scripts
(Arabic, Indic, Tibetan, Khmer) correctly. The alternative is getting
it wrong. We wrap it so the widget system does not know HarfBuzz exists.

Variable font shaping is handled transparently -- HarfBuzz supports
it, lib_ccsp_text exposes font variation axes in `ccsp_text_style_t`.

**Dependencies:** lib_ccsp_base, lib_ccsp_font, HarfBuzz.

### lib_ccsp_widget

Flutter's layout model in C. Constraints down, sizes up, parent sets
position. O(n). Five widget methods. Composition over inheritance.
Accessibility built in.

**Why Flutter's model:** It is correct, it is simple, it is proven
at scale, and the layout performance is a theorem (single pass, O(n))
not a hope. GTK and Qt's layout systems are legacy. Flutter's is the
right answer.

Layout widgets: flex, stack, constrained, padded, expanded, aligned,
scrollable, sized_box, aspect_ratio, wrap.

Leaf widgets: text, image, colored_box, custom_paint.

Gesture detectors: tap, long_press, drag, hover, double_tap, scale.

Accessibility: `ccsp_wid_a11y_t` descriptor per widget, AT-SPI2 tree
generated from widget tree.

Dirty region tracking: `ccsp_wid_mark_dirty()`, per-frame accumulation,
partial repaint.

**Testing:** lib_ccsp_draw_img backend produces pixel-identical output
to the Wayland backend on a reference widget tree. Visual regression
is automated.

**Dependencies:** lib_ccsp_base, lib_ccsp_draw, lib_ccsp_text,
lib_ccsp_event, lib_ccsp_anim.

### lib_ccsp_anim

Animation engine. Float values over time via curves (linear, ease-in,
ease-out, ease-in-out, spring physics). Sequences, parallel groups,
stagger. Driven by lib_ccsp_event timers.

**Dependencies:** lib_ccsp_base, lib_ccsp_event.

### lib_ccsp_input

Input event normalization. `ccsp_input_event_t`. Float coordinates in
widget space, not integer pixels. Text input separate from key events
(IME exists and CJK users need it to work).

**Wayland input:** `wl_seat`, `wl_pointer`, `wl_keyboard`, `wl_touch`.
`zwp-text-input-v3` for IME. Pointer coordinates are already logical
floats in Wayland's model -- we carry this through to the widget system
without conversion.

**evdev:** For applications running without a windowing system. Direct
`/dev/input/eventN` access. Key code translation to `ccsp_input_keycode_t`.

**Other:** X11, Win32, macOS NSEvent adapters.

**Dependencies:** lib_ccsp_base, lib_ccsp_event.

### lib_ccsp_a11y

Accessibility tree platform adapters.

AT-SPI2 on Linux via the D-Bus accessibility bus. AT-SPI2 is the
correct Linux accessibility framework in 2026. It works under both X11
and Wayland. Screen readers (Orca) use it. The Wayland transition did
not break AT-SPI2 because AT-SPI2 was always IPC-based, not X11-based.

NSAccessibility on macOS. UI Automation (UIA) on Windows.

**Dependencies:** lib_ccsp_base, lib_ccsp_widget.

### lib_ccsp_style

Design tokens and theming. `ccsp_style_theme_t` contains all visual
design decisions: typography, colors, geometry, motion timing.

Not CSS. No specificity. No cascade. No `!important`. A widget uses
theme tokens or overrides them explicitly. The override is code.

`ccsp_style_theme_dark()` produces a dark variant. `ccsp_style_theme_high_contrast()`
produces a WCAG AAA high-contrast variant. Runtime switching via
`ccsp_wid_set_theme()`.

**Dependencies:** lib_ccsp_base, lib_ccsp_widget.

### lib_ccsp_app

Application lifecycle, window management, system menus, platform
integration.

On Linux, this is Wayland-native (via lib_ccsp_draw_wayland) with
X11 fallback. Window management uses `xdg-shell` protocol. System
notifications via libnotify or org.freedesktop.Notifications D-Bus.
File pickers via the XDG Desktop Portal (`org.freedesktop.portal.FileChooser`
D-Bus interface) which works under both X11 and Wayland and properly
sandboxes the dialog.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_widget,
lib_ccsp_input, lib_ccsp_a11y.

---

## Layer 7: Tooling

### lib_ccsp_test

Testing infrastructure. Test runner (TAP output, JUnit XML for CI).
Mock allocator (tracking, failing, limited). Capture log backend.
Mock DNS resolver. Mock TLS pair (in-memory, no socket). Property-based
testing with shrinking. Known-answer test vector framework. Visual
regression via lib_ccsp_draw_img.

**Dependencies:** lib_ccsp_base, lib_ccsp_draw_img.

### lib_ccsp_fuzz

Fuzz targets for every parser, every decoder. LibFuzzer and AFL.
OSS-Fuzz integration. 100k iterations per target per PR in CI.
Corpus management. Crash triage.

**Targets include:** lib_ccsp_asn1 DER parse, lib_ccsp_cert parse,
lib_ccsp_json DOM and streaming, lib_ccsp_toml, lib_ccsp_dns response,
lib_ccsp_lineproto input, lib_ccsp_http request/response,
lib_ccsp_tls record/handshake, lib_ccsp_regex pattern and match,
lib_ccsp_ocl aead_open.

**Dependencies:** lib_ccsp_base, lib_ccsp_test.

### ccsp-check

Static analysis against CVE corpus. C source scanner (not a compiler
plugin). CRITICAL findings block CI. HIGH findings block merge with
mandatory suppression justification.

**CRITICAL:** `gets()`, `gethostbyname()`, hand-rolled line reader,
`malloc(a+b)` without overflow check, `sprintf()` (not `snprintf`),
`strcpy()`/`strcat()`, `rand()`/`random()` in crypto context.

**HIGH:** `select()` without FD_SETSIZE check, hand-rolled connect
without SO_ERROR check, `strcmp()` on secrets, `memcmp()` on secrets,
`atoi()`/`atol()` (no error detection), `signal()` (use `sigaction`).

Output includes CVE count, mean latency to disclosure, specific fix API.
Suppression requires justification comment. Suppression audit report
for security reviews.

---

# Part V: Cross-Cutting Concerns

## C17 and What It Eliminates

In 2026, C17 on all Tier 1 platforms means:

`<stdint.h>` -- `uint8_t`, `uint32_t`, `int64_t`, `uintptr_t`,
`SIZE_MAX`. These are the types. We use them.

`<stdbool.h>` -- `bool`, `true`, `false`. No `ccsp_bool_t`. No `TRUE`,
`FALSE` macros.

`<stdatomic.h>` -- `atomic_int`, `atomic_uint64_t`, `atomic_store()`,
`atomic_load()`, `atomic_compare_exchange_strong()`. `once_flag` and
`call_once()`. We wrap these in `ccsp_atomic.h` only for policy (default
to sequential consistency, document exceptions) not as shims.

`<threads.h>` -- `thrd_t`, `mtx_t`, `cnd_t`. Available on GCC 4.8+,
Clang 3.6+, MSVC 2019+. We use `mtx_t` in the platform table's mutex
operations when needed.

`snprintf()` -- returns the number of bytes that would have been written.
Not a shim. C99. It has been C99 for 27 years.

`static_assert()` -- C11 macro. Available. We use it for type size
assertions in `ccsp_platform.h`.

There is no shim table. The shim table has been replaced by "use the
language."

## Error Handling

`ccsp_err_t` with domain string, code, message, file, line.
`[[nodiscard]]` (C23) with `__attribute__((warn_unused_result))` fallback.
Domain strings by library: `"ccsp.base"`, `"ccsp.event"`, `"ccsp.dns"`,
etc. Error codes are stable within major versions.

## Memory Model

One rule: no library calls `malloc` directly except the platform
default implementation in lib_ccsp_base. Enforced by `ccsp-check`
(CRITICAL finding: `direct_malloc_call`) and by build-time symbol
checking.

`ccsp_free_secure()` uses `explicit_bzero()` where available (Linux,
macOS, BSDs), `memset_s()` (C11 Annex K, MSVC), or a volatile-pointer
memset with a compiler barrier. All three are correct. No shim -- all
three are present on the supported platforms.

## Threading Model

Event-driven single-threaded per event loop. One thread owns one
`ccsp_ev_base_t`. All I/O and timer callbacks on that thread. No locking
within a single event loop. Objects shared across threads are either
read-only after construction (immutable after `_create()`) or explicitly
synchronized. Thread-safety is documented per symbol.

io_uring's submission queue polling thread (SQPOLL mode) is managed
internally by the io_uring backend and is not visible to the application.

## Logging

All libraries use `CCSP_LOG()` from lib_ccsp_base. Structured key-value
pairs. Standard key names across the stack:

`conn_id`, `peer_addr`, `hostname`, `port`, `tls_version`,
`cipher_suite`, `alpn_proto`, `cert_subject`, `duration_ms`,
`bytes_read`, `bytes_written`, `status_code`, `method`, `path`,
`error_code`, `error_domain`, `dane_result`, `dnssec_validated`,
`doh_transport`, `dot_transport`, `srv_priority`, `quic_version`.

Consistent key names mean log aggregation works without custom parsers.

## Testing

**Known-answer tests:** NIST CAVP for crypto, Unicode Consortium files
for UTF-8, RFC test vectors for protocols, toml-test for TOML,
JSONTestSuite for JSON.

**Property-based tests:** Fixed seed in CI. Interesting shrunk
counterexamples stored in corpus for regression.

**Fuzz targets:** Every parser. 100k iterations per target per PR.
Crash = build failure. OSS-Fuzz for continuous background fuzzing.

**CI matrix:**

| Platform | Compiler | Sanitizers |
|----------|----------|------------|
| Linux x86_64 | GCC 13 | ASan, UBSan, TSan |
| Linux x86_64 | Clang 17 | ASan, UBSan, TSan, MemSan |
| Linux aarch64 | GCC 13 | ASan, UBSan |
| Linux aarch64 | Clang 17 | ASan, UBSan |
| macOS 14 | Apple Clang 15 | ASan, UBSan |
| FreeBSD 14 | Clang 17 | ASan |
| Windows 11 | MSVC 2022 | ASan |

---

# Part VI: API Guide Table of Contents

## Volume 0: Introduction and Architecture

### Chapter 0.1: How to Read This Guide
- 0.1.1 Guide organization
- 0.1.2 C17 baseline: what this means for your code
- 0.1.3 Notation conventions
- 0.1.4 All code examples are compiled and tested
- 0.1.5 How to find the library you need

### Chapter 0.2: The C17 Baseline
- 0.2.1 What C17 provides that we use directly
- 0.2.2 `<stdint.h>`: the types, without wrappers
- 0.2.3 `<stdbool.h>`: bool without ccsp_bool_t
- 0.2.4 `<stdatomic.h>`: atomics and once-init
- 0.2.5 `<threads.h>`: threads.h on all supported platforms
- 0.2.6 `[[nodiscard]]` (C23) with warn_unused_result fallback
- 0.2.7 `static_assert()` for compile-time invariants
- 0.2.8 What C23 adds that we use where available
- 0.2.9 Compiler minimum versions: GCC 8, Clang 6, MSVC 2019

### Chapter 0.3: Namespace and Naming Conventions
- 0.3.1 The `lib_ccsp_` prefix: library names
- 0.3.2 The `ccsp_` prefix: symbols, types, macros
- 0.3.3 The `ccsp_*_t` convention for types
- 0.3.4 The `ccsp_*_create()` / `ccsp_*_destroy()` convention
- 0.3.5 Header naming: `ccsp_*.h`
- 0.3.6 Error code prefixes by library
- 0.3.7 Log domain strings by library

### Chapter 0.4: The Dependency Architecture
- 0.4.1 The layer model
- 0.4.2 The complete library registry
- 0.4.3 Reading the dependency graph
- 0.4.4 The dependency rule and its enforcement
- 0.4.5 Using subsets: minimal builds
- 0.4.6 External dependencies: libevent, FreeType, HarfBuzz, libpng

### Chapter 0.5: The Error Model
- 0.5.1 `ccsp_err_t`: structure and fields
- 0.5.2 `CCSP_ERR()`: the macro and call-site capture
- 0.5.3 `ccsp_err_wrap()`: propagation with added context
- 0.5.4 `[[nodiscard]]` / `warn_unused_result`: enforcement
- 0.5.5 Passing NULL for err: when acceptable
- 0.5.6 Error propagation patterns (with code examples)
- 0.5.7 Converting platform errno to `ccsp_err_t`

### Chapter 0.6: The Memory Model
- 0.6.1 The platform table: the only path to heap allocation
- 0.6.2 `ccsp_malloc()`, `ccsp_calloc()`, `ccsp_realloc()`, `ccsp_free()`
- 0.6.3 `ccsp_free_secure()`: explicit_bzero, memset_s, compiler barrier
- 0.6.4 The one-malloc rule: enforcement and verification
- 0.6.5 `ccsp_size_add()`, `ccsp_size_mul()`: overflow-checked sizing
- 0.6.6 Caller-provides-storage principle
- 0.6.7 Object creation and ownership
- 0.6.8 Testing with mock allocators

### Chapter 0.7: The Threading Model
- 0.7.1 Event-driven single-threaded per event loop
- 0.7.2 Thread-safety documentation conventions
- 0.7.3 Sharing immutable objects across threads
- 0.7.4 Multiple event loops
- 0.7.5 io_uring SQPOLL: the invisible thread

### Chapter 0.8: UTF-8 Throughout
- 0.8.1 The policy: UTF-8 or bytes, no third option
- 0.8.2 Strict validation: what is rejected
- 0.8.3 `ccsp_str_t`: data pointer and byte length
- 0.8.4 `ccsp_utf8_*`: iteration, decode, encode
- 0.8.5 Converting legacy encodings with lib_ccsp_charconv

### Chapter 0.9: Build System Integration
- 0.9.1 CMake: `find_package(CCSP)`
- 0.9.2 pkg-config: `pkg-config --libs ccsp-base`
- 0.9.3 Selecting components: `CCSP_COMPONENTS`
- 0.9.4 Optional backends: io_uring, Vulkan, OpenSSL
- 0.9.5 Platform detection output
- 0.9.6 Cross-compilation: toolchain requirements
- 0.9.7 Embedded targets: static builds, custom platform table

### Chapter 0.10: ccsp-check Integration
- 0.10.1 Installation and invocation
- 0.10.2 CRITICAL findings: complete list
- 0.10.3 HIGH findings: complete list
- 0.10.4 INFO findings: complete list
- 0.10.5 Suppression syntax and justification requirement
- 0.10.6 The suppression audit report
- 0.10.7 CI integration: exit codes
- 0.10.8 Custom rules: format and registration
- 0.10.9 ccsp-corpus integration: live CVE count in findings

### Chapter 0.11: The CVE Corpus
- 0.11.1 The corpus URL and schema
- 0.11.2 Querying with ccsp-corpus
- 0.11.3 The 12 patterns: definitions and statistics
- 0.11.4 Adding new entries: submission format
- 0.11.5 Classification methodology
- 0.11.6 Using the corpus in security reviews and procurement

---

## Volume 1: lib_ccsp_base

### Chapter 1.1: Getting Started
- 1.1.1 The first program using lib_ccsp_base
- 1.1.2 `ccsp_platform_default()`: the standard configuration
- 1.1.3 `ccsp_ctx_create()`, `ccsp_ctx_destroy()`
- 1.1.4 C17 types in use: uint8_t, bool, size_t
- 1.1.5 Running the conformance suite

### Chapter 1.2: Platform Detection (ccsp_platform.h)
- 1.2.1 The one-file rule: only file with OS/arch ifdefs
- 1.2.2 OS flags: `CCSP_OS_LINUX`, `CCSP_OS_FREEBSD`, `CCSP_OS_MACOS`, `CCSP_OS_WINDOWS`
- 1.2.3 Arch flags: `CCSP_ARCH_X86_64`, `CCSP_ARCH_AARCH64`, `CCSP_ARCH_RISCV64`
- 1.2.4 `CCSP_STRICT_ALIGN`: bus-error platforms
- 1.2.5 Endianness: `CCSP_BIG_ENDIAN`, `CCSP_LITTLE_ENDIAN`
- 1.2.6 Feature flags: `CCSP_HAS_IO_URING`, `CCSP_HAS_GETRANDOM`,
  `CCSP_HAS_EXPLICIT_BZERO`, `CCSP_HAS_AESNI`, `CCSP_HAS_ARM_CRYPTO`
- 1.2.7 `static_assert()` checks for type sizes
- 1.2.8 Adding a new OS or architecture

### Chapter 1.3: The Platform Table
- 1.3.1 `ccsp_platform_t`: the complete struct
- 1.3.2 `ccsp_platform_default()`: standard configuration
- 1.3.3 Replacing the allocator: pool allocator, TLSF
- 1.3.4 Replacing the clock: testing time-dependent code
- 1.3.5 Replacing the log backend
- 1.3.6 NULL threading functions for single-threaded builds
- 1.3.7 Static memory embedded targets

### Chapter 1.4: The Base Context
- 1.4.1 `ccsp_ctx_t`: what it holds
- 1.4.2 Creation, initialization, destruction
- 1.4.3 Passing context through call stacks
- 1.4.4 Context vs global state: the argument from testability

### Chapter 1.5: Error Handling
- 1.5.1 `ccsp_err_t`: code, domain, message, file, line
- 1.5.2 `CCSP_ERR()`: the macro
- 1.5.3 `ccsp_err_wrap()`: adding context on propagation
- 1.5.4 `ccsp_err_clear()`, `ccsp_err_format()`
- 1.5.5 `[[nodiscard]]` and `warn_unused_result`: what happens when ignored
- 1.5.6 Pattern: propagating without losing context
- 1.5.7 Pattern: converting errno
- 1.5.8 Pattern: testing all error paths with mock allocator
- 1.5.9 Complete error code list (all libraries)

### Chapter 1.6: Memory and Allocation
- 1.6.1 `ccsp_malloc()`, `ccsp_calloc()`, `ccsp_realloc()`, `ccsp_free()`
- 1.6.2 `ccsp_free_secure()`: implementation and verification
  - explicit_bzero() on Linux/macOS/BSDs
  - memset_s() on Windows and C11 Annex K
  - Volatile-pointer with barrier: the last resort
  - How to verify the compiler has not elided it: inspecting assembly
- 1.6.3 `ccsp_strdup()`, `ccsp_strndup()`
- 1.6.4 `ccsp_size_add()`, `ccsp_size_mul()`: the pattern for safe sizing
- 1.6.5 The one-malloc rule: grep, build check, ccsp-check rule
- 1.6.6 Valgrind and ASan annotations
- 1.6.7 ASAN_OPTIONS useful for lib_ccsp_base testing

### Chapter 1.7: Strings and UTF-8
- 1.7.1 `ccsp_str_t`: the type
- 1.7.2 `CCSP_STR_LIT("hello")`: literal without strlen
- 1.7.3 `ccsp_str_from_cstr()`, `ccsp_str_cstr()` (allocates null terminator)
- 1.7.4 `ccsp_str_eq()`, `ccsp_str_eq_ci()`: equality
- 1.7.5 `ccsp_str_dup()`, `ccsp_str_free()`: owned copy
- 1.7.6 `ccsp_str_contains()`, `ccsp_str_starts_with()`, `ccsp_str_ends_with()`
- 1.7.7 `ccsp_str_trim()` variants
- 1.7.8 `ccsp_str_split()`: delimiter split
- 1.7.9 `ccsp_utf8_validate()`: strict validation (what is rejected)
- 1.7.10 `ccsp_utf8_decode()`, `ccsp_utf8_encode()`: single codepoint
- 1.7.11 `ccsp_utf8_next()`, `ccsp_utf8_prev()`: iteration
- 1.7.12 `ccsp_utf8_charlen()`: byte length of next codepoint (without decode)
- 1.7.13 `ccsp_utf8_codepoint_count()`: O(n), use carefully
- 1.7.14 Pattern: iterating codepoints
- 1.7.15 Pattern: splitting on ASCII delimiters in UTF-8 text

### Chapter 1.8: Byte Order (ccsp_endian.h)
- 1.8.1 Why byte-by-byte: no pointer casts, no alignment assumptions
- 1.8.2 `ccsp_read_be16/32/64()`, `ccsp_write_be16/32/64()`
- 1.8.3 `ccsp_read_le16/32/64()`, `ccsp_write_le16/32/64()`
- 1.8.4 `ccsp_bswap16/32/64()`: byte swapping
- 1.8.5 Compiler intrinsics: `__builtin_bswap32` on GCC/Clang, `_byteswap_ulong` on MSVC
- 1.8.6 Test vectors

### Chapter 1.9: Intrusive Linked Lists
- 1.9.1 Why intrusive: allocation overhead and cache behavior
- 1.9.2 `ccsp_list_node_t` embedded in containing struct
- 1.9.3 `CCSP_CONTAINER_OF()`: from node back to containing struct
- 1.9.4 `ccsp_list_t`: the sentinel-based head
- 1.9.5 `ccsp_list_init()`, `ccsp_list_insert_head()`, `ccsp_list_insert_tail()`
- 1.9.6 `ccsp_list_remove()`: O(1), no search
- 1.9.7 `ccsp_list_empty()`, `ccsp_list_count()`
- 1.9.8 `CCSP_LIST_FOREACH()`, `CCSP_LIST_FOREACH_SAFE()`
- 1.9.9 `ccsp_slist_t`: singly-linked for stack/LIFO use
- 1.9.10 Pattern: embedding in application structs

### Chapter 1.10: Hash Tables
- 1.10.1 Design: open addressing, insertion order, interned keys
- 1.10.2 `ccsp_hash_create()`, `ccsp_hash_destroy()`
- 1.10.3 `ccsp_hash_set()`, `ccsp_hash_get()`, `ccsp_hash_delete()`
- 1.10.4 `ccsp_hash_count()`, `ccsp_hash_clear()`
- 1.10.5 Key types: `CCSP_HASH_KEY_STRING`, `_BYTES`, `_UINT64`, `_PTR`
- 1.10.6 Interning: what it means, when it matters
- 1.10.7 `ccsp_hash_each()`, `ccsp_hash_iter_t`: always insertion order
- 1.10.8 Load factor, resize, tombstone management
- 1.10.9 Performance: cache behavior analysis

### Chapter 1.11: Buffers and Ring Buffers
- 1.11.1 `ccsp_buf_t`: growable byte buffer
- 1.11.2 `ccsp_buf_append()`, `ccsp_buf_reserve()`, `ccsp_buf_commit()`
- 1.11.3 `ccsp_buf_steal()`: ownership transfer without copy
- 1.11.4 `ccsp_ring_t`: fixed-capacity ring buffer, caller-provided storage
- 1.11.5 `ccsp_ring_readable()`, `ccsp_ring_writable()`
- 1.11.6 `ccsp_ring_peek()`, `ccsp_ring_consume()`: zero-copy read
- 1.11.7 `ccsp_ring_reserve()`, `ccsp_ring_commit()`: zero-copy write
- 1.11.8 `ccsp_ring_peek_all()`: the two-range wraparound case
- 1.11.9 Pattern: zero-copy TLS record writing

### Chapter 1.12: Slab Allocator
- 1.12.1 When to use slab allocation
- 1.12.2 `ccsp_slab_create()`, `ccsp_slab_destroy()`
- 1.12.3 `ccsp_slab_alloc()`, `ccsp_slab_free()`
- 1.12.4 Page-aligned backing, embedded free list
- 1.12.5 Debug canaries for double-free detection

### Chapter 1.13: Monotonic Time
- 1.13.1 Why `time()` and `gettimeofday()` are wrong for intervals
- 1.13.2 `ccsp_monotonic_ns()`: nanoseconds since unspecified epoch
- 1.13.3 `ccsp_elapsed_ns()`, `ccsp_elapsed_ms()`
- 1.13.4 `ccsp_wallclock_us()`: wall clock for human-visible timestamps
- 1.13.5 Platform implementations: CLOCK_MONOTONIC, QPC, mach_absolute_time
- 1.13.6 The macOS timebase conversion: precision and overflow edge case

### Chapter 1.14: Secure Random
- 1.14.1 Why `rand()` is not acceptable (and yes, people still use it)
- 1.14.2 `ccsp_random_bytes()`, `ccsp_random_uint32()`, `ccsp_random_uint64()`
- 1.14.3 `ccsp_random_range()`: uniform in [0, n), rejection sampling
- 1.14.4 Platform implementations: `getrandom(2)`, `getentropy(2)`, `arc4random_buf()`
- 1.14.5 The blocking-before-seeded problem on Linux
- 1.14.6 Statistical test: chi-square uniformity

### Chapter 1.15: Atomics and Once-Init
- 1.15.1 `ccsp_atomic.h`: thin policy wrapper over `<stdatomic.h>`
- 1.15.2 Sequential consistency default: the reasoning
- 1.15.3 Reference counting pattern
- 1.15.4 `ccsp_once_t` over C11 `once_flag`
- 1.15.5 Testing with ThreadSanitizer

### Chapter 1.16: Structured Logging
- 1.16.1 Why structured: logs are queryable data
- 1.16.2 Log levels: DEBUG, INFO, WARN, ERROR, FATAL
- 1.16.3 `CCSP_LOG()`: level, domain, message, key-value pairs
- 1.16.4 Compile-time level filtering: `CCSP_LOG_MIN_LEVEL`
- 1.16.5 Backend types: stderr (default), null, capture, custom
- 1.16.6 Writing a JSON-lines backend
- 1.16.7 Writing a syslog backend
- 1.16.8 Standard log key names (full cross-stack reference)
- 1.16.9 Key-value evaluation: no side effects in macro arguments

### Chapter 1.17: Overflow-Checked Arithmetic
- 1.17.1 `ccsp_size_add()`, `ccsp_size_mul()`, `ccsp_size_align()`
- 1.17.2 Pattern: safe array allocation
- 1.17.3 Pattern: safe string concatenation sizing
- 1.17.4 `__builtin_add_overflow` and `__builtin_mul_overflow` (GCC/Clang)
- 1.17.5 MSVC `UIntAdd`, `UIntMult` equivalents

### Chapter 1.18: Platform Port Guide
- 1.18.1 The porting checklist
- 1.18.2 Required: `ccsp_platform.h` OS and arch detection
- 1.18.3 Required: C17 compiler validation (type sizes, static_assert)
- 1.18.4 Required: monotonic clock source
- 1.18.5 Required: secure entropy source
- 1.18.6 Optional: hardware crypto acceleration
- 1.18.7 Running the conformance suite on a new platform
- 1.18.8 Submitting a port

---

## Volume 2: Core Services

### Chapter 2.1: lib_ccsp_event -- libevent Wrapper
- 2.1.1 Why wrap libevent instead of use directly
- 2.1.2 `ccsp_ev_base_t`: always explicit, never global
- 2.1.3 `ccsp_ev_base_create()`: backend selection
- 2.1.4 `ccsp_ev_base_run()`, `ccsp_ev_base_run_once()`, `ccsp_ev_base_stop()`
- 2.1.5 I/O events: `ccsp_ev_io_t`
  - `ccsp_ev_io_init()`: fd, events, callback
  - `ccsp_ev_io_add()` with optional timeout
  - `ccsp_ev_io_remove()`, `ccsp_ev_io_modify()`
- 2.1.6 Timer events: `ccsp_ev_timer_t`
  - Min-heap: O(log n) insert, O(log n) remove
  - `ccsp_ev_timer_add()`, `ccsp_ev_timer_remove()`, `ccsp_ev_timer_reset()`
- 2.1.7 Signal events: `ccsp_ev_signal_t`
  - Self-pipe mechanism: the one acceptable global
  - Multi-base signal delivery limitation: documented
- 2.1.8 Deferred callbacks: `ccsp_ev_defer()`
- 2.1.9 Idle callbacks: `ccsp_ev_idle_t`
- 2.1.10 Backend details
  - epoll: edge-triggered, EPOLLRDHUP for peer close detection
  - kqueue: the seven documented edge case behaviors
  - IOCP: Windows completion port integration
  - io_uring: submission queue, completion queue, SQPOLL mode
    - When to enable io_uring: high connection count servers
    - When not to: containers with restricted syscalls, security-sensitive
    - io_uring attack surface: documented and referenced
  - poll: fallback, no fd limit
  - select: last resort, FD_SETSIZE enforcement
- 2.1.11 Performance comparison across backends: documented benchmarks
- 2.1.12 Inter-thread wakeup: `ccsp_ev_base_wakeup()`

### Chapter 2.2: lib_ccsp_mpi -- Arbitrary Precision Arithmetic
- 2.2.1 Variable-time vs constant-time: the two namespaces
- 2.2.2 `ccsp_mpi_t` operations: complete API
- 2.2.3 `ccsp_mpi_ct_*` constant-time operations
- 2.2.4 Timing variance tests: dudect methodology, CI thresholds
- 2.2.5 Validation against GMP: test vector format and coverage
- 2.2.6 Performance: GMP comparison on P-256 and RSA-2048

### Chapter 2.3: lib_ccsp_regex -- Regular Expressions
- 2.3.1 NFA engine: O(n) guarantee, why it matters for security
- 2.3.2 Backtracking engine: the compiler warning text and meaning
- 2.3.3 `ccsp_rx_compile()`: flags, engine selection, error format
- 2.3.4 `ccsp_rx_match()`, `ccsp_rx_find_all()`, `ccsp_rx_replace()`
- 2.3.5 Named captures: `(?P<name>...)` and `(?<name>...)`
- 2.3.6 UTF-8 mode: character classes, `\w`, `\d`, Unicode properties
- 2.3.7 Thread safety: compiled patterns are immutable
- 2.3.8 Fuzz results

### Chapter 2.4: lib_ccsp_charconv -- Encoding Conversion
- 2.4.1 The boundary principle
- 2.4.2 `ccsp_conv_to_utf8()`, `ccsp_conv_from_utf8()`
- 2.4.3 `ccsp_conv_detect()`: statistical detection, BOM detection
- 2.4.4 Streaming conversion: `ccsp_conv_ctx_t`
- 2.4.5 ICU as an alternative for normalization and collation
- 2.4.6 Supported encoding list

---

## Volume 3: I/O and Networking

### Chapter 3.1: lib_ccsp_bio -- Buffered I/O
- 3.1.1 Reader/writer separation: why OpenSSL got it wrong
- 3.1.2 `ccsp_bio_reader_t`: the abstract interface
- 3.1.3 `ccsp_bio_writer_t`: the abstract interface
- 3.1.4 Concrete readers: fd (blocking), fd_nb (nonblocking + io_uring),
  memory, FILE*, buf
- 3.1.5 Concrete writers: fd (blocking), fd_nb (nonblocking + io_uring),
  memory, FILE*, null (discard)
- 3.1.6 Filters: base64, hex, zlib/deflate, brotli (where available)
- 3.1.7 `ccsp_bio_fmt_*`: formatted writing
- 3.1.8 Zero-copy path: ring_reserve → write → ring_commit
- 3.1.9 io_uring integration: what changes for the caller (nothing)
- 3.1.10 Writing a custom reader or writer

### Chapter 3.2: lib_ccsp_dns -- Async Resolver
- 3.2.1 The getaddrinfo() problem: still synchronous in 2026
- 3.2.2 `ccsp_dns_resolver_t`, `ccsp_dns_config_t`
- 3.2.3 Transport options: UDP, TCP, DoT, DoH
  - DoT: port 853, TLS certificate validation, SPKI pinning option
  - DoH: via lib_ccsp_http, JSON or DNS wireformat
  - Selecting transport per resolver address
- 3.2.4 `ccsp_dns_resolver_query()`, `ccsp_dns_query_cancel()`
- 3.2.5 `ccsp_dns_response_t`: all fields
- 3.2.6 Record types: complete typed struct list including HTTPS/SVCB
  - HTTPS/SVCB (RFC 9460): ALPN, ECH config, Alt-Svc parameters
- 3.2.7 DNSSEC validation: chain, trust anchors, failure handling
- 3.2.8 DANE: TLSA resolution and caching alongside A/AAAA
- 3.2.9 Cache: TTL, min/max TTL, manual flush
- 3.2.10 Security: per-query random source port and query ID
- 3.2.11 `ccsp_dns_config_from_os()`: platform-specific resolver detection
- 3.2.12 mDNS routing: automatic for `.local`

### Chapter 3.3: lib_ccsp_mdns -- Multicast DNS
- 3.3.1 RFC 6762/6763 scope
- 3.3.2 Announcement, withdrawal, conflict resolution
- 3.3.3 Browsing: on_found, on_lost callbacks
- 3.3.4 DNS-SD: PTR → SRV → A/AAAA chain
- 3.3.5 lib_ccsp_connect integration
- 3.3.6 When mDNS matters in 2026 (embedded, IoT, local networks)

### Chapter 3.4: lib_ccsp_connect -- Stream Connection
- 3.4.1 The SO_ERROR miss: still in production code in 2026
- 3.4.2 `ccsp_conn_config_t`: complete field reference
- 3.4.3 Happy eyeballs RFC 8305: 250ms stagger, parallel IPv4/IPv6
- 3.4.4 QUIC-aware transport field: TCP (default), QUIC (roadmap)
- 3.4.5 HTTPS/SVCB record integration: Alt-Svc without extra RTT
- 3.4.6 SRV selection: RFC 2782 weighted random with secure RNG
- 3.4.7 DANE: TLSA usage 0/1/2/3, selector 0/1, matching 0/1/2
- 3.4.8 The connection state machine: all states and transitions
- 3.4.9 Error taxonomy: complete list
- 3.4.10 `ccsp_conn_stream_t`: established stream API
- 3.4.11 TLS info after handshake: version, cipher, ALPN, ECH status

---

## Volume 4: Protocols

### Chapter 4.1: lib_ccsp_lineproto -- Line Protocol Framework
- 4.1.1 The 47 CVEs: line readers in production code
- 4.1.2 Schema types: cmd, state, proto descriptors
- 4.1.3 Argument types: NONE, STRING, WORD, WORDS, KV, CUSTOM
- 4.1.4 `ccsp_lp_cmd_t`: all fields with semantics
- 4.1.5 `ccsp_lp_state_t`: all fields with semantics
- 4.1.6 `ccsp_lp_proto_t`: all fields with semantics
- 4.1.7 `ccsp_lp_response_t`: response struct and state transitions
- 4.1.8 Server API: `ccsp_lp_server_t`, connection management
- 4.1.9 Connection API: write, write_code, write_multiline, auth state
- 4.1.10 Data mode: DELIMITED, LENGTH, BINARY, CUSTOM
- 4.1.11 Pipelining: ordering guarantee, flush conditions
- 4.1.12 STARTTLS: flush, handshake, state reset, auth clear
- 4.1.13 Rate limiting: `max_per_minute` implementation
- 4.1.14 Client API: same schema, loopback for testing
- 4.1.15 Complete example: SMTP server skeleton
- 4.1.16 Complete example: Redis client
- 4.1.17 Complete example: memcached client
- 4.1.18 Writing a custom protocol: step-by-step

### Chapter 4.2: lib_ccsp_http -- HTTP/1.1 + HTTP/2
- 4.2.1 Version negotiation: ALPN, HTTPS record Alt-Svc
- 4.2.2 `ccsp_http_request_t`, `ccsp_http_response_t`
- 4.2.3 Bodies as `ccsp_bio_reader_t`: streaming by default
- 4.2.4 HTTP/1.1: chunked, keep-alive, Expect, trailers
- 4.2.5 HTTP/2: HPACK, multiplexing, flow control, GOAWAY
- 4.2.6 HTTP/3 roadmap: QPACK, QUIC integration, API compatibility
- 4.2.7 Client API: connection pooling, redirects, timeout
- 4.2.8 Server API: on_request callback, response writer
- 4.2.9 WebSocket upgrade: `ccsp_http_upgrade_websocket()`
- 4.2.10 Compression: gzip/deflate/brotli via bio filters
- 4.2.11 Cookie jar: `ccsp_http_cookie_jar_t`

### Chapter 4.3: lib_ccsp_smtp -- SMTP
- 4.3.1 RFC 5321 coverage, SMTP extensions supported
- 4.3.2 SMTPUTF8 (RFC 6531): internationalized addresses
- 4.3.3 DANE MX: outbound enforcement
- 4.3.4 DKIM: signing and verification
- 4.3.5 SMTP AUTH: PLAIN, XOAUTH2
- 4.3.6 PIPELINING (RFC 2920)
- 4.3.7 DSN (RFC 3461): delivery status notification
- 4.3.8 Server: `on_message` callback, envelope
- 4.3.9 Client: MX resolution, STARTTLS, send

### Chapter 4.4: lib_ccsp_imap -- IMAP4rev2
- 4.4.1 IMAP4rev2 (RFC 9051) vs IMAP4rev1 (RFC 3501): what changed
- 4.4.2 The literal challenge: LENGTH data mode integration
- 4.4.3 Authentication: LOGIN, PLAIN, XOAUTH2, AUTHENTICATE
- 4.4.4 Folder operations: LIST, SELECT, EXAMINE, CREATE, DELETE, RENAME
- 4.4.5 Message operations: FETCH, STORE, COPY, MOVE, EXPUNGE
- 4.4.6 MIME parsing: BODYSTRUCTURE, part addressing, attachment extraction
- 4.4.7 SEARCH: full grammar including UID search
- 4.4.8 IDLE (RFC 2177): push notification
- 4.4.9 CONDSTORE (RFC 7162): incremental sync with HIGHESTMODSEQ
- 4.4.10 Server: state machine, backend interface

### Chapter 4.5: lib_ccsp_dgram -- Datagram Application Framework
- 4.5.1 Why this exists: UDP CVE class and the amplification problem
- 4.5.2 `ccsp_dgram_proto_t`: complete field reference
  - opcode_offset, opcode_size, opcode_network_order
  - max_dgram_size: drop larger datagrams
  - handlers[]: NULL-terminated handler table
  - max_per_second_per_src, max_per_second_global: rate limiting
  - max_response_multiple: amplification protection enforcement
  - Callbacks: on_bind, on_rate_limited, on_amplification_blocked
- 4.5.3 `ccsp_dgram_cmd_t`: opcode, handler callback
- 4.5.4 Handler signature: opcode, datagram bytes, source addr, response builder
- 4.5.5 `ccsp_dgram_respond()`: send response datagram
- 4.5.6 Amplification protection in detail
  - The DNS/NTP/SSDP amplification pattern
  - How max_response_multiple is enforced (before calling handler)
  - Logging: WARN on blocked response, observable
  - Setting N=1 for protocols with no legitimate amplification
- 4.5.7 Rate limiting: per-source and global token buckets
- 4.5.8 Source address tracking: `ccsp_sockaddr_t`, validation
- 4.5.9 Server: `ccsp_dgram_server_create()`, bind, destroy
- 4.5.10 IPv4 + IPv6 simultaneously: dual-stack and two-socket fallback
- 4.5.11 Multicast: `ccsp_dgram_server_join_multicast()`
  - Source-specific multicast (SSM) where available
  - SO_REUSEPORT and SO_REUSEADDR: coexistence with system daemons
- 4.5.12 Client: `ccsp_dgram_client_t`
  - Correlation ID field: offset and size in protocol descriptor
  - Retransmission: configurable count, backoff, timeout
  - Response callback: result or CCSP_DGRAM_ERR_TIMEOUT
- 4.5.13 Known protocol schemas: NTP, Syslog UDP, TFTP, CoAP skeleton
- 4.5.14 Complete example: NTP client
- 4.5.15 Complete example: minimal UDP echo server with rate limiting

### Chapter 4.6: lib_ccsp_dtls -- DTLS 1.2 + 1.3
- 4.6.1 DTLS vs TLS: the unreliable transport problem
- 4.6.2 DTLS 1.2 (RFC 6347) vs DTLS 1.3 (RFC 9147): what changed
- 4.6.3 DTLS 1.0: not implemented, not configurable (deprecated by RFC 9147)
- 4.6.4 `ccsp_dtls_config_t`: configuration
  - Same cert, key, ALPN, cert_policy as lib_ccsp_tls
  - handshake_timeout_ms, handshake_retransmit_count
  - max_connections: connection table size
  - cookie_secret: for HelloVerifyRequest
- 4.6.5 The handshake over UDP
  - Fragmentation and reassembly of handshake messages
  - Retransmission timer: exponential backoff
  - Flight-based retransmission per RFC 6347
- 4.6.6 Cookie exchange (RFC 6347 §4.2.1)
  - Why: prevents handshake amplification
  - HelloVerifyRequest: cookie computation with HMAC
  - Stateless cookie validation: no per-client state before cookie verified
- 4.6.7 Connection table: keyed by (local addr, remote addr)
  - New source address: triggers handshake
  - Known source address: routes to established session
  - Session expiry: configurable idle timeout
- 4.6.8 DTLS 1.3 epoch management
  - Epochs and key material rotation
  - Retaining previous epoch keys briefly after key update
  - Reordering tolerance: accept out-of-order packets across epoch boundary
- 4.6.9 Cipher suites: same list as lib_ccsp_tls, DTLS-applicable subset
- 4.6.10 Integration with lib_ccsp_dgram: wrapping a dgram server
- 4.6.11 Integration with lib_ccsp_coap: `coaps://` URI handling
- 4.6.12 PMTU discovery: path MTU and record size limits

### Chapter 4.7: lib_ccsp_coap -- CoAP (Roadmap)
- 4.7.1 CoAP overview: REST over UDP for constrained devices
- 4.7.2 Why this is post-1.0: scope of observe + block-wise
- 4.7.3 Planned API: server URI routing, client request/response
- 4.7.4 Confirmable vs non-confirmable messages
- 4.7.5 Observe (RFC 7641): push notification subscription
- 4.7.6 Block-wise transfer (RFC 7959): large payloads over UDP
- 4.7.7 Group communication: multicast CoAP
- 4.7.8 Matter device protocol: CoAP usage in Matter
- 4.7.9 `coap://` and `coaps://` URI support

---

## Volume 5: Cryptography and Security

### Chapter 5.1: lib_ccsp_asn1 -- BER/DER Codec
- 5.1.1 The ANY situation: what we implement and what we refuse
  - `ANY DEFINED BY`: dispatch tables, OID-typed parsing
  - Bare `ANY`: opaque capture as `ccsp_asn1_any_t`, warning logged
  - Real-world cases requiring ANY DEFINED BY: X.509, PKCS#8, CMS
  - `CCSP_ASN1_ERR_UNKNOWN_OID`: unknown discriminator OIDs are errors
  - The working group argument: still made, less often won
- 5.1.2 Schema-driven parsing: schema syntax, generator, usage
- 5.1.3 Type support: full ASN.1 type list
  - `ANY DEFINED BY`: dispatch table API
  - Bare `ANY`: opaque `ccsp_asn1_any_t` capture
- 5.1.4 Strict DER: what is rejected and why
- 5.1.5 Security limits: nesting depth, object size, member count
- 5.1.6 String normalization: all types → UTF-8
- 5.1.7 OID handling: binary, dot notation, known OID registry
- 5.1.8 Encode and decode APIs
- 5.1.9 Fuzz results

### Chapter 5.2: lib_ccsp_cert -- Certificate Handling
- 5.2.1 The three modules: certparse, certval, ca-nav
- 5.2.2 certparse: `ccsp_cert_parse()`, `ccsp_cert_t` accessor reference
  - All subject/issuer DN accessors
  - All extension accessors
  - SAN types: DNS, IP, email, URI, otherName
  - Immutability guarantee
- 5.2.3 certval: RFC 5280 path validation
  - `ccsp_certval_verify()`: all parameters
  - `ccsp_certval_result_t`: complete error list
  - `ccsp_certval_policy_t`: all configuration options
  - The `at_time` parameter: never calls `time()`
  - Name constraints, policy OIDs, EKU, path length
- 5.2.4 ca-nav: path building, OCSP, CRL
  - Fetch callback pattern
  - Stapled OCSP
  - Cache TTL management
- 5.2.5 Trust store: system trust store locations per platform
- 5.2.6 DANE module: TLSA validation, usage/selector/matching matrix

### Chapter 5.3: lib_ccsp_pki -- Certificate Generation
- 5.3.1 Key generation: Ed25519 (default), X25519, P-256, P-384, RSA
  - Why Ed25519 is the default in 2026
  - Why RSA is still present: legacy interop
  - What is not present: RSA-1024, DSA
- 5.3.2 Key serialization: PKCS#8 PEM/DER, SubjectPublicKeyInfo PEM/DER
- 5.3.3 Key import: from PEM or DER
- 5.3.4 CSR generation: builder API, subject attributes, SAN, EKU
- 5.3.5 Certificate signing: builder API, validity, serial, extensions
- 5.3.6 Self-signed for DANE usage 3: the operational procedure
- 5.3.7 PKCS#12: import and export with AES-256-CBC + PBKDF2

### Chapter 5.4: lib_ccsp_ocl -- Cryptographic Primitives
- 5.4.1 Two backend modes: native and OpenSSL
  - When to use the OpenSSL backend: FIPS 140-3 compliance programs
  - API is identical: the backend is a compile/link choice
- 5.4.2 The AEAD interface: what applications use
  - `ccsp_ocl_aead_seal()`: internal nonce, structural prevention of reuse
  - `ccsp_ocl_aead_open()`: constant-time authentication
  - Algorithm selection: ChaCha20-Poly1305 vs AES-256-GCM
- 5.4.3 Hash functions: SHA-2, SHA-3, BLAKE3
  - BLAKE3: why it is included (speed, streaming, keyed, KDF modes)
  - Streaming context API for all hash functions
- 5.4.4 HMAC: HMAC-SHA256, HMAC-SHA512
- 5.4.5 Key derivation: HKDF, Argon2id (default for passwords), PBKDF2, scrypt
  - Argon2id parameters: memory cost, time cost, parallelism
- 5.4.6 Asymmetric: Ed25519, Ed448, ECDSA P-256/P-384, RSA PSS/OAEP
  - Ed25519: deterministic, fast, correct by construction
  - ECDSA: RFC 6979 deterministic nonces
  - RSA: PSS for signatures, OAEP for encryption, no PKCS#1 v1.5 signatures
- 5.4.7 Key agreement: X25519, X448, ECDH P-256/P-384
- 5.4.8 Post-quantum: X25519Kyber768 hybrid (ML-KEM, FIPS 203)
  - Why hybrid: classical security maintained if Kyber is broken
  - Compile-time option: not default, explicitly opt-in
- 5.4.9 What is absent and why: MD5, SHA-1, 3DES, RC4, AES-ECB, RSA-1024
- 5.4.10 Constant-time discipline: verification methodology and CI thresholds
- 5.4.11 FIPS 140-3 boundary: for the native backend
- 5.4.12 Hardware acceleration: AES-NI, ARMv8 crypto, runtime detection
- 5.4.13 NIST CAVP test vectors: coverage per algorithm

### Chapter 5.5: lib_ccsp_tls -- TLS 1.2 + 1.3
- 5.5.1 TLS 1.0 and 1.1: not implemented, not configurable, RFC 8996
- 5.5.2 Supported cipher suites: the list and what is excluded
- 5.5.3 The Heartbleed structural fix: record/handshake separation
- 5.5.4 `ccsp_tls_config_t`: complete field reference
- 5.5.5 Explicit state machine: table of states and transitions
- 5.5.6 SNI: required, Encrypted Client Hello (ECH) when negotiated
- 5.5.7 Forward secrecy: mandatory, no config option
- 5.5.8 No renegotiation: close + log behavior
- 5.5.9 ALPN: negotiation and `ccsp_tls_get_alpn()`
- 5.5.10 Session tickets (TLS 1.3): key rotation, replay prevention
- 5.5.11 OCSP stapling: client acceptance, server proactive fetch
- 5.5.12 Certificate validation: certval policy, DANE, pinning
- 5.5.13 QUIC transport: exported seal/open functions for QUIC integration
- 5.5.14 Post-quantum: X25519Kyber768 hybrid key exchange (when enabled)
- 5.5.15 Handshake timing: log events and performance measurement

---

## Volume 6: Data and CLI

### Chapter 6.1: lib_ccsp_data -- Shared Value Type
- 6.1.1 `ccsp_data_value_t`: type enum and union
- 6.1.2 Integer as int64_t: why and what it fixes
- 6.1.3 Dot-path navigation: `ccsp_data_get()`
- 6.1.4 Typed accessors with default values
- 6.1.5 Array and table access
- 6.1.6 Iteration: `ccsp_data_each_array()`, `ccsp_data_each_table()`
- 6.1.7 Serialization to JSON and TOML
- 6.1.8 Immutability: `ccsp_data_clone()` for modification
- 6.1.9 Insertion order: the lib_ccsp_hash guarantee

### Chapter 6.2: lib_ccsp_toml -- TOML 1.0
- 6.2.1 TOML 1.0 spec coverage
- 6.2.2 Parse API: `ccsp_toml_parse()`, `ccsp_toml_parse_file()`
- 6.2.3 Error type: line, column, message
- 6.2.4 Type coverage: all TOML 1.0 types including datetime
- 6.2.5 Multiline strings
- 6.2.6 Path navigation and defaults
- 6.2.7 Round-trip serialization: key order preserved
- 6.2.8 lib_ccsp_cli integration
- 6.2.9 Conformance: toml-test suite results

### Chapter 6.3: lib_ccsp_json -- JSON RFC 8259
- 6.3.1 DOM and streaming APIs
- 6.3.2 Streaming parser: incremental feed, event types
- 6.3.3 lib_ccsp_event integration for network streaming
- 6.3.4 Security limits: depth, string, document
- 6.3.5 UTF-8 validation: invalid sequences are errors
- 6.3.6 Number parsing: strtoll and strtod
- 6.3.7 Serialization: compact and pretty-printed
- 6.3.8 JSONTestSuite compliance

### Chapter 6.4: lib_ccsp_cbor -- CBOR RFC 8949
- 6.4.1 Why CBOR and not protobuf
  - No schema compiler, no generated code, self-describing
  - IETF binary format: COSE, CWT, EDHOC, CoAP, FIDO2/WebAuthn
  - When protobuf is the right choice instead: high-volume RPC, both ends controlled
- 6.4.2 Type mapping: CBOR major types to `ccsp_data_value_t`
  - Integer, negative int → CCSP_DATA_INTEGER
  - Byte string → CCSP_DATA_BYTES
  - Text string → CCSP_DATA_STRING (UTF-8 validated on decode)
  - Array → CCSP_DATA_ARRAY
  - Map → CCSP_DATA_TABLE
  - Tags: preserved as metadata
  - Bignum tags 2 and 3: returned as CCSP_DATA_BYTES with tag
- 6.4.3 DOM decoder: `ccsp_cbor_parse()`
  - Security limits: nesting depth 64, item count 65536, string 16MB
  - CBOR sequences (RFC 8742): multiple items without wrapper
- 6.4.4 Streaming decoder: event-based, incremental feed
  - Same event pattern as lib_ccsp_json streaming
  - For CoAP payloads and constrained devices
  - Indefinite-length arrays and maps: supported on decode
- 6.4.5 Streaming encoder: `ccsp_cbor_enc_t`
  - Always definite-length: smaller and faster to parse
  - `ccsp_cbor_enc_array_begin()`, `ccsp_cbor_enc_map_begin()`
  - `ccsp_cbor_enc_tag()`: CBOR tags for typed values
  - `ccsp_cbor_enc_value()`: encode a `ccsp_data_value_t` directly
- 6.4.6 Non-string map keys: decode support, encode limitation
- 6.4.7 CBOR diagnostic notation: human-readable for debugging
- 6.4.8 COSE integration roadmap: COSE_Sign1, COSE_Encrypt0, COSE_Mac0

### Chapter 6.5: lib_ccsp_cas -- Content-Addressable Storage
- 6.5.1 Object model: blob, tree, commit
- 6.5.2 Store backends: filesystem, in-memory, remote
- 6.5.3 `ccsp_cas_put()`, `ccsp_cas_get()`: write and read with verification
- 6.5.4 Tree operations: entries, pack, unpack
- 6.5.5 Commit operations: tree, parent, metadata
- 6.5.6 Use cases with examples: build systems, packages, config, sync

### Chapter 6.6: lib_ccsp_cli -- Declarative Option Parsing
- 6.6.1 The getopt problem: types, help drift, no source tracking
- 6.6.2 `ccsp_cli_opt_t`: complete field reference
- 6.6.3 All option types with examples and storage patterns
- 6.6.4 `ccsp_cli_cmd_t`: subcommands and dispatch
- 6.6.5 `ccsp_cli_parser_t` configuration
- 6.6.6 Source priority chain: four sources, one rule
- 6.6.7 Source tracking: `ccsp_cli_result_t`, `ccsp_cli_print_effective_config()`
- 6.6.8 Help format: exact output, cannot drift
- 6.6.9 `--no-*` negation: automatic for booleans
- 6.6.10 TOML config integration
- 6.6.11 Complete example: a real tool with subcommands and config file

---

## Volume 7: Graphics and UI

### Chapter 7.1: lib_ccsp_draw -- Backend-Agnostic Drawing
- 7.1.1 Design principles: float coords, text pre-decomposed, frame lifecycle
- 7.1.2 `ccsp_draw_backend_vtable_t`: twelve required, two optional
- 7.1.3 Color model: three layers
  - Storage: `ccsp_draw_color_t` -- four float channels, linear light
    - Why linear light: Porter-Duff is defined there, GPU expects it
    - Range [0.0, 1.0] SDR; above 1.0 valid for HDR and wide-gamut (P3, Rec. 2020)
    - `CCSP_DRAW_COLOR()`: construct from linear-light floats
  - Manipulation: Oklab / OKLCh
    - Why not linear RGB: perceptually non-uniform, gradients go muddy
    - Why not HSL: hue rotation is wrong, lightness is wrong
    - Why Oklab: perceptually uniform, good hue linearity, simple math
    - `ccsp_draw_color_from_oklch()`, `ccsp_draw_color_from_oklab()`
    - `ccsp_draw_color_to_oklch()`: round-trip for inspection
    - `ccsp_draw_color_mix()`: interpolates through Oklab, no hue shift
    - `ccsp_draw_color_lighten()`, `ccsp_draw_color_darken()`: move L axis
    - `ccsp_draw_color_saturate()`, `ccsp_draw_color_desaturate()`: move C axis
    - `ccsp_draw_color_rotate_hue()`: move h axis, no gamut shift
    - `ccsp_draw_color_with_alpha()`: alpha channel only
    - Gradient interpolation uses `ccsp_draw_color_mix()` internally
  - Input: sRGB convenience
    - `ccsp_draw_color_from_srgb8()`: applies actual sRGB transfer function
    - `ccsp_draw_color_from_hex()`: packed 0xRRGGBBAA, sRGB→linear
    - Why from_srgb8(128,128,128) ≠ {0.502, 0.502, 0.502}: the math
    - Named color constants: WHITE, BLACK, TRANSPARENT
  - Backend output: sRGB transfer, wide-gamut swapchain, hardware scan-out
  - Wire formats (RGB565, ARGB8888): live in backend, not in this type
- 7.1.4 Paint types: solid, linear gradient, radial gradient, image pattern
- 7.1.4 Path operations: move, line, quad bezier, cubic bezier, arc, close
- 7.1.5 Drawing operations: fill_rect, stroke_rect, fill_path, stroke_path,
  draw_image, draw_glyph
- 7.1.6 Graphics state: save, restore, clip_rect, clip_path, transform, opacity
- 7.1.7 Frame lifecycle: begin, end, present -- and why they are three operations
- 7.1.8 Off-screen surfaces and compositing
- 7.1.9 Damage tracking: mark_dirty, dirty_rect accumulation
- 7.1.10 Backend detection: `ccsp_draw_backend_detect()` priority order
- 7.1.11 Adding a new backend: checklist and template file

### Chapter 7.2: lib_ccsp_draw_wayland -- Primary Linux Backend
- 7.2.1 Wayland in 2026: why this is primary, not X11
- 7.2.2 libwayland-client: connection, registry, globals
- 7.2.3 xdg-shell: window management protocol
  - `xdg_surface`, `xdg_toplevel`
  - Window title, maximize, minimize, resize, fullscreen
  - Configure/ack_configure cycle
- 7.2.4 xdg-decoration: server-side vs client-side decorations
- 7.2.5 High-DPI: `wl_output.scale`, fractional scaling `wp-fractional-scale-v1`
- 7.2.6 wl_shm: shared memory buffer, software rasterizer path
  - Buffer allocation and format selection (ARGB8888, etc.)
  - Damage: `wl_surface_damage_buffer()` for partial updates
  - Double buffering: frame callbacks for vsync
- 7.2.7 wl_dmabuf: zero-copy GPU texture sharing
- 7.2.8 EGL: OpenGL ES context for GPU backend
- 7.2.9 Vulkan: `VK_KHR_wayland_surface`
- 7.2.10 Input: `wl_seat`, `wl_pointer`, `wl_keyboard`, `wl_touch`
  - Pointer events: logical float coordinates
  - Keyboard: keymap (libxkbcommon), key events, focus
  - Touch: multi-touch, tracking IDs
- 7.2.11 IME: `zwp-text-input-v3` protocol
  - Preedit string delivery
  - Cursor position for IME window placement
  - Commit string
- 7.2.12 Clipboard: `wl_data_device`, `wl_data_source`, `wl_data_offer`
- 7.2.13 Drag and drop: `wl_data_device` drag API
- 7.2.14 Layer shell: `wlr-layer-shell` for panels and overlays
  - Available on: Sway, Wayfire, KWin, Weston
  - Not available on: GNOME (mutter has own protocol)
- 7.2.15 Notifications: org.freedesktop.Notifications D-Bus
- 7.2.16 File picker: XDG Desktop Portal `FileChooser`
  - Works under X11 and Wayland, sandboxing-compatible
- 7.2.17 XWayland compatibility: when the compositor runs XWayland,
  lib_ccsp_draw_x11 works through it; no special handling needed
- 7.2.18 Compositor compatibility matrix: GNOME, KDE Plasma, Sway, Hyprland,
  Wayfire, Weston

### Chapter 7.3: lib_ccsp_draw_x11 -- Legacy/Embedded Backend
- 7.3.1 When to use X11 directly in 2026: embedded, custom compositors
- 7.3.2 XLib + XRender: antialiased compositing
- 7.3.3 XShm: shared memory for zero-copy local display
- 7.3.4 XInput2: multi-touch, high-precision pointer
- 7.3.5 Damage extension: partial repaint
- 7.3.6 EWMH: window manager hints for desktop integration
- 7.3.7 IME via XIM: the painful truth about XIM

### Chapter 7.4: lib_ccsp_draw_fb -- Framebuffer Backend
- 7.4.1 `/dev/fb0` and custom embedded targets
- 7.4.2 Pixel format detection and conversion
- 7.4.3 The `flush()` callback: DMA, SPI, MMIO
- 7.4.4 Software rasterizer: Bresenham lines, scanline fill
- 7.4.5 Double buffering: page-flip where supported, memcpy fallback

### Chapter 7.5: lib_ccsp_draw_img -- Software Rasterizer / Reference
- 7.5.1 The reference implementation role: when GPU differs, img is correct
- 7.5.2 Output formats: PNG (libpng), PPM, raw RGBA, PDF
- 7.5.3 `ccsp_draw_img_diff()`: pixel comparison API
- 7.5.4 Visual regression testing workflow
- 7.5.5 Documentation generation: rendering examples from code

### Chapter 7.6: lib_ccsp_draw_gpu -- Vulkan + GLES3 Backend
- 7.6.1 Why Vulkan in 2026: availability, explicit model, performance
- 7.6.2 Why GLES3 fallback: embedded targets without Vulkan
- 7.6.3 What OpenGL 2.1 is: a museum piece, not a target
- 7.6.4 Vulkan implementation
  - Command buffer per frame
  - Render pass: single subpass for 2D
  - Pipeline state cache: by draw mode and blend state
  - Vertex buffer: updated per frame, dynamic geometry
  - Glyph atlas: VkImage, descriptor set, cache management
  - Path tessellation: libtess2 (Skia tessellator, MIT)
  - Damage: partial framebuffer update via scissor
- 7.6.5 GLES3 fallback implementation
  - VAO, VBO, UBO
  - Same tessellator and glyph atlas approach
  - EGL context management
- 7.6.6 Platform surface integration
  - Wayland: `VK_KHR_wayland_surface`, `wl_egl_window`
  - X11: `VK_KHR_xlib_surface`, GLX
  - Windows: `VK_KHR_win32_surface`, WGL
  - macOS: MoltenVK + `VK_EXT_metal_surface`
- 7.6.7 Shader compilation: GLSL → SPIR-V via glslang at build time,
  not runtime
- 7.6.8 Memory management: VMA (Vulkan Memory Allocator, MIT)

### Chapter 7.7: lib_ccsp_font -- FreeType Wrapper
- 7.7.1 FreeType 2: what it handles, what we add
- 7.7.2 `ccsp_font_load_file()`, `ccsp_font_load_mem()`
- 7.7.3 Variable fonts: variation axes, named instances
- 7.7.4 Glyph rasterization: `ccsp_font_rasterize()`
- 7.7.5 Font metrics: ascender, descender, line height, cap height
- 7.7.6 Kerning: `ccsp_font_kern_advance()` (for non-HarfBuzz path)
- 7.7.7 Glyph atlas: `ccsp_font_atlas_t`, shelf packing, LRU eviction
- 7.7.8 Subpixel rendering: optional, platform-dependent

### Chapter 7.8: lib_ccsp_text -- Text Layout
- 7.8.1 The pipeline: string → HarfBuzz shaped → bidi reordered →
  line broken → positioned glyphs
- 7.8.2 HarfBuzz integration: shaping, OpenType features, scripts
  - Variable font shaping: axis values passed through
  - Script and language tags
  - Feature enable/disable: liga, kern, calt, etc.
- 7.8.3 Fallback shaping: advance-width for Latin without HarfBuzz
- 7.8.4 Unicode UAX#9 bidirectional algorithm: full implementation
- 7.8.5 Unicode UAX#14 line breaking: mandatory, opportunity, prohibited
- 7.8.6 Paragraph layout API: `ccsp_text_paragraph_t`, `ccsp_text_layout_t`
- 7.8.7 Justification: none, left, center, right, full
- 7.8.8 Hyphenation: TeX pattern algorithm, language pattern files
- 7.8.9 Hit testing: codepoint offset from pixel position
- 7.8.10 Selection rectangles: handles bidirectional, multi-line
- 7.8.11 Emoji: emoji presentation, skin tone modifiers, ZWJ sequences
- 7.8.12 Color fonts: COLR/CPAL, sbix, CBDT (via FreeType)

### Chapter 7.9: lib_ccsp_widget -- Flutter Layout Model
- 7.9.1 The three layout rules: constraints↓, sizes↑, parent positions
- 7.9.2 Why O(n) matters: comparison with GTK and Qt layout
- 7.9.3 `ccsp_wid_constraints_t`, `ccsp_wid_size_t`, `CCSP_WID_INFINITY`
- 7.9.4 The five widget methods: the complete interface
- 7.9.5 Layout widgets: complete API for each
  - flex (row/column, main-axis and cross-axis alignment)
  - stack (z-axis layering, positioned children)
  - constrained, padded, expanded, aligned
  - scrollable (axis, scroll position, animate to)
  - sized_box, aspect_ratio, wrap
- 7.9.6 Leaf widgets: complete API for each
  - text (style, overflow, max lines, selectable)
  - image (fit modes, lazy load)
  - colored_box (paint, border radius, border, shadow)
  - custom_paint (measure + paint callbacks)
  - placeholder (invisible spacer)
- 7.9.7 Gesture detectors: tap, long_press, drag, hover, double_tap, scale
  - Gesture arena: conflict resolution algorithm
- 7.9.8 Focus and keyboard navigation
  - focusable wrapper, focus scope, focus traversal
  - Programmatic focus: `ccsp_wid_focus()`, `ccsp_wid_unfocus()`
- 7.9.9 Accessibility: descriptor, roles, AT-SPI2 tree generation
- 7.9.10 Dirty region tracking and partial repaint
- 7.9.11 Custom widget implementation: the template
- 7.9.12 Visual regression testing workflow

### Chapter 7.10: lib_ccsp_anim -- Animation Engine
- 7.10.1 The model: a float over time
- 7.10.2 `ccsp_anim_create()`: all parameters
- 7.10.3 Curves: linear, ease variants, spring physics
  - Spring parameters: mass, stiffness, damping, overshoot
- 7.10.4 Control: start, stop, reverse
- 7.10.5 Sequences, parallel groups, staggered groups
- 7.10.6 Integration with lib_ccsp_widget: the on_value + mark_dirty pattern

### Chapter 7.11: lib_ccsp_input -- Input Normalization
- 7.11.1 `ccsp_input_event_t`: all event types and fields
- 7.11.2 Float coordinates: Wayland's logical model carried through
- 7.11.3 Text input vs key events: why the distinction matters
- 7.11.4 `ccsp_input_keycode_t`: platform-independent key codes
- 7.11.5 Wayland input: wl_seat, wl_pointer, wl_keyboard, wl_touch
- 7.11.6 zwp-text-input-v3: IME protocol under Wayland
- 7.11.7 evdev: direct device input without windowing system
- 7.11.8 X11, macOS, Win32 adapters

### Chapter 7.12: lib_ccsp_a11y -- Accessibility Adapters
- 7.12.1 AT-SPI2 (Linux): D-Bus, Orca compatibility, Wayland support
- 7.12.2 AT-SPI2 role and state mappings: complete table
- 7.12.3 NSAccessibility (macOS): protocol, roles, actions
- 7.12.4 UIA (Windows): IUIAutomationProvider, control types, patterns
- 7.12.5 Testing accessibility: AT-SPI2 Inspector, Accessibility Inspector (macOS)

### Chapter 7.13: lib_ccsp_style -- Theming
- 7.13.1 `ccsp_style_theme_t`: all fields
- 7.13.2 Typography tokens
- 7.13.3 Color tokens: semantic names and their meaning
  - All tokens are `ccsp_draw_color_t` (linear light storage)
  - Define theme colors via `ccsp_draw_color_from_oklch()`: hue, chroma, lightness
  - Derive tonal palette: `ccsp_draw_color_lighten()` / `ccsp_draw_color_darken()` in OKLCh
  - Dark mode: adjust L axis, keep hue and chroma; do not invert RGB
  - Accessible contrast: measure in OKLCh L difference, not relative luminance formula
- 7.13.4 Geometry tokens: spacing system
- 7.13.5 Motion tokens: timing and curves
- 7.13.6 Dark and high-contrast variants
- 7.13.7 Runtime theme switching

### Chapter 7.14: lib_ccsp_app -- Application Lifecycle
- 7.14.1 `ccsp_app_t` and initialization
- 7.14.2 Window management: Wayland-native on Linux
- 7.14.3 System menus: xdg-shell on Wayland, EWMH on X11, native on Win/macOS
- 7.14.4 File pickers: XDG Desktop Portal (Wayland + X11), native Win/macOS
- 7.14.5 System notifications: libnotify/D-Bus (Linux), native Win/macOS
- 7.14.6 Multi-window management
- 7.14.7 Run loop: `ccsp_app_run()`, `ccsp_app_quit()`

---

## Volume 8: Tooling

### Chapter 8.1: lib_ccsp_test -- Testing Infrastructure
- 8.1.1 Test runner: TAP, JUnit XML, timeout per case
- 8.1.2 Assertion macros: the complete list
- 8.1.3 Mock allocator: tracking, failing, limited
- 8.1.4 Capture log backend: assertion API
- 8.1.5 Mock DNS resolver: test table configuration
- 8.1.6 Mock TLS pair: in-memory, no socket required
- 8.1.7 Property-based testing: generators, shrinking, fixed seeds
- 8.1.8 Known-answer test vectors: format and runner
- 8.1.9 Visual regression: workflow, thresholds, update procedure

### Chapter 8.2: lib_ccsp_fuzz -- Fuzzing Infrastructure
- 8.2.1 Complete fuzz target list
  - lib_ccsp_asn1: DER parse, round-trip
  - lib_ccsp_cert: parse, validate
  - lib_ccsp_json: DOM, streaming
  - lib_ccsp_toml: parse
  - lib_ccsp_cbor: DOM, streaming, sequence
  - lib_ccsp_dns: response parse
  - lib_ccsp_lineproto: input dispatch
  - lib_ccsp_dgram: dispatch (any opcode), NTP schema
  - lib_ccsp_http: request, response
  - lib_ccsp_tls: record, server handshake, client handshake
  - lib_ccsp_dtls: record, server handshake, cookie
  - lib_ccsp_regex: pattern compile, match
  - lib_ccsp_ocl: aead_open (must return auth failure, not crash)
  - lib_ccsp_coap: request, observe (when implemented)
- 8.2.2 Writing a new fuzz target: template and checklist
- 8.2.3 LibFuzzer: flags, corpus, sanitizer interaction
- 8.2.4 AFL++: modes, persistent mode, instrumentation
- 8.2.5 OSS-Fuzz integration files
- 8.2.6 Corpus management: minimize, merge, seed corpora
- 8.2.7 CI integration: 100k iterations per target per PR, crash = build failure
- 8.2.8 Crash triage: reproduction, deduplication, minimization

### Chapter 8.3: ccsp-check -- Static Analysis
- 8.3.1 Installation and invocation
- 8.3.2 CRITICAL findings: complete list with vulnerable/fixed examples
  - gets(), gethostbyname(), hand-rolled line reader
  - malloc(a+b/a*b) without overflow check
  - sprintf (not snprintf), strcpy, strcat
  - rand()/random() in crypto context
  - direct malloc call in library code
- 8.3.3 HIGH findings: complete list with vulnerable/fixed examples
  - select() without FD_SETSIZE check
  - hand-rolled connect without SO_ERROR check
  - strcmp/memcmp on secrets
  - atoi/atol/atof (no error detection)
  - signal() (use sigaction)
  - UDP recvfrom loop without per-source rate limiting (v1.1+)
  - UDP response larger than request without explicit amplification limit (v1.1+)
  - hand-rolled DTLS state machine (v1.1+)
  - CBOR decode without nesting depth limit (v1.2+)
- 8.3.4 INFO findings: complete list
  - strlen() in hot path
  - printf/fprintf in library code (use structured logging)
  - exit()/abort() outside main()
  - cyclomatic complexity > 20
  - UDP server not using lib_ccsp_dgram (v1.1+)
- 8.3.5 Suppression syntax and justification requirement
- 8.3.6 The suppression audit report: format and use in security reviews
- 8.3.7 CI integration: exit codes, CRITICAL blocks build, HIGH blocks merge
- 8.3.8 Custom rules: rule format, registration, corpus-driven generation
- 8.3.9 ccsp-corpus export: generate rules from corpus entries
- 8.3.10 Version history: which rules were added in which ccsp-check release

### Chapter 8.4: The CVE Corpus Database
- 8.4.1 Schema: all fields
- 8.4.2 The 12 patterns: definitions, CVE count, mean latency to disclosure
- 8.4.3 UDP amplification pattern: documented separately, added v1.1
- 8.4.4 ccsp-corpus query tool: list, stats, search, export
- 8.4.5 Contributing: format, review, classification
- 8.4.6 Dispute process
- 8.4.7 Using the corpus in procurement and security reviews

---

## Volume 9: Reference

### Chapter 9.1: Complete Error Code Reference
All CCSP_*_ERR_* codes from all libraries, with description, common
cause, and resolution.

### Chapter 9.2: Complete Log Key Reference
All standard log keys: name, type, libraries that emit it, meaning.

### Chapter 9.3: Symbol Namespace Reference
Complete tables: symbol prefixes, header files, error code prefixes,
log domain strings.

### Chapter 9.4: ABI Stability Policy
What is stable within major version. What changes between minors.
The `ccsp-abi` checker: how to run, what it reports, ABI dump format.

### Chapter 9.5: Versioning Policy
Semantic versioning. Release branches. Security patch backport.
EOL policy.

### Chapter 9.6: Platform Support Matrix

**Tier 1:** Full support, CI coverage, release-blocking failures.
Linux x86_64 (GCC 13, Clang 17), Linux aarch64 (GCC 13, Clang 17),
macOS 14 (Apple Clang 15), Windows 11 (MSVC 2022).

**Tier 2:** Best effort, CI coverage, non-blocking.
FreeBSD 14, Linux RISC-V 64.

**Tier 3:** Community maintained, no CI.
NetBSD, OpenBSD, musl-based Linux distributions, embedded bare-metal
with custom platform table.

### Chapter 9.7: Migration Guide from OpenSSL
- API mapping: SSL_CTX, SSL, BIO equivalents
- Error model differences
- BIO vs lib_ccsp_bio: the conceptual rewrite
- TLS config differences
- Certificate handling comparison
- Common pitfalls when porting

### Chapter 9.8: Migration Guide from GLib
- Type mappings: GList, GHashTable, GString, GError
- Why GObject is not needed in lib_ccsp_widget
- GMainLoop to lib_ccsp_event
- GIOChannel to lib_ccsp_bio
- GError to ccsp_err_t

### Chapter 9.9: Migration Guide from libevent 1.x/2.x
- API mapping: event_base, bufferevent, evhttp, evdns
- Callback signature changes
- Buffer management differences

### Chapter 9.10: Security Disclosure Policy
Reporting process. Embargo period (90 days). CVE assignment.
Project member 48-hour advance notification. Public disclosure
format and timeline.

### Chapter 9.11: Contributing
C17 code style. Test requirements for PRs. Documentation requirements.
ccsp-check compliance: no CRITICAL findings. ABI compatibility rules.
Commit message format. Review process and merge criteria.

---

# Part VII: Roadmap

## Delivery Milestones

**Milestone 1.0: Core Networking and Crypto Stack**

Libraries: lib_ccsp_base, lib_ccsp_event, lib_ccsp_mpi, lib_ccsp_regex,
lib_ccsp_charconv, lib_ccsp_bio, lib_ccsp_dns, lib_ccsp_connect,
lib_ccsp_asn1, lib_ccsp_cert, lib_ccsp_pki, lib_ccsp_ocl, lib_ccsp_tls,
lib_ccsp_cli, lib_ccsp_test, lib_ccsp_fuzz, ccsp-check v1.0,
CVE corpus v1.0 (all 12 patterns, 200+ entries).

Criteria: All libraries pass conformance suite on Tier 1 platforms.
ccsp-check passes on all library source. No CRITICAL findings in any
library. FIPS 140-3 validation initiated for lib_ccsp_ocl native
backend.

**Milestone 1.1: Protocol Stack**

Libraries: lib_ccsp_lineproto, lib_ccsp_dgram, lib_ccsp_dtls,
lib_ccsp_http, lib_ccsp_smtp, lib_ccsp_imap, lib_ccsp_mdns, lib_ccsp_pki
(complete).

Criteria: lib_ccsp_lineproto: SMTP server passes RFC 5321 conformance.
lib_ccsp_http: HTTP/1.1 and HTTP/2 pass relevant test suites. DANE
enforcement validated. lib_ccsp_dgram: NTP client and server schemas
validated against reference implementations. lib_ccsp_dtls: DTLS 1.2
and 1.3 interop verified against OpenSSL and wolfSSL.

**Milestone 1.2: Data Stack and Tooling Complete**

Libraries: lib_ccsp_data, lib_ccsp_toml, lib_ccsp_json, lib_ccsp_cbor,
lib_ccsp_cas, ccsp-check v1.1 (dgram amplification rules added).

Criteria: lib_ccsp_json passes JSONTestSuite. lib_ccsp_toml passes
toml-test. lib_ccsp_cbor passes RFC 8949 test vectors and CBOR
interop tests. ccsp-check identifies all 12 CVE patterns plus the
new UDP server patterns. OSS-Fuzz integration live.

**Milestone 2.0: Graphics Stack**

Libraries: lib_ccsp_draw (all backends), lib_ccsp_font, lib_ccsp_text,
lib_ccsp_widget, lib_ccsp_anim, lib_ccsp_input, lib_ccsp_a11y,
lib_ccsp_style, lib_ccsp_app.

Criteria: Flutter layout model test vectors pass. Visual regression
suite passes (imagemap vs Wayland pixel-identical on reference widget
tree). AT-SPI2 validated with Orca. NSAccessibility validated on
macOS 14. UIA validated on Windows 11. Vulkan backend passes visual
regression. GLES3 backend passes visual regression.

**Milestone 2.1: FIPS Validation**

Scope: lib_ccsp_ocl native backend and lib_ccsp_tls.

Criteria: CMVP FIPS 140-3 certificate issued.

**Milestone 2.2: HTTP/3 and QUIC**

Scope: QUIC transport in lib_ccsp_connect, HTTP/3 in lib_ccsp_http,
QUIC integration in lib_ccsp_tls (exported seal/open).

Criteria: HTTP/3 interop against curl, nginx, and Cloudflare.
QUIC draft and RFC 9000 conformance suite.

**Milestone 3.0: IoT and Constrained Devices**

Libraries: lib_ccsp_coap (full implementation), COSE in lib_ccsp_cbor,
lib_ccsp_cbor EDHOC support.

Criteria: CoAP interop against libcoap and Eclipse Californium.
Matter device protocol validation. EDHOC key exchange against IETF
test vectors.

**Milestone 3.1: Ecosystem**

Goals: curl alternate DNS/TLS backend contributed upstream. OpenSSH
contributed patches for lib_ccsp_dns async resolver. LSB behavioral
spec published with lib_ccsp_base as reference implementation. C
committee Annex S TR (Systems Programming Interfaces) published.
ccsp-check integrated into at least one major Linux distribution's
package review tooling.

## New ccsp-check Rules (Post-1.0)

UDP server patterns added in ccsp-check 1.1:

**HIGH findings (block merge, require justification):**
- Any UDP recvfrom loop without per-source rate limiting
- Any UDP server that sends a response larger than the request without
  an explicit `max_response_multiple` configuration
- Any hand-rolled DTLS state machine (use lib_ccsp_dtls)

**INFO findings:**
- Any UDP server not using lib_ccsp_dgram (report, not block -- there
  are legitimate reasons to hand-roll NTP or syslog receivers)

CBOR-specific:
- Any hand-rolled CBOR decoder without nesting depth limit
- Any CBOR decode that uses the result of a tag without validating
  the tag value

## New Fuzz Targets (Post-1.0)

lib_ccsp_dgram:
- `ccsp_fuzz_dgram_dispatch` -- any bytes as incoming datagram, any
  opcode, must not crash
- `ccsp_fuzz_dgram_ntp` -- any bytes as NTP packet

lib_ccsp_dtls:
- `ccsp_fuzz_dtls_record` -- any bytes as DTLS record
- `ccsp_fuzz_dtls_handshake_server` -- any bytes as ClientHello
- `ccsp_fuzz_dtls_cookie` -- any bytes as cookie in HelloVerifyRequest

lib_ccsp_cbor:
- `ccsp_fuzz_cbor_dom` -- any bytes as CBOR document
- `ccsp_fuzz_cbor_streaming` -- any bytes fed incrementally
- `ccsp_fuzz_cbor_sequence` -- any bytes as CBOR sequence

lib_ccsp_coap (when implemented):
- `ccsp_fuzz_coap_request` -- any bytes as CoAP message
- `ccsp_fuzz_coap_observe` -- any bytes as observe notification

---

## Appendix A: The CVE Corpus Patterns

All 12 patterns with: definition, canonical vulnerable code (C17),
canonical fixed code using lib_csp, CVE list from corpus, mean
latency, structural prevention mechanism, ccsp-check rule.

## Appendix B: Design Decisions Not Taken

This appendix documents decisions that come up repeatedly and explains
why they were rejected. The short answer to most of them is Principle
10. The longer answers follow.

### B.1: GLib's GObject

GObject provides runtime type identity, reference counting, a signal
and property system, and object inheritance in C. It requires a
code-generation step (`glib-mkenums`, `glib-genmarshal`), a marshaling
layer, and enough boilerplate per type that experienced GLib developers
use macros to hide it. The result is code that is harder to read than
C++, harder to audit than C, and provides safety guarantees weaker than
either.

lib_ccsp_widget uses a vtable (five function pointers per widget type).
That is the object system. It covers everything the widget system needs.
If you need full GObject -- signals, property change notification,
runtime type queries -- you are building a GNOME application and should
use GTK, which is designed around GObject and handles the complexity on
your behalf.

### B.2: Qt's moc

Qt's meta-object compiler preprocesses C++ to add signals, slots,
runtime type information, and property introspection. It is a separate
compilation step that generates C++ from C++ annotations. The generated
code is not readable. The build system integration is non-trivial.

We do not use C++. moc is irrelevant. This entry exists because the
"why not signals and slots" question comes up independently of Qt, and
the answer is the same: signals and slots are a runtime callback
registration system with dynamic dispatch. We have function pointers
and explicit callback registration. They are the same thing with less
machinery.

### B.3: A Custom Allocator by Default

The platform table allows replacing the allocator. This is for embedded
targets with static memory pools, for testing with the mock allocator,
and for performance-sensitive deployments that want TLSF or jemalloc.

The default is the system allocator. Not tcmalloc. Not jemalloc. Not a
pool. The system allocator because: it is always present, it is what
everyone has debugged against, Valgrind and ASan understand it natively,
and the performance difference for systems code -- as opposed to
allocation-heavy application code -- is usually unmeasurable.

The allocator is replaceable. Replace it when you have a measurement
that justifies it. Not before.

### B.4: Reference Counting in lib_ccsp_base

`ccsp_cert_t` uses reference counting because parsed certificates are
genuinely shared across a connection's lifetime with no clear single
owner. This is the one place in the stack where reference counting
is the correct answer to a real ownership problem.

A general reference counting infrastructure in lib_ccsp_base would be
used everywhere, including places where ownership is clear and explicit
`_create()`/`_destroy()` pairs are sufficient. Reference counting in
those places hides ownership, makes use-after-free bugs harder to find
(the counter is wrong, not the pointer), and adds atomic operations to
every assignment.

If you need shared ownership of something other than certificates,
model the ownership explicitly. If you cannot, that is a design problem
in your code, not a missing feature in lib_ccsp_base.

### B.5: CSS for Styling

CSS is a document styling language retrofitted onto application UI
through a process of accretion that produced specificity rules,
cascade semantics, the box model, flexbox, grid, custom properties,
and `!important`. It is expressive and it is a debugging nightmare.

lib_ccsp_style is a struct. You set fields. The field values are the
design tokens. There is no specificity. There is no cascade. There is
no inheritance except what you implement explicitly by passing a
modified theme downward through the widget tree. Dark mode is a
different struct. High contrast is a different struct. The entire
theming system is greppable.

### B.6: YAML

YAML is a superset of JSON that adds anchors, aliases, multi-line
strings, multiple document streams, and seventeen ways to write a
string literal, four of which behave differently with trailing newlines.
The spec is 80 pages. The Norway problem (the string `NO` parsed as
boolean `false` in some implementations) is a real thing that has
caused real production incidents.

TOML covers configuration files. JSON covers data interchange. YAML
covers neither better than these alternatives and adds complexity that
has repeatedly produced security-relevant parser inconsistencies across
implementations. We do not ship a YAML parser and we do not plan to.

### B.7: XML

XML is appropriate for document markup and for some structured data
interchange formats that predate JSON. This stack uses JSON for data
interchange and TOML for configuration. We do not need XML for either
of these purposes.

If you are implementing a protocol that uses XML (XMPP, SOAP, some
CalDAV/CardDAV operations), write a targeted parser for the specific
schema. A general-purpose XML parser is not a 1.0 deliverable. If
demand is sufficient, it becomes lib_ccsp_xml with strict schema
validation and the same security limits as lib_ccsp_json.

### B.8: ICU for All Unicode Operations

ICU is the authoritative Unicode implementation. It also weighs 30MB,
has a complex build system, and handles many things this stack does not
need (locale-sensitive collation, transliteration, BIDI cursors in
complex editing scenarios, full CLDR data).

lib_ccsp_text uses HarfBuzz for shaping and implements UAX#9 and UAX#14
directly. These cover the cases the widget system needs. ICU is
available as an opt-in for applications that need normalization,
locale-sensitive sorting, or full CLDR locale data. It is not a
dependency of the stack.

### B.9: GTK or Qt as the Widget Foundation

GTK is the GNOME toolkit. It is designed for GNOME applications. It
carries GObject, GLib, Pango, Cairo, ATK, and the GNOME accessibility
infrastructure. It is a framework. It controls your application's
structure. lib_ccsp_widget is a library. You control the structure.

Qt is a C++ framework. We are not writing C++.

Flutter's layout model is the right widget model for this stack:
provably O(n), composable, testable with a software rasterizer, and
not coupled to any desktop environment. We implement it in C. The
result is smaller, faster, and auditable without understanding the
entire toolkit.

### B.10: Write This in Rust

Rust is a good language for systems programming. It provides memory
safety guarantees at compile time that C does not. If you are starting
a new systems project in 2026 and memory safety is a primary concern,
Rust is a reasonable choice.

This stack is C because: C has a 50-year library ecosystem, near-
universal platform support including embedded targets that do not have
Rust toolchains, stable ABI guarantees that Rust does not provide,
and a larger pool of systems programmers who can contribute. The
security properties of this stack come from API design (patterns that
are not expressible incorrectly) and tooling (fuzzers, sanitizers,
static analysis), not from the type system.

We are not anti-Rust. We are pro-using-the-right-tool. This stack is C.
If you want Rust bindings to lib_ccsp_tls or lib_ccsp_dns, they are a
`bindgen` run and a thin safe wrapper away and we will happily host them.

### B.11: Skia for 2D Rendering

Skia is Google's 2D graphics library, used in Chrome and Flutter. It is
correct, fast, and approximately 5 million lines of C++. Its build
system is gn. Its dependencies include libwebp, libjpeg-turbo, zlib,
freetype, harfbuzz, icu, and several others. The audit scope is
unbounded for practical purposes.

lib_ccsp_draw has a software rasterizer backend (lib_ccsp_draw_img), a
Wayland backend, an X11 backend, a framebuffer backend, and a Vulkan/
GLES3 GPU backend. These cover the rendering targets this stack needs.
They are auditable. Their source is in this repository.

For applications that need Skia's full capabilities (PDF rendering,
SVG parsing, complex color management, wide-gamut HDR), use Skia.
It is not a dependency of this stack.

### B.12: Cairo

Cairo is a 2D vector graphics library used by GTK and Firefox (before
Skia). It handles PDF, SVG, PostScript, and X11/Win32/Quartz surfaces.
It does not handle GPU rendering well -- Cairo's GL backend was never
production-quality and the project has not invested in Vulkan.

In 2026, a 2D graphics library that does not have a first-class GPU
path is not the right foundation for a widget system. lib_ccsp_draw_gpu
is Vulkan-primary. Cairo is not the right choice.

### B.13: OpenSSL as the Only Crypto Backend

lib_ccsp_ocl has a native implementation and an optional OpenSSL backend.
The native implementation is the default. The OpenSSL backend exists for
deployments where a compliance program mandates a FIPS 140-3 validated
OpenSSL FIPS module specifically.

Making OpenSSL mandatory would couple every deployment to OpenSSL's
release schedule, OpenSSL's CVEs, OpenSSL's API instability (the 1.x
to 3.x transition broke a large fraction of the ecosystem), and
OpenSSL's license (Apache 2.0, which has patent termination clauses
that matter to some deployments).

The native implementation is auditable in isolation. The OpenSSL backend
is available when needed. Neither is required.

### B.14: io_uring as the Only Event Backend

io_uring is faster than epoll for high-connection-count servers on
Linux 5.1+. It is also a large kernel attack surface (CVE-2022-0847,
CVE-2022-1015, CVE-2022-2586, and others), requires elevated privileges
in some container environments, and is Linux-only.

io_uring is an optional backend in lib_ccsp_event. The default on Linux
is epoll. Enable io_uring when you have a measurement showing it
matters for your workload and your deployment environment permits it.
The decision to enable it is explicit and documented.

### B.15: QUIC as the Only Transport

QUIC is the right transport for HTTP/3 and for low-latency applications
where head-of-line blocking matters. It also requires UDP, which is
blocked or throttled in a significant fraction of enterprise networks.
It requires a QUIC implementation, which is a substantial piece of
code. And TLS 1.3 is integrated into the QUIC handshake in a way that
requires changes to lib_ccsp_tls's internal API.

TCP is available everywhere. The lib_ccsp_connect API surface
accommodates QUIC as a transport option. When QUIC support is complete
it will be selectable alongside TCP. It will not replace TCP.

## Appendix C: Standards Positions

- C.1: IETF: NO to bare ANY in ASN.1 specs; ANY DEFINED BY requires
  complete dispatch table in the spec itself
- C.2: IETF: UTF-8 in all new protocol string fields
- C.3: IETF: DANE as the CA reform path
- C.4: IETF: HTTPS/SVCB (RFC 9460) as Alt-Svc standard
- C.5: IETF: ECH (Encrypted Client Hello) deployment
- C.6: POSIX: proposals pending
- C.7: C committee: C23 adoption recommendations
- C.8: The IPR disclosure record: running log of WG interventions

## Appendix D: External Dependency Reference

| Library | Version | License | Purpose |
|---------|---------|---------|---------|
| libevent | 2.1.x | BSD | Event loop backend |
| FreeType | 2.13+ | FreeType/GPLv2 | Font rasterization |
| HarfBuzz | 8.x+ | MIT | Text shaping |
| libpng | 1.6.x | PNG Reference Lib | PNG encode/decode |
| libjpeg-turbo | 3.x | BSD/IJG | JPEG decode |
| libwebp | 1.x | BSD | WebP decode |
| libwayland-client | 1.22+ | MIT | Wayland protocol |
| libxkbcommon | 1.x | MIT | Keyboard map handling |
| Vulkan headers | 1.3+ | Apache 2.0 | GPU backend |
| libtess2 | MIT | Path tessellation |
| VMA | 3.x | MIT | Vulkan memory allocation |
| glslang | BSD | SPIR-V compilation at build time |

OpenSSL (3.x, Apache 2.0): optional, used only when
`CCSP_OCL_BACKEND=openssl` is selected for FIPS 140-3 contexts.

## Appendix E: Changelog

- 0.1: Initial design document (historical, 1991 framing)
- 0.2: lib_ccsp_ namespace, expanded TOC
- 1.0: C17 baseline. Wayland primary. Vulkan/GLES3. TLS 1.0/1.1
  removed. Ed25519 default. io_uring optional. HTTPS/SVCB. DoH/DoT.
  Post-quantum hybrid option. Principle 10 (We Are Not Fixing C).
  Appendix B expanded to prose.
- 1.1: ANY DEFINED BY policy for lib_ccsp_asn1. Color model: linear
  light storage, Oklab/OKLCh manipulation API, sRGB input convenience.
  lib_ccsp_dgram, lib_ccsp_dtls, lib_ccsp_coap (roadmap) added.
  lib_ccsp_cbor added. Part VII (Roadmap) written. ccsp-check UDP and
  CBOR rules added. Fuzz targets expanded.

---

*The CCSP Stack is maintained by the CCSP Project.*

*Source:* `https://github.com/ccsp-project`
*Issues:* `https://github.com/ccsp-project/ccsp/issues`
*Discussions:* `https://github.com/ccsp-project/ccsp/discussions`
*CVE Corpus:* `https://github.com/ccsp-project/cve-corpus`
*The Book:* `"Systems Programming With A Brain"` -- `https://ccsys.org/book`
