# AGENTS.md

## Project Overview

MarketDataFeedHandler is a C++ market-data infrastructure project implementing a handler for the ITCH market-data protocol.

The core pipeline is intended to evolve toward:

```text
Feed Source
    ↓
Raw Bytes
    ↓
ITCH Protocol Decoder
    ↓
Normalized Market Events
    ↓
Order Book
    ↓
Consumers / API / Visualization
```

The project may eventually support multiple feed sources, including normal kernel networking, replayed data, and potentially DPDK/kernel-bypass implementations.

The primary goals are:

* Correctly decode ITCH messages.
* Reliably reconstruct market state.
* Provide deterministic replay and testing.
* Maintain clean separation between protocol, transport, and market-state logic.
* Provide a foundation for performance experimentation.
* Eventually expose market data to visualization and other downstream consumers.

---

## Repository Structure

```text
.
├── BUILD.bazel
├── MODULE.bazel
├── README.md
├── AGENTS.md
│
├── src/
│   ├── BUILD.bazel
│   │
│   ├── feed/
│   │   └── BUILD.bazel
│   │
│   ├── orderbook/
│   │   ├── BUILD.bazel
│   │   └── OrderBook.H
│   │
│   ├── protocol/
│   │   ├── BUILD.bazel
│   │   ├── Decoder.C
│   │   ├── Decoder.H
│   │   ├── Messages.H
│   │   └── Types.H
│   │
│   └── replay/
│       └── BUILD.bazel
│
└── tests/
    └── BUILD.bazel
```

### `src/protocol/`

Contains ITCH-specific protocol definitions and binary decoding.

Responsibilities include:

* Message definitions.
* Protocol field types.
* Binary parsing.
* Endianness handling.
* Timestamp decoding.
* Price encoding/decoding.
* Message dispatch.
* Protocol validation.

Protocol-specific knowledge should remain isolated to this package whenever practical.

### `src/feed/`

Contains mechanisms for obtaining raw market-data bytes.

Potential implementations include:

* Linux sockets.
* UDP/TCP feed sources.
* DPDK.
* AF_XDP.
* Mock/test sources.

Feed implementations should not contain ITCH message-decoding logic.

### `src/orderbook/`

Contains market-state reconstruction.

Responsibilities include:

* Tracking individual orders.
* Maintaining price levels.
* Processing order lifecycle events.
* Maintaining bid/ask state.
* Providing best bid/offer.
* Generating snapshots.
* Validating book consistency.

The order book should not depend directly on ITCH binary layouts.

Prefer domain-level events or interfaces over coupling the order book to `protocol/Messages.H`.

### `src/replay/`

Contains historical feed replay functionality.

Replay should use the same downstream processing pipeline as live feeds whenever practical.

Potential functionality:

* File-based replay.
* Original-timestamp replay.
* Accelerated replay.
* Deterministic replay.
* Symbol filtering.
* Message filtering.
* Timestamp seeking.

### `tests/`

Contains unit, integration, regression, performance, and protocol validation tests.

Tests should be organized around behavior rather than implementation details whenever practical.

---

# Architecture Principles

## 1. Keep protocol and domain logic separate

The order book should not need to understand ITCH byte offsets, message headers, or binary encoding.

Prefer:

```text
ITCH bytes
    ↓
ITCH message
    ↓
domain event
    ↓
order book
```

rather than:

```text
ITCH bytes
    ↓
order book directly
```

This makes the system easier to test and allows other market-data protocols to be added later.

---

## 2. Keep feed sources interchangeable

The downstream application should not care whether data came from:

```text
Socket
DPDK
AF_XDP
File
Replay
Mock
```

Prefer a common feed/source abstraction.

Conceptually:

```text
FeedSource
    ├── SocketFeed
    ├── DpdkFeed
    ├── FileFeed
    └── MockFeed
```

All sources should ultimately provide data through a common interface to the protocol layer.

---

## 3. Do not prematurely optimize

Correctness comes before performance.

The expected progression is:

```text
Correctness
    ↓
Tests
    ↓
Benchmark
    ↓
Profile
    ↓
Optimize
    ↓
Benchmark again
```

Do not introduce DPDK, lock-free structures, custom allocators, or complicated concurrency solely because they appear appropriate for high-frequency trading.

Performance improvements should be supported by measurements.

---

## 4. Preserve deterministic behavior

Where practical, the same input feed should produce the same sequence of decoded events and order-book states.

This is particularly important for:

* Replay.
* Regression tests.
* Debugging.
* Performance comparisons.
* Visualizer demonstrations.

Avoid introducing nondeterminism into the core processing pipeline without a clear reason.

---

# C++ Guidelines

## General

* Prefer modern C++ practices.
* Use RAII.
* Prefer value semantics where practical.
* Avoid unnecessary raw owning pointers.
* Use `const` aggressively.
* Prefer explicit types for protocol fields.
* Avoid implicit narrowing conversions.
* Keep interfaces small.
* Avoid global mutable state.

## Headers

Headers should contain declarations and types unless there is a specific reason for header-only implementation.

Prefer:

```text
Decoder.H
Decoder.C
```

over putting substantial implementation into `Decoder.H`.

Use include guards or the project's chosen equivalent consistently.

Avoid unnecessary includes in headers.

Prefer forward declarations where appropriate.

---

# Protocol Implementation Guidelines

ITCH is a binary protocol. Correctness at the byte level is critical.

When implementing a message:

1. Consult the authoritative protocol specification.
2. Confirm every field's size.
3. Confirm field ordering.
4. Confirm endianness.
5. Confirm timestamp representation.
6. Confirm price representation.
7. Confirm message length.
8. Add a representative byte-level test.
9. Add malformed-input tests where relevant.

Do not infer protocol layouts solely from existing code or examples.

Each message implementation should ideally have a corresponding test fixture.

For example:

```text
Add Order
    ↓
raw bytes
    ↓
decoder
    ↓
expected AddOrder structure
```

---

# Order Book Guidelines

The order book is a state machine.

Each incoming event should produce a well-defined state transition.

For example:

```text
OrderAdded
    → add order

OrderExecuted
    → reduce remaining quantity

OrderCancelled
    → reduce/cancel quantity

OrderDeleted
    → remove order

OrderReplaced
    → modify/remove old order and create replacement state
```

Invalid transitions should be handled deliberately.

Do not silently ignore inconsistencies unless the behavior is explicitly documented.

The book should eventually provide validation facilities capable of detecting issues such as:

* Unknown order IDs.
* Negative remaining quantity.
* Incorrect price-level aggregation.
* Invalid replacements.
* Inconsistent aggregate quantities.

---

# Bazel Guidelines

Use Bazel as the primary build and test system.

Prefer:

```bash
bazel build //...
bazel test //...
```

over ad-hoc compiler commands.

Use `BUILD.bazel` for new BUILD files and `MODULE.bazel` for Bzlmod configuration.

Keep Bazel targets relatively modular.

For example:

```text
//src/protocol:protocol
//src/orderbook:orderbook
//src/feed:feed
//src/replay:replay
```

Prefer depending on a library target rather than directly reaching into another package's source files.

Avoid broad dependencies such as:

```python
deps = ["//visibility:public"]
```

unless genuinely necessary.

Keep dependency direction intentional.

---

# Testing

Every significant feature should include tests.

At minimum, consider:

### Unit tests

Test individual components in isolation.

Examples:

* Binary integer decoding.
* Timestamp decoding.
* Price decoding.
* Individual ITCH message decoding.
* Order addition.
* Order cancellation.
* Order execution.
* Order replacement.

### Integration tests

Test multiple components together.

Examples:

```text
raw ITCH bytes
    ↓
decoder
    ↓
event
    ↓
order book
```

### Regression tests

When a bug is discovered:

1. Add a test reproducing the bug.
2. Fix the implementation.
3. Keep the regression test permanently.

### Fuzz testing

The decoder should eventually be fuzz-tested against malformed and arbitrary byte sequences.

The decoder must not crash or invoke undefined behavior on malformed input.

---

# Performance

Performance is an important project concern, but measurements should drive optimization.

Potential metrics include:

* Messages per second.
* Bytes per second.
* Decode latency.
* Order-book update latency.
* End-to-end latency.
* p50 latency.
* p95 latency.
* p99 latency.
* p99.9 latency.
* CPU utilization.
* Memory usage.
* Allocations per message.
* Packet drops.

When comparing networking approaches, benchmark equivalent workloads.

Potential implementations:

```text
Linux sockets
AF_XDP
DPDK
```

should feed the same decoder and order-book implementation where practical.

This allows meaningful comparisons.

---

# Concurrency

Do not introduce multithreading merely because the system processes market data.

Start with a simple, deterministic processing model.

If concurrency is introduced:

* Clearly document ownership.
* Clearly document synchronization.
* Avoid shared mutable state where possible.
* Benchmark synchronization overhead.
* Test for races.
* Keep the order-book state machine easy to reason about.

A single-threaded hot path is a valid and potentially desirable design.

---

# Error Handling

Errors should be explicit and observable.

Distinguish between:

* Malformed protocol data.
* Network errors.
* Sequence gaps.
* Unknown order IDs.
* Invalid book state.
* Configuration errors.
* Internal programming errors.

Do not silently discard data without recording why it was discarded.

For market-data infrastructure, observability is part of correctness.

---

# Logging and Observability

Eventually expose useful operational information such as:

```text
messages received
messages decoded
messages rejected
sequence gaps
bytes received
orders currently tracked
symbols tracked
trades processed
book inconsistencies
```

Avoid excessive logging in the hot path.

Prefer counters/metrics for high-frequency events.

---

# GitHub Issues

GitHub issues should generally represent one independently implementable piece of work.

Good issue:

```text
Implement ITCH Add Order message decoder
```

Less useful issue:

```text
Implement protocol
```

Protocol message issues should include:

* Protocol reference.
* Expected fields.
* Acceptance criteria.
* Tests required.
* Dependencies, if any.

Useful labels include:

```text
area:protocol
area:feed
area:orderbook
area:replay
area:testing
area:performance
area:web

type:feature
type:bug
type:test
type:refactor
type:research

difficulty:1
difficulty:2
difficulty:3
difficulty:4

protocol:message
good-first-issue
stretch
```

---

# Pull Requests

PRs should be focused.

Prefer:

```text
Implement Add Order decoder
```

over:

```text
Implement half of ITCH
```

A PR should generally:

* Explain what changed.
* Explain why it changed.
* Include tests.
* Avoid unrelated refactoring.
* Keep Bazel targets/builds passing.

Before submitting a PR:

```bash
bazel build //...
bazel test //...
```

should pass.

---

# Documentation

Significant architectural decisions should be documented.

Useful documents will eventually include:

```text
docs/
├── architecture.md
├── protocol.md
├── message-flow.md
├── orderbook.md
├── replay.md
└── performance.md
```

Document the reasoning behind important architectural choices, not just what the code does.

---

# Scope Control

The project should prioritize the following order:

1. ITCH decoding correctness.
2. Feed ingestion.
3. Order-book reconstruction.
4. Deterministic replay.
5. Testing and validation.
6. Performance measurement.
7. Web visualization.
8. Advanced networking / kernel bypass.
9. Additional protocols and experimental features.

Potential extensions include:

* DPDK.
* AF_XDP.
* PCAP ingestion.
* Multiple symbols.
* Market statistics.
* Historical book reconstruction.
* WebSocket streaming.
* Interactive order-book visualization.
* Alternative market-data protocols.
* Hardware/network performance experiments.

Extensions should not compromise the correctness or maintainability of the core system.

