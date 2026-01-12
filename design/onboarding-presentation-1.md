---
marp: true
theme: default
style: |
  section {
    font-size: 24px;
    padding-top: 20px;
  }
  h1 {
    font-size: 36px;
  }
  h2 {
    font-size: 32px;
    margin-top: 0;
  }
  pre, code {
    font-size: 18px;
  }
  table {
    font-size: 20px;
  }
---

# Go MCP SDK: Maintainer Onboarding

## Presentation 1: High-Level Overview

---

# Agenda (60 minutes)

1. **Introduction to MCP** (15-20 min)
   - What is MCP?
   - Where does this SDK fit?
   - The 4-level architecture

2. **High-Level APIs** (25-30 min)
   - Servers & Clients
   - Sessions & Connections
   - Feature binding (Tools, Prompts, Resources)
   - Transports

3. **Working on the Repository** (15-20 min)
   - Running tests
   - Conformance tests
   - Testing with MCP Inspector

---

# Part 1: Introduction to MCP

---

## What is MCP?

**MCP** is an open standard for connecting AI assistants to external tools and data.

```
┌────────────────────────────────┐
│           Host                 │
│   (Claude Desktop, IDE, etc.)  │
│                                │
│   ┌────────────────────────┐   │        ┌─────────────────────┐
│   │      MCP Client        │   │        │     MCP Server      │
│   │   (provided by SDK)    │<──────────>│  (Tools, Resources, │
│   └────────────────────────┘   │  MCP   │   Prompts, etc.)    │
│                                │        └─────────────────────┘
└────────────────────────────────┘
```

**Key Roles:**
- **Host**: AI application that wants to access external capabilities
- **MCP Client**: Protocol layer (SDK) that connects to servers
- **MCP Server**: Exposes tools, resources, and prompts

**This SDK provides both client and server implementations.**

---

## MCP SDKs

Official SDKs at [github.com/modelcontextprotocol](https://github.com/modelcontextprotocol):

| SDK | Collaboration |
|-----|---------------|
| TypeScript, Python | Anthropic |
| **Go** | **Google** |
| Java | Spring AI |
| Kotlin | JetBrains |
| C# | Microsoft |
| Ruby | Shopify |
| PHP | PHP Foundation + Symfony |
| Swift, Rust | Community |

---

## Design Principles

From `design/design.md`:

- **Complete** - Implement every feature of the MCP spec
- **Idiomatic** - Follow Go conventions and `net/http` patterns
- **Robust** - Well tested; enable testability for users
- **Future-proof** - Allow spec evolution without breaking changes

---

## The 4-Level Architecture

```
Level 1: Application Layer
   ├─ Client: manages multiple ClientSessions
   └─ Server: manages multiple ServerSessions

Level 2: Session Layer
   ├─ ClientSession: API for calling server methods
   └─ ServerSession: API for calling client methods

Level 3: JSON-RPC Protocol Layer
   └─ jsonrpc2.Connection: bidirectional message exchange

Level 4: Transport Layer
   ├─ CommandTransport (stdio client-side)
   ├─ StdioTransport (stdio server-side)
   ├─ SSEClientTransport / SSEHTTPHandler
   ├─ StreamableClientTransport / StreamableHTTPHandler
   └─ InMemoryTransport (testing)
```

Each layer has a single responsibility.

---

## Connection Flow

```
Client                                                   Server
  ⇅                          (jsonrpc2)                     ⇅
ClientSession ⇄ Client Transport ⇄ Server Transport ⇄ ServerSession
```

**Data Flow Example (Tool Call):**

1. `session.CallTool(ctx, "greet", args)` on `Client`
2. `Client` marshals to JSON-RPC request
3. `Transport` sends to server
4. `Server` dispatches to tool handler
5. Tool executes and returns result
6. Response flows back through `Transport`
7. `Client` unmarshals result

---

# Part 2: High-Level APIs

---

## Package Layout

```
go-sdk/
├── mcp/                    # Core user-facing API
│   ├── client.go          # Client implementation
│   ├── server.go          # Server implementation
│   ├── session.go         # Session types
│   ├── transport.go       # Transport abstractions
│   ├── tool.go            # Tool binding
│   ├── protocol.go        # Auto-generated protocol types
│   ├── streamable.go      # Streamable HTTP transport
│   └── sse.go             # SSE transport
├── jsonrpc/               # Public JSON-RPC type aliases
├── internal/jsonrpc2/     # Internal JSON-RPC implementation
├── auth/                  # OAuth primitives
└── examples/              # Working examples
```

Single `mcp` package (like `net/http`).

*V2 consideration: more granular packages could make the SDK more digestible.*

---

## Creating a Server

```go
// Create a server with implementation info
server := mcp.NewServer(&mcp.Implementation{
    Name:    "my-server",
    Version: "v1.0.0",
}, nil)  // nil = default options

// Add features
mcp.AddTool(server, tool, handler)
server.AddPrompt(prompt, handler)
server.AddResource(resource, handler)

// Run on stdio transport
if err := server.Run(ctx, &mcp.StdioTransport{}); err != nil {
    log.Fatal(err)
}
```

**Key Points:**
- `Server` can handle multiple concurrent sessions
- Features can be added/removed dynamically
- Connected clients are notified of changes

---

## Creating a Client

```go
client := mcp.NewClient(&mcp.Implementation{
    Name:    "my-client",
    Version: "v1.0.0",
}, nil)

transport := &mcp.CommandTransport{
    Command: exec.Command("path/to/server"),
}

session, err := client.Connect(ctx, transport, nil)
if err != nil {
    log.Fatal(err)
}
defer session.Close()

result, err := session.CallTool(ctx, &mcp.CallToolParams{Name: "greet"})
```

**Key Points:**
- `Client` can connect to multiple servers (multiple sessions)
- `Connect` returns a `ClientSession` for interacting with the server
- Session methods: `ListTools()`, `CallTool()`, `GetPrompt()`, etc.

---

## Sessions: The Connection Abstraction

**Why separate Client/Server from ClientSession/ServerSession?**

```go
// Server can have multiple sessions (n:1)
server := mcp.NewServer(impl, nil)

// Each connection creates a new session
for session := range server.Sessions() {
    // Handle each connected client
}

// Client can also have multiple sessions
client := mcp.NewClient(impl, nil)
session1, _ := client.Connect(ctx, transport1, nil)
session2, _ := client.Connect(ctx, transport2, nil)
```

**Session provides the peer API:**
- `ClientSession`: `ListTools()`, `CallTool()`, `GetPrompt()`, etc.
- `ServerSession`: `ListRoots()`, `CreateMessage()`, `NotifyProgress()`, etc.

---

## Feature Binding: Tools

**Two approaches to add tools:**

**Low-Level (ToolHandler)**
```go
server.AddTool(&mcp.Tool{
    Name:        "greet",
    InputSchema: mySchema,
}, func(ctx context.Context, req *mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    // Manual argument parsing
    return result, nil
})
```

**High-Level (Generic AddTool)**
```go
type GreetArgs struct {
    Name string `json:"name" jsonschema:"the person to greet"`
}

mcp.AddTool(server, &mcp.Tool{
    Name: "greet",
}, func(ctx context.Context, req *mcp.CallToolRequest, args GreetArgs) (*mcp.CallToolResult, string, error) {
    return nil, "Hello " + args.Name, nil
})
```

**The generic version automatically:**
- Infers input schema from Go struct
- Validates input against schema
- Handles error wrapping

---

## Feature Binding: Prompts & Resources

**Prompts**
```go
server.AddPrompt(&mcp.Prompt{
    Name:        "code_review",
    Description: "Review code for best practices",
    Arguments: []*mcp.PromptArgument{
        {Name: "code", Required: true},
    },
}, func(ctx context.Context, ss *mcp.ServerSession, params *mcp.GetPromptParams) (*mcp.GetPromptResult, error) {
    return &mcp.GetPromptResult{
        Messages: []mcp.SamplingMessage{{
            Role: mcp.RoleUser,
            Content: &mcp.TextContent{Text: "Review: " + params.Arguments["code"]},
        }},
    }, nil
})
```

**Resources**
```go
server.AddResource(&mcp.Resource{
    URI:      "file:///config.json",
    Name:     "Config File",
    MIMEType: "application/json",
}, func(ctx context.Context, ss *mcp.ServerSession, params *mcp.ReadResourceParams) (*mcp.ReadResourceResult, error) {
    return mcp.NewTextResourceContents(params.URI, "application/json", configData), nil
})
```

---

## Transports

Transports abstract the communication layer. A `Connection` is a bidirectional stream.

```go
type Transport interface {
    Connect(ctx context.Context) (Connection, error)
}

type Connection interface {
    Read(ctx context.Context) (JSONRPCMessage, error)
    Write(ctx context.Context, JSONRPCMessage) error
    Close() error
    SessionID() string  // V2: remove
}
```

---

## Available Transports

| Transport | Side | Use Case |
|-----------|------|----------|
| `CommandTransport` | Client | Run server as subprocess |
| `StdioTransport` | Server | Communicate via stdin/stdout |
| `SSEClientTransport` | Client | HTTP SSE (legacy) |
| `SSEHTTPHandler` | Server | HTTP SSE handler |
| `StreamableClientTransport` | Client | Modern HTTP transport |
| `StreamableHTTPHandler` | Server | Modern HTTP handler |
| `InMemoryTransport` | Both | In-process testing |

---

## HTTP Server Example

```go
server := mcp.NewServer(impl, nil)
mcp.AddTool(server, tool, handler)

// Create HTTP handler (one server per session, or shared)
handler := mcp.NewStreamableHTTPHandler(
    func(req *http.Request) *mcp.Server {
        return server  // Return same server for all sessions
    },
    nil,  // default options
)

// Mount and serve
http.Handle("/mcp", handler)
log.Fatal(http.ListenAndServe(":8080", nil))
```

**For per-session state:**
```go
handler := mcp.NewStreamableHTTPHandler(
    func(req *http.Request) *mcp.Server {
        // Create new server for each session
        s := mcp.NewServer(impl, nil)
        // Add session-specific features
        return s
    },
    nil,
)
```

---

## Middleware

**MCP-level middleware for cross-cutting concerns:**

```go
type MethodHandler func(ctx context.Context, method string, req Request) (Result, error)
type Middleware func(MethodHandler) MethodHandler

// Logging middleware example
func loggingMiddleware(next mcp.MethodHandler) mcp.MethodHandler {
    return func(ctx context.Context, method string, req mcp.Request) (mcp.Result, error) {
        log.Printf("Request: %s", method)
        result, err := next(ctx, method, req)
        log.Printf("Response: %v, err: %v", result, err)
        return result, err
    }
}

// Add to server
server.AddReceivingMiddleware(loggingMiddleware)
server.AddSendingMiddleware(loggingMiddleware)
```

**Receiving:** Runs before handling incoming requests
**Sending:** Runs before sending outgoing requests

---

# Part 3: Working on the Repository

---

## Running Tests

```bash
go test ./...
```

Internal conformance tests in `mcp/testdata/conformance/server/*.txtar`:

```bash
go test -run TestServerConformance ./mcp           # Run (Go 1.24+/1.25+)
go test -run TestServerConformance ./mcp -update   # Update expected output
```

---

## Conformance Tests (Official)

Official framework at [github.com/modelcontextprotocol/conformance](https://github.com/modelcontextprotocol/conformance).

The SDK includes a script to run against the SDK's conformance server:

```bash
# Run official conformance tests
./scripts/conformance.sh

# Save results to a directory
./scripts/conformance.sh --result_dir ./conformance-results

# Run against local conformance repo checkout
./scripts/conformance.sh --conformance_repo ~/src/conformance
```

---

## Documentation Generation

**README.md is auto-generated - don't edit directly!**

```bash
# Update README.md from source
go generate ./internal/readme

# Update all documentation
go generate ./internal/docs
```

### Source Files
- `internal/readme/readme.src.md` - README source
- `internal/docs/*.src.md` - Other doc sources

**After editing source files, always regenerate!**

---

## Testing with MCP Inspector

[MCP Inspector](https://github.com/modelcontextprotocol/inspector) - web UI for testing MCP servers.

```bash
# Launch inspector with a stdio server
npx @modelcontextprotocol/inspector go run ./examples/server/hello

# Or just launch the UI and connect via browser
npx @modelcontextprotocol/inspector
# Opens http://localhost:6274 - connect to HTTP servers from there
```

Supports stdio, SSE, and streamable HTTP transports.

---

## Testing with Go Workspaces

**Test SDK changes against projects that use it:**

```bash
# Create workspace with your project and the SDK
go work init ./your-project ./go-sdk

# Now your-project uses local go-sdk changes
cd your-project
go test ./...
```

This allows testing real-world usage without publishing SDK changes.

---

## Resources

- **Design Document:** `design/design.md`
- **API Reference:** https://pkg.go.dev/github.com/modelcontextprotocol/go-sdk/mcp
- **MCP Spec:** https://modelcontextprotocol.io/specification/
- **Examples:** `examples/` directory
- **CONTRIBUTING.md:** Contribution guidelines

---

## Q&A

Questions about the SDK architecture or development workflow?

---

# Practical Exercise: Your First PR

---

## Submitting a PR

1. Fork the repository on GitHub
2. Clone your fork and create a branch
3. Make your change
4. Run tests: `go test ./...`
5. Commit with proper format:
   ```
   mcp: brief description

   Longer explanation if needed.

   Fixes #123
   ```
6. Push and open a PR against `main`

See `CONTRIBUTING.md` for full details.

---

## Suggested First PRs

Pick one of these trivial tasks:

| Task |
|------|
| Move `TestToolForSchemas` to `tool_test.go` (`mcp/server_test.go:803`) |
| Fix typos: `failedd`, `capabilitiy`, `availble` |
| Extract `"Last-Event-ID"` to a constant (`mcp/streamable_client_test.go:72`) |
| Update `CONTRIBUTING.md` to reference MCP Code of Conduct, not Go's |

Search for `TODO` in the codebase for more opportunities!

---

# Next Sessions

**Presentation 2: Deep Dive**
- Tool Binding (schema inference, validation, error handling)
- Streamable Transport (protocol, sessions, resumption)

**Presentation 3: Open Source Processes**
- Governance & Working Group
- Issue & PR Management
- Tracking the MCP Spec
- Releases & Maintenance
