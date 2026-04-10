# CCSP — Correct C Systems Programming

A C17 library stack for building secure, auditable systems software.

This should have existed in 1987. We are fixing that now.

Yes, we called it "correct." Forty years of buffer overflows, use-after-frees, and billion-dollar CVEs have earned the word.

## Status

Design phase. No implementation yet.

## What This Is

Every serious C program needs the same things: integer types that don't lie about their size, strings that aren't null-terminated traps, an allocator you can replace for testing, data structures that don't malloc per node, a monotonic clock that doesn't jump, secure random bytes, structured logging, and an error type that tells you where things went wrong.

Instead, everyone either pulls in GLib (and gets an object system, an event loop, and a bookmark file parser they didn't ask for), or rolls their own (and gets CVEs they didn't ask for either).

CCSP is the library stack that should have shipped with ANSI C:

- **lib_ccsp_base** — types, strings, memory, errors, UTF-8, data structures, time, random, atomics, logging
- **lib_ccsp_event** — event loop (epoll, kqueue, io_uring)
- **lib_ccsp_bio** — buffered I/O
- **lib_ccsp_connect** — TCP/TLS connection establishment
- **lib_ccsp_dns** — async DNS with DNSSEC, DoH, DoT
- **lib_ccsp_tls** — TLS 1.3
- **lib_ccsp_http** — HTTP/1.1, HTTP/2
- **lib_ccsp_dgram** — UDP servers with rate limiting and amplification protection
- And more (see design docs)

## Why

87% of CVEs in C systems software fall into 12 patterns. CCSP prevents them by design:

- No direct malloc/free — all allocation through `ccsp_alloc.h`
- Explicit error handling with location tracking — `ccsp_err_t`
- UTF-8 only, validated at boundaries
- No hidden global state
- Static analysis integration via `ccsp-check`

## Design Documents

| Document | Contents |
|----------|----------|
| [ccsp-design.md](ccsp-design.md) | Master design — all libraries, rationale, API guide |
| [design-base.md](design-base.md) | lib_ccsp_base detailed design |

## License

- Documentation (*.md): CC-BY-SA-4.0
- Code: MIT

## Links

- Source: https://github.com/ccsp-project
- Issues: https://github.com/ccsp-project/ccsp/issues
- Book: https://ccsys.org/book
