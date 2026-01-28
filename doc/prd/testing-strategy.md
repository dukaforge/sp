# PRD: Testing Strategy and Quality Assurance

**Issue**: sp-ms6.5
**Status**: Draft
**Author**: Claude
**Date**: 2026-01-27

## Overview

This document defines the comprehensive testing strategy for SP. It covers unit tests for individual components, integration tests for end-to-end flows, benchmarks for performance validation, and quality gates for continuous integration.

## Requirements

### Functional Requirements

| ID | Requirement |
|----|-------------|
| TS-1 | Unit tests cover all exported functions and types |
| TS-2 | Integration tests verify end-to-end message flows |
| TS-3 | Benchmarks measure latency and throughput |
| TS-4 | Race detector validates concurrency correctness |
| TS-5 | Tests cover error conditions and edge cases |
| TS-6 | No goroutine leaks in any test scenario |

### Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NF-1 | Unit tests run in < 30 seconds |
| NF-2 | Integration tests run in < 2 minutes |
| NF-3 | All tests pass with `-race` flag |
| NF-4 | Code coverage > 80% for core packages |
| NF-5 | Benchmarks are reproducible (< 10% variance) |

## Test Organization

### Directory Structure

```
sp/
├── transport/
│   ├── unix.go
│   ├── unix_test.go        # Unit tests
│   ├── ip.go
│   └── ip_test.go
├── internal/
│   ├── pool/
│   │   ├── buffer.go
│   │   └── buffer_test.go
│   ├── io/
│   │   ├── worker.go
│   │   └── worker_test.go
│   └── protocol/
│       ├── reqrep.go
│       └── reqrep_test.go
├── socket.go
├── socket_test.go
└── test/
    ├── integration_test.go  # End-to-end tests
    ├── bench_test.go        # Benchmarks
    └── testutil/            # Test helpers
        ├── transport.go     # Mock transports
        └── helpers.go       # Common utilities
```

### Test Naming Conventions

| Pattern | Description | Example |
|---------|-------------|---------|
| `Test<Component><Scenario>` | Unit test | `TestBufferPoolGet` |
| `TestIntegration<Flow>` | Integration test | `TestIntegrationReqRep` |
| `Benchmark<Operation>` | Benchmark | `BenchmarkSendRecvLatency` |
| `Example<Usage>` | Runnable example | `ExampleReqSocket` |

## Unit Test Strategy

### Transport Layer Tests

```go
// transport/unix_test.go

func TestUnixTransportSend(t *testing.T) {
    // Setup: create paired sockets
    // Action: send message
    // Assert: message received correctly
}

func TestUnixTransportRecv(t *testing.T) {
    // Setup: create paired sockets, queue message
    // Action: recv
    // Assert: message matches, address correct
}

func TestUnixTransportMessageBoundary(t *testing.T) {
    // Setup: create paired sockets
    // Action: send multiple messages rapidly
    // Assert: each message received intact, no merging
}

func TestUnixTransportClose(t *testing.T) {
    // Setup: create transport
    // Action: close
    // Assert: subsequent ops return ErrClosed
}

func TestUnixTransportDeadline(t *testing.T) {
    // Setup: create transport, set short deadline
    // Action: recv with no data available
    // Assert: returns ErrTimeout
}

func TestUnixTransportConcurrent(t *testing.T) {
    // Setup: create paired sockets
    // Action: concurrent sends from multiple goroutines
    // Assert: all messages received, no data corruption
}
```

### Shared Infrastructure Tests

```go
// internal/pool/buffer_test.go

func TestBufferPoolGetPut(t *testing.T) {
    pool := NewBufferPool(64 * 1024)

    buf := pool.Get(1024)
    assert.True(t, len(buf) >= 1024)

    pool.Put(buf)
    // Get should return pooled buffer
    buf2 := pool.Get(1024)
    assert.Same(t, buf, buf2)  // May be same underlying array
}

func TestBufferPoolConcurrent(t *testing.T) {
    pool := NewBufferPool(64 * 1024)
    var wg sync.WaitGroup

    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < 1000; j++ {
                buf := pool.Get(1024)
                pool.Put(buf)
            }
        }()
    }
    wg.Wait()
    // No panics, no races
}

func TestBufferPoolStats(t *testing.T) {
    pool := NewBufferPool(64 * 1024)

    pool.Get(1024)
    pool.Get(1024)

    stats := pool.Stats()
    assert.Equal(t, uint64(2), stats.Gets)
}
```

### I/O Worker Tests

```go
// internal/io/worker_test.go

func TestRecvWorkerDeliversMessages(t *testing.T) {
    transport := newMockTransport()
    recvCh := make(chan *Message, 16)
    pool := NewBufferPool(64 * 1024)

    worker := NewRecvWorker(transport, recvCh, pool)
    worker.Start(context.Background())

    // Inject message into mock transport
    transport.InjectRecv([]byte("hello"), mockAddr)

    // Should appear on channel
    msg := <-recvCh
    assert.Equal(t, []byte("hello"), msg.Data)

    worker.Stop()
}

func TestSendWorkerWritesMessages(t *testing.T) {
    transport := newMockTransport()
    sendCh := make(chan *Message, 16)

    worker := NewSendWorker(transport, sendCh)
    worker.Start(context.Background())

    // Send message
    sendCh <- &Message{Data: []byte("hello")}

    // Should be written to transport
    data, _ := transport.WaitSend(time.Second)
    assert.Equal(t, []byte("hello"), data)

    worker.Stop()
}

func TestWorkerPairShutdown(t *testing.T) {
    transport := newMockTransport()
    pool := NewBufferPool(64 * 1024)

    wp := NewWorkerPair(transport, pool, DefaultWorkerConfig())
    wp.Start()

    // Queue some messages
    wp.SendCh() <- &Message{Data: []byte("pending")}

    // Stop should drain
    done := make(chan struct{})
    go func() {
        wp.Stop()
        close(done)
    }()

    select {
    case <-done:
        // Good, stopped cleanly
    case <-time.After(time.Second):
        t.Fatal("worker pair did not stop in time")
    }
}
```

### Protocol Engine Tests

```go
// internal/protocol/reqrep_test.go

func TestReqStateMachine(t *testing.T) {
    tests := []struct {
        name      string
        initial   ReqState
        action    string
        wantState ReqState
        wantErr   error
    }{
        {"send from idle", ReqStateIdle, "send", ReqStateRequestSent, nil},
        {"recv from idle", ReqStateIdle, "recv", ReqStateIdle, ErrInvalidState},
        {"send while awaiting", ReqStateAwaitingReply, "send", ReqStateRequestSent, nil},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Test state transitions
        })
    }
}

func TestReqAutoResend(t *testing.T) {
    req := newTestReqSocket(WithResendTime(50 * time.Millisecond))
    rep := newTestRepSocket()
    connect(req, rep)

    // Send request
    req.Send([]byte("hello"))

    // Don't reply, wait for resend
    time.Sleep(100 * time.Millisecond)

    // Should have received request twice
    assert.Equal(t, 2, rep.RecvCount())
}

func TestRepBacktracePreserved(t *testing.T) {
    req := newTestReqSocket()
    rep := newTestRepSocket()
    connect(req, rep)

    // Send request
    go req.Send([]byte("request"))

    // Receive and reply
    rep.Recv()
    rep.Send([]byte("reply"))

    // Request should get correct reply
    reply, _ := req.Recv()
    assert.Equal(t, []byte("reply"), reply)
}
```

### Socket API Tests

```go
// socket_test.go

func TestSocketCreate(t *testing.T) {
    req, err := NewReqSocket()
    assert.NoError(t, err)
    assert.NotNil(t, req)
    req.Close()
}

func TestSocketOptions(t *testing.T) {
    req, _ := NewReqSocket(
        WithSendTimeout(5 * time.Second),
        WithRecvTimeout(10 * time.Second),
    )
    defer req.Close()

    // Verify options applied
    assert.Equal(t, 5*time.Second, req.GetSendTimeout())
    assert.Equal(t, 10*time.Second, req.GetRecvTimeout())
}

func TestSocketCloseIdempotent(t *testing.T) {
    req, _ := NewReqSocket()

    err1 := req.Close()
    err2 := req.Close()
    err3 := req.Close()

    assert.NoError(t, err1)
    assert.NoError(t, err2)
    assert.NoError(t, err3)
}

func TestSocketDialInvalidAddress(t *testing.T) {
    req, _ := NewReqSocket()
    defer req.Close()

    err := req.Dial("invalid://address")
    assert.ErrorIs(t, err, ErrInvalidAddress)
}
```

## Integration Test Strategy

### End-to-End Tests

```go
// test/integration_test.go

func TestIntegrationReqRepUnix(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    addr := "unix:///tmp/sp-test-" + randomID() + ".sock"
    defer os.Remove(addr[7:])  // Remove socket file

    // Start replier
    rep, _ := NewRepSocket()
    rep.Listen(addr)
    defer rep.Close()

    go func() {
        for {
            data, err := rep.Recv()
            if err != nil {
                return
            }
            rep.Send(append([]byte("echo:"), data...))
        }
    }()

    // Start requester
    req, _ := NewReqSocket()
    req.Dial(addr)
    defer req.Close()

    // Wait for connection
    time.Sleep(50 * time.Millisecond)

    // Exchange messages
    for i := 0; i < 100; i++ {
        msg := []byte(fmt.Sprintf("msg-%d", i))
        req.Send(msg)
        reply, _ := req.Recv()
        assert.Equal(t, append([]byte("echo:"), msg...), reply)
    }
}

func TestIntegrationReqRepIP(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    addr := "ip://127.0.0.1:" + randomPort()

    // Similar to Unix test but with IP transport
    // ...
}

func TestIntegrationMultipleClients(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    addr := "unix:///tmp/sp-multi-" + randomID() + ".sock"
    defer os.Remove(addr[7:])

    // Start replier
    rep, _ := NewRepSocket()
    rep.Listen(addr)
    defer rep.Close()

    go func() {
        for {
            data, err := rep.Recv()
            if err != nil {
                return
            }
            rep.Send(data)
        }
    }()

    // Start multiple requesters
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            req, _ := NewReqSocket()
            req.Dial(addr)
            defer req.Close()

            time.Sleep(50 * time.Millisecond)

            for j := 0; j < 10; j++ {
                msg := []byte(fmt.Sprintf("client-%d-msg-%d", id, j))
                req.Send(msg)
                reply, _ := req.Recv()
                assert.Equal(t, msg, reply)
            }
        }(i)
    }
    wg.Wait()
}

func TestIntegrationReconnect(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    addr := "unix:///tmp/sp-reconnect-" + randomID() + ".sock"

    // Start initial replier
    rep1, _ := NewRepSocket()
    rep1.Listen(addr)

    // Requester with reconnect
    req, _ := NewReqSocket(WithReconnect(100*time.Millisecond, time.Second))
    req.Dial(addr)
    time.Sleep(50 * time.Millisecond)

    // Exchange works
    go func() { rep1.Recv(); rep1.Send([]byte("1")) }()
    req.Send([]byte("ping"))
    reply1, _ := req.Recv()
    assert.Equal(t, []byte("1"), reply1)

    // Kill replier
    rep1.Close()
    os.Remove(addr[7:])

    // Start new replier
    rep2, _ := NewRepSocket()
    rep2.Listen(addr)
    defer rep2.Close()
    defer os.Remove(addr[7:])

    // Wait for reconnect
    time.Sleep(200 * time.Millisecond)

    // Exchange should work again
    go func() { rep2.Recv(); rep2.Send([]byte("2")) }()
    req.Send([]byte("ping"))
    reply2, _ := req.Recv()
    assert.Equal(t, []byte("2"), reply2)

    req.Close()
}
```

### Goroutine Leak Detection

```go
// test/integration_test.go

func TestNoGoroutineLeaks(t *testing.T) {
    initialGoroutines := runtime.NumGoroutine()

    // Create and close many sockets
    for i := 0; i < 100; i++ {
        req, _ := NewReqSocket()
        rep, _ := NewRepSocket()

        addr := "unix:///tmp/sp-leak-" + randomID() + ".sock"
        rep.Listen(addr)
        req.Dial(addr)

        time.Sleep(10 * time.Millisecond)

        req.Close()
        rep.Close()
        os.Remove(addr[7:])
    }

    // Allow goroutines to exit
    time.Sleep(100 * time.Millisecond)
    runtime.GC()

    finalGoroutines := runtime.NumGoroutine()

    // Allow small variance for runtime goroutines
    assert.InDelta(t, initialGoroutines, finalGoroutines, 5,
        "goroutine leak detected: started with %d, ended with %d",
        initialGoroutines, finalGoroutines)
}
```

## Benchmark Strategy

### Latency Benchmarks

```go
// test/bench_test.go

func BenchmarkReqRepLatency(b *testing.B) {
    addr := "unix:///tmp/sp-bench-" + randomID() + ".sock"
    defer os.Remove(addr[7:])

    rep, _ := NewRepSocket()
    rep.Listen(addr)
    defer rep.Close()

    go func() {
        for {
            data, err := rep.Recv()
            if err != nil {
                return
            }
            rep.Send(data)
        }
    }()

    req, _ := NewReqSocket()
    req.Dial(addr)
    defer req.Close()
    time.Sleep(50 * time.Millisecond)

    msg := make([]byte, 64)

    b.ResetTimer()
    b.ReportAllocs()

    for i := 0; i < b.N; i++ {
        req.Send(msg)
        req.Recv()
    }
}

func BenchmarkUnixTransportLatency(b *testing.B) {
    // Raw transport benchmark without protocol overhead
    // Target: < 10μs round-trip
}

func BenchmarkIPTransportLatency(b *testing.B) {
    // Raw IP transport benchmark
    // Target: < 100μs round-trip (localhost)
}
```

### Throughput Benchmarks

```go
func BenchmarkReqRepThroughput(b *testing.B) {
    // Measure messages per second
    // Target: > 50K req/sec
}

func BenchmarkBufferPoolThroughput(b *testing.B) {
    pool := NewBufferPool(64 * 1024)

    b.ResetTimer()
    b.ReportAllocs()

    for i := 0; i < b.N; i++ {
        buf := pool.Get(1024)
        pool.Put(buf)
    }
    // Target: < 50ns per Get/Put cycle
}
```

### Zero-Allocation Benchmarks

```go
func BenchmarkSendRecvZeroAlloc(b *testing.B) {
    // Setup connected req/rep pair
    // ...

    msg := make([]byte, 1024)

    b.ResetTimer()
    b.ReportAllocs()

    for i := 0; i < b.N; i++ {
        req.Send(msg)
        req.Recv()
    }

    // Target: 0 allocs/op in steady state
}
```

## Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Unix transport latency | < 10μs | Round-trip time |
| IP transport latency | < 100μs | Round-trip localhost |
| REQ/REP latency | < 20μs | End-to-end round-trip |
| REQ/REP throughput | > 50K msg/sec | Messages per second |
| Buffer pool Get/Put | < 50ns | Cycle time |
| Socket creation | < 1ms | New to ready |
| Socket close | < 10ms | Close to released |
| Steady-state allocs | 0 | Per send/recv |

## Test Helpers

```go
// test/testutil/transport.go

// MockTransport implements Transport for testing.
type MockTransport struct {
    recvCh chan recvResult
    sendCh chan sendCall
    closed atomic.Bool
}

func NewMockTransport() *MockTransport

func (m *MockTransport) InjectRecv(data []byte, addr Addr)
func (m *MockTransport) WaitSend(timeout time.Duration) ([]byte, Addr)
func (m *MockTransport) Send(data []byte, addr Addr) (int, error)
func (m *MockTransport) Recv() ([]byte, Addr, error)
func (m *MockTransport) Close() error

// test/testutil/helpers.go

func RandomID() string
func RandomPort() string
func WaitFor(t *testing.T, condition func() bool, timeout time.Duration)
func AssertEventually(t *testing.T, condition func() bool, msg string)
```

## CI/CD Integration

### GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - name: Run unit tests
        run: go test -v -race ./...

      - name: Run integration tests
        run: go test -v -race -tags=integration ./test/...

      - name: Run benchmarks
        run: go test -bench=. -benchmem ./test/...

      - name: Check coverage
        run: |
          go test -coverprofile=coverage.out ./...
          go tool cover -func=coverage.out | grep total
```

### Quality Gates

| Gate | Criteria |
|------|----------|
| Unit tests | All pass |
| Race detector | No races detected |
| Coverage | > 80% for core packages |
| Benchmarks | No significant regression |
| Integration tests | All pass |

## Acceptance Criteria

1. **Unit Tests Complete**: All components have comprehensive unit tests
2. **Integration Tests Complete**: End-to-end flows tested
3. **Benchmarks Defined**: Performance targets documented and measured
4. **Race Detector Clean**: All tests pass with `-race`
5. **Coverage Met**: > 80% for transport, protocol, socket packages
6. **No Leaks**: Goroutine leak detection passes
7. **CI Configured**: GitHub Actions workflow runs all tests
8. **Test Helpers**: Mock transport and utilities available

## Dependencies

- All previous PRDs (sp-ms6.1 through sp-ms6.4)
- Go testing package
- Go race detector

## References

- [Go testing package](https://pkg.go.dev/testing)
- [Go race detector](https://go.dev/doc/articles/race_detector)
- [Go benchmarks](https://pkg.go.dev/testing#hdr-Benchmarks)
