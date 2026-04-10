> **This library should have existed in 1987.**
>
> It could have existed in 1987. BSD 4.3 had sockets, select(), and
> gettimeofday(). GCC 1.0 had shipped. X11 was released. Every primitive
> was in place. The only missing piece was /dev/urandom, and that's a
> 200-line kernel driver.
>
> I wanted this library in 1987. I needed it in 1987. Every C programmer
> who ever wrote a network daemon needed it in 1987.
>
> There is no excuse for it not existing in 1987.
>
> Instead, we got 40 years of the same bugs. 47 CVEs from hand-rolled line
> parsers. 28 CVEs from gethostbyname's global state. 19 CVEs from
> forgetting to check SO_ERROR after nonblocking connect. The same errors,
> exposed in different codebases, exposed by different maintainers,
> exposed decade after decade. Mean time from introduction to disclosure:
> 5.8 years. Per bug. Exposed. Fixed. Reintroduced elsewhere. Exposed
> again.
>
> The knowledge existed. The primitives existed. The need was obvious.
> Nobody built it.
>
> This document describes what should have been on the shelf in 1987.

---

## Why Nobody Did This in 1987

**The Unix Wars.** 1987 was the peak of fragmentation. AT&T vs BSD vs
Sun vs IBM vs HP vs DEC vs Apollo vs NeXT. Every vendor was building
incompatible variants, fighting for market share, building lock-in. A
shared foundation library required cooperation. Cooperation was
weakness. The incentives pointed exactly the wrong direction.

**No infrastructure for collaboration.** Usenet existed. Email existed.
But coordinating a cross-vendor open source project across corporate
boundaries, across continents, without the web, without GitHub, without
even consistent FTP access? The logistics were nearly impossible. The
people who could have done this were in different companies, different
countries, different Usenet hierarchies. They couldn't even agree on
whether to top-post or bottom-post.

**Commercial incentives rewarded lock-in.** Sun made money when you
couldn't move off Sun. HP made money when you couldn't move off HP. A
portable foundation library would reduce switching costs. No vendor
would fund work that made it easier to leave. And there was no neutral
funding source -- no Apache Foundation, no Linux Foundation, no Google
Open Source Programs Office. The model didn't exist.

**The culture valued cleverness over correctness.** "Real programmers"
knew the tricks. Hand-rolling a parser was a rite of passage. Using a
library was admitting you couldn't figure it out yourself. The hacker
ethic celebrated tight, clever, minimal code. Robust, defensive,
"paranoid" code was bloat. Security wasn't a concern because networks
were trusted, users were colleagues, and the Morris Worm hadn't happened
yet (November 1988).

**Nobody was counting the bodies.** No CVE database. No security
advisories. No public disclosure. When code broke, you fixed it locally
and moved on. There was no aggregated evidence that the same twelve
patterns kept causing the same bugs. Each team thought their bugs were
unique. They weren't. But nobody could see the pattern because nobody
was keeping score.

**Everyone was waiting for ANSI C.** The standard was in committee from
1983 to 1989. Why build on K&R C when the language was about to change?
Why invest in portability when `void *` wasn't even standard yet? The
reasonable thing was to wait. And wait. And then when C89 shipped, the
Unix Wars were still raging, and the moment had passed.

**BSD was supposed to be this.** Berkeley thought they were building the
foundation. They were close. But they were building an operating system,
not a portable library. They were entangled with AT&T's license. And in
1992 the lawsuit hit and BSD's momentum collapsed for three years.
During those three years, Linux happened instead.

**GNU was supposed to be this.** Stallman started in 1983. The GNU
project was building all the pieces. But Stallman's focus was
replacement, not foundation -- replace Unix, don't complement it.
Replace the compiler, the editor, the shell, the kernel (eventually).
The ideological goal was to eliminate proprietary Unix, not to make
proprietary Unix better. And the GPL v1 (1989) was intentionally
incompatible with proprietary use. This library needed to be in
*everything*, including proprietary code. GNU's license was a barrier
by design.

**The people who needed it most had no power.** The programmers getting
bitten by these bugs were implementers, not architects. They didn't set
technical direction. The architects were fighting the Unix Wars,
building empires, accumulating headcount. They didn't want a shared
foundation -- they wanted differentiation. The implementers knew
something was wrong but couldn't articulate it, couldn't fund it,
couldn't coordinate it.

**Egos.** The people technically capable of doing this had strong
opinions about how it should be done. Berkeley's way. AT&T's way. GNU's
way. Sun's way. Each camp thought the others were doing it wrong. A
neutral foundation library required someone to say "I don't care whose
way wins, I just want the problem solved." That person didn't exist.
Everyone who cared enough to work on it cared too much about winning.

**It wasn't anyone's job.** At a company, you work on what you're paid
to work on. Nobody was paid to build cross-platform foundations. They
were paid to ship products. A foundation library is infrastructure --
valuable, essential, but not a product. Infrastructure gets built when
someone has the luxury to think long-term. In 1987, everyone was
scrambling to survive the Unix Wars. There was no long-term.

So nothing happened. The bugs accumulated. Forty years of the same
twelve patterns. Exposed. Patched. Reintroduced. Exposed again.

The cost of coordination failure is measured in CVEs.

---

*This document is written as if from a better 1987, where this was created.*

---

# The CCSP Stack: Unified Design, Rationale, and Library Guide

**Namespace:** `lib_ccsp_*`
**Version:** 0.2-draft
**Status:** Master Design Document
**License:** MIT or GPL-2.0-or-later, at your option

---

## A Note on Naming

All libraries in this stack use the `ccsp_` and `lib_ccsp_`
prefixes. CCSP stands for **Correct C Systems Programming**. The
prefix serves two purposes:

First, collision avoidance. `libbase`, `libevent`, `libjson`, `libdns`
and their header files all conflict with existing or future libraries.
`lib_ccsp_base`, `lib_ccsp_event`, `lib_ccsp_json`, `lib_ccsp_dns` do not.
The prefix is long enough to be globally unique and short enough to be
tolerable in code.

Second, identity. Code that uses `ccsp_hash_create()` or
`ccsp_connect()` is visibly using this stack. The provenance is traceable.
The audit trail is clear. When a security researcher finds a
vulnerability they can search for the prefix and find every affected
application.

Header files follow the same convention: `ccsp_base.h`, `ccsp_event.h`,
`ccsp_dns.h`. The `ccsp_` prefix on symbols means `ccsp_malloc()`,
`ccsp_str_t`, `ccsp_err_t`. No symbol in this stack is a bare name that
could conflict with any existing code.

---

## Table of Contents

- [Preamble](#preamble)
- [Part I: Why This Exists](#part-i-why-this-exists)
  - [The Problem Statement](#the-problem-statement)
  - [The Evidence](#the-evidence)
  - [The Argument Against Rolling Your Own](#the-argument-against-rolling-your-own)
  - [The Argument Against Large Frameworks](#the-argument-against-large-frameworks)
  - [What We Are Building Instead](#what-we-are-building-instead)
  - [Licensing](#licensing)
- [Part II: Architectural Principles](#part-ii-architectural-principles)
  - [Principle 1: Separation of Concerns Is a Security Property](#principle-1-separation-of-concerns-is-a-security-property)
  - [Principle 2: Never Own What Someone Else Should Own](#principle-2-never-own-what-someone-else-should-own)
  - [Principle 3: Every Shim Has a Deletion Condition](#principle-3-every-shim-has-a-deletion-condition)
  - [Principle 4: Caller Provides Storage](#principle-4-caller-provides-storage)
  - [Principle 5: Explicit Over Implicit](#principle-5-explicit-over-implicit)
  - [Principle 6: One ifdef Location Per Platform Concern](#principle-6-one-ifdef-location-per-platform-concern)
  - [Principle 7: UTF-8 Is the Only Text Encoding](#principle-7-utf-8-is-the-only-text-encoding)
  - [Principle 8: Audit Scope Is Bounded By Design](#principle-8-audit-scope-is-bounded-by-design)
  - [Principle 9: The Reference Implementation Controls the Standard](#principle-9-the-reference-implementation-controls-the-standard)
  - [Principle 10: Running Code or It Didn't Happen](#principle-10-running-code-or-it-didnt-happen)
- [Part III: The Dependency Architecture](#part-iii-the-dependency-architecture)
  - [The Library Registry](#the-library-registry)
  - [The Dependency Graph](#the-dependency-graph)
  - [The Dependency Rule](#the-dependency-rule)
  - [Why Libraries Not a Framework](#why-libraries-not-a-framework)
  - [Versioning and ABI Stability](#versioning-and-abi-stability)
- [Part IV: Library Catalog](#part-iv-library-catalog)
  - [Layer 0: Platform and Foundation](#layer-0-platform-and-foundation)
  - [Layer 1: Core Services](#layer-1-core-services)
  - [Layer 2: I/O and Networking](#layer-2-io-and-networking)
  - [Layer 3: Protocols](#layer-3-protocols)
  - [Layer 4: Cryptography and Security](#layer-4-cryptography-and-security)
  - [Layer 5: Data and Parsing](#layer-5-data-and-parsing)
  - [Layer 6: Graphics and UI](#layer-6-graphics-and-ui)
  - [Layer 7: Tooling](#layer-7-tooling)
- [Part V: Cross-Cutting Concerns](#part-v-cross-cutting-concerns)
  - [Error Handling Across the Stack](#error-handling-across-the-stack)
  - [Memory Model Across the Stack](#memory-model-across-the-stack)
  - [Threading Model Across the Stack](#threading-model-across-the-stack)
  - [Logging Across the Stack](#logging-across-the-stack)
  - [Testing Across the Stack](#testing-across-the-stack)
  - [The CVE Corpus](#the-cve-corpus)
- [Part VI: API Guide Table of Contents](#part-vi-api-guide-table-of-contents)
- [Part VII: Roadmap and Shim Status](#part-vii-roadmap-and-shim-status)
- [Appendices](#appendices)

---

# Preamble

This document describes a unified stack of C libraries covering systems
programming from integer types to TLS, from async I/O to line protocols,
from widget layout to GPU rendering. Every library uses the `lib_ccsp_`
prefix to prevent namespace collision with existing or future libraries.

Each library in the stack is small, focused, independently auditable,
and independently deployable. None of them is required to use the others
except where the dependency is explicit and justified. Together they cover
the ground that every serious C application has to cover and that every
serious C application currently covers badly, expensively, and
dangerously.

---

# Part I: Why This Exists

## The Problem Statement

C systems software has a predictable failure mode. It is not that C is
unsafe, though C is not safe. It is not that systems programming is hard,
though it is hard. The predictable failure mode is this:

**Every project reinvents the same infrastructure, badly, on a deadline,
with different bugs than the last project that reinvented it.**

The infrastructure being reinvented includes: event loops, line parsers,
option parsers, DNS resolvers, connection state machines, buffer
management, hash tables, string handling, error propagation, and logging.
These are not novel problems. They are solved problems. They are solved
problems that keep being re-solved because the solved versions are not
available in a form that can be adopted without accepting a large
framework or a restrictive license or a GObject system or all three.

The cost of this reinvention is measured in CVEs. The CVE corpus
maintained alongside this stack classifies every major C systems
vulnerability by root cause pattern. Twelve patterns cover 87% of
vulnerabilities. Every one of those twelve patterns has a structural
fix in this stack. A structural fix means the pattern is not possible
in code that uses the library correctly, not merely less likely.

## The Evidence

The twelve patterns, with representative CVE counts from the corpus:

```tbl
.TS
center box;
l n l
l n l.
Pattern	CVEs	Library That Prevents It
_
Hand-rolled line reader, fixed buffer	47	lib_ccsp_lineproto
Caller-supplied length trusted without validation	31	lib_ccsp_bio, lib_ccsp_ocl
gethostbyname non-reentrant global state	28	lib_ccsp_dns
Integer overflow before allocation	24	lib_ccsp_base (ccsp_size_*)
SO_ERROR miss in nonblocking connect	19	lib_ccsp_connect
select() FD_SETSIZE unchecked	17	lib_ccsp_event
Nonce/IV reuse in symmetric encryption	15	lib_ccsp_ocl
sprintf without bounds	22	lib_ccsp_base (ccsp_buf_*)
ASN.1 ANY type parser confusion	12	lib_ccsp_asn1
Implicit integer size assumption	34	lib_ccsp_base (ccsp_types.h)
Non-constant-time secret comparison	11	lib_ccsp_ocl
Non-reentrant global in crypto path	9	lib_ccsp_ocl
.TE
```

Mean time from introduction to disclosure across all patterns: 5.8 years.

## The Argument Against Rolling Your Own

The developer who says "I'll write my own event loop, it's only 200 lines"
is not wrong about the line count. A naive event loop is 200 lines. A
correct event loop is 2,000 lines and 5 years of production bug reports.
The developer does not have those 5 years. The correct version should
have been on the shelf.

**"How many people know kqueue well enough to write an event state machine
with zero bugs from a cold start? Zero. Not even the best systems
programmers alive. Because if they could, they would have. The bugs in
their codebases prove it."**

## The Argument Against Large Frameworks

GLib, Qt, and their descendants get the motivation right and the design
wrong. The motivation -- provide a common foundation so applications
don't reinvent infrastructure -- is correct. The design -- put everything
in one library, couple it to an object system, make it load-bearing for
a desktop environment -- produces a different category of problem.

Auditing an application that depends on GLib means auditing everything
GLib contains. The audit scope is unbounded. The bookmark file parser
is in the TCB whether you want it there or not.

**Audit scope is a security property.** A library that does one thing
has a bounded audit scope. A library that does everything has an
unbounded audit scope.

## What We Are Building Instead

A stack of small libraries. Each one does one thing. Each one depends
only on the things it genuinely needs. Each one can be audited in
isolation. Each one can be replaced if you have a better version.

`lib_ccsp_base.so` is in memory on any system that runs any of these
libraries. It costs nothing extra. Its audit scope is bounded. Its API
is stable. Everything else builds on it.

## Licensing

All libraries: **MIT License OR GNU General Public License version 2.0
or later, at your option.**

MIT for commercial use. GPL-2.0-or-later for GPL projects. LGPL is not
offered -- MIT already covers the linking-from-proprietary case without
LGPL's complexity. Two options. Every use case covered.

We added the GPL-2.0 so the GNU people would stop bikeshedding and just
use this.

---

# Part II: Architectural Principles

## Principle 1: Separation of Concerns Is a Security Property

Every library boundary is also an audit boundary. The boundaries are not
arbitrary. They are drawn where the audit scope should be bounded.
Heartbleed was a record layer bug made possible by the absence of
separation between the record layer and the handshake layer. Separation
would have made it impossible.

## Principle 2: Never Own What Someone Else Should Own

If a well-maintained, widely-deployed standard or library owns a problem,
our job is to have a clean interface to it. We own things only when
nothing suitable exists, explicitly, minimally, and with a deletion
condition.

## Principle 3: Every Shim Has a Deletion Condition

```c
/*
 * TEMPORARY SHIM -- delete when [precise condition]
 * See: CCSP-SHIM-NNN (mail ccsp-bugs@ccsys.org for status)
 * Last reviewed: YYYY-MM-DD
 */
```

The shim ticket is open. It is reviewed on schedule. The shim never
becomes permanent by neglect.

## Principle 4: Caller Provides Storage

Functions that produce output take caller-provided buffers. The library
calls `ccsp_malloc()` only when the caller explicitly requests object
creation via `_create()`. A grep for `malloc` across the entire stack
finds exactly one match: the platform default implementation in
lib_ccsp_base.

## Principle 5: Explicit Over Implicit

Every piece of state a function needs is passed as a parameter. No
global variables. No thread-local state except where the platform
requires it (errno). No ambient authority.

## Principle 6: One ifdef Location Per Platform Concern

Platform-specific ifdefs belong in one file per concern. No other file
contains `#ifdef __linux__` or `#ifdef _WIN32`. When something breaks
on AIX you look in `ccsp_platform.h` and the relevant backend file.

## Principle 7: UTF-8 Is the Only Text Encoding

Inside this stack, text is UTF-8 or it is bytes. There is no third
option. Strictly validated: overlong encodings rejected, surrogates
rejected, codepoints above U+10FFFF rejected. Legacy encodings are
converted at the boundary by lib_ccsp_charconv.

## Principle 8: Audit Scope Is Bounded By Design

An auditor reviewing a TLS application built on this stack can bound
scope precisely to the specific libraries involved. Each boundary is
auditable independently. Trust is compositional.

## Principle 9: The Reference Implementation Controls the Standard

Every standards proposal from this project comes with a reference
implementation and a test suite. Not a position paper. Running code.
The test suite becomes the conformance mechanism.

## Principle 10: Running Code or It Didn't Happen

Nothing in this stack is proposed, argued for, or defended without a
working implementation. Running code ends debates that argument cannot.

---

# Part III: The Dependency Architecture

## The Library Registry

```tbl
.TS
center box;
l l l l
l l l l.
Library	Symbol Prefix	Header	Purpose
_
lib_ccsp_base	ccsp_	ccsp_base.h	Foundation: types, alloc, str, utf8, etc.
lib_ccsp_event	ccsp_ev_	ccsp_event.h	I/O event dispatch, timers, signals
lib_ccsp_mpi	ccsp_mpi_	ccsp_mpi.h	Arbitrary precision integers
lib_ccsp_regex	ccsp_rx_	ccsp_regex.h	Regular expressions, NFA and backtracking
lib_ccsp_charconv	ccsp_conv_	ccsp_charconv.h	Encoding conversion to/from UTF-8
lib_ccsp_bio	ccsp_bio_	ccsp_bio.h	Buffered I/O, readers, writers, filters
lib_ccsp_dns	ccsp_dns_	ccsp_dns.h	Async DNS resolver, DNSSEC
lib_ccsp_mdns	ccsp_mdns_	ccsp_mdns.h	Multicast DNS, DNS-SD
lib_ccsp_connect	ccsp_conn_	ccsp_connect.h	Stream connection, happy eyeballs, SRV, DANE
lib_ccsp_lineproto	ccsp_lp_	ccsp_lineproto.h	Line protocol server and client framework
lib_ccsp_http	ccsp_http_	ccsp_http.h	HTTP/1.1 and HTTP/2
lib_ccsp_smtp	ccsp_smtp_	ccsp_smtp.h	SMTP client and server
lib_ccsp_imap	ccsp_imap_	ccsp_imap.h	IMAP4 client and server
lib_ccsp_asn1	ccsp_asn1_	ccsp_asn1.h	BER/DER/CER codec
lib_ccsp_cert	ccsp_cert_	ccsp_cert.h	Certificate parse, validate, path build
lib_ccsp_pki	ccsp_pki_	ccsp_pki.h	Certificate generation and CA operations
lib_ccsp_ocl	ccsp_ocl_	ccsp_ocl.h	Crypto primitives, schemes, AEAD
lib_ccsp_tls	ccsp_tls_	ccsp_tls.h	TLS protocol layer
lib_ccsp_data	ccsp_data_	ccsp_data.h	Shared structured data value type
lib_ccsp_toml	ccsp_toml_	ccsp_toml.h	TOML parser
lib_ccsp_json	ccsp_json_	ccsp_json.h	JSON parser, streaming and DOM
lib_ccsp_cas	ccsp_cas_	ccsp_cas.h	Content-addressable storage
lib_ccsp_cli	ccsp_cli_	ccsp_cli.h	Declarative command-line parsing
lib_ccsp_draw	ccsp_draw_	ccsp_draw.h	2D drawing primitives, backend agnostic
lib_ccsp_draw_x11	ccsp_dx11_	ccsp_draw_x11.h	X11 drawing backend
lib_ccsp_draw_fb	ccsp_dfb_	ccsp_draw_fb.h	Framebuffer drawing backend
lib_ccsp_draw_img	ccsp_dimg_	ccsp_draw_img.h	Imagemap/software rasterizer backend
lib_ccsp_draw_gpu	ccsp_dgpu_	ccsp_draw_gpu.h	OpenGL GPU drawing backend
lib_ccsp_font	ccsp_font_	ccsp_font.h	Font loading and glyph rasterization
lib_ccsp_text	ccsp_text_	ccsp_text.h	Text layout, shaping, bidi, line breaking
lib_ccsp_widget	ccsp_wid_	ccsp_widget.h	Flutter-model widget system
lib_ccsp_anim	ccsp_anim_	ccsp_anim.h	Animation engine
lib_ccsp_input	ccsp_input_	ccsp_input.h	Input event normalization
lib_ccsp_a11y	ccsp_a11y_	ccsp_a11y.h	Accessibility platform adapters
lib_ccsp_style	ccsp_style_	ccsp_style.h	Theming and design tokens
lib_ccsp_app	ccsp_app_	ccsp_app.h	Application lifecycle
lib_ccsp_test	ccsp_test_	ccsp_test.h	Testing infrastructure
lib_ccsp_fuzz	ccsp_fuzz_	ccsp_fuzz.h	Fuzzing harness infrastructure
.TE
```

Tools (not runtime libraries):

```tbl
.TS
center box;
l l l
l l l.
Tool	Binary	Purpose
_
ccsp_check	ccsp-check	Static analysis against CVE corpus
ccsp_abi	ccsp-abi	ABI compatibility checker
ccsp_corpus	ccsp-corpus	CVE corpus query tool
.TE
```

## The Dependency Graph

Process with: `pic depgraph.pic | troff -ms | ... `

```pic
.PS
scale = 0.75
boxwid = 1.8; boxht = 0.4

# LAYER 0
L0: box "lib_ccsp_base"
    "depends: libc only" at L0.s - (0, 0.25)

    arrow down 0.5 from L0.s
    "all libraries below" ljust " depend on lib_ccsp_base" ljust

# First tier - three branches
    line down 0.3
    line left 1.8; line right 3.6
    move to last line.start; move down 0.15

EV: box "lib_ccsp_event" with .n at L0.s - (1.8, 1.4)
MPI: box "lib_ccsp_mpi" with .n at L0.s - (0, 1.4)
RE: box "lib_ccsp_regex" with .n at L0.s + (1.8, -1.4)

# Event branch -> bio
    arrow down 0.4 from EV.s
BIO: box "lib_ccsp_bio"

# MPI branch -> asn1 -> cert
    arrow down 0.4 from MPI.s
ASN: box "lib_ccsp_asn1"
    arrow down 0.4 from ASN.s
CERT: box "lib_ccsp_cert"

# Merge bio and cert into ocl
    line from BIO.s down 0.6
    line from CERT.s down 0.3
    line from last line.end to 2nd last line.end
    arrow down 0.3 from last line.c

OCL: box "lib_ccsp_ocl" wid 2
    "AES, SHA-2/3, ChaCha20, RSA, ECDSA, EdDSA" at OCL.s - (0, 0.2)

    arrow down 0.5 from OCL.s
TLS: box "lib_ccsp_tls"
    "TLS 1.2/1.3, no renegotiation" at TLS.s - (0, 0.2)

    arrow down 0.5 from TLS.s
DNS: box "lib_ccsp_dns"
    "async, DNSSEC, DANE, SRV" at DNS.s - (0, 0.2)

# DNS branches to mdns and connect
    line down 0.3 from DNS.s
    line left 1.2; line right 2.4
    move to last line.start; move down 0.15

MDNS: box "lib_ccsp_mdns" with .n at DNS.s - (1.2, 0.6)
CONN: box "lib_ccsp_connect" with .n at DNS.s + (1.2, -0.6)
    "happy eyeballs, SO_ERROR" at CONN.s - (0, 0.2)

# Connect branches to protocols
    line down 0.3 from CONN.s
    line left 1.5; line right 3
    move to last line.start; move down 0.15

LP: box "lib_ccsp_lineproto" with .n at CONN.s - (1.5, 0.6)
HTTP: box "lib_ccsp_http" with .n at CONN.s - (0, 0.6)
PKI: box "lib_ccsp_pki" with .n at CONN.s + (1.5, -0.6)

# Lineproto branches to smtp and imap
    line down 0.3 from LP.s
    line left 0.8; line right 1.6

SMTP: box "lib_ccsp_smtp" with .n at LP.s - (0.8, 0.6)
IMAP: box "lib_ccsp_imap" with .n at LP.s + (0.8, -0.6)


# DATA LAYER - separate cluster
move to L0.e + (3.5, 0)

D0: box "lib_ccsp_data"
    arrow left 0.5 from D0.w
DJ: box "lib_ccsp_json" with .e at D0.w - (0.5, 0)
    arrow down 0.4 from D0.n
DT: box "lib_ccsp_toml" with .s at D0.n + (0, 0.4)
    arrow down 0.4 from D0.s
CAS: box "lib_ccsp_cas" with .n at D0.s - (0, 0.4)
    arrow down 0.4 from CAS.s
CLI: box "lib_ccsp_cli" with .n at CAS.s - (0, 0.4)


# GRAPHICS LAYER - separate cluster
move to D0.e + (3, 0)

GD: box "lib_ccsp_draw"
    arrow right 0.5 from GD.e
X11: box "lib_ccsp_draw_x11" with .w at GD.e + (0.5, 0)
    box "lib_ccsp_draw_fb" with .n at X11.s - (0, 0.15)
    box "lib_ccsp_draw_gpu" with .n at last box.s - (0, 0.15)

    arrow down 0.4 from GD.s
GF: box "lib_ccsp_font" with .n at GD.s - (0, 0.4)
    arrow down 0.4 from GF.s
GT: box "lib_ccsp_text" with .n at GF.s - (0, 0.4)
    arrow down 0.4 from GT.s
GW: box "lib_ccsp_widget" with .n at GT.s - (0, 0.4)
    arrow down 0.4 from GW.s
GA: box "lib_ccsp_app" with .n at GW.s - (0, 0.4)


# TOOLING - bottom
move to L0.s - (0, 8.5)

"TOOLING (not runtime libraries)" ljust
move down 0.3
T1: box "lib_ccsp_test"
    arrow right 0.4 from T1.e
T2: box "lib_ccsp_fuzz" with .w at T1.e + (0.4, 0)
    arrow right 0.4 from T2.e
T3: box "ccsp-check" with .w at T2.e + (0.4, 0)
    arrow right 0.4 from T3.e
T4: box "CVE corpus" with .w at T3.e + (0.4, 0)

.PE
```

To render: `pic file.ms | troff -ms | dpost | ps2pdf - graph.pdf`
or with groff: `groff -p -ms -Tps file.ms | ps2pdf - graph.pdf`

## The Dependency Rule

**A library may depend only on libraries in lower or equal layers.**

Enforced by the build system at configure time. A `CMakeLists.txt` that
creates an upward dependency fails with a descriptive error before a
single file is compiled. The rule is not enforced by code review alone.

## Why Libraries Not a Framework

A framework controls your application. A library serves it. This stack
is libraries. Take the ones you need. An embedded device with no display
can use the full networking and crypto stack without pulling in a single
widget.

## Versioning and ABI Stability

Each library versions independently. ABI stability: struct layouts stable
within major version, new functions in minor versions, removals only in
major versions. The `ccsp-abi` tool checks ABI compatibility against the
previous release on every release candidate.

---

# Part IV: Library Catalog

## Layer 0: Platform and Foundation

### lib_ccsp_base

**What it is:** The foundation. Every other library depends on this.
Every symbol is prefixed `ccsp_`. Every header is `ccsp_*.h`.

**Why this exists (1987 context):**

Every Unix vendor ships a different libc. Sun's `sprintf` returns
`char *`. BSD's returns `int`. System V's `memcpy` arguments are in
different order than BSD's `bcopy`. HP-UX has `random()`, Ultrix has
`rand()`, neither seeds properly. Every project wastes its first month
writing portability shims. Every project writes them differently. Every
project has different bugs.

The X3J11 committee is standardizing C. When ANSI C ships, some of this
pain goes away. But ANSI C will not ship until 1989 at earliest, and
vendor compilers will not catch up for years after that. We cannot wait.
We ship what ANSI C should have included but did not: fixed-width
integers, portable byte order, overflow-checked arithmetic, and a real
string type.

This library is the boundary layer between your code and the Unix Wars.
You write to `ccsp_*` interfaces. We fight with SunOS, BSD, System V,
Ultrix, HP-UX, AIX, and Domain/OS so you do not have to.

**Portability targets:**

- BSD 4.3 (VAX, Tahoe)
- SunOS 3.5+ (68020, SPARC)
- System V Release 3 (3B2, i386)
- Ultrix 2.0+ (VAX, MIPS)
- HP-UX 6.0+ (HP 9000 series)
- AIX 2.2+ (RT PC)
- Domain/OS SR10+ (Apollo 68k)
- MIPS RISC/os 4.0+
- NeXT (when it ships)

**Architecture concerns:**

- **Word size:** 32-bit is standard on workstations, but some code still
  runs on 16-bit systems. `ccsp_size_t` is `unsigned long` everywhere.
  Never assume `sizeof(int) == sizeof(long)`.

- **Endianness:** SPARC, 68k, MIPS, and IBM RT are big-endian. VAX and
  i386 are little-endian. Network protocols are big-endian. Never cast
  pointers to access multi-byte integers. Use `ccsp_read_be32()` and
  friends.

- **Alignment:** SPARC and MIPS trap on unaligned access. 68k and VAX
  are permissive but slow. Never assume you can read a 32-bit value from
  an odd address. The byte-by-byte accessors in `ccsp_endian.h` are
  always safe.

- **Signed char:** Some compilers default `char` to signed, others to
  unsigned. Always use `unsigned char` for byte data. `ccsp_str_t` uses
  `unsigned char *` internally.

**Language constraints:**

We write to K&R C with an eye toward the draft ANSI standard. Function
prototypes in headers (guarded by `#if __STDC__`), K&R definitions in
source files for maximum compiler compatibility. No `//` comments. No
`long long` (64-bit integers are `ccsp_uint64_t`, typedef'd per platform
to whatever works: `unsigned long` on 64-bit, struct of two longs on
32-bit, or compiler extension where available). No `inline` keyword (use
macros for performance-critical small functions). No `const` in
positions that older compilers reject.

When ANSI C is ratified and deployed, delete the `#if __STDC__` guards.
Until then, we compile on everything.

**Modules:**

`ccsp_platform.h` -- OS and architecture detection. The only file in the
entire stack containing OS/arch ifdefs. Defines `CCSP_OS_SUNOS`,
`CCSP_OS_BSD`, `CCSP_OS_SYSV`, `CCSP_OS_ULTRIX`, `CCSP_OS_HPUX`,
`CCSP_OS_AIX`, `CCSP_OS_DOMAIN`, `CCSP_ARCH_VAX`, `CCSP_ARCH_M68K`,
`CCSP_ARCH_SPARC`, `CCSP_ARCH_MIPS`, `CCSP_ARCH_I386`, `CCSP_BIG_ENDIAN`,
`CCSP_STRICT_ALIGN`, `CCSP_HAS_VOID_PTR` (some ancient compilers lack
`void *`), `CCSP_HAS_CONST`, `CCSP_HAS_PROTOTYPES`. Every platform quirk
is detected here and nowhere else.

```c
/* Example: the SPARC requires aligned access */
#if defined(sun) && defined(sparc)
#define CCSP_STRICT_ALIGN 1
#define CCSP_BIG_ENDIAN 1
#define CCSP_OS_SUNOS 1
#define CCSP_ARCH_SPARC 1
#endif
```

`ccsp_types.h` -- Fixed-width integer types. `ccsp_int8_t`,
`ccsp_uint8_t`, `ccsp_int16_t`, `ccsp_uint16_t`, `ccsp_int32_t`,
`ccsp_uint32_t`, `ccsp_int64_t`, `ccsp_uint64_t`. Also `ccsp_size_t`,
`ccsp_ssize_t`, `ccsp_ptrdiff_t`, `ccsp_intptr_t`, `ccsp_uintptr_t`.
Also `ccsp_bool_t` with `CCSP_TRUE` and `CCSP_FALSE`. Every type is
typedef'd per-platform to whatever the compiler provides. On VAX Ultrix,
`ccsp_uint32_t` is `unsigned long`. On 68k SunOS, same. We verified
every combination.

```c
/* ccsp_types.h excerpt */
#if defined(CCSP_ARCH_VAX) || defined(CCSP_ARCH_M68K)
typedef unsigned char  ccsp_uint8_t;
typedef unsigned short ccsp_uint16_t;
typedef unsigned long  ccsp_uint32_t;
typedef long           ccsp_int32_t;
/* 64-bit: struct because VAX has no 64-bit type */
typedef struct { ccsp_uint32_t hi; ccsp_uint32_t lo; } ccsp_uint64_t;
#endif
```

Deletion condition: when ANSI C ships and all target compilers support
`<limits.h>` properly. We will switch to ANSI types and keep `ccsp_*`
as typedefs for compatibility.

`ccsp_err.h` -- Error handling. Every function that can fail returns
`ccsp_err_t`. The error carries: numeric code, domain string (which
library), human message, source file, line number. The `CCSP_ERR()` macro
captures `__FILE__` and `__LINE__` at the call site. Errors propagate up;
callers can wrap with additional context or pass through.

```c
ccsp_err_t err;
err = ccsp_file_open(&f, path, CCSP_O_RDONLY);
if (err.code != 0) {
    /* err.file is "ccsp_file.c", err.line is 234 */
    /* err.domain is "ccsp.file", err.message is "open failed" */
    return err;  /* propagate */
}
```

This is what `errno` should have been. Global `errno` is not thread-safe
(yes, threads are coming, even if vendors pretend otherwise), not
composable, and loses information at every layer. `ccsp_err_t` is
value-typed, explicit, and carries provenance.

`ccsp_alloc.h` -- Memory allocation wrappers. `ccsp_malloc()`,
`ccsp_realloc()`, `ccsp_free()`, `ccsp_calloc()`, `ccsp_strdup()`,
`ccsp_free_secure()`. The last one zeroes memory before freeing --
essential for secrets that should not linger in freed heap.

Why wrappers? Three reasons:

1. **Debugging.** Compile with `CCSP_ALLOC_DEBUG` and every allocation
   is tracked. On exit, report leaks with file:line of allocation site.

2. **Replacement.** Some systems have terrible `malloc`. Some embedded
   targets have no heap. Pass a custom allocator in the platform table.

3. **Accounting.** Count bytes allocated per subsystem. Find the memory
   hog without running Purify.

`ccsp_platform_table.h` -- Function pointer table for platform services.
Allocator, time source, random source, mutex operations, logging backend.
Initialized once at startup, never modified. Every `ccsp_ctx_t` holds a
pointer to this table. Every allocation, every timestamp, every random
byte goes through the table. Mock any of them for testing.

```c
struct ccsp_platform_table {
    void *(*malloc_fn)(size_t);
    void *(*realloc_fn)(void *, size_t);
    void  (*free_fn)(void *);
    ccsp_uint64_t (*monotonic_usec_fn)(void);
    int   (*random_bytes_fn)(void *, size_t);
    /* ... */
};
```

`ccsp_ctx.h` -- The base context. Passed to every stateful operation.
Holds platform table pointer plus library-wide settings (log level,
allocation limits, debug flags). Never use globals; always pass context.
This is how you run two independent instances in one process, how you
test with mocked time, how you embed in a larger application that has
its own allocation strategy.

`ccsp_str.h` -- String type. `ccsp_str_t` is a pointer and a length.
Not null-terminated internally, though `ccsp_str_cstr()` will give you
a null-terminated copy when you need one for legacy interfaces.

```c
typedef struct {
    const unsigned char *data;
    ccsp_size_t len;
} ccsp_str_t;

#define CCSP_STR_LIT(s) { (const unsigned char *)(s), sizeof(s) - 1 }
```

Why not null-terminated C strings? Because every function that takes a
C string must scan to find the length. `strlen()` in a loop is O(n^2).
Substring operations require copying. Binary data with embedded nulls
is unrepresentable. The null-terminated string was a mistake in 1972
and it is still a mistake in 1987. We do not compound it.

`ccsp_utf8.h` -- UTF-8 support. The X3L2 committee is working on what
will become Unicode. We are betting that Ken Thompson's FSS-UTF proposal
(or something like it) will win for interchange. `ccsp_utf8_validate()`
checks a byte sequence for UTF-8 well-formedness. `ccsp_utf8_decode()`
extracts codepoints. `ccsp_utf8_encode()` produces bytes. These will be
essential when internationalization stops being an afterthought.

`ccsp_endian.h` -- Byte order conversion. Read and write 16, 32, and 64
bit integers in big-endian or little-endian format, regardless of host
byte order.

```c
ccsp_uint32_t ccsp_read_be32(const unsigned char *p);
void ccsp_write_be32(unsigned char *p, ccsp_uint32_t v);
ccsp_uint32_t ccsp_read_le32(const unsigned char *p);
void ccsp_write_le32(unsigned char *p, ccsp_uint32_t v);
/* ... and 16-bit, 64-bit variants */
```

Implementation is byte-by-byte. Never `*(uint32_t *)p`. The SPARC will
bus error. The MIPS will bus error. Even on VAX where it works, the
optimizer cannot prove alignment and generates defensive code anyway.
Byte-by-byte is correct everywhere and fast enough.

`ccsp_list.h` -- Intrusive doubly-linked list. The node is embedded in
your struct; no separate allocation per list entry. BSD `queue.h` got
this right; we follow that model but with a consistent API and a
sentinel node that simplifies empty-list handling.

```c
struct my_connection {
    int fd;
    ccsp_list_node_t node;  /* embedded */
    /* ... */
};

/* iterate without allocation */
ccsp_list_foreach(node, &conn_list) {
    struct my_connection *c = CCSP_CONTAINER_OF(node, struct my_connection, node);
    /* ... */
}
```

Why intrusive? Because network servers have 10,000 connections and
allocating a list node per connection is death by malloc. Intrusive
lists have zero allocation overhead. The BSD kernel uses them. We use
them.

`ccsp_slist.h` -- Intrusive singly-linked list. When you only traverse
forward and memory is tight. One pointer per node instead of two.

`ccsp_hash.h` -- Hash table. Open addressing, linear probing, resize at
75% load. Keys are interned (string keys are copied into a pool and
compared by pointer thereafter). Iteration order is insertion order,
always, because debugging is hard enough without hash iteration order
changing between runs.

`ccsp_buf.h` -- Growable byte buffer. Start with inline storage, grow
to heap when needed, double on each growth. The pattern every project
implements, usually with off-by-one errors in the growth calculation.
We wrote it once, tested it, proved it correct.

```c
ccsp_buf_t buf;
ccsp_buf_init(&buf, ctx);
ccsp_buf_append(&buf, "hello ", 6);
ccsp_buf_append(&buf, "world", 5);
/* buf now contains "hello world", heap-allocated if > inline size */
```

`ccsp_ring.h` -- Fixed-size ring buffer. Caller provides storage. No
allocation after initialization. Essential for bounded I/O buffers where
you cannot afford allocation in the read/write path. `ccsp_ring_peek()`
returns up to two pointer/length pairs to handle wraparound without
copying.

`ccsp_slab.h` -- Slab allocator for fixed-size objects. Pre-allocate a
pool, hand out objects from free list, return to free list on release.
Zero per-object malloc overhead. Essential for high-frequency allocation
patterns (parse tree nodes, connection state objects).

`ccsp_size.h` -- Overflow-checked arithmetic for sizes.

```c
ccsp_size_t total;
if (!ccsp_size_mul(&total, count, sizeof(struct foo))) {
    /* overflow: count * sizeof(foo) does not fit in size_t */
    return CCSP_ERR_OVERFLOW;
}
p = ccsp_malloc(ctx, total);
```

Every integer overflow vulnerability in allocation size calculation --
and there are dozens in the wild -- is preventable by using these
functions. We will not have CVEs from this class of bug.

`ccsp_time.h` -- Time. `ccsp_monotonic_usec()` returns microseconds from
an unspecified epoch, monotonically increasing, unaffected by wall clock
adjustment. Essential for timeouts. Implemented via `gettimeofday()` on
BSD (with drift correction), `times()` on System V (scaled from clock
ticks), whatever each platform provides. You get microseconds; we handle
the insanity.

`ccsp_wallclock_usec()` returns wall clock time in microseconds since
1970. For logging timestamps, not for measuring durations.

`ccsp_sleep_usec()` sleeps. Interruptible by signals on most platforms.
Returns early if interrupted; check remaining time if you need exact
duration.

`ccsp_random.h` -- Secure random bytes. `ccsp_random_bytes(buf, len)`
fills a buffer with cryptographically strong random data.

1987 has no `/dev/urandom`. No `arc4random()`. No kernel entropy pool
on any shipping Unix. We ship kernel drivers:

- BSD: Character device driver, harvests interrupt timing jitter,
  maintains entropy pool, provides `/dev/urandom`.
- SunOS: Same model, loadable via `modload` on SunOS 4.x.
- System V: STREAMS module on SVR3.2+, character device on others.

Yes, this is extreme. Yes, shipping kernel code is a support burden.
But every cryptographic protocol needs random numbers, and "seed with
time(0) ^ getpid()" is how you get broken key generation for a decade.
We do this right or we do not do it.

`ccsp_random_uint32()` and `ccsp_random_range(min, max)` are convenience
wrappers. Use `ccsp_random_bytes()` for key material.

`ccsp_atomic.h` -- Atomic operations. On single-CPU systems (most
systems in 1987), these are trivial: disable interrupts or just do the
operation. On multi-CPU systems (some VAX configurations, Sequent
Balance, Encore Multimax), we use whatever hardware provides: VAX
interlocked instructions, or spin locks, or disable-migration-and-
interrupts.

```c
ccsp_atomic_t counter;
ccsp_atomic_init(&counter, 0);
ccsp_atomic_fetch_add(&counter, 1);  /* returns old value */
old = ccsp_atomic_exchange(&counter, new);
```

Also `ccsp_once_t` for one-time initialization. Essential when threads
become common. Essential even now for signal handlers.

`ccsp_log.h` -- Structured logging.

```c
CCSP_LOG_INFO(ctx, "connection accepted",
    CCSP_LOG_INT("fd", fd),
    CCSP_LOG_STR("peer", peer_addr));
```

Output is machine-parseable: timestamp, level, message, key=value pairs.
Not `printf` spaghetti that requires regex to extract fields. Filtering
by level at compile time (release builds omit DEBUG entirely). Backend
is pluggable: stderr default, syslog adapter, null for tests, capture
for test assertions.

**Dependencies:** libc only (plus our kernel drivers for random).

**Size:** ~4,000 lines across 18 headers, ~2,000 lines platform shims.

**Symbol count:** ~150 public symbols.

**Build:** `./configure` detects platform, writes `ccsp_config.h`,
generates Makefile. `make` builds `libccsp_base.a`. `make test` runs
test suite. `make install` installs headers and library. No autoconf
(autoconf is new and not yet proven). Hand-written configure in
`/bin/sh`, tested on every target.

---

## Layer 1: Core Services

### lib_ccsp_event

**What it is:** I/O event dispatch, timers, signals. Nothing else.

**What it is not:** DNS resolver. HTTP client. Buffer manager. Thread
pool.

**Symbol prefix:** `ccsp_ev_`

**Core types:**
- `ccsp_ev_base_t` -- event loop base, always explicit, never global
- `ccsp_ev_io_t` -- fd readability/writability watcher
- `ccsp_ev_timer_t` -- monotonic timer, min-heap implementation
- `ccsp_ev_signal_t` -- signal watcher via self-pipe

**Backends (selected at `ccsp_ev_base_create()` time):**
- `ccsp_ev_backend_select` -- universal fallback, FD_SETSIZE checked
- `ccsp_ev_backend_poll` -- POSIX.1-2001, no fd limit
- `ccsp_ev_backend_kqueue` -- FreeBSD, macOS, NetBSD, OpenBSD
- `ccsp_ev_backend_epoll` -- Linux 2.5.44+
- `ccsp_ev_backend_devpoll` -- Solaris `/dev/poll`

**Backend vtable:** add(fd, events), remove(fd), dispatch(timeout).
Four operations. Adding a new backend is one file.

**Seven kqueue behaviors handled:**
EV_EOF with pending data, EVFILT_SIGNAL vs EVFILT_PROC ordering,
kevent batch change/event interaction, EVFILT_READ on listening socket,
EV_CLEAR vs EV_ONESHOT semantics, kevent per-event error in result
list, timer precision variation across BSD variants.

**Dependencies:** lib_ccsp_base.

### lib_ccsp_mpi

**What it is:** Arbitrary precision integer arithmetic.

**Symbol prefix:** `ccsp_mpi_`

**Two namespaces:**
- `ccsp_mpi_*` -- variable-time, for public data
- `ccsp_mpi_ct_*` -- constant-time, for secret data

The namespace distinction is enforced by naming, not flags. Using
`ccsp_mpi_mul()` on secret data is a code review finding. Using
`ccsp_mpi_ct_mul()` on public data is slower than necessary but not
a vulnerability.

**Validation:** All variable-time operations validated against GMP
test vectors. Constant-time operations have timing variance tests in
the conformance suite.

**Dependencies:** lib_ccsp_base.

### lib_ccsp_regex

**What it is:** Regular expression compilation and matching.

**Symbol prefix:** `ccsp_rx_`

**Two engines:**
- `CCSP_RX_ENGINE_NFA` (default) -- Thompson NFA, O(n) guaranteed,
  no backreferences, no ReDoS
- `CCSP_RX_ENGINE_BACKTRACK` -- supports backreferences, no time
  guarantee, compiler warning on use

**Named captures:** First-class. `${name}` syntax. Unnamed captures
are syntactic sugar.

**UTF-8 mode:** `CCSP_RX_FLAG_UTF8`. `.` matches codepoints. `\w`
uses Unicode properties. Unicode property table ships as separate
data file: `ccsp_rx_unicode_props.dat`.

**Compile errors:** Include position in pattern, not just error code.
`"unmatched '(' at position 3"` not `CCSP_RX_ERR_PAREN`.

**Thread safety:** Compiled pattern (`ccsp_rx_pattern_t`) is immutable
and explicitly documented as safe to share across threads without
locking.

**Dependencies:** lib_ccsp_base.

### lib_ccsp_charconv

**What it is:** Encoding conversion. To UTF-8 from everything else.
From UTF-8 to everything else. Used at system boundaries only.

**Symbol prefix:** `ccsp_conv_`

**Supported encodings:**
ISO 8859-1 through ISO 8859-16, Shift-JIS, EUC-JP, EUC-KR, Big5,
GB2312, GBK, GB18030, UCS-2 BE/LE, UTF-16 BE/LE, UTF-32 BE/LE, ASCII.

**`ccsp_conv_detect()`:** Statistical encoding detection with
`confidence` output (0.0--1.0). Below 0.8: tell the user you are not
certain. BOM detection for UTF-8/16/32 before statistical analysis.

**BOM stripping:** Automatic for UTF-8 with BOM. The BOM is a Windows
artifact. Your UTF-8 does not need one. Emit a `CCSP_LOG_INFO` when
stripping a BOM so the event is observable.

**Dependencies:** lib_ccsp_base.

---

## Layer 2: I/O and Networking

### lib_ccsp_bio

**What it is:** Buffered I/O. Readers, writers, filters, formatting.

**Symbol prefix:** `ccsp_bio_`

**The separation OpenSSL's BIO failed to make:**
- `ccsp_bio_reader_t` and `ccsp_bio_writer_t` are different types
- Buffering (`ccsp_ring_t`), filtering (`ccsp_bio_filter_*`), and
  formatting (`ccsp_bio_fmt_*`) are separate modules
- TLS is not a filter. lib_ccsp_tls takes a reader and writer as inputs.
  It is not a `ccsp_bio_filter_t`.

**Concrete readers:**
`ccsp_bio_reader_from_fd()` -- file descriptor, blocking or nonblocking
`ccsp_bio_reader_from_mem()` -- read-only memory buffer
`ccsp_bio_reader_from_file()` -- stdio FILE*
`ccsp_bio_reader_from_ev()` -- libevent-integrated, async

**Concrete writers:**
`ccsp_bio_writer_to_fd()`, `ccsp_bio_writer_to_mem()`,
`ccsp_bio_writer_to_file()`, `ccsp_bio_writer_to_ev()`

**Filters (reader wrappers):**
`ccsp_bio_filter_base64_reader()` -- base64 decode on read
`ccsp_bio_filter_base64_writer()` -- base64 encode on write
`ccsp_bio_filter_zlib_reader()` -- inflate on read (when zlib available)
`ccsp_bio_filter_zlib_writer()` -- deflate on write

**Formatting (`ccsp_bio_fmt_*`):**
`ccsp_bio_fmt_write()` -- printf-style into writer
`ccsp_bio_fmt_write_hex()` -- hex dump
`ccsp_bio_fmt_write_pem()` -- PEM encoding with label

**Zero-copy path:** `ccsp_ring_reserve()` -> write directly -> 
`ccsp_ring_commit()`. No intermediate copy in the TLS record layer.

**Dependencies:** lib_ccsp_base, lib_ccsp_event.

### lib_ccsp_dns

**What it is:** Async DNS resolver. Full record type support. DNSSEC.
DANE. SRV. The replacement for `gethostbyname()`.

**Symbol prefix:** `ccsp_dns_`

**Record types fully parsed (no raw RDATA exposed):**
A, AAAA, NS, CNAME, SOA, PTR, MX, TXT, AAAA, SRV, NAPTR, DS, RRSIG,
NSEC, DNSKEY, TLSA, CAA.

**`ccsp_dns_response_t`:** answers, authority, additional sections.
`resolved_name` field shows what actually resolved (search domain
expansion visible to caller). `from_search` flag. `dnssec_validated`
flag.

**Security:** Per-query random source port (bound socket per query)
plus random query ID. Requires attacker to guess both port and ID for
cache poisoning. Random port uses `ccsp_random_uint16()`. Not `rand()`.

**TCP fallback:** Automatic on truncation (`tc` bit set). TCP connection
per nameserver reused for subsequent queries (multiple queries per TCP
connection per DNS spec, but most resolvers don't bother -- we do).

**DNSSEC validation chain:**
Query -> RRSIG -> DNSKEY -> DS (parent) -> ... -> root KSK trust anchor.
Trust anchors in `ccsp_dns_config_t`. Updated when IANA rotates root KSK.

**Cancellation guarantee:** After `ccsp_dns_query_cancel()` returns,
the callback will never fire. No races. No "might fire one more time."
Guaranteed by the event loop integration design.

**`ccsp_dns_config_from_os()`:** Platform-specific: reads
`/etc/resolv.conf` on POSIX, registry on Windows, `scutil` output on
macOS. The calling code does not know which.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio.

### lib_ccsp_mdns

**What it is:** RFC 6762 Multicast DNS and RFC 6763 DNS Service
Discovery. Zero-configuration networking for local networks.

**Symbol prefix:** `ccsp_mdns_`

**Integration with lib_ccsp_dns:** `.local` queries automatically route
to mDNS. No caller configuration. The resolver chooses.

**`ccsp_mdns_announce()`:** Service announcement with automatic renewal
at RFC-specified intervals. Conflict detection and resolution per RFC.
Name change callback when conflict forces a rename.

**`ccsp_mdns_browse()`:** Service discovery. `on_found` and `on_lost`
callbacks. Service instance includes name, hostname, port, TXT records.

**DNS-SD (RFC 6763):** PTR record browse for service type discovery.
SRV resolution for instance details. TXT record metadata. The full
chain: type browse -> instance browse -> SRV resolve -> A/AAAA resolve ->
connect. lib_ccsp_connect follows this chain automatically for `.local`
service names.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio,
lib_ccsp_dns.

### lib_ccsp_connect

**What it is:** Stream connection. TCP connection establishment with
every correctness property that hand-rolled code misses.

**Symbol prefix:** `ccsp_conn_`

**The SO_ERROR check:** After a nonblocking connect, writability does
not mean connected. `SO_ERROR` must be checked. 19 CVEs in the corpus.
lib_ccsp_connect checks `SO_ERROR`. Always. Without exception.

**Happy eyeballs (RFC 6555):**
- Start IPv6 attempt
- After 50ms if not connected, start IPv4 attempt in parallel  
- Use first to connect, cancel the other
- 50ms: enough for IPv6 to succeed if working, not enough for timeout
- Maximum user-visible penalty for broken IPv6: 50ms

**SRV-based selection (RFC 2782):**
- Weighted random within priority group using `ccsp_random_uint64()`
- Not `rand()`. Predictable selection enables steering attacks.
- `weight + 1` for zero-weight records per RFC spec
- Priority group fallback on connection failure

**DANE enforcement:**
- Automatic TLSA query for connection target
- DNSSEC-validated TLSA -> enforce against TLS certificate
- Usage 3 (domain-issued): bypass CA hierarchy
- `CCSP_CONN_DANE_OPPORTUNISTIC` / `_REQUIRED` / `_DISABLED`

**Error taxonomy:** `CCSP_CONN_ERR_DNS_NXDOMAIN`,
`CCSP_CONN_ERR_CONNECT_REFUSED`, `CCSP_CONN_ERR_CONNECT_TIMEOUT`,
`CCSP_CONN_ERR_ALL_FAILED`, `CCSP_CONN_ERR_BUDGET_EXCEEDED`,
`CCSP_CONN_ERR_TLS_HANDSHAKE`, `CCSP_CONN_ERR_TLS_CERT`,
`CCSP_CONN_ERR_TLS_DANE_MISMATCH`. Source tracking on all errors.

**`ccsp_conn_service_info()`:** After connection, service TXT records
available. Used by DNS-SD applications.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio,
lib_ccsp_dns, lib_ccsp_tls.

---

## Layer 3: Protocols

### lib_ccsp_lineproto

**What it is:** A declarative state machine engine for line-based
protocols. SMTP, IMAP, POP3, FTP, Redis, memcached, IRC -- all share
the same structure. lib_ccsp_lineproto provides it once.

**Symbol prefix:** `ccsp_lp_`

**The 47 CVEs in the corpus for hand-rolled line readers** are the
reason this library exists. Every one of them is structurally
impossible in lib_ccsp_lineproto because the library reads lines,
parses verbs, dispatches to handlers, and enforces state -- and the
application never touches a buffer.

**Schema types:**
- `ccsp_lp_cmd_t` -- verb, argument type, allowed states bitmask,
  pipeline_ok flag, auth_required flag, max_per_minute, callback,
  help text
- `ccsp_lp_state_t` -- id, name, timeout_ms, greeting code/message,
  error code/message, on_enter/on_timeout/on_exit callbacks
- `ccsp_lp_proto_t` -- command table, state table, initial state,
  max_line_len, CRLF policy, pipeline config, STARTTLS config,
  on_connect/on_disconnect callbacks

**`allowed_states` bitmask:** The library enforces which commands are
valid in which states before calling the handler. The application never
writes `if (!authenticated) { send_error(); return; }` -- that logic
is in the schema.

**Data mode:** `ccsp_lp_data_mode_t`. DELIMITED (SMTP dot-stuffing),
LENGTH (IMAP literals `{1234}`), BINARY (FTP image mode), CUSTOM
(application-defined framing). The library switches to data mode,
reads the data, fires `on_complete`, returns to line mode.

**Client:** Same protocol descriptor. Server and client share the
schema. A test creates both in the same event loop with no network.
End-to-end protocol testing without a socket.

**STARTTLS:** After handler returns `CCSP_LP_STATE_PUSH`, library sends
response, flushes, wraps fd with lib_ccsp_tls, performs handshake,
resets state machine to initial_state, clears auth status. Correct per
RFC 3207.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio,
lib_ccsp_connect.

### lib_ccsp_http

**What it is:** HTTP/1.1 (RFC 7230--7235) and HTTP/2 (RFC 7540).
Protocol only. Not a web framework.

**Symbol prefix:** `ccsp_http_`

**Version negotiation:** ALPN in TLS (`h2` vs `http/1.1`). Automatic.
The caller does not know which version was negotiated unless they ask.

**HTTP/1.1 implementation:** Built on lib_ccsp_lineproto. Headers are
line protocol. Body is data mode. Chunked transfer encoding is a
bio_filter.

**HTTP/2 implementation:** Binary framing protocol. HPACK header
compression. Stream multiplexing. Per-stream flow control. Server push
(optional, configurable). Built separately from HTTP/1.1 -- sharing
code between them causes more complexity than it saves.

**Request/response body:** Always a `ccsp_bio_reader_t`. No buffering
of entire bodies. A 10GB response body is a stream, not a buffer.
Applications that want buffering wrap the reader with a `ccsp_buf_t`.

**Connection pooling (client):** Per-host connection pool. H2 uses one
connection per host (multiplexed). H1.1 uses configurable pool size.
Keep-alive managed automatically.

**Server routing:** Not in lib_ccsp_http. The library calls
`on_request()`. Routing is the application's concern.

**WebSocket upgrade:** `ccsp_http_upgrade_websocket()`. After upgrade
the connection is a `ccsp_bio_reader_t` / `ccsp_bio_writer_t` pair with
WebSocket framing. The HTTP library steps aside.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio,
lib_ccsp_connect, lib_ccsp_lineproto, lib_ccsp_tls.

### lib_ccsp_smtp

**What it is:** RFC 5321 SMTP. Client and server. STARTTLS. DANE
enforcement for MX connections. DKIM signing and verification.
SMTP-AUTH.

**Symbol prefix:** `ccsp_smtp_`

**The sendmail motivation:** The sendmail CVE hall of shame documents
approximately one major vulnerability per year through the 1990s. Most
are line parsing bugs in the class lib_ccsp_lineproto prevents. The
SMTP server in lib_ccsp_smtp is lib_ccsp_lineproto applied to RFC 5321.

**DANE MX:** When sending to a domain, lib_ccsp_smtp looks up MX
records, then TLSA records for each MX host. If DNSSEC validates the
TLSA records, certificate validation is against the TLSA record.
Authenticated encrypted email between DANE-enabled servers without CA
involvement.

**DKIM:** Signing outbound messages. Verifying inbound DKIM signatures.
Key management via DNS TXT records. Per-domain key selection.

**Pipeline support:** SMTP pipelining (RFC 2920). The DATA command
correctly flushes the pipeline before switching to data mode.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio,
lib_ccsp_connect, lib_ccsp_lineproto, lib_ccsp_dns, lib_ccsp_ocl.

### lib_ccsp_imap

**What it is:** IMAP4rev1 (RFC 3501). Client and server.

**Symbol prefix:** `ccsp_imap_`

**The literal challenge:** IMAP has literal strings (`{1234}` preceding
exactly 1234 bytes). The state machine must switch from line mode to
length-delimited data mode mid-command. lib_ccsp_lineproto's
`CCSP_LP_DATA_LENGTH` mode handles this. The IMAP implementation uses
it.

**Client operations:**
Authenticate (LOGIN, PLAIN, XOAUTH2), SELECT/EXAMINE folders, FETCH
messages with MIME parsing, SEARCH with full IMAP search grammar,
STORE flags, COPY/MOVE messages, SUBSCRIBE/UNSUBSCRIBE, LIST/LSUB,
IDLE for push notification.

**MIME parsing:** BODYSTRUCTURE response parsed into a typed tree.
Individual MIME parts fetchable. Attachment extraction without
buffering the entire message.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio,
lib_ccsp_connect, lib_ccsp_lineproto, lib_ccsp_ocl.

---

## Layer 4: Cryptography and Security

### lib_ccsp_asn1

**What it is:** BER/DER/CER encoder and decoder. The ASN.1 codec.

**Symbol prefix:** `ccsp_asn1_`

**The ANY policy:** lib_ccsp_asn1 does not implement `ANY` or
`ANY DEFINED BY`. Any specification requiring `ANY` is an incomplete
specification. The response when encountered in an IETF working group:
"Write the type. If you cannot write the type, you have not finished
the design. Come back when you have finished the design."

**Schema-driven parsing:** Schemas are written in ASN.1 module syntax.
A verified generator produces the parser. The generator is verified
once. All generated parsers are correct by construction for the grammar.

**Strict DER enforcement:**
- Indefinite-length encoding: parse error
- Non-minimal length encodings: parse error
- Non-canonical boolean (anything other than 0x00 or 0xFF): parse error
- Constructed encoding where primitive required: parse error

**Security limits:**
- Maximum nesting depth: 64 (configurable, hard-coded maximum 128)
- Maximum object size: configurable, default 16MB
- Maximum sequence/set member count: 1024

**String normalization:** All string types returned as UTF-8 regardless
of ASN.1 type. UTF8String, PrintableString, IA5String, T61String,
BMPString, UniversalString: all normalized to UTF-8 before the caller
sees them. The caller never sees `ASN1_T61STRING`.

**Unknown critical extensions:** Parse error, not silent acceptance.
Unknown non-critical extensions: returned as opaque bytes with OID.
The caller decides whether to process them.

**Dependencies:** lib_ccsp_base.

### lib_ccsp_cert

**What it is:** Three sub-libraries. One header. Clean interfaces
between them.

**Symbol prefix:** `ccsp_cert_`

**certparse module:**
- Input: DER bytes
- Output: `ccsp_cert_t *` (opaque, immutable, reference-counted)
- Uses lib_ccsp_asn1 for DER parsing
- Parsed certificate is frozen. No mutation. Build a new cert via
  lib_ccsp_pki.
- All strings normalized to UTF-8 by lib_ccsp_asn1 before certparse
  sees them
- Subject and issuer DN exposed as ordered arrays of typed attributes,
  not opaque strings
- `ccsp_cert_get_string(cert, "CN")` for simple access
- `ccsp_cert_get_san_dns(cert, index)` for SAN DNS names
- `ccsp_cert_get_san_ip(cert, index)` for SAN IP addresses

**certval module:**
- RFC 5280 path validation algorithm
- Implemented from the formal Coq specification
- Explicit `ccsp_time_t at_time` parameter -- never calls `time()`
  internally, making it fully testable for time-sensitive validation
- Policy OID validation
- Name constraints
- Extended key usage
- `ccsp_cert_validate()` returns typed result: valid, or specific error
  (expired, not yet valid, unknown CA, revoked, policy mismatch, etc.)

**ca-nav module:**
- AIA chasing: fetch intermediate certificates from issuer URL
- OCSP: query revocation status
- CRL: fetch and parse certificate revocation lists
- The fetch callback: `int (*fetch)(const char *url, ccsp_bio_writer_t
  *out, void *userdata)`. lib_ccsp_http provides this callback for
  production use. A mock provides it for testing.
- Result caching with TTL: next OCSP check time, CRL next update
- Stapled OCSP responses accepted from TLS handshake

**DANE module (within lib_ccsp_cert):**
- `ccsp_cert_validate_dane()`: validate certificate against TLSA record
- TLSA usage 0/1/2/3 handling
- TLSA selector 0 (full cert) and 1 (public key)
- TLSA matching type 0 (exact), 1 (SHA-256), 2 (SHA-512)
- Constant-time comparison always. The timing of DANE validation must
  not leak information.

**Dependencies:** lib_ccsp_base, lib_ccsp_asn1, lib_ccsp_ocl (for
TLSA hash computation).

### lib_ccsp_pki

**What it is:** Certificate and key generation. The write side of PKI.
Where lib_ccsp_cert reads, lib_ccsp_pki writes.

**Symbol prefix:** `ccsp_pki_`

**Key generation:**
- `ccsp_pki_keygen_rsa()` -- RSA, 2048/3072/4096 bits
- `ccsp_pki_keygen_ec()` -- ECDSA/ECDH, P-256/P-384/P-521/X25519/X448
- `ccsp_pki_keygen_ed()` -- EdDSA, Ed25519/Ed448
- All return opaque `ccsp_pki_key_t *`. Key material never exposed as
  struct member. Serialization only via explicit export functions.

**CSR generation:**
- `ccsp_pki_csr_create()` -- builds CSR from key and subject attributes
- Subject attributes as typed key-value pairs, not raw DN string
- SAN extensions: DNS names, IP addresses, email addresses, URIs
- Returns DER-encoded CSR as `ccsp_buf_t`

**Certificate signing:**
- `ccsp_pki_cert_sign()` -- sign a CSR with a CA key
- Self-signed: `ccsp_pki_cert_self_sign()` for DANE usage 3
- Serial number generation: random 128-bit, or sequential with caller
  control
- Validity period: explicit start and end times, not "now + N days"
- Extension handling: EKU, basic constraints, key usage, SAN, AIA, CDP

**PKCS#12:**
- `ccsp_pki_p12_import()` -- extract key and cert chain from .p12/.pfx
- `ccsp_pki_p12_export()` -- pack key and cert chain into .p12/.pfx
- Password-based encryption: AES-256-CBC with PBKDF2 (not the legacy
  RC2/DES that many PKCS#12 files still use)

**Dependencies:** lib_ccsp_base, lib_ccsp_asn1, lib_ccsp_cert,
lib_ccsp_ocl, lib_ccsp_mpi.

### lib_ccsp_ocl

**What it is:** Cryptographic primitives, schemes, and AEAD
constructions. The OCL stands for Open Crypto Library.

**Symbol prefix:** `ccsp_ocl_`

**Primitives layer** (pure functions, no state, caller provides output
buffer):

Symmetric:
- `ccsp_ocl_aes128_block()`, `ccsp_ocl_aes256_block()` -- single block
- `ccsp_ocl_chacha20_block()` -- 64-byte keystream block

Hash:
- `ccsp_ocl_sha256()`, `ccsp_ocl_sha384()`, `ccsp_ocl_sha512()`
- `ccsp_ocl_sha3_256()`, `ccsp_ocl_sha3_512()`
- `ccsp_ocl_blake2b()`, `ccsp_ocl_blake3()`

MAC:
- `ccsp_ocl_poly1305()` -- Poly1305 one-time MAC
- `ccsp_ocl_ghash()` -- GHASH for GCM mode

**Schemes layer** (stateful operations on key objects):

`ccsp_ocl_hmac_*` -- HMAC-SHA256, HMAC-SHA512. `_init()`, `_update()`,
`_final()`. Streaming interface.

`ccsp_ocl_hkdf_*` -- HKDF extract and expand.

`ccsp_ocl_pbkdf2_*` -- Password-based key derivation.

`ccsp_ocl_argon2_*` -- Memory-hard password hashing. Argon2id default.

`ccsp_ocl_rsa_*` -- RSA sign, verify, encrypt, decrypt. PKCS#1 v1.5
(legacy support only) and OAEP/PSS. Private operations constant-time
via lib_ccsp_mpi constant-time.

`ccsp_ocl_ecdsa_*` -- ECDSA sign and verify. P-256, P-384, P-521.
Deterministic nonce generation (RFC 6979) by default.

`ccsp_ocl_ecdh_*` -- ECDH key agreement. P-256, P-384, X25519, X448.

`ccsp_ocl_eddsa_*` -- EdDSA sign and verify. Ed25519, Ed448.

**AEAD layer** (the blessed interface -- what applications should use):

`ccsp_ocl_aead_t` -- opaque key context

`ccsp_ocl_aead_create()` -- algorithm (AES-256-GCM or
CHACHA20-POLY1305), key material

`ccsp_ocl_aead_seal()` -- encrypt and authenticate. Nonce generated
internally from a per-context counter + random base. **The application
cannot reuse a nonce.** This is not optional. The interface makes it
structurally impossible.

`ccsp_ocl_aead_open()` -- authenticate and decrypt. Rejects any
ciphertext with invalid authentication tag. Constant-time tag comparison.

`ccsp_ocl_aead_destroy()` -- zeroing free of key material

**Constant-time discipline:**
All operations on secret data are constant-time. The definition of
"secret data": private keys, AEAD keys, MAC keys, private scalars,
MAC tags (comparison), password hashes (comparison). Any value that
must not be inferred from timing.

The constant-time property is tested. The test suite includes timing
variance measurements using dudect methodology: `t-test` on timing
samples from two different inputs. A failing timing test is a build
failure.

**FIPS 140-3 boundary:**
Explicitly designed in. The FIPS module boundary, the approved algorithm
list, the conditional and power-on self-tests, and the key zeroization
procedures are not retrofitted. FIPS validation is a certification
exercise, not a porting project.

**NIST CAVP test vectors:** Every algorithm has known-answer tests from
the NIST Cryptographic Algorithm Validation Program. No algorithm ships
without passing all applicable CAVP vectors.

**Hardware acceleration:** AES-NI on x86/x86_64. ARMv8 crypto
extensions on AArch64. Detected at runtime. The software path is always
available as fallback and as reference for testing.

**Dependencies:** lib_ccsp_base, lib_ccsp_mpi, lib_ccsp_asn1 (for key
serialization).

### lib_ccsp_tls

**What it is:** TLS 1.2 (RFC 5246) and TLS 1.3 (RFC 8446) protocol
implementation.

**Symbol prefix:** `ccsp_tls_`

**Why Heartbleed was structurally inevitable in OpenSSL:** The record
layer and handshake layer shared buffer management. The handshake layer
passed a user-supplied length field to the record layer without
validation. In lib_ccsp_tls the record layer and handshake layer have
separate buffer management. The record layer has no access to
user-supplied handshake fields. This class of vulnerability is
structurally impossible.

**Explicit state machine:** A table of states and transitions. A
transition table entry: (current state, event) -> (action, next state).
Invalid transitions are detected at init time, not at runtime when the
invalid input arrives. No implicit state scattered across function calls.

**SNI:** Required in ClientHello. Not optional. `ccsp_tls_config_t`
requires a `server_name` field. A TLS connection without SNI is an
error at config validation time, before the handshake begins. Privacy
note: SNI leaks the target hostname. Encrypted SNI (RFC draft) is
tracked as a planned feature.

**Forward secrecy:** Mandatory. All cipher suites use ephemeral key
exchange. ECDHE or DHE. There is no configuration option to disable
this. The configuration option would be: `ccsp_tls_config_t.i_understand
_that_this_makes_past_sessions_decryptable = true`. We are not adding
this option.

**No renegotiation:** TLS renegotiation (RFC 5746) was a design mistake.
It is not implemented. A renegotiation request from a peer results in
the connection being closed with `CCSP_TLS_ERR_RENEGOTIATION_REQUESTED`.
The error is logged at `CCSP_LOG_WARN` with the peer address so the
event is observable.

**ALPN:** Required for protocol negotiation. Supported protocols
declared in `ccsp_tls_config_t.alpn_protocols[]`. The negotiated
protocol is available via `ccsp_tls_get_alpn()` after handshake.

**Session tickets (TLS 1.3 only):** For 0-RTT and resumption. Ticket
keys rotated on schedule. Old keys retained briefly for in-flight
resumptions. Ticket replay detection.

**Certificate validation:** Calls lib_ccsp_cert certval. The validation
policy (required leaf EKU, allowed signature algorithms, minimum key
sizes) is configurable via `ccsp_tls_config_t.cert_policy`.

**OCSP stapling:** Accepted from server, passed to lib_ccsp_cert certval.
Fetched proactively by lib_ccsp_cert ca-nav for server mode.

**Dependencies:** lib_ccsp_base, lib_ccsp_ocl, lib_ccsp_cert, lib_ccsp_bio.

---

## Layer 5: Data and Parsing

### lib_ccsp_data

**What it is:** The shared structured data value type used across the
data layer.

**Symbol prefix:** `ccsp_data_`

**`ccsp_data_value_t` type enum:**
`CCSP_DATA_NULL`, `CCSP_DATA_BOOL`, `CCSP_DATA_INTEGER` (int64_t),
`CCSP_DATA_FLOAT` (double), `CCSP_DATA_STRING` (UTF-8 ccsp_str_t),
`CCSP_DATA_BYTES` (raw bytes, base64 in JSON), `CCSP_DATA_ARRAY`,
`CCSP_DATA_TABLE` (ordered, via lib_ccsp_hash)

**Integer is int64_t, not double.** JSON's double-for-all-numbers
is wrong for integers above 2^53 (9007199254740992). lib_ccsp_data
distinguishes integer and float. A number `42` without decimal or
exponent is integer. A number `42.0` is float. Applications that need
integer precision get it without configuration.

**`ccsp_data_get(root, "database.server.port", NULL)`:** Dot-separated
path navigation. Returns NULL if path absent. Default value parameter
for typed accessors.

**Typed accessors with defaults:**
```c
const char *host = ccsp_data_get_string(cfg, "db.host", "localhost");
int64_t port     = ccsp_data_get_integer(cfg, "db.port", 5432);
bool verbose     = ccsp_data_get_bool(cfg, "logging.verbose", false);
```

**Immutability:** Parsed structures are immutable. `ccsp_data_clone()`
creates a mutable deep copy if modification is needed.

**Insertion order preserved:** Tables are `ccsp_hash_t` instances.
lib_ccsp_hash preserves insertion order. TOML files round-trip with
the same key order. JSON objects round-trip with the same key order.

**Serialization:** `ccsp_data_write_json()` and `ccsp_data_write_toml()`
to `ccsp_bio_writer_t`. Compact or pretty-printed.

**Dependencies:** lib_ccsp_base (including lib_ccsp_hash).

### lib_ccsp_toml

**What it is:** TOML parser producing `ccsp_data_value_t`.

**Symbol prefix:** `ccsp_toml_`

**The TOML specification** is published alongside the library as
`TOML-1.0-ccsp.md`. The implementation is the reference implementation
of that specification. Test suite passes `toml-test` (the canonical
TOML conformance suite).

**Types:** String (UTF-8), Integer (int64_t), Float (double), Boolean,
Offset Date-Time (ISO 8601), Local Date-Time, Local Date, Local Time,
Array (homogeneous), Inline Table, Table, Array of Tables.

**Parse errors:** Line number, column number, human-readable message.
Not error codes. `ccsp_toml_err_t` extends `ccsp_err_t` with `line` and
`column` fields.

**Multiline strings:** Both basic (`"""`) and literal (`'''`). Escape
sequences only in basic strings.

**lib_ccsp_cli integration:** `ccsp_cli_config_t.config_file` reads a
TOML file. Config keys specified in the option descriptor as
`config_key = "section.key"`. Source tracking reports
`CCSP_CLI_SOURCE_CONFIG_FILE` with the key path.

**Dependencies:** lib_ccsp_base, lib_ccsp_data.

### lib_ccsp_json

**What it is:** JSON parser. Both DOM (full parse to value tree) and
streaming SAX-style events. RFC 8259 conformant.

**Symbol prefix:** `ccsp_json_`

**DOM parser:** `ccsp_json_parse()` takes bytes, returns
`ccsp_data_value_t *`. Full document buffered.

**Streaming parser:**
```c
ccsp_json_parser_t *p = ccsp_json_parser_create(on_event_cb, userdata);
ccsp_json_parser_feed(p, chunk, chunk_len);  // call as chunks arrive
ccsp_json_parser_finish(p, &err);
```
Events: `CCSP_JSON_EVENT_NULL`, `_BOOL`, `_INTEGER`, `_FLOAT`,
`_STRING`, `_ARRAY_START`, `_ARRAY_END`, `_OBJECT_START`,
`_OBJECT_KEY`, `_OBJECT_END`. String event gives pointer into input
buffer -- zero allocation for string values.

**lib_ccsp_event integration:** Feed the streaming parser from the
`on_readable` callback of a `ccsp_bio_reader_t`. Events fire as
complete tokens arrive from the network. No need to buffer the entire
document before processing.

**Security limits:**
- `max_depth`: default 64, configurable, hard max 512
- `max_string_len`: default 1MB, configurable
- `max_document_size`: default 16MB, configurable
- UTF-8 validation: invalid UTF-8 in a string is a parse error

**Number parsing:** `strtoll()` for integers, `strtod()` for floats.
Not hand-rolled. `strtoll()` range: [INT64_MIN, INT64_MAX]. Out-of-range
integers are a parse error, not silent truncation.

**JSONTestSuite:** All mandatory test cases pass. Optional cases
documented.

**Dependencies:** lib_ccsp_base, lib_ccsp_data, lib_ccsp_bio (streaming
parser).

### lib_ccsp_cas

**What it is:** Content-addressable storage. SHA-256 hash of content
equals its address.

**Symbol prefix:** `ccsp_cas_`

**Object types:**
- Blob: raw bytes
- Tree: ordered map of (name, mode, hash) entries
- Commit: tree hash + optional parent hash + metadata (author, timestamp,
  message)

These are Git's objects. Git's data model insight is correct.
lib_ccsp_cas provides the model without the version control UI.

**`ccsp_cas_put(store, data, len, hash_out)`:** Content-addressed write.
Returns SHA-256 of content. Idempotent: writing the same content twice
returns the same hash.

**`ccsp_cas_get(store, hash, data_out, len_out)`:** Content-addressed
read. Returns NULL if hash not found. Hash verification on read:
computed hash must match requested hash or data is corrupt.

**Store backends:**
- Filesystem: one file per object, content hash as filename
- Memory: hash table, for testing and transient stores
- Remote: delegate to another store (for distributed scenarios)

**Use cases documented with examples:**
- Build system: skip recompile when input hash unchanged
- Package management: package identity is content hash
- Configuration management: system state as CAS tree, changes as commits
- Distributed sync: content-addressed data replicates without coordination

**Dependencies:** lib_ccsp_base, lib_ccsp_ocl (SHA-256).

### lib_ccsp_cli

**What it is:** Declarative command-line option parsing. Schema-driven.
Type-safe. Callback-based. Source-tracked.

**Symbol prefix:** `ccsp_cli_`

**Option types:**
`CCSP_CLI_OPT_BOOL`, `CCSP_CLI_OPT_STRING`, `CCSP_CLI_OPT_INT`,
`CCSP_CLI_OPT_UINT`, `CCSP_CLI_OPT_FLOAT`, `CCSP_CLI_OPT_SIZE` (with
k/m/g/kb/mb/gb suffixes), `CCSP_CLI_OPT_DURATION` (with ms/s/m/h
suffixes, stored as uint64_t microseconds), `CCSP_CLI_OPT_PATH`,
`CCSP_CLI_OPT_ENUM`, `CCSP_CLI_OPT_LIST` (accumulates values),
`CCSP_CLI_OPT_COUNT` (counts occurrences of flag)

**The `store` union:** `store.i = &my_int`. The parser writes directly
into application variables. No copy step. No intermediate struct.

**Source priority chain:** Command line -> environment variable -> TOML
config file -> default value. Implemented once. Correctly.

**Source tracking:** After parsing, `ccsp_cli_result_t` reports which
source set each option and what value came from that source.
`ccsp_cli_print_effective_config()` shows all options, their values,
and their sources. "Why is timeout 30000 instead of 5000? It was set
by environment variable APP_TIMEOUT."

**Help generation:** Generated from the schema. Cannot drift. Format:
```
  -h, --host=HOSTNAME   server hostname [required] [$APP_HOST]
  -p, --port=PORT       server port (1-65535) [default: 443] [$APP_PORT]
  -t, --timeout=DURATION connection timeout e.g. 5s, 500ms [default: 30s]
```

**Subcommands:** Nested `ccsp_cli_cmd_t`. The parser dispatches to the
selected subcommand's `run()` callback. No dispatch loop in application
code.

**Validation:** min/max on numeric types. Regex pattern on string types
(via lib_ccsp_regex). Custom validator callback. Errors before `run()` is
called, not inside it.

**`--no-*` negation:** Boolean options automatically get `--no-*`
variants. `--verbose` sets true. `--no-verbose` sets false.

**Dependencies:** lib_ccsp_base, lib_ccsp_toml (config file),
lib_ccsp_regex (pattern validation).

---

## Layer 6: Graphics and UI

### lib_ccsp_draw

**What it is:** 2D drawing primitives. Backend-agnostic. All drawing
calls are identical regardless of output target.

**Symbol prefix:** `ccsp_draw_`

**The backend interface:**
```c
typedef struct {
    const char *name;
    ccsp_draw_canvas_t *(*canvas_create)(...);
    void               (*canvas_destroy)(ccsp_draw_canvas_t *);
    void               (*frame_begin)(ccsp_draw_canvas_t *);
    void               (*frame_end)(ccsp_draw_canvas_t *);
    void               (*frame_present)(ccsp_draw_canvas_t *);
    void               (*fill_rect)(...);
    void               (*stroke_path)(...);
    void               (*fill_path)(...);
    void               (*draw_image)(...);
    void               (*draw_glyph)(...);
    void               (*save)(...);
    void               (*restore)(...);
    void               (*clip_rect)(...);
    void               (*clip_path)(...);
    void               (*transform)(...);
    void               (*set_opacity)(...);
    ccsp_draw_size_t    (*measure_glyph)(...);
    /* optional -- NULL if unsupported */
    void               (*composite)(...);
    ccsp_draw_canvas_t *(*create_offscreen)(...);
} ccsp_draw_backend_vtable_t;
```

Twelve required operations. Two optional. Adding a new backend is one
file that implements these operations.

**Coordinates are floats.** High-DPI displays have logical coordinates
that do not map 1:1 to pixels. The widget system never thinks in integer
pixels. The backend handles DPI scaling.

**Text decomposed before the backend sees it.** lib_ccsp_text decomposes
text into `ccsp_draw_glyph_t` objects with positions. The backend draws
pre-positioned glyphs. Backends do not do text layout. Text layout is
in lib_ccsp_text, done once for all backends.

**Paint types:** `ccsp_draw_paint_t` is a union: solid color (RGBA),
linear gradient (start, end, color stops), radial gradient, image
pattern, or custom (backend-defined). Solid color is the common case
and is always supported. Gradients are optional per backend.

**Path operations:** `ccsp_draw_path_t`. Move-to, line-to, quadratic
bezier, cubic bezier, arc, close. Paths are immutable after creation.

**Frame lifecycle:** `frame_begin` (start building frame), `frame_end`
(finish building, may submit to GPU), `frame_present` (make visible,
may sync). Between `frame_begin` and `frame_end` no blocking. Between
`frame_end` and `frame_present` GPU submission may occur asynchronously.

**`ccsp_draw_backend_detect()`:** Prefer GPU if available. Fall back to
X11 if DISPLAY set. Fall back to framebuffer if `/dev/fb0` accessible.
Fall back to imagemap always. The imagemap backend always succeeds.

**No X11 calls above lib_ccsp_draw.** This is enforced by the build
system's symbol checker. When Wayland replaces X11, only
lib_ccsp_draw_x11 changes.

**Dependencies:** lib_ccsp_base, lib_ccsp_event.

### lib_ccsp_draw_x11

X11 drawing backend. XLib + XRender when available. XShm for local
display (zero-copy via shared memory). Double-buffered. Damage tracking:
only repaints changed regions per frame. XRender for compositing and
antialiased rendering (X11R6.8+). Xft/FreeType for client-side font
rendering.

### lib_ccsp_draw_fb

Linux framebuffer (`/dev/fb0`) backend. Also supports custom embedded
targets via the `flush()` callback. Double-buffered: front and back
buffer, page-flip or memcpy depending on hardware. Software rasterizer:
Bresenham lines, scanline fill for paths, bilinear filter for images.
The `flush()` callback is the platform hook for DMA, SPI, or memory-
mapped register updates.

### lib_ccsp_draw_img

Software rasterizer to memory buffer. No display required. Output:
PNG (via libpng), PPM (no dependencies), raw RGBA, or PDF (path-
recording mode).

This is the reference implementation. When GPU output differs from
imagemap output, imagemap is correct. Used for visual regression testing
(pixel-perfect comparison), documentation generation (all code examples
rendered from actual code), server-side rendering, and print/PDF output.

`ccsp_draw_img_diff()`: compare two canvases pixel by pixel. Returns
max pixel difference and changed-pixel count. Used by lib_ccsp_test for
visual regression assertions.

### lib_ccsp_draw_gpu

OpenGL 2.1+ backend. Drawing calls build a command buffer between
`frame_begin` and `frame_end`. `frame_end` submits to GPU. Zero CPU-GPU
synchronization per draw call. Paths tessellated to triangles.
Text rendered via glyph atlas (one draw call per paragraph). Images as
texture quads. Opacity via blend state. Clip rect via scissor test.
Transform via model matrix. Complex clip paths fall back to software
for rasterization then upload as alpha texture.

### lib_ccsp_font

**What it is:** Font loading and glyph rasterization. A clean adapter
over FreeType.

**Symbol prefix:** `ccsp_font_`

**Why not write our own:** FreeType handles font format parsing (TTF,
OTF, WOFF), hinting, anti-aliasing, and metrics. Our job is to wrap
FreeType so the rest of the stack uses fonts without knowing FreeType
exists.

**`ccsp_font_load_file()`:** Load font from file path.
**`ccsp_font_load_mem()`:** Load font from memory buffer.
**`ccsp_font_rasterize()`:** Rasterize one glyph to a bitmap.
**`ccsp_font_metrics()`:** Ascender, descender, line height, advance.
**`ccsp_font_kerning()`:** Kerning adjustment between two glyphs.

**Glyph atlas management:** For GPU rendering. A large texture
containing rasterized glyphs. `ccsp_font_atlas_t` tracks glyph
positions. Shelf-packing algorithm. Cache miss triggers rasterization
and atlas update.

**Dependencies:** lib_ccsp_base, FreeType (external, wrapped).

### lib_ccsp_text

**What it is:** Text layout. The library between a string and a list
of positioned glyphs.

**Symbol prefix:** `ccsp_text_`

**Shaping:** OpenType feature processing (ligatures, kerning, glyph
substitution). HarfBuzz integration (external, wrapped) for complex
scripts. Fallback: basic advance-width positioning for Latin scripts
without HarfBuzz.

**Bidirectional text:** Unicode Bidirectional Algorithm (UAX #9). Full
implementation. Level calculation, reordering, bracket matching.

**Line breaking:** Unicode Line Break Algorithm (UAX #14). Mandatory
breaks (newline, paragraph separator). Opportunity breaks (spaces,
after hyphens, between CJK ideographs). Prohibited breaks (before
closing punctuation, within sequences that must not be separated).

**Paragraph layout:**
- Input: UTF-8 string, font, size, width constraint
- Output: array of `ccsp_text_line_t`, each containing positioned glyphs
- Justification: none, left, center, right, full (distributes space)
- Hyphenation: optional, pattern-based (TeX hyphenation algorithm via
  lib_ccsp_regex)
- Ellipsis: truncation with `...` when text exceeds height constraint

**`ccsp_text_hit_test()`:** Given a pixel position, return the
codepoint offset in the original string. For cursor placement.

**`ccsp_text_selection_rects()`:** Given start and end codepoint
offsets, return rectangles covering the selection. For selection
rendering.

**Dependencies:** lib_ccsp_base, lib_ccsp_font, lib_ccsp_regex
(hyphenation patterns).

### lib_ccsp_widget

**What it is:** The widget system. Flutter's layout model in C.

**Symbol prefix:** `ccsp_wid_`

**The layout contract:**
1. Constraints go down: parent sends min/max width/height to child
2. Sizes go up: child reports its size to parent
3. Parent sets position: parent places child at a specific offset

This contract is never broken. By any widget. Ever. Layout is a single
tree traversal. O(n) in the number of widgets. No layout cycles. No
layout thrashing.

**The five widget methods** (the entire widget interface):
```c
typedef struct {
    ccsp_wid_size_t  (*measure)(ccsp_wid_t *, ccsp_wid_constraints_t);
    void            (*layout)(ccsp_wid_t *, ccsp_wid_size_t);
    void            (*paint)(ccsp_wid_t *, ccsp_draw_canvas_t *);
    bool            (*event)(ccsp_wid_t *, const ccsp_input_event_t *);
    bool            (*hit_test)(ccsp_wid_t *, ccsp_wid_point_t);
} ccsp_wid_vtable_t;
```

Every widget -- built-in or application-defined -- implements these
five methods. Nothing else is required.

**Layout widgets** (containers that implement the Flutter layout model):

`ccsp_wid_flex()` -- row or column. Children with flex factors take
proportional share of remaining space. Cross-axis alignment:
start/center/end/stretch. Main-axis alignment: start/center/end/
space-between/space-around/space-evenly.

`ccsp_wid_stack()` -- children layered. Explicit positioning relative
to the stack bounds. First child is bottommost.

`ccsp_wid_constrained()` -- imposes min/max constraints on child.
Forces child to be exactly a given size, or at least/at most a size.

`ccsp_wid_padded()` -- insets on all sides. `ccsp_wid_insets_t`: top,
right, bottom, left. Can be asymmetric.

`ccsp_wid_expanded()` -- takes remaining space in a flex container.
`flex_factor` determines relative share when multiple expanded widgets
compete.

`ccsp_wid_aligned()` -- positions child within available space. x_align
and y_align as floats: 0.0=start, 0.5=center, 1.0=end.

`ccsp_wid_scrollable()` -- clips child and adds scroll offset. Axes:
horizontal, vertical, or both. Scroll position as observable state.
Scroll bar rendering: optional, provided as a separate widget.

`ccsp_wid_sized_box()` -- forces a specific width and/or height.
Child fills the box.

`ccsp_wid_aspect_ratio()` -- constrains child to a specific aspect
ratio within the available space.

`ccsp_wid_wrap()` -- flex container that wraps to new rows/columns
when children overflow.

**Leaf widgets:**

`ccsp_wid_text()` -- UTF-8 text. Uses lib_ccsp_text for layout. Wraps
by default. Overflow ellipsis configurable. Max lines configurable.
`ccsp_wid_text_style_t`: font, size, weight, color, letter spacing,
line height, decoration.

`ccsp_wid_image()` -- image with fit modes: fill, contain, cover,
scale-down, none.

`ccsp_wid_colored_box()` -- rectangle with paint (solid, gradient).
Border radius. Border width and color.

`ccsp_wid_custom_paint()` -- escape hatch. Caller provides size and
paint callback. For custom rendering that cannot be composed from
existing widgets.

`ccsp_wid_placeholder()` -- invisible widget with specified size.
For spacing and layout debugging.

**Gesture detectors** (wrap any widget):

`ccsp_wid_tap()` -- single tap. `on_tap_down`, `on_tap_up`,
`on_tap_cancel`, `on_tap` (fires on clean tap).

`ccsp_wid_long_press()` -- press held for duration (default 500ms).
`on_long_press_start`, `on_long_press_update`, `on_long_press_end`.

`ccsp_wid_drag()` -- pointer drag. `on_drag_start`, `on_drag_update`,
`on_drag_end`. Delta and absolute position.

`ccsp_wid_hover()` -- pointer enter/exit. `on_enter`, `on_exit`,
`on_move`.

`ccsp_wid_double_tap()` -- two taps within threshold (default 300ms).

`ccsp_wid_scale()` -- pinch-to-scale gesture. Two-pointer.

**Composition examples:**

A button:
```c
ccsp_wid_t *button(ccsp_wid_t *content, ccsp_tap_fn on_pressed,
                   void *ud) {
    return ccsp_wid_tap(
        ccsp_wid_padded(
            ccsp_wid_colored_box_with_child(&button_bg, content),
            button_padding
        ),
        on_pressed, ud
    );
}
```

A button that contains a progress bar is possible because composition
is unrestricted. There is no widget hierarchy preventing it.

**Accessibility descriptor** (required per widget type, auto-generated
for standard widgets, explicit for `ccsp_wid_custom_paint()`):
```c
typedef struct {
    ccsp_wid_a11y_role_t  role;      /* button, checkbox, heading, etc */
    ccsp_str_t            label;     /* accessible label, UTF-8 */
    ccsp_str_t            hint;      /* additional description */
    bool                 focusable;
    bool                 enabled;
    /* role-specific state */
} ccsp_wid_a11y_t;
```

**Dirty region tracking:** Widgets call `ccsp_wid_mark_dirty()` when
state changes. The framework tracks the union of dirty regions. Each
frame repaints only dirty regions. `frame_present` sends only the dirty
rect to the backend.

**Testing with lib_ccsp_draw_img:** The same widget tree that renders
to X11 renders identically to an imagemap. Visual regression tests
are pixel-perfect comparisons. No human required to observe regressions.

**Dependencies:** lib_ccsp_base, lib_ccsp_draw, lib_ccsp_text,
lib_ccsp_event.

### lib_ccsp_anim

**What it is:** Animation engine. Produces float values over time.

**Symbol prefix:** `ccsp_anim_`

**Core type:** `ccsp_anim_t`. Created with from, to, duration_ms, curve,
on_value callback, on_done callback. Started explicitly. Driven by
lib_ccsp_event timer.

**Curves:**
- `CCSP_ANIM_LINEAR` -- constant rate
- `CCSP_ANIM_EASE_IN` -- slow start, fast end (cubic)
- `CCSP_ANIM_EASE_OUT` -- fast start, slow end (cubic)
- `CCSP_ANIM_EASE_IN_OUT` -- slow start and end (cubic)
- `CCSP_ANIM_SPRING` -- physically-based spring simulation. Parameters:
  mass, stiffness, damping. May overshoot.
- `CCSP_ANIM_CUSTOM` -- caller provides curve function

**Sequences and staggered animations:**
`ccsp_anim_sequence()` -- run animations in order
`ccsp_anim_parallel()` -- run animations simultaneously
`ccsp_anim_stagger()` -- same animation with delay between each

**Reversing:** `ccsp_anim_reverse()` -- runs from current value back
to start. Preserves curve shape.

**Dependencies:** lib_ccsp_base, lib_ccsp_event.

### lib_ccsp_input

**What it is:** Input event normalization. Produces `ccsp_input_event_t`
from platform-specific input.

**Symbol prefix:** `ccsp_input_`

**`ccsp_input_event_t` type enum:**
`CCSP_INPUT_POINTER_MOVE`, `CCSP_INPUT_POINTER_DOWN`,
`CCSP_INPUT_POINTER_UP`, `CCSP_INPUT_POINTER_SCROLL`,
`CCSP_INPUT_KEY_DOWN`, `CCSP_INPUT_KEY_UP`, `CCSP_INPUT_TEXT_INPUT`,
`CCSP_INPUT_FOCUS_IN`, `CCSP_INPUT_FOCUS_OUT`, `CCSP_INPUT_RESIZE`,
`CCSP_INPUT_CLOSE_REQUEST`.

**Float coordinates:** Pointer events carry (x, y) as floats in widget
coordinates, not integer pixels. Touch screens report sub-pixel
precision. High-DPI displays have logical coordinate spaces. The widget
system receives floats and never converts to integers.

**Text input vs key events:** `CCSP_INPUT_TEXT_INPUT` carries a UTF-8
encoded codepoint (up to 4 bytes). `CCSP_INPUT_KEY_DOWN` carries a
platform-independent keycode. These are different events. An IME
(Input Method Editor) consumes key events and synthesizes text input
events. CJK input requires this distinction. Applications that conflate
the two cannot correctly handle CJK input.

**Platform adapters:**
- `ccsp_input_source_evdev()` -- Linux evdev (`/dev/input/eventN`)
- `ccsp_input_source_x11()` -- X11 input events
- `ccsp_input_source_windows()` -- Win32 WM_* messages
- `ccsp_input_source_macos()` -- NSEvent
- `ccsp_input_source_custom()` -- application-defined source

**IME support:** `ccsp_input_ime_t`. Platform IME integration. Preedit
string (text being composed) delivered separately from committed text.
Cursor position within preedit string for rendering the composition
underline.

**Dependencies:** lib_ccsp_base, lib_ccsp_event.

### lib_ccsp_a11y

**What it is:** Accessibility tree platform adapters.

**Symbol prefix:** `ccsp_a11y_`

**Input:** The accessibility tree generated by lib_ccsp_widget from
widget `ccsp_wid_a11y_t` descriptors.

**Platform adapters:**
- AT-SPI2 (Linux/GNOME): D-Bus accessibility bus. Role mappings, event
  emission on state changes, tree navigation for screen readers
  (Orca, Accerciser).
- NSAccessibility (macOS): Cocoa accessibility protocol. NSAccessibilityElement
  bridging.
- UI Automation (Windows): IUIAutomationProvider implementation. COM
  interface bridging.

**Role mappings:** `CCSP_WID_A11Y_ROLE_BUTTON` -> `ATK_ROLE_PUSH_BUTTON`
(AT-SPI2), `NSAccessibilityButtonRole` (macOS), `UIA_ButtonControlTypeId`
(Windows). Maintained as a mapping table, one file per platform.

**Event emission:** Widget state changes trigger accessibility events.
A checkbox state change emits `ATK_STATE_CHECKED` change on AT-SPI2.
Focus changes emit focus events. Label changes emit name changes.

**Dependencies:** lib_ccsp_base, lib_ccsp_widget.

### lib_ccsp_style

**What it is:** Theming and design tokens.

**Symbol prefix:** `ccsp_style_`

**`ccsp_style_theme_t`:** A struct containing all visual design decisions.

Typography tokens:
```c
ccsp_text_style_t  body;          /* 16sp, regular */
ccsp_text_style_t  body_small;    /* 14sp, regular */
ccsp_text_style_t  heading1;      /* 32sp, bold */
ccsp_text_style_t  heading2;      /* 24sp, semibold */
ccsp_text_style_t  heading3;      /* 20sp, medium */
ccsp_text_style_t  caption;       /* 12sp, regular */
ccsp_text_style_t  monospace;     /* 14sp, monospace font */
ccsp_text_style_t  button_label;  /* 14sp, medium, letter-spaced */
```

Color tokens:
```c
ccsp_draw_color_t  primary;           /* brand color */
ccsp_draw_color_t  primary_variant;   /* darker/lighter brand */
ccsp_draw_color_t  secondary;         /* accent color */
ccsp_draw_color_t  background;        /* page/window background */
ccsp_draw_color_t  surface;           /* card/panel background */
ccsp_draw_color_t  error;             /* error state */
ccsp_draw_color_t  on_primary;        /* text on primary background */
ccsp_draw_color_t  on_secondary;      /* text on secondary background */
ccsp_draw_color_t  on_background;     /* text on background */
ccsp_draw_color_t  on_surface;        /* text on surface */
ccsp_draw_color_t  on_error;          /* text on error background */
ccsp_draw_color_t  divider;           /* separator lines */
ccsp_draw_color_t  overlay;           /* modal scrim */
```

Geometry tokens:
```c
float             border_radius_sm;   /* 4dp */
float             border_radius_md;   /* 8dp */
float             border_radius_lg;   /* 16dp */
float             elevation_shadow_1; /* 2dp shadow blur */
float             elevation_shadow_2; /* 4dp shadow blur */
ccsp_wid_insets_t  content_padding;
ccsp_wid_insets_t  component_padding;
```

Motion tokens:
```c
uint32_t          duration_short;   /* 150ms */
uint32_t          duration_medium;  /* 300ms */
uint32_t          duration_long;    /* 500ms */
ccsp_anim_curve_t  enter_curve;      /* ease out */
ccsp_anim_curve_t  exit_curve;       /* ease in */
ccsp_anim_curve_t  emphasis_curve;   /* ease in-out */
```

**Not CSS:** No specificity. No cascade. No `!important`. A widget uses
theme values or overrides them explicitly for itself and its subtree by
passing a modified theme downward. The override is explicit in code.

**`ccsp_style_theme_dark(base)`:** Takes a light theme and returns a
dark variant with colors inverted appropriately. Not just "invert
everything" -- semantic tokens like `primary` stay the brand color,
only `background` and `surface` flip.

**`ccsp_style_theme_high_contrast(base)`:** High-contrast variant for
accessibility. Increases contrast ratios to WCAG AAA levels.

**Runtime switching:** `ccsp_wid_set_theme(root, new_theme)` triggers
a full repaint. No style sheet recompilation. No layout invalidation
unless typography changed.

**Dependencies:** lib_ccsp_base, lib_ccsp_widget.

### lib_ccsp_app

**What it is:** Application lifecycle and platform integration.

**Symbol prefix:** `ccsp_app_`

**`ccsp_app_t`:** Represents a running application. One per process.

**Window management:**
`ccsp_app_window_create()` -- creates a platform window hosting a widget
`ccsp_app_window_title()` -- set window title (UTF-8)
`ccsp_app_window_size()` -- get/set window size
`ccsp_app_window_fullscreen()` -- enter/exit fullscreen
`ccsp_app_window_minimize()` -- minimize
`ccsp_app_window_close()` -- request close (fires `CCSP_INPUT_CLOSE_REQUEST`)

**System menus:**
`ccsp_app_menu_create()` -- build a menu bar or context menu from a
declarative description. Items: action (callback), submenu, separator,
checkbox, radio group. Platform-native rendering on macOS and Windows.
Custom rendering via lib_ccsp_widget on Linux/X11.

**Platform integration:**
`ccsp_app_set_dock_icon()` -- application icon (macOS dock, Windows
taskbar, Linux/X11 window icon)
`ccsp_app_badge_count()` -- notification badge number
`ccsp_app_notify()` -- system notification (title, body, optional image)
`ccsp_app_open_url()` -- open URL in system browser
`ccsp_app_pick_file()` -- native file picker dialog
`ccsp_app_pick_directory()` -- native directory picker
`ccsp_app_pick_color()` -- native color picker (where available)

**Run loop:** `ccsp_app_run()` enters the event loop. Returns when all
windows are closed or `ccsp_app_quit()` is called. Integrates with
lib_ccsp_event: the event loop is lib_ccsp_event's `ccsp_ev_base_run()`.

**Dependencies:** lib_ccsp_base, lib_ccsp_event, lib_ccsp_widget,
lib_ccsp_input, lib_ccsp_a11y.

---

## Layer 7: Tooling

### lib_ccsp_test

**What it is:** Testing infrastructure shared across the entire stack.

**Symbol prefix:** `ccsp_test_`

**Test runner:**
- TAP (Test Anything Protocol) output
- `ccsp_test_case()` / `ccsp_test_assert()` / `ccsp_test_assert_eq()`
- Setup and teardown per suite and per case
- Timeout per test case (wall clock, via lib_ccsp_event)
- Parallel test execution
- XML output for CI integration (JUnit-compatible)

**Mock allocator:**
- `ccsp_test_alloc_tracking()` -- counts allocations, detects leaks
- `ccsp_test_alloc_failing()` -- fails after N allocations
- `ccsp_test_alloc_limited()` -- limits total allocation bytes
- Pattern: set up mock allocator, run operation, assert no leaks,
  run again with failing allocator, assert clean failure handling

**Capture log backend:**
- `ccsp_test_log_capture()` -- stores all log events in a buffer
- `ccsp_test_log_assert_contains()` -- assert a log event with given
  level and message was emitted
- `ccsp_test_log_assert_not_contains()` -- assert a log event was not
  emitted
- `ccsp_test_log_clear()` -- reset captured log

**Mock DNS resolver:**
- `ccsp_test_dns_mock()` -- responds to queries from a test table
- Test table maps query name+type to pre-built responses
- Supports simulating timeouts, SERVFAIL, NXDOMAIN

**Mock TLS context:**
- `ccsp_test_tls_pair()` -- creates a connected client/server TLS pair
  backed by memory buffers, no network
- Used to test protocol state machines without a TCP connection

**Property-based testing:**
- `ccsp_test_prop_check()` -- runs a test function with random inputs
- Shrinks failing inputs to minimal reproducing case
- Configurable number of iterations (default 1000)
- Fixed seed for reproducible CI runs

**Known-answer test vectors:**
- `ccsp_test_kat_load()` -- load test vectors from JSON file
- `ccsp_test_kat_run()` -- run all vectors against a function
- Standard format: `{input: hex, expected: hex, description: str}`
- NIST CAVP test vector format parser included

**Visual regression (for GUI tests):**
- `ccsp_test_visual_assert()` -- render widget tree to imagemap, compare
  to reference PNG, fail if pixel difference exceeds threshold
- `ccsp_test_visual_update()` -- update reference PNGs (run with
  `CCSP_TEST_UPDATE_REFS=1`)
- Difference image written to test output directory on failure

**Dependencies:** lib_ccsp_base, lib_ccsp_draw_img (for visual tests).

### lib_ccsp_fuzz

**What it is:** Fuzzing harness infrastructure.

**Symbol prefix:** `ccsp_fuzz_`

**Fuzz target catalog** (one target per parser/decoder/formatter):

lib_ccsp_asn1:
- `ccsp_fuzz_asn1_der_parse` -- any DER bytes must not crash
- `ccsp_fuzz_asn1_roundtrip` -- parse then serialize must reproduce input

lib_ccsp_cert:
- `ccsp_fuzz_cert_parse` -- any bytes as cert DER must not crash
- `ccsp_fuzz_cert_validate` -- any cert + any chain must not crash

lib_ccsp_json:
- `ccsp_fuzz_json_dom` -- any bytes as JSON must not crash
- `ccsp_fuzz_json_streaming` -- feed in arbitrary chunks must not crash

lib_ccsp_toml:
- `ccsp_fuzz_toml_parse` -- any bytes as TOML must not crash

lib_ccsp_dns:
- `ccsp_fuzz_dns_response` -- any bytes as DNS response must not crash

lib_ccsp_lineproto:
- `ccsp_fuzz_lineproto_input` -- any bytes as line protocol input must
  not crash

lib_ccsp_http:
- `ccsp_fuzz_http_request` -- any bytes as HTTP request must not crash
- `ccsp_fuzz_http_response` -- any bytes as HTTP response must not crash

lib_ccsp_tls:
- `ccsp_fuzz_tls_record` -- any bytes as TLS record must not crash
- `ccsp_fuzz_tls_handshake_server` -- any bytes as client hello must
  not crash
- `ccsp_fuzz_tls_handshake_client` -- any bytes as server hello must
  not crash

lib_ccsp_regex:
- `ccsp_fuzz_regex_pattern` -- any bytes as regex pattern must not crash
  or loop infinitely (timeout enforced)
- `ccsp_fuzz_regex_match` -- any compiled pattern against any input must
  not crash

lib_ccsp_ocl:
- `ccsp_fuzz_aead_open` -- any bytes as AEAD ciphertext+tag must not
  crash (must return authentication failure, not crash)

**Build integration:**
- LibFuzzer target: `cmake -DENABLE_FUZZING=ON -DFUZZ_ENGINE=libfuzzer`
- AFL target: `cmake -DENABLE_FUZZING=ON -DFUZZ_ENGINE=afl`
- OSS-Fuzz integration files provided for continuous fuzzing

**Corpus management:**
- `ccsp_fuzz_corpus_minimize()` -- reduce corpus to minimal set
- `ccsp_fuzz_corpus_merge()` -- merge corpora from multiple runs
- Seed corpus: manually crafted interesting inputs per target
  (valid examples, boundary cases, known-bad inputs)

**CI integration:** Run each fuzz target for 100,000 iterations on
every PR. Report: iterations run, crashes found, coverage delta. A
crash in any fuzz target is a blocking failure.

**Dependencies:** lib_ccsp_base, lib_ccsp_test.

### ccsp-check (lib_ccsp_check)

**What it is:** Static analysis tool. Scans C source for patterns from
the CVE corpus. Not a compiler plugin. A source code scanner.

**Binary:** `ccsp-check`

**The finding list:**

CRITICAL findings (stop the build in CI):
- Any call to `gets()` anywhere
- Any call to `gethostbyname()` -- use `ccsp_dns_resolver_query()`
- Any `char buf[N]; read(fd, buf, ...)` without length validation
- Any `malloc(a + b)` or `malloc(a * b)` without overflow check
- Any `sprintf()` (no `n`) -- use `ccsp_bio_fmt_write()` or `snprintf()`
- Any `strcpy()` or `strcat()` -- use `ccsp_str_*` or `strlcpy()`/`strlcat()`
- Any hand-rolled line reader pattern (fixed buf + read in loop + strchr)
- Any `rand()` or `random()` in a context that includes crypto headers

HIGH findings (block merge, allow override with justification):
- Any `select()` without `FD_SETSIZE` check -- use `ccsp_ev_*`
- Any hand-rolled connect loop without explicit `SO_ERROR` check
- Any `strcmp()` used to compare values that could be secrets
- Any `memcmp()` used to compare values that could be secrets
- Any `strtol()` without range check on result
- Any `atoi()` / `atol()` / `atof()` -- no error detection
- Any `signal()` (unreliable) -- use `ccsp_ev_signal_t` or `sigaction()`

INFO findings (reported, not blocking):
- Any `strlen()` in a hot path (O(n), document if intentional)
- Any `printf()` / `fprintf()` in library code (use structured logging)
- Any `exit()` / `abort()` outside of `main()` (document the panic)
- Any function with cyclomatic complexity > 20

**Output format:**
```
CRITICAL src/resolver.c:234: gethostbyname_nonreentrant
  cvss_pattern: gethostbyname_nonreentrant
  bug_count: 28 (see CCSP-BUGS.TXT, pattern: gethostbyname)
  mean_latency: 4.2 years to disclosure
  fix: ccsp_dns_resolver_query() -- async, reentrant, full record types
  suppress: ccsp-check:suppress gethostbyname_nonreentrant reason="legacy compat"
```

**Suppression:** `// ccsp-check:suppress PATTERN reason="explanation"`.
Suppressions are logged to a suppression report. The suppression report
is reviewed in security audits. A suppression without a reason string
is a parse error.

**CI integration:** `ccsp-check --critical-exit-code=1 src/` returns 1
if any CRITICAL findings exist. CI blocks merge on exit code 1.

### Bug Pattern Database

**Location:** `ftp://ftp.ccsys.org/pub/ccsp/bugs/`
**Query tool:** `ccsp-bugs` (structured grep over CCSP-BUGS.TXT)

**Format:** One record per bug, pipe-delimited flat file (CCSP-BUGS.TXT):
```
CERT-CA-1988-01|sendmail_debug|sendmail < 5.59|1983|1988|5.0|...
CERT-CA-1989-01|fingerd_gets|fingerd (Morris worm)|1988|1988|0.1|...
```

Fields: advisory_id, pattern, affected_software, introduced_year,
disclosed_year, years_latent, description, vulnerable_code, fixed_code.

**Query examples:**
```sh
ccsp-bugs -p hand_rolled_line_reader   # list by pattern
ccsp-bugs -s                           # summary statistics
ccsp-bugs -l sendmail                  # all sendmail bugs
awk -F'|' '{print $2}' CCSP-BUGS.TXT | sort | uniq -c | sort -rn  # DIY
```

**Contributing:** Mail new entries to `ccsp-bugs@ccsys.org` with subject
"BUG SUBMISSION: [advisory-id]". Include: advisory ID, pattern
classification, affected software, and both vulnerable and fixed code.
Disputes discussed on `comp.lang.c.ccsp`.

---

# Part V: Cross-Cutting Concerns

## Error Handling Across the Stack

All libraries use `ccsp_err_t` from lib_ccsp_base. Domain strings by
library are registered in `ccsp_err_domains.h`.

```tbl
.TS
center box;
l l l
l l l.
Library	Domain String	Error Code Prefix
_
lib_ccsp_base	"ccsp.base"	CCSP_ERR_
lib_ccsp_event	"ccsp.event"	CCSP_EV_ERR_
lib_ccsp_bio	"ccsp.bio"	CCSP_BIO_ERR_
lib_ccsp_dns	"ccsp.dns"	CCSP_DNS_ERR_
lib_ccsp_connect	"ccsp.connect"	CCSP_CONN_ERR_
lib_ccsp_lineproto	"ccsp.lineproto"	CCSP_LP_ERR_
lib_ccsp_http	"ccsp.http"	CCSP_HTTP_ERR_
lib_ccsp_asn1	"ccsp.asn1"	CCSP_ASN1_ERR_
lib_ccsp_cert	"ccsp.cert"	CCSP_CERT_ERR_
lib_ccsp_ocl	"ccsp.ocl"	CCSP_OCL_ERR_
lib_ccsp_tls	"ccsp.tls"	CCSP_TLS_ERR_
lib_ccsp_json	"ccsp.json"	CCSP_JSON_ERR_
lib_ccsp_toml	"ccsp.toml"	CCSP_TOML_ERR_
lib_ccsp_cli	"ccsp.cli"	CCSP_CLI_ERR_
lib_ccsp_draw	"ccsp.draw"	CCSP_DRAW_ERR_
lib_ccsp_widget	"ccsp.widget"	CCSP_WID_ERR_
.TE
```

## Memory Model Across the Stack

One rule: **no library calls `malloc`, `realloc`, or `free` directly
except in `lib_ccsp_base/src/platform/ccsp_platform_default.c`.**

Enforced by:
1. `ccsp-check` CRITICAL rule: `direct_malloc_call`
2. Build system symbol check: `nm` output scanned for `U malloc`
   in any object file other than the allowed one
3. Code review

`ccsp_free_secure()` for all memory containing key material, passwords,
nonces, or other secret data. The zeroing is verified at each
optimization level in CI by inspecting generated assembly.

## Threading Model Across the Stack

Event-driven single-threaded per event loop. One thread, one
`ccsp_ev_base_t`. All callbacks on that thread. No locking within a
single event loop.

Objects are not shared between event loops. Read-only shared objects
(compiled regex patterns, parsed certificates, TLS configs) are safe
to share as they are immutable after construction.

Thread-safety is documented per API symbol. Default assumption:
not thread-safe. `@threadsafe` in doc comment means verified.

## Logging Across the Stack

All libraries use `CCSP_LOG()` from lib_ccsp_base. Standard log key
names:

```tbl
.TS
center box;
l l l
l l l.
Key	Type	Used By
_
conn_id	string	connect, tls, http, smtp, imap
peer_addr	string	connect, tls, dns
hostname	string	dns, connect, tls
port	int	connect, http
tls_version	string	tls
cipher_suite	string	tls
alpn_proto	string	tls, http
cert_subject	string	cert, tls
cert_issuer	string	cert, tls
duration_ms	int	dns, connect, http
bytes_read	int	bio, tls, http
bytes_written	int	bio, tls, http
status_code	int	http
method	string	http
path	string	http
error_code	int	all
error_domain	string	all
dane_result	string	connect, tls
dnssec_validated	bool	dns
srv_priority	int	connect
srv_weight	int	connect
.TE
```

Consistent key names mean log aggregation queries work across all
libraries without custom parsers.

## Testing Across the Stack

Three test categories per library:

**Known-answer tests:** Fixed inputs, expected outputs. NIST CAVP for
crypto. Unicode Consortium test files for UTF-8. RFC test vectors for
protocols. `toml-test` for TOML. `JSONTestSuite` for JSON. Run on every
build. Deterministic. Never different results on the same platform.

**Property-based tests:** Invariants for any input. Run with fixed seed
in CI for reproducibility. Interesting shrunk counterexamples stored in
corpus for regression testing.

**Fuzz targets:** Every parser, every decoder. Run for 100,000 iterations
per target per PR. Crash = blocking failure. Corpus maintained and grown.
OSS-Fuzz integration for continuous background fuzzing.

CI matrix:

```tbl
.TS
center box;
l l l
l l l.
Platform	Compiler	Sanitizers
_
Linux x86_64	GCC 12	ASan, UBSan, TSan
Linux x86_64	Clang 16	ASan, UBSan, TSan, MemSan
Linux aarch64	GCC 12	ASan, UBSan
Linux aarch64	Clang 16	ASan, UBSan
macOS 13	Apple Clang	ASan, UBSan
FreeBSD 13	Clang 16	ASan
NetBSD 9	GCC 12	(no sanitizers)
.TE
```

## The CVE Corpus

The corpus is the evidence base. Updated with every new C systems CVE
that matches a pattern. Public. Auditable. libcheck queries it.
Conference talks cite it. Procurement checklists reference it.

When libcheck flags a pattern, it shows:
- CVE count for the pattern
- Mean years to disclosure
- The specific API call that prevents it
- Link to corpus entry

The decision to ignore a finding is documented with a suppression
comment. The suppression is in the audit trail. The CVE count is in
the suppression record. The risk acceptance is explicit.

---

# Part VI: API Guide Table of Contents

## Volume 0: Introduction and Architecture

### Chapter 0.1: How to Read This Guide
- 0.1.1 Guide organization: volumes, chapters, sections
- 0.1.2 Notation conventions
- 0.1.3 Code example conventions (all examples are tested)
- 0.1.4 Error handling conventions in examples
- 0.1.5 How to find the library you need

### Chapter 0.2: The Dependency Architecture
- 0.2.1 The library registry: all 37 libraries
- 0.2.2 The dependency graph: reading the diagram
- 0.2.3 The dependency rule and its enforcement
- 0.2.4 Taking only what you need: minimal builds
- 0.2.5 The `lib_ccsp_` namespace: why and how

### Chapter 0.3: The `ccsp_` Symbol Namespace
- 0.3.1 Symbol prefix conventions by library
- 0.3.2 Header naming conventions
- 0.3.3 Error code prefix conventions
- 0.3.4 Log domain string conventions
- 0.3.5 Avoiding collisions with application code

### Chapter 0.4: The Error Model
- 0.4.1 `ccsp_err_t`: structure and semantics
- 0.4.2 `CCSP_ERR()`: the error-setting macro
- 0.4.3 `ccsp_err_wrap()`: adding call-site context
- 0.4.4 `warn_unused_result`: compile-time enforcement
- 0.4.5 Passing NULL for err
- 0.4.6 Error propagation patterns
- 0.4.7 The complete error code reference

### Chapter 0.5: The Memory Model
- 0.5.1 The platform table and `ccsp_malloc()`
- 0.5.2 The caller-provides-storage principle
- 0.5.3 `ccsp_free_secure()`: zeroing secrets
- 0.5.4 Object creation and ownership
- 0.5.5 Testing with mock allocators
- 0.5.6 The malloc rule and how it is enforced

### Chapter 0.6: The Threading Model
- 0.6.1 Event-driven single-threaded model
- 0.6.2 Thread-safety documentation conventions
- 0.6.3 Sharing objects between threads: what is safe
- 0.6.4 Multiple event loops: one per thread

### Chapter 0.7: The UTF-8 Commitment
- 0.7.1 Why UTF-8 and nothing else
- 0.7.2 `ccsp_str_t`: the string type
- 0.7.3 Strict validation: what is rejected and why
- 0.7.4 Handling legacy encodings with lib_ccsp_charconv
- 0.7.5 The boundary principle: convert at the edge, not inside

### Chapter 0.8: Build System Integration
- 0.8.1 CMake: `find_package(CCSP)`
- 0.8.2 pkg-config: `pkg-config --libs ccsp-base`
- 0.8.3 Selecting libraries: `CCSP_COMPONENTS`
- 0.8.4 Platform detection output: reading `CMakeCache.txt`
- 0.8.5 Cross-compilation: toolchain file requirements
- 0.8.6 Embedded targets: static build, custom platform table
- 0.8.7 Shim detection: which shims are active in your build

### Chapter 0.9: ccsp-check Integration
- 0.9.1 Installing ccsp-check
- 0.9.2 Running ccsp-check on your codebase
- 0.9.3 CI integration: exit codes and reporting
- 0.9.4 Interpreting findings: CRITICAL vs HIGH vs INFO
- 0.9.5 Writing suppression comments with justification
- 0.9.6 The suppression audit report
- 0.9.7 Adding custom rules

### Chapter 0.10: The CVE Corpus
- 0.10.1 The corpus URL and schema
- 0.10.2 Querying with ccsp-corpus
- 0.10.3 The 12 patterns: definitions and CVE counts
- 0.10.4 How new entries are added
- 0.10.5 Using the corpus in security reviews

---

## Volume 1: lib_ccsp_base

### Chapter 1.1: Getting Started
- 1.1.1 The first program using lib_ccsp_base
- 1.1.2 Initializing with `ccsp_platform_default()`
- 1.1.3 Creating a context with `ccsp_ctx_create()`
- 1.1.4 The "Hello, UTF-8" example
- 1.1.5 Running the conformance suite

### Chapter 1.2: Platform Detection (ccsp_platform.h)
- 1.2.1 OS detection macros: `CCSP_OS_*`
- 1.2.2 Architecture detection macros: `CCSP_ARCH_*`
- 1.2.3 `CCSP_STRICT_ALIGN`: platforms that bus-error on misalignment
- 1.2.4 `CCSP_BIG_ENDIAN` / `CCSP_LITTLE_ENDIAN`
- 1.2.5 Feature flags: `CCSP_HAS_CLOCK_MONOTONIC`, `CCSP_HAS_GETRANDOM`, etc.
- 1.2.6 The one-file rule: why ifdefs are banned elsewhere
- 1.2.7 Adding a new OS to the detection matrix
- 1.2.8 Adding a new architecture to the detection matrix

### Chapter 1.3: Integer Types (ccsp_types.h)
- 1.3.1 The type table: `ccsp_uint8_t` through `ccsp_uint64_t`
- 1.3.2 `ccsp_uintptr_t` and `ccsp_ptrdiff_t`
- 1.3.3 `ccsp_size_t` and `ccsp_ssize_t`
- 1.3.4 `ccsp_bool_t`, `CCSP_TRUE`, `CCSP_FALSE`
- 1.3.5 The C99 shim: when it is active and when it will be deleted
- 1.3.6 Platform conformance test: what is checked
- 1.3.7 The rule: never use bare `int` for a sized value

### Chapter 1.4: The Platform Table
- 1.4.1 `ccsp_platform_t`: the complete struct
- 1.4.2 `ccsp_platform_default()`: the standard configuration
- 1.4.3 Replacing `malloc`/`realloc`/`free` with a pool allocator
- 1.4.4 Replacing the monotonic clock for testing
- 1.4.5 Replacing `getrandom` with a hardware RNG
- 1.4.6 Replacing the log backend with a custom writer
- 1.4.7 NULL threading functions: single-threaded builds
- 1.4.8 Embedded targets: static memory, no heap

### Chapter 1.5: The Base Context
- 1.5.1 `ccsp_ctx_t`: what it holds
- 1.5.2 `ccsp_ctx_create()`: initialization
- 1.5.3 `ccsp_ctx_destroy()`: cleanup and verification
- 1.5.4 Context vs global state: a worked example
- 1.5.5 Passing context through call stacks

### Chapter 1.6: Error Handling
- 1.6.1 `ccsp_err_t`: structure
- 1.6.2 `CCSP_ERR()`: the macro, line capture, NULL safety
- 1.6.3 `ccsp_err_wrap()`: propagation with context
- 1.6.4 `ccsp_err_clear()`: resetting
- 1.6.5 `ccsp_err_format()`: human-readable output
- 1.6.6 `CCSP_ERR_WARN_UNUSED`: compiler enforcement
- 1.6.7 Pattern: propagating errors without losing context
- 1.6.8 Pattern: converting platform errno to `ccsp_err_t`
- 1.6.9 Pattern: testing error paths with mock allocator
- 1.6.10 The complete `CCSP_ERR_*` code list

### Chapter 1.7: Memory and Allocation
- 1.7.1 `ccsp_malloc()`, `ccsp_realloc()`, `ccsp_free()`: platform wrappers
- 1.7.2 `ccsp_calloc()`: zeroing allocation
- 1.7.3 `ccsp_free_secure()`: zeroing before free
- 1.7.4 `ccsp_strdup()`: string duplication
- 1.7.5 `ccsp_size_add()`, `ccsp_size_mul()`: overflow-checked arithmetic
- 1.7.6 The one-malloc rule: enforcement and verification
- 1.7.7 Assembly inspection: verifying `ccsp_free_secure()` is not elided
- 1.7.8 Valgrind annotations
- 1.7.9 AddressSanitizer annotations

### Chapter 1.8: UTF-8 and Strings
- 1.8.1 `ccsp_str_t`: data and len
- 1.8.2 `CCSP_STR_LIT()`: literal without strlen
- 1.8.3 `ccsp_str_from_cstr()`: null-terminated to `ccsp_str_t`
- 1.8.4 `ccsp_str_cstr()`: `ccsp_str_t` to null-terminated (allocates)
- 1.8.5 `ccsp_str_eq()`, `ccsp_str_eq_ci()`: equality comparison
- 1.8.6 `ccsp_str_dup()`, `ccsp_str_free()`: owned copy
- 1.8.7 `ccsp_str_contains()`, `ccsp_str_starts_with()`, `ccsp_str_ends_with()`
- 1.8.8 `ccsp_str_trim()`, `ccsp_str_trim_left()`, `ccsp_str_trim_right()`
- 1.8.9 `ccsp_str_split()`: split on delimiter
- 1.8.10 `ccsp_utf8_validate()`: strict validation
- 1.8.11 `ccsp_utf8_decode()`: decode one codepoint
- 1.8.12 `ccsp_utf8_encode()`: encode one codepoint
- 1.8.13 `ccsp_utf8_next()`, `ccsp_utf8_prev()`: iteration without decoding
- 1.8.14 `ccsp_utf8_charlen()`: byte length of next codepoint
- 1.8.15 `ccsp_utf8_codepoint_count()`: O(n) count, use carefully
- 1.8.16 Pattern: iterating UTF-8 codepoints
- 1.8.17 Pattern: splitting on ASCII delimiters in UTF-8 text
- 1.8.18 The strlen/strcat antipatterns and why they are banned

### Chapter 1.9: Byte Order Utilities (ccsp_endian.h)
- 1.9.1 Why byte-by-byte access instead of pointer casts
- 1.9.2 `ccsp_read_be16/32/64()`: big-endian reads
- 1.9.3 `ccsp_write_be16/32/64()`: big-endian writes
- 1.9.4 `ccsp_read_le16/32/64()`, `ccsp_write_le16/32/64()`: little-endian
- 1.9.5 `ccsp_bswap16/32/64()`: byte swapping
- 1.9.6 `CCSP_HTON32()`, `CCSP_NTOH32()`: compile-time constants
- 1.9.7 Platform optimization: `__builtin_bswap32` on GCC/Clang
- 1.9.8 Test vectors: known byte sequences and expected values

### Chapter 1.10: Intrusive Linked Lists
- 1.10.1 Why intrusive lists: allocation and cache behavior
- 1.10.2 `ccsp_list_node_t`: the embedded node
- 1.10.3 `ccsp_list_t`: the list head with sentinel
- 1.10.4 `CCSP_CONTAINER_OF()`: node to containing struct
- 1.10.5 `ccsp_list_init()`: initialization
- 1.10.6 `ccsp_list_insert_head()`, `ccsp_list_insert_tail()`: O(1)
- 1.10.7 `ccsp_list_insert_before()`, `ccsp_list_insert_after()`
- 1.10.8 `ccsp_list_remove()`: O(1) removal
- 1.10.9 `ccsp_list_empty()`: emptiness check
- 1.10.10 `CCSP_LIST_FOREACH()`: iteration
- 1.10.11 `CCSP_LIST_FOREACH_SAFE()`: iteration safe for removal during
- 1.10.12 `ccsp_slist_t`: singly-linked for stack operations
- 1.10.13 Pattern: embedding lists in application structs
- 1.10.14 The sentinel invariant: why it matters

### Chapter 1.11: Hash Tables
- 1.11.1 Design: open addressing, insertion order, typed keys
- 1.11.2 `ccsp_hash_create()`: key type, capacity, options
- 1.11.3 `ccsp_hash_destroy()`: cleanup
- 1.11.4 `ccsp_hash_set()`: insert or update
- 1.11.5 `ccsp_hash_get()`: lookup
- 1.11.6 `ccsp_hash_delete()`: remove
- 1.11.7 `ccsp_hash_count()`: O(1) count
- 1.11.8 `ccsp_hash_clear()`: remove all, keep allocation
- 1.11.9 `CCSP_HASH_KEY_STRING`: interned string keys
- 1.11.10 `CCSP_HASH_KEY_BYTES`: interned bytes keys
- 1.11.11 `CCSP_HASH_KEY_UINT64`: integer keys
- 1.11.12 `CCSP_HASH_KEY_PTR`: pointer identity keys
- 1.11.13 `ccsp_hash_each()`: iteration in insertion order, always
- 1.11.14 `ccsp_hash_iter_t`: explicit iterator
- 1.11.15 Load factor, resize policy, tombstone management
- 1.11.16 Performance: cache behavior vs chaining
- 1.11.17 Thread safety: the wrapper pattern

### Chapter 1.12: Buffers and Ring Buffers
- 1.12.1 `ccsp_buf_t`: the growable buffer
- 1.12.2 `ccsp_buf_init()`, `ccsp_buf_destroy()`
- 1.12.3 `ccsp_buf_append()`, `ccsp_buf_append_str()`
- 1.12.4 `ccsp_buf_reserve()`, `ccsp_buf_commit()`: pre-allocated writes
- 1.12.5 `ccsp_buf_truncate()`, `ccsp_buf_clear()`
- 1.12.6 `ccsp_buf_steal()`: ownership transfer
- 1.12.7 Growth policy: doubling, `ccsp_buf_shrink_to_fit()`
- 1.12.8 `ccsp_ring_t`: fixed-capacity ring buffer
- 1.12.9 `ccsp_ring_init()`: caller-provided storage
- 1.12.10 `ccsp_ring_readable()`, `ccsp_ring_writable()`
- 1.12.11 `ccsp_ring_peek()`, `ccsp_ring_consume()`: zero-copy read
- 1.12.12 `ccsp_ring_reserve()`, `ccsp_ring_commit()`: zero-copy write
- 1.12.13 `ccsp_ring_peek_all()`: handling the wraparound case
- 1.12.14 Pattern: zero-copy TLS record writing

### Chapter 1.13: Slab Allocator
- 1.13.1 When to use slab allocation
- 1.13.2 `ccsp_slab_create()`, `ccsp_slab_destroy()`
- 1.13.3 `ccsp_slab_alloc()`, `ccsp_slab_free()`
- 1.13.4 Internal structure: page-aligned backing, embedded free list
- 1.13.5 Debug mode: canary detection for double-free
- 1.13.6 Thread safety: one slab per thread

### Chapter 1.14: Monotonic Time and Sleep
- 1.14.1 Why `time()` and `gettimeofday()` are wrong for intervals
- 1.14.2 `ccsp_monotonic_ns()`: nanoseconds since unspecified epoch
- 1.14.3 `ccsp_elapsed_ns()`, `ccsp_elapsed_ms()`: interval computation
- 1.14.4 Why unsigned wraparound is defined and correct
- 1.14.5 `ccsp_wallclock_us()`: wall clock for timestamps
- 1.14.6 `ccsp_sleep_ns()`: sleep with EINTR retry
- 1.14.7 Platform implementations: CLOCK_MONOTONIC, QPC, mach_absolute_time
- 1.14.8 The macOS timebase conversion: precision and overflow
- 1.14.9 The shim deletion condition

### Chapter 1.15: Secure Random
- 1.15.1 Why `rand()` is not acceptable
- 1.15.2 `ccsp_random_bytes()`: fill buffer with secure random
- 1.15.3 `ccsp_random_uint32()`, `ccsp_random_uint64()`
- 1.15.4 `ccsp_random_range()`: uniform in [0, n) without modulo bias
- 1.15.5 Rejection sampling: the algorithm and termination proof
- 1.15.6 Statistical test: chi-square uniformity
- 1.15.7 Platform shim: getrandom, getentropy, arc4random_buf, /dev/urandom
- 1.15.8 The blocking-before-seeded problem
- 1.15.9 Error handling: platforms with no entropy source

### Chapter 1.16: Atomic Operations
- 1.16.1 When to use atomics
- 1.16.2 `ccsp_atomic_t` and its operations
- 1.16.3 Sequential consistency only: the design decision
- 1.16.4 Reference counting pattern
- 1.16.5 `ccsp_once_t` and `ccsp_once_run()`
- 1.16.6 Platform shim: C11, GCC builtins, MSVC, mutex fallback
- 1.16.7 Testing with ThreadSanitizer

### Chapter 1.17: Structured Logging
- 1.17.1 Why structured logging: logs are data
- 1.17.2 Log levels: DEBUG, INFO, WARN, ERROR, FATAL
- 1.17.3 `CCSP_LOG()`: the primary macro
- 1.17.4 Key-value pairs: the structured part
- 1.17.5 Compile-time level filtering: `CCSP_LOG_MIN_LEVEL`
- 1.17.6 `ccsp_log_set_backend()`: installing a backend
- 1.17.7 The default stderr backend: format and timestamp
- 1.17.8 The null backend: discarding output
- 1.17.9 The capture backend for testing
- 1.17.10 Writing a JSON-lines backend
- 1.17.11 Writing a syslog backend
- 1.17.12 Standard log key names (cross-stack reference)
- 1.17.13 Macro argument evaluation: no side effects

### Chapter 1.18: Overflow-Checked Arithmetic
- 1.18.1 `ccsp_size_add()`: addition with overflow check
- 1.18.2 `ccsp_size_mul()`: multiplication with overflow check
- 1.18.3 `ccsp_size_align()`: round up to alignment
- 1.18.4 Pattern: safe array allocation
- 1.18.5 Pattern: safe string concatenation sizing

### Chapter 1.19: Platform Port Guide
- 1.19.1 The porting checklist
- 1.19.2 Required: OS detection in `ccsp_platform.h`
- 1.19.3 Required: architecture detection
- 1.19.4 Required: integer type sizes (validation test)
- 1.19.5 Required: byte order
- 1.19.6 Required: strict alignment flag
- 1.19.7 Required: monotonic time source
- 1.19.8 Required: secure entropy source
- 1.19.9 Optional: native atomic operations
- 1.19.10 Optional: native once-initialization
- 1.19.11 Running the conformance suite on a new platform
- 1.19.12 The platform report format
- 1.19.13 Submitting a port

### Chapter 1.20: Shim Reference
- 1.20.1 Active shims: list, deletion conditions, tracker links
- 1.20.2 The shim review schedule
- 1.20.3 How to delete a shim: the checklist
- 1.20.4 The `.tracker` file format

---

## Volume 2: Core Services

### Chapter 2.1: lib_ccsp_event -- I/O Event Dispatch
- 2.1.1 Design philosophy: the event base is always explicit
- 2.1.2 `ccsp_ev_base_create()`, `ccsp_ev_base_destroy()`
- 2.1.3 `ccsp_ev_base_run()`, `ccsp_ev_base_stop()`
- 2.1.4 `ccsp_ev_base_run_once()`: single iteration
- 2.1.5 I/O events: `ccsp_ev_io_t`
  - `ccsp_ev_io_init()`: fd, events (READ|WRITE|PERSIST), callback
  - `ccsp_ev_io_add()`: activate with optional timeout
  - `ccsp_ev_io_remove()`: deactivate
  - `ccsp_ev_io_modify()`: change watched events
  - Callback signature: `(fd, events, userdata)`
- 2.1.6 Timer events: `ccsp_ev_timer_t`
  - `ccsp_ev_timer_init()`: callback, userdata
  - `ccsp_ev_timer_add()`: timeout in milliseconds
  - `ccsp_ev_timer_remove()`: cancel
  - `ccsp_ev_timer_reset()`: restart from now
  - The min-heap implementation: O(log n) insert/remove
- 2.1.7 Signal events: `ccsp_ev_signal_t`
  - `ccsp_ev_signal_init()`: signal number, callback
  - `ccsp_ev_signal_add()`, `ccsp_ev_signal_remove()`
  - The self-pipe mechanism: the one global state
  - Multi-base signal delivery: documented limitation
- 2.1.8 Backends
  - `ccsp_ev_backend_select`: universal, FD_SETSIZE checked
  - `ccsp_ev_backend_poll`: POSIX, no fd limit
  - `ccsp_ev_backend_kqueue`: the seven behaviors handled
  - `ccsp_ev_backend_epoll`: edge-triggered mode
  - `ccsp_ev_backend_devpoll`: Solaris
  - Backend selection: `ccsp_ev_backend_detect()`
  - Writing a new backend: the vtable
- 2.1.9 Advanced patterns
  - Chaining event loops: inter-thread wakeup
  - Priority events: urgent callbacks
  - Deferred callbacks: `ccsp_ev_defer()`
  - Idle callbacks: `ccsp_ev_idle_t`

### Chapter 2.2: lib_ccsp_mpi -- Arbitrary Precision Arithmetic
- 2.2.1 Variable-time vs constant-time: the two namespaces
- 2.2.2 `ccsp_mpi_t`: the integer type
- 2.2.3 Variable-time operations: `ccsp_mpi_*`
  - `ccsp_mpi_create()`, `ccsp_mpi_free()`
  - `ccsp_mpi_set_u64()`, `ccsp_mpi_get_u64()`
  - `ccsp_mpi_from_bytes_be()`, `ccsp_mpi_to_bytes_be()`
  - `ccsp_mpi_add()`, `ccsp_mpi_sub()`, `ccsp_mpi_mul()`
  - `ccsp_mpi_div()`, `ccsp_mpi_mod()`
  - `ccsp_mpi_powmod()`: modular exponentiation
  - `ccsp_mpi_gcd()`, `ccsp_mpi_modinv()`
  - `ccsp_mpi_is_prime()`: Miller-Rabin probabilistic test
  - `ccsp_mpi_cmp()`, `ccsp_mpi_cmp_u64()`
  - `ccsp_mpi_bits()`: bit length
- 2.2.4 Constant-time operations: `ccsp_mpi_ct_*`
  - Identical API, guaranteed constant time
  - The timing variance test methodology
  - Which operations MUST use constant-time
  - Performance implications: documented and measured
- 2.2.5 Validation against GMP: test vector format
- 2.2.6 Performance notes: GMP comparison

### Chapter 2.3: lib_ccsp_regex -- Regular Expressions
- 2.3.1 Design: NFA default, backtracking opt-in
- 2.3.2 The NFA engine: why O(n) matters for security
- 2.3.3 The backtracking engine: the compiler warning and what it means
- 2.3.4 `ccsp_rx_compile()`: pattern string to compiled pattern
  - Flag: `CCSP_RX_FLAG_UTF8`
  - Flag: `CCSP_RX_FLAG_CASE_INSENSITIVE`
  - Flag: `CCSP_RX_FLAG_MULTILINE`
  - Flag: `CCSP_RX_FLAG_DOT_ALL`
  - Engine: `CCSP_RX_ENGINE_NFA` (default), `CCSP_RX_ENGINE_BACKTRACK`
  - Error output: position and message
- 2.3.5 `ccsp_rx_free()`: free compiled pattern
- 2.3.6 `ccsp_rx_match()`: match against input
  - `ccsp_rx_match_t`: start, len, captures
  - `capture.name`, `capture.start`, `capture.len`
  - Thread safety: patterns are immutable
- 2.3.7 `ccsp_rx_find_all()`: callback for all matches
- 2.3.8 `ccsp_rx_replace()`: substitution with capture references
  - Replacement syntax: `$0` whole, `$1` first, `${name}` named
- 2.3.9 Named captures: syntax `(?P<name>...)` and `(?<name>...)`
- 2.3.10 UTF-8 mode: character classes, `.`, `\w`, `\d`, `\s`
- 2.3.11 Unicode property file: `ccsp_rx_unicode_props.dat`
- 2.3.12 Fuzz results: iterations run, interesting inputs found

### Chapter 2.4: lib_ccsp_charconv -- Encoding Conversion
- 2.4.1 The boundary principle: convert at the edge
- 2.4.2 `ccsp_conv_to_utf8()`: convert any encoding to UTF-8
  - `lconv_encoding_t`: the full enum
  - Output to `ccsp_bio_writer_t`
  - Error on unconvertible character
- 2.4.3 `ccsp_conv_from_utf8()`: convert UTF-8 to any encoding
  - Replacement character policy: configurable
  - Unmappable character handling
- 2.4.4 `ccsp_conv_detect()`: statistical detection
  - BOM detection: UTF-8/16/32
  - Statistical method: n-gram frequency analysis
  - `confidence` output: interpretation
  - When to trust detection vs ask the user
- 2.4.5 BOM handling: strip on input, never emit on output
- 2.4.6 Streaming conversion: `ccsp_conv_ctx_t` for chunked input
- 2.4.7 Supported encoding list: completeness and gaps

---

## Volume 3: I/O and Networking

### Chapter 3.1: lib_ccsp_bio -- Buffered I/O
- 3.1.1 Design: separation of reader, writer, filter, formatter
- 3.1.2 `ccsp_bio_reader_t`: the abstract reader interface
  - `read(ctx, buf, len, got)`: the operation
  - `close(ctx)`: cleanup
  - `available(ctx)`: non-blocking available bytes query
- 3.1.3 `ccsp_bio_writer_t`: the abstract writer interface
  - `write(ctx, buf, len, sent)`: the operation
  - `flush(ctx)`: drain write buffer
  - `close(ctx)`: cleanup
- 3.1.4 Concrete readers
  - `ccsp_bio_reader_from_fd()`: blocking file descriptor
  - `ccsp_bio_reader_from_fd_nb()`: nonblocking fd with libevent
  - `ccsp_bio_reader_from_mem()`: read-only memory buffer
  - `ccsp_bio_reader_from_file()`: stdio FILE*
  - `ccsp_bio_reader_from_buf()`: `ccsp_buf_t` as reader
- 3.1.5 Concrete writers
  - `ccsp_bio_writer_to_fd()`, `ccsp_bio_writer_to_fd_nb()`
  - `ccsp_bio_writer_to_mem()`: write to `ccsp_buf_t`
  - `ccsp_bio_writer_to_file()`
  - `ccsp_bio_writer_to_null()`: discard (useful for testing)
- 3.1.6 Filters: reader wrappers
  - `ccsp_bio_filter_base64_reader()`: decode on read
  - `ccsp_bio_filter_base64_writer()`: encode on write
  - `ccsp_bio_filter_hex_reader()`, `ccsp_bio_filter_hex_writer()`
  - `ccsp_bio_filter_zlib_reader()`, `ccsp_bio_filter_zlib_writer()`
  - Composing filters: `ccsp_bio_filter_base64_reader(zlib_reader(fd_reader))`
- 3.1.7 Formatting: `ccsp_bio_fmt_*`
  - `ccsp_bio_fmt_write()`: printf-style
  - `ccsp_bio_fmt_write_hex()`: hex dump with offset
  - `ccsp_bio_fmt_write_pem()`: PEM with label
  - `ccsp_bio_fmt_write_base64()`: plain base64
- 3.1.8 The zero-copy path
  - `ccsp_ring_reserve()` -> write directly -> `ccsp_ring_commit()`
  - TLS record layer pattern: no intermediate copy
  - When zero-copy is and is not applicable
- 3.1.9 lib_ccsp_event integration
  - `ccsp_bio_bufev_t`: buffered event, libevent-backed
  - Backpressure: write buffer full -> `CCSP_BIO_ERR_WOULD_BLOCK`
  - Drain callback: `on_drain` fires when write buffer empties
- 3.1.10 Writing a custom reader or writer

### Chapter 3.2: lib_ccsp_dns -- Async Resolver
- 3.2.1 Why gethostbyname() is a CVE factory
- 3.2.2 `ccsp_dns_resolver_t`: the resolver context
  - `ccsp_dns_resolver_create()`: config and libevent base
  - `ccsp_dns_resolver_destroy()`
- 3.2.3 `ccsp_dns_config_t`: configuration
  - Nameserver list: address, port
  - Search domains: list and mode (NDOTS, ALWAYS, NONE)
  - Timeouts: per-query, per-retry
  - Retry: count, delay, backoff
  - Cache: max entries, min/max TTL
  - TCP: force, fallback-on-truncation
  - DNSSEC: validate flag, trust anchors
- 3.2.4 `ccsp_dns_config_from_resolv_conf()`: parse /etc/resolv.conf
- 3.2.5 `ccsp_dns_config_from_os()`: platform-specific resolver config
- 3.2.6 `ccsp_dns_resolver_query()`: submit async query
  - name, type, class, callback, userdata
  - Returns `ccsp_dns_query_t *` handle
- 3.2.7 `ccsp_dns_query_cancel()`: cancellation guarantee
- 3.2.8 `ccsp_dns_response_t`: the result
  - `status`: NOERROR, NXDOMAIN, SERVFAIL, TIMEOUT, etc.
  - `authoritative`, `truncated` flags
  - `dnssec_validated` flag
  - `resolved_name`, `from_search` fields
  - `answers[]`, `authority[]`, `additional[]`
- 3.2.9 Record types: parsed structs for each
  - `ccsp_dns_rr_a_t`: IPv4 address
  - `ccsp_dns_rr_aaaa_t`: IPv6 address
  - `ccsp_dns_rr_mx_t`: preference + exchange name
  - `ccsp_dns_rr_srv_t`: priority, weight, port, target
  - `ccsp_dns_rr_txt_t`: text string array
  - `ccsp_dns_rr_tlsa_t`: usage, selector, matching_type, data
  - `ccsp_dns_rr_naptr_t`: order, preference, flags, service, regexp, replacement
  - `ccsp_dns_rr_soa_t`: all SOA fields
  - `ccsp_dns_rr_cname_t`, `ccsp_dns_rr_ns_t`, `ccsp_dns_rr_ptr_t`
- 3.2.10 DNSSEC validation
  - Trust anchors: IANA root zone KSK
  - Validation chain: RRSIG -> DNSKEY -> DS -> parent
  - `ccsp_dns_rr_dnskey_t`, `ccsp_dns_rr_rrsig_t`, `ccsp_dns_rr_ds_t`
  - `ccsp_dns_rr_nsec_t`, `ccsp_dns_rr_nsec3_t`
  - What to do when validation fails: configurable policy
- 3.2.11 Cache behavior
  - TTL enforcement: entries expire at TTL
  - `cache_min_ttl`: floor for zero-TTL records
  - `cache_max_ttl`: ceiling for very long TTLs
  - Cache invalidation: manual flush
- 3.2.12 Security: per-query random source port
- 3.2.13 TCP fallback: connection reuse behavior
- 3.2.14 mDNS integration: .local routing

### Chapter 3.3: lib_ccsp_mdns -- Multicast DNS
- 3.3.1 RFC 6762 overview and scope
- 3.3.2 `ccsp_mdns_t`: the mDNS context
  - `ccsp_mdns_create()`: event base, interface
  - `ccsp_mdns_destroy()`
- 3.3.3 Service announcement
  - `ccsp_mdns_announce()`: name, service type, port, TXT records
  - `ccsp_mdns_withdraw()`: remove announcement
  - Renewal timing: RFC-specified intervals, automatic
  - Conflict detection: probe-then-announce per RFC 6762
  - Conflict resolution callback: name has changed
- 3.3.4 Service browsing
  - `ccsp_mdns_browse()`: service type, on_found, on_lost callbacks
  - `ccsp_mdns_service_t`: name, hostname, port, TXT, address
  - `ccsp_mdns_resolve()`: resolve instance to address+port
- 3.3.5 DNS-SD (RFC 6763)
  - PTR record browse: service type discovery
  - SRV -> A/AAAA resolution chain
  - TXT metadata access
  - `ccsp_mdns_browse_types()`: what service types are present
- 3.3.6 lib_ccsp_connect integration: .local service names
- 3.3.7 Multi-interface behavior: which interface responds

### Chapter 3.4: lib_ccsp_connect -- Stream Connection
- 3.4.1 The SO_ERROR miss: 19 CVEs and the fix
- 3.4.2 `ccsp_conn_config_t`: configuration
  - `host`, `port`: direct connection
  - `host`, `service`, `proto`: SRV-based discovery
  - `dns_timeout_ms`, `connect_timeout_ms`, `total_timeout_ms`
  - `max_attempts`: across all addresses
  - `retry_delay_ms`: between attempts
  - `happy_eyeballs`: RFC 6555, 50ms stagger
  - `prefer_ipv6`, `ipv4_only`, `ipv6_only`
  - `tls`: optional TLS config (NULL = plaintext)
  - `dane_mode`: DISABLED, OPPORTUNISTIC, REQUIRED
  - `bind_addr`, `bind_port`: local binding
  - `on_connect`, `on_read`, `on_drain`, `on_close` callbacks
- 3.4.3 `ccsp_conn_connect()`: submit connection, nonblocking
- 3.4.4 `ccsp_conn_cancel()`: cancel in-progress connection
- 3.4.5 `ccsp_conn_stream_t`: the established stream
  - `ccsp_conn_write()`: write to stream
  - `ccsp_conn_close()`: close stream
  - `ccsp_conn_get_peer()`: remote address
  - `ccsp_conn_is_tls()`: TLS status
  - `ccsp_conn_get_tls_info()`: negotiated version, cipher, ALPN
  - `ccsp_conn_service_info()`: DNS-SD TXT records
- 3.4.6 Happy eyeballs in detail
  - The 50ms stagger: why this value
  - IPv6 and IPv4 attempt interleaving
  - Cancellation of the losing attempt
- 3.4.7 SRV selection in detail
  - Priority group ordering
  - Weighted random within group
  - Why secure random prevents steering attacks
  - RFC 2782 weight-zero handling
- 3.4.8 DANE enforcement in detail
  - TLSA record lookup: when it happens
  - DNSSEC validation requirement
  - Usage 0/1/2/3 handling
  - Selector 0/1 handling
  - Matching type 0/1/2 handling
  - Certificate expiry check even for usage 3
  - `CCSP_CONN_ERR_TLS_DANE_MISMATCH`: what happens
- 3.4.9 The connection state machine
  - States: resolving_srv, resolving_direct, selecting_candidate,
    resolving_target, connecting, handshaking, connected
  - Transitions: detailed diagram
  - Error handling per state
- 3.4.10 Error taxonomy: complete list with descriptions
- 3.4.11 `ccsp_conn_service_info()`: TXT record access post-connection

---

## Volume 4: Protocols

### Chapter 4.1: lib_ccsp_lineproto -- Line Protocol Framework
- 4.1.1 The 47 CVEs and why this library exists
- 4.1.2 Architecture: schema, dispatch, application callbacks
- 4.1.3 Argument types: NONE, STRING, WORD, WORDS, KV, CUSTOM
- 4.1.4 `ccsp_lp_arg_desc_t`: argument descriptor
- 4.1.5 `ccsp_lp_args_t`: parsed arguments
  - `raw`, `raw_len`: original argument string
  - `words[]`, `word_lens[]`, `word_count`
  - `kv[]`, `kv_count`
  - `custom`: application-defined parsed data
- 4.1.6 `ccsp_lp_response_t`: response struct
  - `code`, `message`: single-line response
  - `multiline`, `write_lines`: multi-line response callback
  - `next_state`: `CCSP_LP_STATE_SAME`, `CCSP_LP_STATE_CLOSE`,
    `CCSP_LP_STATE_PUSH`, or application state ID
- 4.1.7 `ccsp_lp_cmd_t`: command descriptor
  - `verb`: case-folded automatically
  - `args`: argument descriptor
  - `allowed_states`: bitmask of valid states
  - `pipeline_ok`: safe to pipeline
  - `auth_required`: enforced before calling handler
  - `max_per_minute`: rate limiting
  - `handle`: the callback
  - `help`: for HELP verb auto-generation
- 4.1.8 `ccsp_lp_state_t`: state descriptor
  - `id`, `name`
  - `timeout_ms`: per-state timeout
  - `greeting_code`, `greeting_msg`: sent on entering state
  - `error_code`, `error_msg`: sent on invalid command
  - `on_enter`, `on_timeout`, `on_exit` callbacks
- 4.1.9 `ccsp_lp_proto_t`: protocol descriptor
  - `name`, `version`
  - `max_line_len`: enforced, connection killed on exceed
  - `crlf_required`, `crlf_output`
  - `cmds[]`: NULL-terminated command table
  - `states[]`: NULL-terminated state table
  - `initial_state`
  - `auto_quit`, `auto_noop`, `auto_help`
  - `pipeline`: enable
  - `pipeline_max`: max queued commands
  - `starttls`, `tls_config`
  - `on_connect`, `on_disconnect`
  - `on_unknown`: handler for unrecognized verbs
- 4.1.10 `ccsp_lp_server_config_t` and `ccsp_lp_server_create()`
  - `bind_addr`, `bind_port`, `backlog`
  - `max_connections`, `max_connections_per_ip`
  - `connect_timeout_ms`
- 4.1.11 Connection API: `ccsp_lp_conn_t`
  - `ccsp_lp_conn_write()`: formatted write
  - `ccsp_lp_conn_write_code()`: numeric response
  - `ccsp_lp_conn_write_multiline()`: multi-line response
  - `ccsp_lp_conn_set_auth()`: mark as authenticated
  - `ccsp_lp_conn_get_peer()`: remote address
  - `ccsp_lp_conn_is_tls()`: TLS status
  - `ccsp_lp_conn_get_userdata()`, `ccsp_lp_conn_set_userdata()`
  - `ccsp_lp_conn_get_state()`: current state
- 4.1.12 Data mode: `ccsp_lp_data_mode_t`
  - `CCSP_LP_DATA_DELIMITED`: SMTP dot-stuffing
  - `CCSP_LP_DATA_LENGTH`: IMAP literals
  - `CCSP_LP_DATA_BINARY`: FTP image mode
  - `CCSP_LP_DATA_CUSTOM`: application framing
  - `on_complete` callback: data received, resume line mode
  - `max_size`: prevent unbounded data reception
- 4.1.13 Pipelining
  - Which commands can be pipelined
  - `pipeline_ok = false`: DATA and similar flush the pipeline
  - Pipeline depth limit
  - Response ordering guarantee
- 4.1.14 STARTTLS integration
  - Response, flush, TLS handshake, state machine reset
  - Auth status cleared: why this is required per RFC
- 4.1.15 Client: `ccsp_lp_client_t`
  - `ccsp_lp_client_create()`: event base, resolver, proto
  - `ccsp_lp_client_send()`: async command send
  - Response callback: code, message, userdata
  - Client-server loopback: testing without network
- 4.1.16 Rate limiting: `max_per_minute` implementation
- 4.1.17 Complete example: SMTP server skeleton (with annotations)
- 4.1.18 Complete example: Redis client
- 4.1.19 Complete example: memcached client
- 4.1.20 Writing a custom protocol: step by step

### Chapter 4.2: lib_ccsp_http -- HTTP Client and Server
- 4.2.1 Design: protocol only, not a framework
- 4.2.2 Version negotiation via ALPN: transparent to caller
- 4.2.3 `ccsp_http_request_t`
  - method, url, headers, body reader
  - `ccsp_http_header_t`: name (lowercase), value
  - Header canonicalization: lowercase names, trim whitespace
- 4.2.4 `ccsp_http_response_t`
  - status code, status text, headers, body reader
  - Body as `ccsp_bio_reader_t`: streaming, no buffer
- 4.2.5 HTTP/1.1 details
  - Chunked transfer encoding: transparent decode/encode
  - Keep-alive: connection reuse
  - Expect: 100-continue
  - Trailers: after chunked body
- 4.2.6 HTTP/2 details
  - HPACK header compression
  - Stream multiplexing: many requests per connection
  - Flow control: per-stream and per-connection
  - Server push: disabled by default, configurable
  - GOAWAY handling: graceful connection close
- 4.2.7 Client API
  - `ccsp_http_client_create()`: event base, resolver, config
  - `ccsp_http_client_request()`: async request
  - Response callback: `(response, err, userdata)`
  - Connection pooling: max connections per host
  - Redirect following: configurable max redirects
  - Cookie jar: `ccsp_http_cookie_jar_t`
  - Timeout: connect, headers, body
- 4.2.8 Server API
  - `ccsp_http_server_create()`: bind address, port, config
  - `on_request` callback: `(request, response_writer, userdata)`
  - `ccsp_http_response_writer_t`: write headers, write body
  - `ccsp_http_server_set_error_handler()`: 4xx/5xx handler
  - Max connections, max request size
- 4.2.9 WebSocket upgrade
  - `ccsp_http_upgrade_websocket()`: from a request handler
  - WebSocket frame types: text, binary, ping, pong, close
  - `ccsp_ws_conn_t`: the WebSocket connection
  - Fragmentation: transparent reassembly
- 4.2.10 TLS configuration: passed through to lib_ccsp_tls
- 4.2.11 HTTP/2 push: server-initiated streams
- 4.2.12 Compression: gzip/deflate via lib_ccsp_bio filter

### Chapter 4.3: lib_ccsp_smtp -- SMTP
- 4.3.1 The sendmail CVE motivation
- 4.3.2 RFC 5321 coverage: what is and is not implemented
- 4.3.3 Server: `ccsp_smtp_server_create()`
  - SMTP state machine: GREETING->EHLO->MAIL->RCPT->DATA->SESSION
  - EHLO capability advertisement
  - Auth mechanisms: PLAIN, LOGIN, XOAUTH2
  - Message size limit
  - Recipient limit
  - `on_message` callback: envelope + body reader
- 4.3.4 Client: `ccsp_smtp_client_create()`
  - MX record resolution via lib_ccsp_dns
  - STARTTLS negotiation
  - SMTP AUTH
  - PIPELINING
  - Message submission: `ccsp_smtp_send()`
- 4.3.5 DANE MX enforcement
  - TLSA lookup for MX target
  - DNSSEC validation requirement
  - Fallback policy when DANE unavailable
  - Logging: what was validated, what was not
- 4.3.6 DKIM signing
  - `ccsp_smtp_dkim_sign()`: sign outbound message
  - Key selection: domain and selector
  - Canonicalization: simple and relaxed
  - Hash algorithm: SHA-256
- 4.3.7 DKIM verification
  - `ccsp_smtp_dkim_verify()`: verify inbound DKIM header
  - DNS lookup for public key
  - Signature validation
  - Result: pass, fail, neutral, tempfail, permerror
- 4.3.8 Envelope handling: MAIL FROM, RCPT TO parsing
- 4.3.9 Message parsing: RFC 5322 headers, MIME
- 4.3.10 Bounce handling: DSN (RFC 3461)

### Chapter 4.4: lib_ccsp_imap -- IMAP
- 4.4.1 RFC 3501 coverage
- 4.4.2 The literal challenge: `{1234}` mid-command
- 4.4.3 Client: `ccsp_imap_client_create()`
  - Authentication: LOGIN, PLAIN, XOAUTH2, AUTHENTICATE
  - Capability negotiation: IMAP4rev1, LITERAL+, CONDSTORE, etc.
- 4.4.4 Folder operations
  - `ccsp_imap_list()`: folder listing with attributes
  - `ccsp_imap_select()`, `ccsp_imap_examine()`
  - `ccsp_imap_create()`, `ccsp_imap_delete()`, `ccsp_imap_rename()`
  - `ccsp_imap_subscribe()`, `ccsp_imap_unsubscribe()`
- 4.4.5 Message operations
  - `ccsp_imap_fetch()`: fetch message parts
  - BODY[HEADER], BODY[TEXT], BODY[1.2.3]: MIME part addressing
  - `ccsp_imap_store()`: flag modification
  - `ccsp_imap_copy()`, `ccsp_imap_move()` (RFC 6851)
  - `ccsp_imap_expunge()`: delete flagged messages
- 4.4.6 Search
  - `ccsp_imap_search()`: full IMAP search grammar
  - Date, size, flag, header, body criteria
  - UID search: stable identifiers
- 4.4.7 MIME parsing
  - `ccsp_imap_mime_t`: BODYSTRUCTURE result
  - Part addressing: `1`, `1.1`, `1.2`, `2`, etc.
  - Attachment extraction: `on_part` callback
  - Content-Transfer-Encoding: decode BASE64/QP transparently
- 4.4.8 IDLE (RFC 2177)
  - `ccsp_imap_idle()`: push notification for new messages
  - Keepalive: re-issue IDLE before 30 minute server timeout
  - `on_exists`, `on_expunge` callbacks during IDLE
- 4.4.9 CONDSTORE (RFC 7162)
  - HIGHESTMODSEQ: only fetch messages changed since last sync
  - Efficient incremental sync
- 4.4.10 Server: `ccsp_imap_server_create()`
  - State machine: NOT_AUTHENTICATED->AUTHENTICATED->SELECTED->LOGOUT
  - Backend interface: `ccsp_imap_backend_t` for message storage

---

## Volume 5: Cryptography and Security

### Chapter 5.1: lib_ccsp_asn1 -- BER/DER Codec
- 5.1.1 The ANY policy: why it is not implemented
- 5.1.2 ASN.1 type support
  - Primitive: BOOLEAN, INTEGER, BIT STRING, OCTET STRING, NULL,
    OBJECT IDENTIFIER, REAL, ENUMERATED
  - String: UTF8String, PrintableString, IA5String, BMPString,
    VisibleString, NumericString, TeletexString (T61String),
    UniversalString, GeneralString, VideotexString
  - Time: UTCTime, GeneralizedTime
  - Constructed: SEQUENCE, SEQUENCE OF, SET, SET OF
  - Tagging: EXPLICIT, IMPLICIT, AUTOMATIC
  - CHOICE, ANY (rejected at parse time with error)
- 5.1.3 Schema-driven parsing
  - ASN.1 module syntax: writing a schema
  - The generator: converting schema to C parser
  - Adding a new schema: step by step
- 5.1.4 Strict DER mode vs permissive BER mode
  - What strict DER rejects: indefinite-length, non-minimal lengths,
    non-canonical booleans, constructed where primitive required
  - When to use permissive BER: legacy data only
- 5.1.5 Security limits
  - `max_depth`: default 64, hard max 128
  - `max_object_size`: default 16MB
  - `max_members`: default 1024
  - Configuring for untrusted input vs trusted local data
- 5.1.6 String normalization
  - All string types -> UTF-8 on output
  - T61String: the non-standard encoding problem
  - BMPString: UCS-2 -> UTF-16 -> UTF-8 conversion
- 5.1.7 OID handling
  - `ccsp_asn1_oid_t`: binary representation
  - `ccsp_asn1_oid_to_string()`: dot-notation
  - `ccsp_asn1_oid_from_string()`: dot-notation to binary
  - OID registry: known OIDs with names
- 5.1.8 Encode API
  - `ccsp_asn1_encode_*()`: per-type encoding functions
  - `ccsp_asn1_encode_sequence()`: constructed type
  - Output to `ccsp_bio_writer_t`
- 5.1.9 Decode API
  - `ccsp_asn1_decode_*()`: per-type decoding functions
  - Schema-driven: `ccsp_asn1_decode_schema()`
  - Error details: type, expected vs actual, offset
- 5.1.10 Fuzz results and corpus

### Chapter 5.2: lib_ccsp_cert -- Certificate Handling
- 5.2.1 The three modules: certparse, certval, ca-nav
- 5.2.2 certparse: DER to `ccsp_cert_t`
  - `ccsp_cert_parse()`: bytes in, immutable cert out
  - `ccsp_cert_ref()`, `ccsp_cert_unref()`: reference counting
  - `ccsp_cert_t` accessors:
    - `ccsp_cert_version()`: 1, 2, or 3
    - `ccsp_cert_serial()`: serial number as bytes
    - `ccsp_cert_subject()`: DN as `ccsp_cert_dn_t`
    - `ccsp_cert_issuer()`: DN as `ccsp_cert_dn_t`
    - `ccsp_cert_not_before()`, `ccsp_cert_not_after()`: time_t
    - `ccsp_cert_pubkey_type()`: RSA, EC, Ed25519, etc.
    - `ccsp_cert_pubkey_der()`: DER-encoded public key
    - `ccsp_cert_san_dns_count()`, `ccsp_cert_san_dns()`: SAN DNS names
    - `ccsp_cert_san_ip_count()`, `ccsp_cert_san_ip()`: SAN IP addresses
    - `ccsp_cert_san_email_count()`, `ccsp_cert_san_email()`
    - `ccsp_cert_eku()`: extended key usage flags
    - `ccsp_cert_basic_constraints()`: CA flag, path length
    - `ccsp_cert_key_usage()`: key usage bits
    - `ccsp_cert_aia_ocsp()`: OCSP URL from AIA extension
    - `ccsp_cert_aia_issuers()`: issuer URL from AIA extension
    - `ccsp_cert_cdp()`: CRL distribution point URLs
    - `ccsp_cert_policy_oids()`: certificate policy OIDs
    - `ccsp_cert_extension_count()`, `ccsp_cert_extension()`: all extensions
    - `ccsp_cert_der()`: original DER bytes
  - `ccsp_cert_dn_t` accessors:
    - `ccsp_cert_dn_attr_count()`: number of attributes
    - `ccsp_cert_dn_attr()`: attribute by index: OID, type, value (UTF-8)
    - `ccsp_cert_dn_get()`: get attribute value by OID shortname ("CN", "O", etc.)
    - `ccsp_cert_dn_to_string()`: RFC 4514 string representation
  - Immutability guarantee: no modification functions exist
- 5.2.3 certval: RFC 5280 path validation
  - `ccsp_certval_verify()`: the main function
    - leaf cert, chain array, trust store, policy, at_time
    - Returns `ccsp_certval_result_t`: valid or specific error
  - `ccsp_certval_result_t` error values:
    - `CCSP_CERTVAL_OK`
    - `CCSP_CERTVAL_ERR_EXPIRED`
    - `CCSP_CERTVAL_ERR_NOT_YET_VALID`
    - `CCSP_CERTVAL_ERR_UNKNOWN_CA`
    - `CCSP_CERTVAL_ERR_REVOKED`
    - `CCSP_CERTVAL_ERR_INVALID_SIGNATURE`
    - `CCSP_CERTVAL_ERR_POLICY_MISMATCH`
    - `CCSP_CERTVAL_ERR_NAME_CONSTRAINTS`
    - `CCSP_CERTVAL_ERR_PATH_LEN`
    - `CCSP_CERTVAL_ERR_KEY_USAGE`
    - `CCSP_CERTVAL_ERR_EKU`
    - `CCSP_CERTVAL_ERR_UNKNOWN_CRITICAL_EXTENSION`
  - `ccsp_certval_policy_t`: validation policy configuration
    - Required EKU flags
    - Allowed signature algorithms
    - Minimum RSA key size
    - Minimum EC curve
    - Allow self-signed (for DANE usage 3 testing)
  - The explicit `at_time` parameter: never calls `time()`
  - Name constraints validation: subtree matching
  - Policy OID validation: intersection and mapping
- 5.2.4 ca-nav: path building and revocation
  - `ccsp_ca_nav_t`: the navigator
  - `ccsp_ca_nav_create()`: with fetch callback
  - `ccsp_ca_nav_build_path()`: AIA chasing to build chain
  - `ccsp_ca_nav_check_ocsp()`: OCSP query
  - `ccsp_ca_nav_check_crl()`: CRL fetch and parse
  - `ccsp_ca_nav_set_staple()`: accept stapled OCSP response
  - The fetch callback: `int (*fetch)(url, writer, userdata)`
  - lib_ccsp_http as the fetch callback
  - Mock fetch for testing
  - Result caching: TTL-based
- 5.2.5 Trust store: `ccsp_cert_store_t`
  - `ccsp_cert_store_create()`
  - `ccsp_cert_store_add()`: add trust anchor
  - `ccsp_cert_store_load_file()`: PEM bundle
  - `ccsp_cert_store_load_system()`: OS trust store
  - System trust store locations by platform
- 5.2.6 DANE module
  - `ccsp_cert_validate_dane()`: validate against TLSA record
  - `ccsp_cert_tlsa_record_t`: usage, selector, matching_type, data
  - Usage 0: CA constraint
  - Usage 1: service certificate constraint
  - Usage 2: trust anchor assertion
  - Usage 3: domain-issued certificate
  - Selector 0: full certificate
  - Selector 1: public key only
  - Matching type 0: exact, 1: SHA-256, 2: SHA-512
  - Constant-time comparison: always
  - Expiry check even for usage 3: why

### Chapter 5.3: lib_ccsp_pki -- Certificate Generation
- 5.3.1 Key generation API
  - `ccsp_pki_keygen_rsa()`: bits, public exponent
  - `ccsp_pki_keygen_ec()`: curve (P-256, P-384, P-521, X25519, X448)
  - `ccsp_pki_keygen_ed()`: algorithm (Ed25519, Ed448)
  - `ccsp_pki_key_t`: the opaque key
  - `ccsp_pki_key_free()`: zeroing free
  - Key material never in public struct fields
- 5.3.2 Key serialization
  - `ccsp_pki_key_export_private_pem()`: PKCS#8 PEM
  - `ccsp_pki_key_export_private_der()`: PKCS#8 DER
  - `ccsp_pki_key_export_public_pem()`: SubjectPublicKeyInfo PEM
  - `ccsp_pki_key_export_public_der()`: SubjectPublicKeyInfo DER
  - `ccsp_pki_key_import_private()`: from PEM or DER
  - `ccsp_pki_key_import_public()`: from PEM or DER
- 5.3.3 CSR generation
  - `ccsp_pki_csr_builder_t`: the builder
  - `ccsp_pki_csr_builder_set_cn()`, `_set_org()`, `_set_country()`, etc.
  - `ccsp_pki_csr_builder_add_san_dns()`, `_add_san_ip()`, `_add_san_email()`
  - `ccsp_pki_csr_builder_add_eku()`: extended key usage
  - `ccsp_pki_csr_builder_sign()`: sign with key, produce DER CSR
- 5.3.4 Certificate signing
  - `ccsp_pki_cert_builder_t`: the builder
  - `ccsp_pki_cert_builder_from_csr()`: start from a CSR
  - `ccsp_pki_cert_builder_set_validity()`: explicit start and end times
  - `ccsp_pki_cert_builder_set_serial()`: explicit or `_set_random_serial()`
  - `ccsp_pki_cert_builder_set_basic_constraints()`: CA, path length
  - `ccsp_pki_cert_builder_add_aia_ocsp()`: OCSP URL
  - `ccsp_pki_cert_builder_add_cdp()`: CRL distribution point
  - `ccsp_pki_cert_builder_sign()`: sign with CA key and cert
- 5.3.5 Self-signed certificates for DANE usage 3
  - `ccsp_pki_cert_builder_self_sign()`: no CA needed
  - Why self-signed + DANE is secure
  - The operational key rotation procedure
- 5.3.6 PKCS#12 operations
  - `ccsp_pki_p12_import()`: extract key, cert, chain
  - `ccsp_pki_p12_export()`: pack key, cert, chain
  - Encryption: AES-256-CBC + PBKDF2 (not RC2/DES)
  - Password handling: UTF-8, zeroed after use

### Chapter 5.4: lib_ccsp_ocl -- Cryptographic Primitives
- 5.4.1 Layer design: primitives, schemes, AEAD
- 5.4.2 The AEAD interface: what applications should use
  - `ccsp_ocl_aead_create()`: algorithm + key material
  - `ccsp_ocl_aead_seal()`: encrypt and authenticate
    - Nonce generated internally: structurally prevents reuse
    - AAD: additional authenticated data
    - Output: nonce + ciphertext + tag
  - `ccsp_ocl_aead_open()`: authenticate and decrypt
    - Input: nonce + ciphertext + tag
    - Constant-time tag comparison
    - Returns error on authentication failure
  - `ccsp_ocl_aead_destroy()`: zeroing free
  - Algorithm `CCSP_OCL_AEAD_AES256GCM`
  - Algorithm `CCSP_OCL_AEAD_CHACHA20POLY1305`
- 5.4.3 Hash functions
  - `ccsp_ocl_sha256()`, `ccsp_ocl_sha384()`, `ccsp_ocl_sha512()`
  - `ccsp_ocl_sha3_256()`, `ccsp_ocl_sha3_512()`
  - `ccsp_ocl_blake2b()`, `ccsp_ocl_blake3()`
  - Streaming: `ccsp_ocl_hash_ctx_t`, `_init()`, `_update()`, `_final()`
- 5.4.4 HMAC
  - `ccsp_ocl_hmac_sha256()`, `ccsp_ocl_hmac_sha512()`
  - Streaming: `ccsp_ocl_hmac_ctx_t`
  - Key material zeroed in context after `_final()`
- 5.4.5 Key derivation
  - `ccsp_ocl_hkdf_extract()`, `ccsp_ocl_hkdf_expand()`
  - `ccsp_ocl_pbkdf2_sha256()`: iteration count, output length
  - `ccsp_ocl_argon2id()`: memory cost, time cost, parallelism
- 5.4.6 RSA operations
  - `ccsp_ocl_rsa_sign_pss()`: RSASSA-PSS with SHA-256
  - `ccsp_ocl_rsa_verify_pss()`: PSS verification
  - `ccsp_ocl_rsa_encrypt_oaep()`: RSAES-OAEP
  - `ccsp_ocl_rsa_decrypt_oaep()`: OAEP decryption, constant-time
  - PKCS#1 v1.5: available, labeled legacy
- 5.4.7 ECDSA operations
  - `ccsp_ocl_ecdsa_sign()`: P-256, P-384, P-521
  - `ccsp_ocl_ecdsa_verify()`: verification
  - Deterministic nonce per RFC 6979: no RNG needed for signing
  - DER-encoded signature output
- 5.4.8 ECDH key agreement
  - `ccsp_ocl_ecdh_generate_keypair()`: ephemeral key
  - `ccsp_ocl_ecdh_compute_shared()`: shared secret
  - Curves: P-256, P-384, X25519, X448
- 5.4.9 EdDSA operations
  - `ccsp_ocl_eddsa_sign()`: Ed25519, Ed448
  - `ccsp_ocl_eddsa_verify()`: verification
- 5.4.10 Constant-time discipline
  - What is secret: private keys, AEAD keys, MAC tags
  - How constant-time is verified: dudect methodology
  - The timing test in CI: what it measures and thresholds
  - Compiler warnings that can break constant-time
- 5.4.11 FIPS 140-3 boundary
  - Module boundary: what is inside
  - Self-tests: power-on, conditional
  - Key zeroization: when and how
  - Approved algorithm list
  - Error states: what happens when a self-test fails
- 5.4.12 Hardware acceleration
  - AES-NI: x86/x86_64 detection and use
  - ARMv8 crypto: AArch64 detection and use
  - Software fallback: always available
  - Testing: CI runs both paths
- 5.4.13 NIST CAVP test vectors: coverage per algorithm

### Chapter 5.5: lib_ccsp_tls -- TLS Protocol
- 5.5.1 Why Heartbleed was structurally inevitable: the analysis
- 5.5.2 TLS version support
  - TLS 1.3 (RFC 8446): primary
  - TLS 1.2 (RFC 5246): for compatibility
  - TLS 1.1 and below: not implemented, not configurable
- 5.5.3 `ccsp_tls_config_t`: configuration
  - `server_name`: required for clients (SNI)
  - `cert`, `key`: server certificate and key
  - `alpn_protocols[]`: offered protocols
  - `cert_policy`: `ccsp_certval_policy_t`
  - `session_ticket_keys`: for resumption
  - `session_ticket_lifetime_s`: ticket validity
  - `min_version`: minimum TLS version
  - `cipher_suites[]`: allowed cipher suites
- 5.5.4 Cipher suites supported
  - TLS 1.3: TLS_AES_256_GCM_SHA384, TLS_CHACHA20_POLY1305_SHA256,
    TLS_AES_128_GCM_SHA256
  - TLS 1.2: ECDHE-ECDSA-AES256-GCM-SHA384,
    ECDHE-RSA-AES256-GCM-SHA384,
    ECDHE-ECDSA-CHACHA20-POLY1305,
    ECDHE-RSA-CHACHA20-POLY1305
  - Excluded: all non-AEAD, all RSA key exchange, all CBC
- 5.5.5 The explicit state machine
  - State table: all states enumerated
  - Transition table: (state, event) -> (action, next state)
  - Invalid transition handling: log and close
  - Testing: every state reachable, every transition exercised
- 5.5.6 SNI: required, not optional
  - `server_name` field in config: validated at create time
  - Empty server_name: error at config validation
  - Server-side: SNI used for certificate selection
  - The privacy note: SNI visible to network observers
  - Encrypted SNI path: tracked, not yet implemented
- 5.5.7 Forward secrecy: mandatory
  - All cipher suites use ECDHE or DHE
  - No configuration to disable
  - The hypothetical config flag: why it does not exist
- 5.5.8 No renegotiation
  - TLS renegotiation: the design mistake
  - How lib_ccsp_tls handles a renegotiation request: close + log
  - `CCSP_TLS_ERR_RENEGOTIATION_REQUESTED`: log level and content
- 5.5.9 ALPN
  - `alpn_protocols[]` in config
  - Negotiated protocol: `ccsp_tls_get_alpn()`
  - No ALPN: connection proceeds, `get_alpn()` returns empty
- 5.5.10 Session resumption (TLS 1.3)
  - Session tickets: `ccsp_tls_config_t.session_ticket_keys[]`
  - Key rotation: old keys retained briefly
  - Replay prevention: NewSessionTicket only once
  - 0-RTT: not implemented in initial version
- 5.5.11 OCSP stapling
  - Client: accept and validate stapled response
  - Server: fetch from OCSP responder, cache, staple in handshake
  - `ccsp_tls_set_occsp_staple()`: for server mode
- 5.5.12 Certificate validation
  - Calls lib_ccsp_cert certval
  - Policy from `ccsp_tls_config_t.cert_policy`
  - DANE: if lib_ccsp_connect provides TLSA records
  - Pinning: `ccsp_tls_config_t.pinned_pubkey_sha256[]`
- 5.5.13 lib_ccsp_connect integration
  - How lib_ccsp_connect creates and configures TLS contexts
  - DANE result passed to lib_ccsp_tls for certificate validation
  - ALPN set from service protocol
- 5.5.14 Handshake timing: log events for observability

---

## Volume 6: Data and Parsing

### Chapter 6.1: lib_ccsp_data -- Shared Value Type
- 6.1.1 The motivation: one type for all structured data
- 6.1.2 `ccsp_data_value_t`: the type enum and union
- 6.1.3 Integer vs float: why we distinguish
- 6.1.4 `ccsp_data_get()`: dot-separated path navigation
- 6.1.5 Typed accessors with default values
  - `ccsp_data_get_string()`, `ccsp_data_get_integer()`
  - `ccsp_data_get_float()`, `ccsp_data_get_bool()`
  - `ccsp_data_get_datetime()`, `ccsp_data_get_bytes()`
- 6.1.6 Array access: `ccsp_data_array_len()`, `ccsp_data_array_get()`
- 6.1.7 Table access: `ccsp_data_table_len()`, `ccsp_data_table_key()`,
  `ccsp_data_table_val()`, `ccsp_data_table_get()`
- 6.1.8 Iteration: `ccsp_data_each_array()`, `ccsp_data_each_table()`
- 6.1.9 Serialization to JSON and TOML
- 6.1.10 Immutability: `ccsp_data_clone()` for modification
- 6.1.11 Insertion order preservation: the lib_ccsp_hash connection

### Chapter 6.2: lib_ccsp_toml -- TOML Parser
- 6.2.1 The TOML spec: our version, published alongside
- 6.2.2 `ccsp_toml_parse()`: bytes to `ccsp_data_value_t`
- 6.2.3 `ccsp_toml_parse_file()`: from file path
- 6.2.4 Error type: `ccsp_toml_err_t` with line and column
- 6.2.5 Type coverage: all TOML types
- 6.2.6 Multiline strings: basic and literal
- 6.2.7 Arrays: inline and block (array of tables)
- 6.2.8 Path navigation and default values
- 6.2.9 Round-trip serialization: key order preserved
- 6.2.10 lib_ccsp_cli integration: config file source
- 6.2.11 Conformance: toml-test suite results

### Chapter 6.3: lib_ccsp_json -- JSON Parser
- 6.3.1 DOM parsing: `ccsp_json_parse()`
- 6.3.2 Streaming parser: `ccsp_json_parser_t`
  - `ccsp_json_parser_create()`: event callback
  - `ccsp_json_parser_feed()`: incremental input
  - `ccsp_json_parser_finish()`: flush and validate
  - Event types and their data
  - Zero-allocation string events: pointer into input buffer
- 6.3.3 lib_ccsp_event integration: streaming from network
- 6.3.4 Security limits: depth, string, document
- 6.3.5 UTF-8 validation: invalid sequences are parse errors
- 6.3.6 Number parsing: strtoll and strtod, no hand-rolled
- 6.3.7 Serialization: `ccsp_json_write_value()`
  - Compact and pretty-printed modes
  - Number format: integer without decimal, float with
- 6.3.8 JSONTestSuite compliance: mandatory and optional results
- 6.3.9 Fuzz results

### Chapter 6.4: lib_ccsp_cas -- Content-Addressable Storage
- 6.4.1 Object model: blob, tree, commit
- 6.4.2 `ccsp_cas_store_t`: the store
  - `ccsp_cas_store_create_fs()`: filesystem backend
  - `ccsp_cas_store_create_mem()`: in-memory for testing
  - `ccsp_cas_store_create_remote()`: delegate to another store
- 6.4.3 `ccsp_cas_put()`: write, get hash
- 6.4.4 `ccsp_cas_get()`: read by hash, verify on read
- 6.4.5 Tree operations
  - `ccsp_cas_put_tree()`: entries array to hash
  - `ccsp_cas_get_tree()`: hash to entries array
  - `ccsp_cas_tree_entry_t`: name, mode, hash
- 6.4.6 Commit operations
  - `ccsp_cas_put_commit()`: tree + parent + metadata
  - `ccsp_cas_get_commit()`: hash to commit struct
  - `ccsp_cas_commit_t`: tree_hash, parent_hash, author, timestamp, message
- 6.4.7 Build system pattern: skip rebuild on hash match
- 6.4.8 Package management pattern: package is a tree
- 6.4.9 Distributed sync pattern: content-addressed replication

### Chapter 6.5: lib_ccsp_cli -- Command-Line Parsing
- 6.5.1 The getopt problem: what it lacks
- 6.5.2 `ccsp_cli_opt_t`: the option descriptor
  - `long_name`, `short_name`
  - `type`: all option types with examples
  - `default_value`, `env_var`, `config_key`
  - `min_value`, `max_value`: validation
  - `required`: error if not set from any source
  - `pattern`: regex validation for strings
  - `hidden`: exists but not shown in default help
  - `deprecated`, `deprecated_use`
  - `metavar`, `help`, `help_long`
  - `on_set`: callback on value assignment
  - `store`: typed union of application variable pointers
- 6.5.3 `ccsp_cli_cmd_t`: the command descriptor
  - `name`, `help`, `help_long`
  - `opts[]`: NULL-terminated option array
  - `subcmds[]`: NULL-terminated subcommand array
  - `positionals[]`: positional argument descriptors
  - `run()`: the command callback
- 6.5.4 `ccsp_cli_parser_t` and `ccsp_cli_parser_create()`
  - `prog_name`, `version`, `description`, `epilog`
  - `config_file`, `config_section`: TOML config
  - `allow_unknown`, `stop_at_nonopt`, `interspersed`
  - `out`, `err`: `ccsp_bio_writer_t` for output
- 6.5.5 `ccsp_cli_parse()`: argc/argv processing
- 6.5.6 `ccsp_cli_run()`: dispatch to selected command
- 6.5.7 `ccsp_cli_help()`: generate help for a command
- 6.5.8 Source priority chain: all four sources
- 6.5.9 `ccsp_cli_result_t`: source tracking
  - Which source set each option
  - What value came from each source
  - `ccsp_cli_print_effective_config()`: debugging output
- 6.5.10 `ccsp_cli_opt_type_t` types in detail
  - `CCSP_CLI_OPT_SIZE`: suffix parsing (k/m/g, kb/mb/gb)
  - `CCSP_CLI_OPT_DURATION`: suffix parsing (ms/s/m/h)
  - `CCSP_CLI_OPT_ENUM`: allowed values list
  - `CCSP_CLI_OPT_LIST`: accumulation
  - `CCSP_CLI_OPT_COUNT`: occurrence counting
- 6.5.11 Help format: the exact output format documented
- 6.5.12 `--no-*` boolean negation: automatic
- 6.5.13 Subcommand dispatch: how the parser routes
- 6.5.14 TOML config integration: section and key mapping
- 6.5.15 Complete example: a real tool with subcommands

---

## Volume 7: Graphics and UI

### Chapter 7.1: lib_ccsp_draw -- Drawing Primitives
- 7.1.1 Design: backend-agnostic, float coordinates, text pre-decomposed
- 7.1.2 `ccsp_draw_canvas_t`: the drawing surface
- 7.1.3 `ccsp_draw_backend_vtable_t`: the twelve required operations
- 7.1.4 `ccsp_draw_paint_t`: paint types
  - Solid color: RGBA float channels
  - Linear gradient: start, end, `ccsp_draw_color_stop_t[]`
  - Radial gradient: center, radius, color stops
  - Image pattern: tile a `ccsp_draw_image_t`
- 7.1.5 `ccsp_draw_path_t`: vector paths
  - `ccsp_draw_path_create()`, `ccsp_draw_path_free()`
  - `ccsp_draw_path_move_to()`, `ccsp_draw_path_line_to()`
  - `ccsp_draw_path_quad_to()`, `ccsp_draw_path_cubic_to()`
  - `ccsp_draw_path_arc()`: center, radius, start/end angle
  - `ccsp_draw_path_close()`
  - `ccsp_draw_path_rect()`: convenience
  - `ccsp_draw_path_rounded_rect()`: convenience with radius
  - `ccsp_draw_path_circle()`: convenience
- 7.1.6 Drawing operations
  - `ccsp_draw_fill_rect()`, `ccsp_draw_stroke_rect()`
  - `ccsp_draw_fill_path()`, `ccsp_draw_stroke_path()`
  - `ccsp_draw_draw_image()`: src rect, dst rect, filter mode
  - `ccsp_draw_draw_glyph()`: positioned glyph from lib_ccsp_text
- 7.1.7 Graphics state
  - `ccsp_draw_save()`, `ccsp_draw_restore()`: state stack
  - `ccsp_draw_clip_rect()`, `ccsp_draw_clip_path()`
  - `ccsp_draw_transform()`: `ccsp_draw_matrix_t`
  - `ccsp_draw_set_opacity()`: 0.0 to 1.0
- 7.1.8 Frame lifecycle
  - `ccsp_draw_frame_begin()`: start frame
  - `ccsp_draw_frame_end()`: finish frame (may submit to GPU)
  - `ccsp_draw_frame_present()`: make visible
  - Why these are separate: GPU pipeline explanation
- 7.1.9 Off-screen surfaces
  - `ccsp_draw_create_offscreen()`: render to texture
  - `ccsp_draw_composite()`: compose onto main canvas
  - Use for layered effects
- 7.1.10 Damage tracking
  - `ccsp_draw_mark_dirty()`: region has changed
  - `ccsp_draw_dirty_rect()`: query accumulated dirty region
  - Backends use dirty rect to minimize work
- 7.1.11 Backend detection: `ccsp_draw_backend_detect()`
- 7.1.12 X11 backend specifics
  - XShm: shared memory for zero-copy local display
  - XRender: alpha compositing and antialiasing
  - Double buffering: back buffer, XCopyArea
  - Damage extension: server-side tracking
- 7.1.13 Framebuffer backend specifics
  - `/dev/fb0` or custom target
  - `flush()` callback: DMA/SPI/register write
  - Double buffering in software
  - Pixel format conversion: RGB565, ARGB8888, etc.
- 7.1.14 Imagemap backend specifics
  - Software rasterizer: Bresenham, scanline fill
  - Output formats: PNG (libpng), PPM, raw RGBA, PDF
  - `ccsp_draw_img_diff()`: pixel comparison for testing
  - Visual regression workflow
- 7.1.15 GPU backend specifics
  - OpenGL 2.1+ minimum
  - Command buffer: built between frame_begin and frame_end
  - Path tessellation: triangulation algorithm
  - Glyph atlas: shelf packing, cache miss handling
  - Complex clip paths: software fallback
  - VBO/VAO usage
- 7.1.16 Adding a new backend: the checklist and template

### Chapter 7.2: lib_ccsp_font -- Font Loading and Rasterization
- 7.2.1 FreeType integration: what we wrap and what we hide
- 7.2.2 `ccsp_font_t`: the font handle
  - `ccsp_font_load_file()`: from filesystem path
  - `ccsp_font_load_mem()`: from memory buffer
  - `ccsp_font_free()`
  - Font collection support: .ttc files
- 7.2.3 Font metrics: `ccsp_font_metrics_t`
  - Ascender, descender, line gap, cap height, x height
  - Units: pixels at a given point size and DPI
- 7.2.4 Glyph operations
  - `ccsp_font_glyph_index()`: codepoint to glyph ID
  - `ccsp_font_rasterize()`: glyph ID to bitmap
    - `ccsp_font_bitmap_t`: pixels, width, height, left, top, advance
    - Antialiasing: grayscale
    - Hinting: enabled by default
  - `ccsp_font_kern_advance()`: kerning between two glyph IDs
  - `ccsp_font_metrics_for_size()`: at a given pixel size
- 7.2.5 Glyph atlas: `ccsp_font_atlas_t`
  - `ccsp_font_atlas_create()`: texture size, initial capacity
  - `ccsp_font_atlas_get_glyph()`: rasterize and cache
  - `ccsp_font_atlas_flush()`: upload changed regions to GPU
  - Shelf-packing algorithm: bin packing for glyphs by height
  - Cache eviction: LRU when atlas is full
- 7.2.6 Subpixel rendering: optional, platform-dependent

### Chapter 7.3: lib_ccsp_text -- Text Layout
- 7.3.1 The pipeline: string -> shaped glyphs -> laid-out lines
- 7.3.2 Shaping with HarfBuzz (when available)
  - OpenType feature processing: liga, kern, calt, etc.
  - Script and language tags
  - Feature enable/disable per run
  - HarfBuzz as optional dependency: graceful degradation
- 7.3.3 Fallback shaping: advance-width positioning
  - For Latin scripts without HarfBuzz
  - Kerning via lib_ccsp_font
- 7.3.4 Bidirectional text: Unicode UAX #9
  - Paragraph level detection
  - Level runs: LTR and RTL within one paragraph
  - Reordering: visual to logical and back
  - Bracket matching
  - Override and isolate formatting characters
- 7.3.5 Line breaking: Unicode UAX #14
  - Mandatory breaks: LF, CR, CRLF, paragraph separator
  - Opportunity breaks: spaces, after hyphens, CJK
  - Prohibited breaks: before closing punctuation, etc.
  - Emergency breaks: long runs of non-breaking content
- 7.3.6 Paragraph layout API
  - `ccsp_text_paragraph_t`: the layout input
    - UTF-8 string with inline style runs
    - `ccsp_text_style_t`: font, size, weight, color, spacing
    - Width constraint
    - Max lines (with ellipsis)
    - Justification: none, left, center, right, full
    - Hyphenation: enabled/disabled, language
  - `ccsp_text_layout_t`: the layout output
    - Array of `ccsp_text_line_t`
    - Each line: array of `ccsp_text_run_t`
    - Each run: font, glyphs with positions
    - Line bounds: origin, ascent, descent, width
    - Total bounds: bounding box of all lines
- 7.3.7 Hit testing: `ccsp_text_hit_test()`
  - Input: point relative to paragraph origin
  - Output: codepoint offset, leading/trailing half
  - Use: cursor placement on click
- 7.3.8 Selection: `ccsp_text_selection_rects()`
  - Input: start and end codepoint offsets
  - Output: array of rectangles covering selection
  - Handles: bidirectional text, multi-line
- 7.3.9 Hyphenation: pattern-based TeX algorithm
  - Language code -> hyphenation pattern set
  - `ccsp_text_hyphen_dict_t`: loaded from pattern file
  - Standard pattern files: en, de, fr, etc.
- 7.3.10 Emoji and combining characters: handling notes

### Chapter 7.4: lib_ccsp_widget -- The Widget System
- 7.4.1 The Flutter layout model: the three rules
- 7.4.2 Why O(n) layout matters: the GTK comparison
- 7.4.3 `ccsp_wid_constraints_t`: min/max width and height
- 7.4.4 `ccsp_wid_size_t`: width and height (result of measure)
- 7.4.5 The five widget methods: the complete interface
- 7.4.6 Defining a custom widget: the template
- 7.4.7 Layout widgets in detail
  - `ccsp_wid_flex()`: the primary layout widget
    - Row vs column: `CCSP_WID_AXIS_HORIZONTAL/VERTICAL`
    - Main-axis alignment: all six options explained
    - Cross-axis alignment: all four options explained
    - Children: `ccsp_wid_t **`, count
  - `ccsp_wid_stack()`: z-axis layering
    - Child positioning within stack bounds
    - `ccsp_wid_positioned()`: explicit x/y within stack
  - `ccsp_wid_constrained()`: size constraints
    - Force exact size: width and height
    - Force minimum: min_width, min_height
    - Force maximum: max_width, max_height
    - `CCSP_WID_INFINITY`: unbounded in a dimension
  - `ccsp_wid_padded()`: insets
    - `ccsp_wid_insets_t`: top, right, bottom, left
    - Symmetric: `ccsp_wid_insets_all()`, `ccsp_wid_insets_symmetric()`
  - `ccsp_wid_expanded()`: take remaining flex space
    - `flex_factor`: proportional share
    - Works only inside `ccsp_wid_flex()`
  - `ccsp_wid_aligned()`: position within available space
    - x_align, y_align: 0.0=start, 0.5=center, 1.0=end
  - `ccsp_wid_scrollable()`: clip and scroll
    - `CCSP_WID_SCROLL_HORIZONTAL`, `_VERTICAL`, `_BOTH`
    - `ccsp_wid_scroll_pos_t`: get/set scroll offset
    - `ccsp_wid_scroll_to()`: animate to position
  - `ccsp_wid_sized_box()`: force size
  - `ccsp_wid_aspect_ratio()`: constrain to ratio
  - `ccsp_wid_wrap()`: wrapping flex container
    - Wrap spacing: gap between rows/columns
- 7.4.8 Leaf widgets in detail
  - `ccsp_wid_text()`: UTF-8 text rendering
    - `ccsp_wid_text_style_t`: all typography properties
    - Overflow: ellipsis, clip
    - Max lines: 0 = unlimited
    - Selectable text: optional
  - `ccsp_wid_image()`: image display
    - Fit modes: fill, contain, cover, scale-down, none
    - Alignment within bounds when not fill
    - Lazy loading: `on_load` callback
  - `ccsp_wid_colored_box()`: filled rectangle
    - `ccsp_draw_paint_t`: solid or gradient
    - Border radius: uniform or per-corner
    - Border: width and color
    - Shadow: offset, blur, spread, color
  - `ccsp_wid_custom_paint()`: escape hatch
    - `measure`: report size to layout system
    - `paint`: receive canvas, draw anything
    - Accessibility descriptor: required
- 7.4.9 Gesture detectors in detail
  - `ccsp_wid_tap()`: all callbacks
  - `ccsp_wid_long_press()`: duration threshold
  - `ccsp_wid_drag()`: axis lock, min distance threshold
  - `ccsp_wid_hover()`: enter, exit, move
  - `ccsp_wid_double_tap()`: time threshold
  - `ccsp_wid_scale()`: two-pointer pinch
  - Gesture arena: conflict resolution when multiple detectors compete
- 7.4.10 Focus and keyboard navigation
  - `ccsp_wid_focusable()`: wraps a widget to be keyboard-focusable
  - Focus traversal order: tree order by default, explicit override
  - `ccsp_wid_focus_scope()`: group with local focus management
  - `ccsp_wid_focus()`, `ccsp_wid_unfocus()`: programmatic
  - Key event routing: focused widget first, then ancestors
- 7.4.11 Accessibility in detail
  - `ccsp_wid_a11y_t`: complete struct documentation
  - `ccsp_wid_a11y_role_t`: all roles with AT-SPI2/macOS/UIA mappings
  - Auto-generated descriptors: which widgets provide them automatically
  - Custom descriptors: required for `ccsp_wid_custom_paint()`
  - `ccsp_wid_a11y_announce()`: live region announcement
  - `ccsp_wid_a11y_focus()`: move accessibility focus programmatically
- 7.4.12 Dirty region tracking and repaint
  - `ccsp_wid_mark_dirty()`: signal state change
  - Dirty accumulation: union across frame
  - Repaint: only dirty regions, only changed subtrees
  - Performance: measured frame time budget
- 7.4.13 Visual regression testing
  - `ccsp_test_visual_assert()`: complete API
  - Reference image workflow
  - `CCSP_TEST_UPDATE_REFS`: how to update
  - Threshold tuning: acceptable pixel difference
  - Difference image format: red highlighting

### Chapter 7.5: lib_ccsp_anim -- Animation Engine
- 7.5.1 The animation model: a float over time
- 7.5.2 `ccsp_anim_t`: `ccsp_anim_create()` parameters
- 7.5.3 `ccsp_anim_curve_t`: all built-in curves with graphs
- 7.5.4 `ccsp_anim_spring_params_t`: mass, stiffness, damping
- 7.5.5 `ccsp_anim_start()`, `ccsp_anim_stop()`, `ccsp_anim_reverse()`
- 7.5.6 Sequences: `ccsp_anim_sequence_create()`
- 7.5.7 Parallel: `ccsp_anim_parallel_create()`
- 7.5.8 Stagger: `ccsp_anim_stagger_create()`
- 7.5.9 Integration with lib_ccsp_widget: the on_value callback pattern

### Chapter 7.6: lib_ccsp_input -- Input Event Normalization
- 7.6.1 `ccsp_input_event_t`: complete struct documentation
- 7.6.2 Float coordinates: why and how
  - Logical vs physical pixels
  - DPI scaling factor
  - Touch screen sub-pixel precision
- 7.6.3 Text input vs key events: the distinction
  - What IME does: preedit, commit
  - Why conflating them breaks CJK
  - `CCSP_INPUT_TEXT_INPUT`: UTF-8 codepoint, not keycode
- 7.6.4 Key codes: `ccsp_input_keycode_t`
  - Platform-independent key code table
  - Modifier key flags: shift, ctrl, alt, meta, caps, num
- 7.6.5 Platform adapters
  - evdev: event types, code mapping
  - X11: KeySym to ccsp_input_keycode mapping
  - macOS: NSEvent to ccsp_input_event
  - Windows: WM_* to ccsp_input_event
- 7.6.6 IME integration
  - `ccsp_input_ime_t`: platform IME handle
  - `ccsp_input_ime_set_cursor()`: position for IME window
  - `ccsp_input_ime_get_preedit()`: current composition string
  - Preedit underline rendering

### Chapter 7.7: lib_ccsp_a11y -- Accessibility
- 7.7.1 The accessibility tree: source from lib_ccsp_widget
- 7.7.2 AT-SPI2 adapter (Linux/GNOME)
  - D-Bus registration
  - Role mappings: complete table
  - State mappings: focused, checked, expanded, etc.
  - Event emission: focus change, value change, state change
  - Text interface: for text widgets
  - Action interface: for interactive widgets
- 7.7.3 NSAccessibility adapter (macOS)
  - NSAccessibilityElement protocol
  - Role and attribute mappings
  - Action methods
  - `accessibilityActivate`, `accessibilityPress`
- 7.7.4 UIA adapter (Windows)
  - IUIAutomationProvider implementation
  - Control type mappings
  - Pattern support: InvokePattern, TogglePattern, ValuePattern
- 7.7.5 Testing accessibility: tools and techniques

### Chapter 7.8: lib_ccsp_style -- Theming
- 7.8.1 `ccsp_style_theme_t`: complete field documentation
- 7.8.2 Typography tokens: all fields and default values
- 7.8.3 Color tokens: semantic meaning of each
- 7.8.4 Geometry tokens: spacing system
- 7.8.5 Motion tokens: timing and curves
- 7.8.6 `ccsp_style_theme_dark()`: how colors are inverted
- 7.8.7 `ccsp_style_theme_high_contrast()`: WCAG AAA targets
- 7.8.8 Custom themes: creating from scratch
- 7.8.9 Theme inheritance: modify specific tokens
- 7.8.10 Runtime theme switching: what repaints, what does not

### Chapter 7.9: lib_ccsp_app -- Application Lifecycle
- 7.9.1 `ccsp_app_t` and `ccsp_app_create()`
- 7.9.2 `ccsp_app_run()` and `ccsp_app_quit()`
- 7.9.3 Window management: complete API
- 7.9.4 System menu construction: declarative description
- 7.9.5 Platform integration: dock, taskbar, notification
- 7.9.6 File pickers: native dialogs
- 7.9.7 System notifications
- 7.9.8 Multi-window applications

---

## Volume 8: Tooling

### Chapter 8.1: lib_ccsp_test -- Testing Infrastructure
- 8.1.1 Test runner: `ccsp_test_suite_t`, `ccsp_test_case_t`
- 8.1.2 Assertion macros: complete list with messages
- 8.1.3 Setup and teardown
- 8.1.4 Timeout per test case
- 8.1.5 TAP output format
- 8.1.6 JUnit XML output for CI
- 8.1.7 Mock allocator: all three variants
- 8.1.8 Capture log backend: assertion API
- 8.1.9 Mock DNS resolver: configuration
- 8.1.10 Mock TLS pair: usage
- 8.1.11 Property-based testing: `ccsp_test_prop_check()`
  - Input generators: integers, strings, bytes, structs
  - Shrinking: how counterexamples are minimized
  - Fixed seed for CI reproducibility
- 8.1.12 Known-answer test vectors: `ccsp_test_kat_*`
- 8.1.13 Visual regression: complete workflow

### Chapter 8.2: lib_ccsp_fuzz -- Fuzzing Infrastructure
- 8.2.1 Fuzz target list: all 20+ targets
- 8.2.2 Writing a new fuzz target
- 8.2.3 LibFuzzer integration
- 8.2.4 AFL integration
- 8.2.5 OSS-Fuzz integration files
- 8.2.6 Corpus management: minimize, merge, seed
- 8.2.7 CI integration: 100k iterations per PR
- 8.2.8 Crash triage: reproduction and deduplication

### Chapter 8.3: ccsp-check -- Static Analysis
- 8.3.1 Installation and invocation
- 8.3.2 CRITICAL findings: complete list with examples
- 8.3.3 HIGH findings: complete list with examples
- 8.3.4 INFO findings: complete list
- 8.3.5 Output format: finding, CVE count, fix suggestion
- 8.3.6 Suppression syntax and the suppression report
- 8.3.7 CI integration: exit codes
- 8.3.8 Adding custom rules: rule format
- 8.3.9 ccsp-corpus export: generate rules from corpus

### Chapter 8.4: The CVE Corpus
- 8.4.1 Schema: all fields documented
- 8.4.2 The 12 patterns: definitions, CVE count, mean latency
- 8.4.3 `ccsp-corpus` query tool
  - `list --pattern=<pattern>`
  - `stats`: summary
  - `search --library=<lib>`
  - `export --format=libcheck`
- 8.4.4 Contributing: submission format, review process
- 8.4.5 Classification methodology: how patterns are assigned
- 8.4.6 Dispute process: contesting a classification

---

## Volume 9: Reference

### Chapter 9.1: Complete Error Code Reference
- All `CCSP_ERR_*` codes from all libraries
- Code, domain, description, common cause, resolution

### Chapter 9.2: Complete Log Key Reference
- All standard log keys: name, type, which libraries emit, meaning

### Chapter 9.3: Symbol Namespace Reference
- Complete symbol prefix table
- Header file table
- Error code prefix table
- Log domain string table

### Chapter 9.4: ABI Stability Policy
- What is stable within major version
- What may change in minor version
- What may change in patch version
- The `ccsp-abi` checker: how to run, what it reports
- ABI dump format

### Chapter 9.5: Versioning Policy
- Semantic versioning: major.minor.patch
- Release branches
- Security patch backport policy
- EOL policy

### Chapter 9.6: Platform Support Matrix
- Tier 1: full support, CI coverage, release-blocking failures
- Tier 2: best effort, CI coverage, non-blocking failures
- Tier 3: community maintained, no CI
- Platform table: OS, architecture, compiler, tier

### Chapter 9.7: Shim Status and Deletion Conditions
- All active shims: library, file, deletion condition, tracker link
- Shim review schedule
- Deletion procedure checklist

### Chapter 9.8: Migration Guide from OpenSSL
- API mapping: OpenSSL function -> lib_csp equivalents
- Conceptual differences: error model, buffer model, BIO model
- Certificate handling differences
- TLS configuration differences
- Common pitfalls

### Chapter 9.9: Migration Guide from GLib
- Type mappings: GList -> ccsp_list, GHashTable -> ccsp_hash
- Why GObject is not needed: vtables and function pointers
- GMainLoop -> lib_ccsp_event
- GIOChannel -> lib_ccsp_bio
- GError -> ccsp_err_t
- Common pitfalls

### Chapter 9.10: Migration Guide from libevent 1.x
- API mapping: event_base -> ccsp_ev_base_t
- Bufferevent -> lib_ccsp_bio + lib_ccsp_event
- evhttp -> lib_ccsp_http
- evdns -> lib_ccsp_dns
- Changes in callback signatures

### Chapter 9.11: Security Disclosure Policy
- Reporting process
- Embargo period
- CVE assignment
- Notification list (project members: 48-hour advance notice)
- Public disclosure format

### Chapter 9.12: Contributing
- Code style
- Test requirements: what every PR must include
- Documentation requirements
- ccsp-check compliance: no CRITICAL findings
- ABI compatibility: what requires a major version bump
- Commit message format
- Review process

---

## Appendix A: The CVE Corpus Patterns

Full documentation for all 12 patterns:

- A.1: `hand_rolled_line_reader` -- 47 CVEs, 5.2yr mean latency
- A.2: `caller_supplied_length_trusted` -- 31 CVEs, 4.8yr mean latency
- A.3: `gethostbyname_nonreentrant` -- 28 CVEs, 6.1yr mean latency
- A.4: `integer_overflow_before_alloc` -- 24 CVEs, 4.3yr mean latency
- A.5: `so_error_miss` -- 19 CVEs, 3.9yr mean latency
- A.6: `select_fd_setsize_unchecked` -- 17 CVEs, 7.2yr mean latency
- A.7: `iv_nonce_reuse` -- 15 CVEs, 5.5yr mean latency
- A.8: `sprintf_no_bounds` -- 22 CVEs, 5.8yr mean latency
- A.9: `asn1_any_parser_confusion` -- 12 CVEs, 8.3yr mean latency
- A.10: `implicit_integer_size` -- 34 CVEs, 6.7yr mean latency
- A.11: `non_constant_time_secret_compare` -- 11 CVEs, 4.1yr mean latency
- A.12: `nonreentrant_global_crypto` -- 9 CVEs, 5.0yr mean latency

Each entry includes: definition, canonical vulnerable code, canonical
fixed code, CVE list, mean latency, structural prevention mechanism,
and the ccsp-check rule that catches it.

---

## Appendix B: Design Decisions Not Taken

- B.1: Why not GLib's GObject -- vtables are sufficient
- B.2: Why not Qt's moc -- code generation in the build is wrong
- B.3: Why not a custom allocator by default
- B.4: Why not reference counting in lib_ccsp_base
- B.5: Why not CSS for styling -- specificity is a design failure
- B.6: Why not a scripting language integration
- B.7: Why not YAML -- TOML is sufficient and simpler to parse
- B.8: Why not XML -- TOML and JSON cover all use cases
- B.9: Why not a plugin/module system -- dlopen is sufficient
- B.10: Why not libuv -- and how this stack differs
- B.11: Why not write this in Rust
- B.12: Why not write this in C++
- B.13: Why not OpenSSL as the crypto backend
- B.14: Why not build on top of existing libraries instead

---

## Appendix C: Standards Positions

- C.1: IETF: NO to ANY in ASN.1 protocol specs -- the argument
- C.2: IETF: UTF-8 in all new protocol string fields -- the argument
- C.3: IETF: SRV records as the service discovery standard
- C.4: IETF: DANE as the CA reform path -- the timeline
- C.5: POSIX: clock_gettime, getrandom, sigaction proposals
- C.6: C committee: stdint, stdbool, snprintf, designated initializers,
  inline, restrict -- contribution history
- C.7: LSB: behavioral spec, not implementation mandate -- the argument
- C.8: The IPR disclosure record -- running log of WG interventions

---

## Appendix D: Changelog

- D.1: 0.1 -- initial design document
- D.2: 0.2 -- lib_ccsp_ namespace, expanded TOC
- D.3: Planned 1.0 -- lib_ccsp_base through lib_ccsp_tls complete
- D.4: Planned 1.1 -- lib_ccsp_http through lib_ccsp_imap complete
- D.5: Planned 1.2 -- lib_ccsp_json through lib_ccsp_cli complete
- D.6: Planned 2.0 -- full graphics stack complete
- D.7: Planned 2.1 -- FIPS 140-3 validation complete

---

# Part VII: Roadmap and Shim Status

## Shim Deletion Schedule

```tbl
.TS
center box;
l l l n
l l l n.
Shim	File	Deletion Condition	Year
_
Integer types	ccsp_types.h	C99 on all min targets	1999
// comments	build config	C99 on all min targets	1999
stdbool.h	ccsp_types.h	C99 on all min targets	1999
snprintf semantics	ccsp_buf.c	C99 on all min targets	1999
inline keyword	build config	C99 on all min targets	1999
restrict keyword	build config	C99 on all min targets	1999
Designated initializers	build config	C99 on all min targets	1999
sigaction shim	ccsp_signal.c	POSIX on all targets	1996
poll shim	lib_ccsp_event	poll on all targets	1997
Atomic operations	ccsp_atomic.h	C11 on all targets	2011
clock_gettime	ccsp_time.c	POSIX 2001 on all targets	2005
getrandom	ccsp_random.c	stdc_secure_random() std	2025
TLI socket shim	ccsp_sock_tli.c	Solaris drops TLI	2000
Once-init	ccsp_once.h	C11 on all targets	2011
.TE
```

## Delivery Milestones

**Milestone 1.0: Core Networking and Crypto Stack**
Libraries: lib_ccsp_base, lib_ccsp_event, lib_ccsp_bio, lib_ccsp_dns,
lib_ccsp_connect, lib_ccsp_asn1, lib_ccsp_cert, lib_ccsp_mpi,
lib_ccsp_ocl, lib_ccsp_tls, lib_ccsp_cli, lib_ccsp_test

Criteria: All libraries pass conformance suite on Tier 1 platforms.
ccsp-check passes on all library source. FIPS 140-1 validation initiated.
No CRITICAL ccsp-check findings in any library source.

**Milestone 1.1: Protocol Stack**
Libraries: lib_ccsp_lineproto, lib_ccsp_http, lib_ccsp_smtp,
lib_ccsp_imap, lib_ccsp_mdns, lib_ccsp_pki

Criteria: RFC 5321 SMTP conformance. HTTP/1.1 and HTTP/2 pass
relevant test suites. DANE enforcement validated.

**Milestone 1.2: Data Stack and Tooling**
Libraries: lib_ccsp_data, lib_ccsp_toml, lib_ccsp_json, lib_ccsp_cas,
lib_ccsp_charconv, lib_ccsp_regex, lib_ccsp_fuzz

Tools: ccsp-check v1.0, CVE corpus v1.0 (all 12 patterns, 200+ entries)

Criteria: JSONTestSuite pass. toml-test pass. ccsp-check identifies
all 12 patterns. libfuzzer and AFL integration complete.

**Milestone 2.0: Graphics Stack**
Libraries: lib_ccsp_draw (all backends), lib_ccsp_font, lib_ccsp_text,
lib_ccsp_widget, lib_ccsp_anim, lib_ccsp_input, lib_ccsp_a11y,
lib_ccsp_style, lib_ccsp_app

Criteria: Flutter layout model test vectors pass. Visual regression
suite passes. AT-SPI2, NSAccessibility, UIA adapters validated.
Imagemap backend produces pixel-identical output to X11 backend on
reference widget tree.

**Milestone 2.1: FIPS Validation**
Scope: lib_ccsp_ocl and lib_ccsp_tls

Criteria: CMVP FIPS 140-3 certificate issued for lib_ccsp_ocl.

**Milestone 3.0: Ecosystem**
Goals: GNU coreutils DNS and connect tools migrated. curl backend
contributed upstream. OpenSSH backend contributed upstream. LSB
behavioral spec published with lib_ccsp_base as reference
implementation. C committee TR published for Annex S proposal.

---

*The CCSP Stack is maintained by the CCSP Project.*

*Source:* `ftp://ftp.ccsys.org/pub/ccsp/`
*Bugs:* `ccsp-bugs@ccsys.org`
*Usenet:* `comp.lang.c.ccsp`
*UUCP:* `...!uunet!ccsys!ccsp`
*Tape:* Send formatted 9-track 6250bpi or QIC-150 cartridge to CCSP Project, Berkeley CA
