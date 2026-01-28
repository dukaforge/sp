# PRD: Transport Abstraction Layer

**Issue**: sp-ms6.1
**Status**: Draft
**Author**: Claude
**Date**: 2026-01-27

## Overview

The Transport Abstraction Layer provides a unified interface for message-oriented communication across different underlying transports. SP supports two transports:

1. **Unix Domain Sockets (unixgram)** - Local IPC with datagram semantics
2. **Raw IP Sockets** - Network communication without TCP overhead

This PRD defines the transport interface, implementation requirements, and testing strategy.

## Requirements

### Functional Requirements

| ID | Requirement |
|----|-------------|
| TR-1 | Transport interface abstracts dial, listen, send, and recv operations |
| TR-2 | Both transports preserve message boundaries (no fragmentation/reassembly) |
| TR-3 | Transport selection happens at socket creation time |
| TR-4 | Transports expose consistent addressing schemes |
| TR-5 | Close operations release system resources deterministically |
| TR-6 | Transports report errors using consistent error types |

### Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NF-1 | Unix socket latency: < 10μs for local message round-trip |
| NF-2 | Memory allocation: zero-alloc steady-state for send/recv |
| NF-3 | Thread safety: all transport operations are goroutine-safe |
| NF-4 | Resource cleanup: no leaked file descriptors on Close() |

## Design

### Transport Interface

```go
// Transport represents a message-oriented communication channel.
type Transport interface {
    // Send transmits a message to the connected peer or specified address.
    // Returns the number of bytes sent or an error.
    // For connected transports, addr is ignored.
    // For unconnected transports, addr specifies the destination.
    Send(data []byte, addr Addr) (int, error)

    // Recv receives a message from the transport.
    // Returns the message data, sender address, and any error.
    // The returned byte slice is valid until the next Recv call.
    Recv() ([]byte, Addr, error)

    // Close releases all resources associated with the transport.
    // After Close, all operations return ErrClosed.
    Close() error

    // LocalAddr returns the local address of the transport.
    LocalAddr() Addr

    // SetDeadline sets read and write deadlines.
    SetDeadline(t time.Time) error
    SetReadDeadline(t time.Time) error
    SetWriteDeadline(t time.Time) error
}

// Listener accepts incoming connections or messages.
type Listener interface {
    // Accept waits for and returns the next connection/message source.
    // For datagram transports, this may return immediately with a
    // transport bound to the listener's address.
    Accept() (Transport, error)

    // Close stops listening and releases resources.
    Close() error

    // Addr returns the listener's address.
    Addr() Addr
}

// Dialer creates outbound connections.
type Dialer interface {
    // Dial connects to the specified address and returns a Transport.
    Dial(addr Addr) (Transport, error)
}

// Addr represents a transport address.
type Addr interface {
    // Network returns the transport type: "unixgram" or "ip".
    Network() string

    // String returns the address in string form.
    String() string
}
```

### Address Formats

| Transport | Format | Examples |
|-----------|--------|----------|
| Unix | `unix://<path>` | `unix:///tmp/sp.sock`, `unix:///var/run/agent.sock` |
| Raw IP | `ip://<host>:<port>` | `ip://192.168.1.1:5555`, `ip://[::1]:5555` |

### Unix Domain Socket Implementation

```go
// UnixTransport implements Transport over Unix datagram sockets.
type UnixTransport struct {
    conn     *net.UnixConn
    addr     *net.UnixAddr
    recvBuf  []byte        // Preallocated receive buffer
    closed   atomic.Bool
    mu       sync.Mutex    // Protects concurrent access
}

// UnixListener implements Listener for Unix sockets.
type UnixListener struct {
    conn   *net.UnixConn
    addr   *net.UnixAddr
    closed atomic.Bool
}
```

**Key Design Decisions**:

1. **Datagram mode (`unixgram`)**: Preserves message boundaries automatically
2. **Preallocated buffers**: `recvBuf` is sized to max message size (64KB default)
3. **Abstract namespace support** (Linux): Addresses starting with `@` use abstract namespace
4. **File cleanup**: Listener removes socket file on Close() (non-abstract only)

### Raw IP Socket Implementation

```go
// IPTransport implements Transport over raw IP sockets.
type IPTransport struct {
    conn     *net.UDPConn  // UDP for message boundaries
    addr     *net.UDPAddr
    recvBuf  []byte
    closed   atomic.Bool
    mu       sync.Mutex
}

// IPListener implements Listener for IP sockets.
type IPListener struct {
    conn   *net.UDPConn
    addr   *net.UDPAddr
    closed atomic.Bool
}
```

**Key Design Decisions**:

1. **UDP over raw IP**: Simpler, provides message boundaries, widely supported
2. **No TCP**: Avoids connection state, head-of-line blocking, Nagle delays
3. **IPv4 and IPv6**: Both supported via Go's `net.UDPConn`
4. **Port reuse**: `SO_REUSEADDR` enabled for quick restart

### Message Boundary Preservation

Both transports guarantee message boundaries:

| Transport | Mechanism | Max Message Size |
|-----------|-----------|------------------|
| Unix (unixgram) | Datagram semantics | 64KB (configurable) |
| Raw IP (UDP) | Datagram semantics | 65507 bytes (UDP limit) |

**No framing required**: Unlike TCP streams, these transports deliver complete messages or fail. The library does not implement length-prefix framing.

### Error Handling

```go
var (
    // ErrClosed indicates the transport has been closed.
    ErrClosed = errors.New("transport: closed")

    // ErrTimeout indicates a deadline was exceeded.
    ErrTimeout = errors.New("transport: timeout")

    // ErrMessageTooLarge indicates the message exceeds transport limits.
    ErrMessageTooLarge = errors.New("transport: message too large")

    // ErrAddrInUse indicates the address is already bound.
    ErrAddrInUse = errors.New("transport: address in use")

    // ErrConnRefused indicates the peer is not listening.
    ErrConnRefused = errors.New("transport: connection refused")
)
```

**Error mapping**: Platform-specific errors (EAGAIN, ECONNREFUSED, etc.) are mapped to these transport-level errors.

### Backpressure

Transports implement backpressure via blocking:

1. **Send blocks** when the kernel send buffer is full
2. **Recv blocks** when no messages are available
3. **Deadlines** prevent indefinite blocking

The protocol layer uses these blocking semantics for natural flow control. No explicit backpressure signals are needed at the transport level.

### Configuration

```go
// TransportConfig holds transport configuration options.
type TransportConfig struct {
    // MaxMessageSize is the maximum message size in bytes.
    // Default: 65536 (64KB)
    MaxMessageSize int

    // SendBufferSize is the kernel send buffer size.
    // Default: 0 (use system default)
    SendBufferSize int

    // RecvBufferSize is the kernel receive buffer size.
    // Default: 0 (use system default)
    RecvBufferSize int

    // ReuseAddr enables SO_REUSEADDR for quick restart.
    // Default: true
    ReuseAddr bool
}
```

## Testing Strategy

### Unit Tests

| Test | Description |
|------|-------------|
| `TestUnixSendRecv` | Basic send/receive on Unix socket |
| `TestUnixMessageBoundary` | Multiple messages maintain boundaries |
| `TestUnixConcurrent` | Concurrent send/recv from multiple goroutines |
| `TestUnixClose` | Close releases resources, subsequent ops fail |
| `TestUnixDeadline` | Deadline timeout returns ErrTimeout |
| `TestIPSendRecv` | Basic send/receive on IP socket |
| `TestIPMessageBoundary` | Multiple messages maintain boundaries |
| `TestIPConcurrent` | Concurrent operations |
| `TestIPClose` | Resource cleanup |
| `TestIPDeadline` | Timeout handling |

### Integration Tests

| Test | Description |
|------|-------------|
| `TestUnixClientServer` | Full client-server exchange via Unix |
| `TestIPClientServer` | Full client-server exchange via IP |
| `TestTransportSwitch` | Same protocol code works with both transports |

### Benchmarks

| Benchmark | Target |
|-----------|--------|
| `BenchmarkUnixLatency` | < 10μs round-trip |
| `BenchmarkUnixThroughput` | > 100K msg/sec (1KB messages) |
| `BenchmarkIPLatency` | < 100μs round-trip (localhost) |
| `BenchmarkIPThroughput` | > 50K msg/sec (1KB messages) |
| `BenchmarkZeroAlloc` | 0 allocs in steady-state send/recv |

## Acceptance Criteria

1. **Interface Complete**: Transport, Listener, Dialer, Addr interfaces defined
2. **Unix Implementation**: Full implementation passing all unit tests
3. **IP Implementation**: Full implementation passing all unit tests
4. **Error Handling**: All error conditions mapped to transport errors
5. **Benchmarks Pass**: Meet latency and throughput targets
6. **Zero-Alloc**: No allocations in steady-state operations
7. **Documentation**: GoDoc comments on all exported types/methods

## Open Questions

| Question | Status | Resolution |
|----------|--------|------------|
| Should we support TCP as a third transport? | Deferred | Focus on datagram transports first; TCP adds complexity |
| Abstract namespace default for Linux Unix sockets? | Open | Cleaner (no file cleanup) but less portable |
| Should max message size be runtime-configurable? | Resolved | Yes, via TransportConfig |

## Dependencies

- Go standard library: `net`, `syscall`
- No external dependencies

## References

- [Go net package](https://pkg.go.dev/net)
- [Unix domain sockets](https://man7.org/linux/man-pages/man7/unix.7.html)
- [UDP sockets](https://man7.org/linux/man-pages/man7/udp.7.html)
- SP ARCHITECTURE.md - Transport Layer section
