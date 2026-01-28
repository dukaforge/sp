# SP: Scalability Protocols for Go

SP is a pure Go library implementing the Scalability Protocols specification. We provide high-performance messaging patterns over Unix sockets and raw IP, designed for agent coordination in network management systems.

## Features

SP implements six messaging patterns:

| Pattern | Use Case |
|---------|----------|
| REQ/REP | Synchronous request-reply with automatic correlation |
| PUB/SUB | One-to-many broadcast with topic filtering |
| PIPELINE | Work distribution with round-robin load balancing |
| SURVEY | Broadcast query with deadline-based response collection |
| BUS | Many-to-many peer mesh communication |
| PAIR | Exclusive bidirectional point-to-point channel |

## Installation

```bash
go get github.com/dukaforge/sp
```

## Quick Start

```go
package main

import (
    "fmt"
    "github.com/dukaforge/sp"
)

func main() {
    // Create a REQ socket
    req, _ := sp.NewReqSocket(sp.ReqConfig{})
    defer req.Close()

    // Create a REP socket
    rep, _ := sp.NewRepSocket(sp.RepConfig{})
    defer rep.Close()

    // Connect
    rep.Listen("unix:///tmp/sp.sock")
    req.Dial("unix:///tmp/sp.sock")

    // Send request
    req.Send([]byte("hello"))

    // Receive and reply
    msg, _ := rep.Recv()
    fmt.Printf("Server received: %s\n", msg)
    rep.Send([]byte("world"))

    // Receive reply
    reply, _ := req.Recv()
    fmt.Printf("Client received: %s\n", reply)
}
```

## Why SP?

SP addresses specific needs in agent coordination:

- **Transport abstraction**: Same API for local IPC (Unix sockets) and distributed communication (raw IP)
- **Low latency**: Avoids TCP overhead by using datagram transports
- **Go-native**: Goroutines and channels instead of callbacks
- **Wire-protocol compatible**: Interoperates with other Scalability Protocol implementations

## Documentation

- [VISION.md](doc/VISION.md) - Project goals and non-goals
- [ARCHITECTURE.md](doc/ARCHITECTURE.md) - System design and component overview

### Product Requirements Documents

| Category                                   | Documents                                                             |
|--------------------------------------------|-----------------------------------------------------------------------|
| [Protocols](doc/prd/protocols/)            | REQ/REP, PUB/SUB, PIPELINE, SURVEY, BUS, PAIR pattern specifications  |
| [Infrastructure](doc/prd/infrastructure/)  | Transport abstraction, shared infrastructure, I/O workers, socket API |
| [API](doc/prd/api/)                        | Async API design, testing strategy                                    |

## Project Status

SP is in the design and documentation phase. Implementation has not yet begun. See the PRDs in `doc/prd/` for detailed specifications.

## License

MIT
