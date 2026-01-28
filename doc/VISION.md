# SP Vision: Scalability Protocols for Go

## The Problem

Agent coordination systems in network management require a high-performance messaging substrate that can:
- Seamlessly switch between local inter-process communication (tight coupling, shared memory speeds) and distributed network communication
- Minimize latency overhead—both from protocol complexity and transport mechanisms (TCP adds unnecessary overhead for network messages)
- Enable clear separation between transport concerns and protocol logic across multiple messaging patterns
- Provide Go-idiomatic abstractions that leverage goroutines and channels rather than forcing traditional callback models

Existing solutions either couple transport and protocol tightly, impose significant overhead for local communication, or provide non-idiomatic APIs for Go environments.

## What This Does

SP is a pure Go implementation of the Scalability Protocols specification that acts as a high-performance messaging substrate. It provides:

- **Transport abstraction**: Same API whether you're using Unix domain sockets for local IPC or raw IP sockets for inter-host communication
- **Wire-protocol compatibility**: Can interoperate with other Scalability Protocol implementations while maintaining our own transport optimizations
- **Multiple messaging patterns**: REQ/REP, PUB/SUB, PIPELINE, SURVEY, BUS, and PAIR patterns for different coordination scenarios
- **Clean layered architecture**: I/O workers handle syscalls, protocol engines handle semantics, application code stays isolated from both
- **Go-native API**: Goroutines, channels, and contexts—not callbacks or event loops

## What Success Looks Like

1. **Performance**: Local communication via Unix sockets matches shared-memory communication speeds; raw IP communication dramatically reduces overhead vs. TCP alternatives
2. **Correctness**: Wire-protocol compatibility verified across test suites; all SP patterns implement their state machines correctly
3. **Usability**: Synchronous blocking API works reliably; asynchronous Go-idiomatic API provides natural integration with channels and select statements
4. **Reliability**: Clear separation of concerns makes goroutine leaks, deadlocks, and race conditions detectable and preventable
5. **Adoption**: AEON agent coordination systems can use SP as their messaging foundation, enabling both tight local orchestration and distributed agent networks

## What This Is NOT

- **Not a drop-in NNG replacement**: We're wire-protocol compatible with the Scalability Protocol specification, not API-compatible with the NNG library
- **Not a general-purpose RPC framework**: SP focuses on patterns and reliability, not request/response with automatic serialization and service discovery
- **Not a queue broker**: Messages don't persist; this is synchronous point-to-point and pub/sub coordination
- **Not a replacement for gRPC or HTTP**: Those tools excel for service-to-service communication; SP is optimized for agent-to-agent messaging patterns
- **Not async-first**: We start with straightforward synchronous semantics, then add Go-idiomatic async on top
