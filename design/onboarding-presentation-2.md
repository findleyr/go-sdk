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

## Presentation 2: Deep Dive - Tool Binding & Streamable Transport

---

# Agenda (60 minutes)

1. **Tool Binding Deep Dive** (30 min)
   - The two-tier API design
   - Generic type inference with `AddTool[In, Out]`
   - JSON Schema generation
   - Input validation & defaults
   - Error handling patterns

2. **Streamable Transport Deep Dive** (30 min)
   - Protocol overview
   - Server-side architecture
   - Client-side architecture
   - Session & stream management
   - Reconnection & resumption

---

# Part 1: Tool Binding Deep Dive

---

## Low-Level API: `Server.AddTool`

Full control over schema and argument parsing:

```go
server.AddTool(&mcp.Tool{
    Name:        "greet",
    InputSchema: &jsonschema.Schema{
        Type: "object",
        Properties: map[string]*jsonschema.Schema{
            "name": {Type: "string"},
        },
        Required: []string{"name"},
    },
}, func(ctx context.Context, req *mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    var args map[string]any
    json.Unmarshal(req.Params.Arguments, &args)
    // Manual handling...
    return result, nil
})
```

- You provide the `InputSchema`
- You parse `req.Params.Arguments` yourself
- You construct the `CallToolResult`

---

## High-Level API: Generic `mcp.AddTool`

Automatic schema inference and validation:

```go
type Args struct {
    Name string `json:"name" jsonschema:"the person's name"`
}

mcp.AddTool(server, &mcp.Tool{Name: "greet"},
    func(ctx context.Context, req *mcp.CallToolRequest, args Args) (*mcp.CallToolResult, string, error) {
        return nil, "Hello " + args.Name, nil
    })
```

- Schema inferred from `Args` struct
- Input validated automatically
- Output marshaled to `StructuredContent`
- Errors wrapped appropriately

---

## Type Definitions

**Location:** `mcp/tool.go`

```go
// Low-level handler - raw JSON arguments
type ToolHandler func(context.Context, *CallToolRequest) (*CallToolResult, error)

// High-level handler - typed arguments and output
type ToolHandlerFor[In, Out any] func(
    ctx context.Context,
    request *CallToolRequest,
    input In,
) (result *CallToolResult, output Out, err error)
```

The `ToolHandlerFor` has three return values:
1. `*CallToolResult` - Optional, for setting Content manually
2. `Out` - Typed output, becomes StructuredContent
3. `error` - Tool errors vs protocol errors

---

## The Generic AddTool Function

**Location:** `mcp/server.go:447`

```go
func AddTool[In, Out any](s *Server, t *Tool, h ToolHandlerFor[In, Out]) {
    tt, hh, err := toolForErr(t, h)
    if err != nil {
        panic(fmt.Sprintf("AddTool: tool %q: %v", t.Name, err))
    }
    s.AddTool(tt, hh)
}
```

**What `toolForErr` does:**
1. Infers InputSchema from `In` type (if not provided)
2. Infers OutputSchema from `Out` type (if not `any`)
3. Creates a wrapper handler that:
   - Validates input against schema
   - Applies default values
   - Unmarshals to typed struct
   - Handles error wrapping

---

## JSON Schema Generation

**Package:** `github.com/google/jsonschema-go/jsonschema`

**From Go Types**
```go
type AddArgs struct {
    X int `json:"x" jsonschema:"first number to add"`
    Y int `json:"y" jsonschema:"second number to add"`
}

// jsonschema.For[T]() generates:
schema, _ := jsonschema.For[AddArgs](nil)
// {
//   "type": "object",
//   "properties": {
//     "x": {"type": "integer", "description": "first number to add"},
//     "y": {"type": "integer", "description": "second number to add"}
//   },
//   "required": ["x", "y"]
// }
```

**Struct Tag Support**
- `json:"fieldname"` - JSON field name
- `json:",omitempty"` - Field is optional
- `jsonschema:"description"` - Field description
- `jsonschema_extras:"key=value"` - Additional schema properties

---

## Schema Inference in toolForErr

**Location:** `mcp/server.go:278-306`

```go
func toolForErr[In, Out any](t *Tool, h ToolHandlerFor[In, Out]) (*Tool, ToolHandler, error) {
    tt := *t  // Copy tool to avoid mutating original

    // Special case: "any" input treated as empty object
    if reflect.TypeFor[In]() == reflect.TypeFor[any]() && t.InputSchema == nil {
        tt.InputSchema = &jsonschema.Schema{Type: "object"}
    }

    // Infer input schema if not provided
    var inputResolved *jsonschema.Resolved
    if _, err := setSchema[In](&tt.InputSchema, &inputResolved); err != nil {
        return nil, nil, fmt.Errorf("input schema: %w", err)
    }

    // Infer output schema (only if Out is not "any")
    var outputResolved *jsonschema.Resolved
    if t.OutputSchema != nil || reflect.TypeFor[Out]() != reflect.TypeFor[any]() {
        elemZero, err = setSchema[Out](&tt.OutputSchema, &outputResolved)
        if err != nil {
            return nil, nil, fmt.Errorf("output schema: %v", err)
        }
    }
    // ...
}
```

Schema is only inferred if not already provided, allowing manual override.

---

## Input Validation & Defaults

**Location:** `mcp/tool.go:68-106`

```go
func applySchema(data json.RawMessage, resolved *jsonschema.Resolved) (json.RawMessage, error) {
    if resolved != nil {
        v := make(map[string]any)
        if len(data) > 0 {
            if err := json.Unmarshal(data, &v); err != nil {
                return nil, fmt.Errorf("unmarshaling arguments: %w", err)
            }
        }

        // Apply default values from schema
        if err := resolved.ApplyDefaults(&v); err != nil {
            return nil, fmt.Errorf("applying schema defaults:\n%w", err)
        }

        // Validate against schema
        if err := resolved.Validate(&v); err != nil {
            return nil, err
        }

        // Re-marshal with defaults applied
        data, err = json.Marshal(v)
    }
    return data, nil
}
```

---

## Default Values Example

```go
type IncArgs struct {
    X int `json:"x,omitempty"`
}

// Set up schema with default value:
inSchema, _ := jsonschema.For[IncArgs](nil)
inSchema.Properties["x"].Default = json.RawMessage(`6`)

mcp.AddTool(server, &mcp.Tool{Name: "inc", InputSchema: inSchema}, incHandler)
```

**What happens:**
- Client calls with `{}` (empty args)
- `applySchema` applies default: `{x: 6}`
- Handler receives `IncArgs{X: 6}`

---

## The Generated Handler Wrapper

**Location:** `mcp/server.go:308-400`

```go
th := func(ctx context.Context, req *CallToolRequest) (*CallToolResult, error) {
    // 1. Get raw arguments
    var input json.RawMessage
    if req.Params.Arguments != nil {
        input = req.Params.Arguments
    }

    // 2. Validate and apply defaults
    input, err = applySchema(input, inputResolved)
    if err != nil {
        return nil, fmt.Errorf("%w: validating \"arguments\": %v",
            jsonrpc2.ErrInvalidParams, err)
    }

    // 3. Unmarshal to typed struct
    var in In
    if input != nil {
        if err := json.Unmarshal(input, &in); err != nil {
            return nil, fmt.Errorf("%w: %v", jsonrpc2.ErrInvalidParams, err)
        }
    }

    // 4. Call the typed handler
    res, out, err := h(ctx, req, in)

    // 5. Handle errors (next slide)
    // 6. Marshal output (next slide)
}
```

---

## Error Handling: Tool Errors vs Protocol Errors

**Location:** `mcp/server.go:330-344`

```go
if err != nil {
    // Check if this is already a JSON-RPC error (protocol error)
    if wireErr, ok := err.(*jsonrpc.Error); ok {
        return nil, wireErr  // Return as-is, breaks the call
    }

    // Regular errors become tool errors (IsError=true)
    var errRes CallToolResult
    errRes.setError(err)  // Sets IsError=true, Content=[TextContent{err.Error()}]
    return &errRes, nil   // Return success at protocol level!
}
```

**Distinction:**
- **Protocol Error:** Returns `error` → Connection may break
- **Tool Error:** Returns `&CallToolResult{IsError: true}` → Normal response

**When to use each:**
```go
// Protocol error - something fundamentally wrong
return nil, fmt.Errorf("%w: database unavailable", jsonrpc2.ErrInternal)

// Tool error - expected failure case
return nil, errors.New("file not found")  // Wrapped automatically
```

---

## Output Handling

**Location:** `mcp/server.go:346-400`

```go
if res == nil {
    res = &CallToolResult{}
}

// Marshal output to StructuredContent
var outval any = out
if elemZero != nil {
    // Handle typed nil pointers - use zero value instead
    var z Out
    if any(out) == any(z) {
        outval = elemZero
    }
}

// Set StructuredContent (for spec 2025-03-26+)
outData, err := json.Marshal(outval)
res.StructuredContent = json.RawMessage(outData)

// Auto-populate Content if not set
if res.Content == nil {
    res.Content = []Content{&TextContent{Text: string(outData)}}
}

// Validate output against schema
if outputResolved != nil {
    if err := outputResolved.Validate(outval); err != nil {
        return nil, fmt.Errorf("output validation: %w", err)
    }
}
```

---

## Tool Name Validation

**Location:** `mcp/tool.go:108-141`

```go
func validateToolName(name string) error {
    if name == "" {
        return fmt.Errorf("tool name cannot be empty")
    }
    if len(name) > 128 {
        return fmt.Errorf("tool name exceeds maximum length of 128 characters")
    }

    var invalidChars []string
    seen := make(map[rune]bool)
    for _, r := range name {
        if !validToolNameRune(r) {
            if !seen[r] {
                invalidChars = append(invalidChars, fmt.Sprintf("%q", string(r)))
                seen[r] = true
            }
        }
    }
    // ...
}

func validToolNameRune(r rune) bool {
    return (r >= 'a' && r <= 'z') ||
           (r >= 'A' && r <= 'Z') ||
           (r >= '0' && r <= '9') ||
           r == '_' || r == '-' || r == '.'
}
```

**Valid characters:** `a-z`, `A-Z`, `0-9`, `_`, `-`, `.`

*V2 issue: Invalid tool names only log an error instead of panicking (SEP-986 landed post-v1).*

---

## Tool Binding: Complete Flow

```
AddTool[Args, Result](server, tool, handler)
         │
         ▼
    toolForErr()
         │
    ┌────┴────┐
    │         │
    ▼         ▼
Infer     Create wrapper
schemas   handler
    │         │
    └────┬────┘
         │
         ▼
   server.AddTool(tool, wrappedHandler)
         │
         ▼
   Store in server.tools featureSet


═══════════════════════════════════════════

Client calls tool:
         │
         ▼
   wrappedHandler(ctx, req)
         │
    ┌────┴────────────────┐
    │                     │
    ▼                     ▼
applySchema()        json.Unmarshal()
(validate+defaults)   (to typed struct)
    │                     │
    └─────────┬───────────┘
              │
              ▼
      userHandler(ctx, req, typedArgs)
              │
              ▼
      Handle errors, marshal output
              │
              ▼
      Return CallToolResult
```

---

# Part 2: Streamable Transport Deep Dive

---

## Streamable HTTP: Protocol Overview

**Spec:** MCP 2025-03-26+

- Single HTTP endpoint for all operations
- Supports both request/response and server-push
- Session management via `Mcp-Session-Id` header
- Optional stream resumption via SSE event IDs

**HTTP Methods:**
| Method | Purpose |
|--------|---------|
| POST | Send JSON-RPC messages to server |
| GET | Establish standalone SSE stream |
| DELETE | Terminate session |

---

## Message Flow

```
Client                                   Server
   │                                        │
   │──── POST /mcp (initialize) ───────────>│
   │<─── 200 OK + Mcp-Session-Id header ────│
   │                                        │
   │──── POST /mcp (tools/call) ───────────>│
   │<─── 200 OK (text/event-stream) ────────│
   │     event: message                     │
   │     data: {"jsonrpc":"2.0",...}        │
   │                                        │
   │──── GET /mcp (standalone SSE) ────────>│
   │<─── 200 OK (text/event-stream) ────────│
   │     [server can push messages here]    │
   │                                        │
   │──── DELETE /mcp ──────────────────────>│
   │<─── 204 No Content ────────────────────│
```

---

## Server-Side Architecture

**Location:** `mcp/streamable.go`

```
┌──────────────────────────────────────────┐
│         StreamableHTTPHandler            │
│  - Routes incoming HTTP requests         │
│  - Manages session map                   │
│  - Creates transports per session        │
└──────────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│      StreamableServerTransport           │
│  - One per session                       │
│  - Implements Transport interface        │
│  - Creates streamableServerConn          │
└──────────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│        streamableServerConn              │
│  - Implements Connection interface       │
│  - Manages multiple streams              │
│  - Routes messages to correct stream     │
└──────────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│              stream                      │
│  - One per POST request or GET           │
│  - Tracks pending requests               │
│  - Delivers responses                    │
└──────────────────────────────────────────┘
```

---

## StreamableHTTPHandler

**Location:** `mcp/streamable.go:46-184`

```go
type StreamableHTTPHandler struct {
    getServer func(*http.Request) *Server
    opts      StreamableHTTPOptions

    mu       sync.Mutex
    sessions map[string]*sessionInfo  // keyed by session ID
}

type sessionInfo struct {
    session   *ServerSession
    transport *StreamableServerTransport
    userID    string           // For session hijacking prevention
    timeout   time.Duration    // Idle timeout
    timer     *time.Timer      // Cleanup timer
}
```

- Implements `http.Handler` — mount at any path
- `getServer` called per-request: return shared server or create per-session
- `sessions` map tracks active sessions by ID
- `sessionInfo` binds session to transport and tracks user for security

---

## StreamableHTTPOptions

```go
type StreamableHTTPOptions struct {
    Stateless      bool          // No session tracking
    JSONResponse   bool          // Prefer JSON over SSE
    Logger         *slog.Logger
    EventStore     EventStore    // For stream resumption
    SessionTimeout time.Duration // Idle cleanup
}
```

| Option | Default | Purpose |
|--------|---------|---------|
| `Stateless` | false | Skip session management (load balancers, serverless) |
| `JSONResponse` | false | Return JSON instead of SSE |
| `Logger` | nil | Enable debug logging |
| `EventStore` | nil | Enable stream resumption |
| `SessionTimeout` | 0 | Auto-cleanup idle sessions |

---

## Request Routing (ServeHTTP)

**Location:** `mcp/streamable.go:209-472`

```go
func (h *StreamableHTTPHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    // 1. Validate Accept header
    // POST/DELETE need: application/json AND text/event-stream
    // GET needs: text/event-stream

    // 2. Look up or create session
    sessionID := req.Header.Get("Mcp-Session-Id")
    sessInfo := h.sessions[sessionID]

    // 3. Validate protocol version header
    protocolVersion := req.Header.Get("Mcp-Protocol-Version")

    // 4. Route by method
    switch req.Method {
    case http.MethodGet:
        sessInfo.transport.ServeHTTP(w, req)  // Standalone or resumed SSE
    case http.MethodPost:
        sessInfo.transport.ServeHTTP(w, req)  // Messages
    case http.MethodDelete:
        sessInfo.session.Close()
        w.WriteHeader(http.StatusNoContent)
    }
}
```

---

## streamableServerConn

**Location:** `mcp/streamable.go:567-662`

```go
type streamableServerConn struct {
    sessionID      string
    incoming       chan jsonrpc.Message      // From client
    streams        map[string]*stream        // Logical streams
    requestStreams map[jsonrpc.ID]string     // Request → stream mapping
}
```

**How it works:**
- `Read()`: pulls from `incoming` channel (fed by POST handlers)
- `Write()`: looks up which stream owns the request, delivers response there
- `requestStreams`: maps each pending request ID to its stream ID
- `streams`: maps stream ID to the actual stream (HTTP response writer)

When a response comes back, the conn finds the right stream and writes to it.

---

## Streams

```go
type stream struct {
    id       string                    // "" for standalone SSE
    w        http.ResponseWriter       // Where to write SSE events
    done     chan struct{}             // Closed when complete
    requests map[jsonrpc.ID]struct{}   // Pending request IDs
    lastIdx  int                       // Event index for resumption
}
```

**Two types:**
- **Standalone SSE** (id="") — long-lived GET connection for server-initiated messages
- **Request stream** (random id) — created per-POST, closes when all responses sent

**Lifecycle:**
1. POST arrives → new stream created, request IDs registered
2. Responses written → delivered to stream's `http.ResponseWriter`
3. All requests answered → stream closes, HTTP response ends

---

## POST Handling (servePOST)

**Location:** `mcp/streamable.go:998-1207`

```go
func (c *streamableServerConn) servePOST(w http.ResponseWriter, req *http.Request) {
    // 1. Read and parse incoming messages
    body, _ := io.ReadAll(req.Body)
    incoming, isBatch, _ := readBatch(body)

    // 2. Identify calls (need responses) vs notifications
    calls := make(map[jsonrpc.ID]struct{})
    for _, msg := range incoming {
        if req, ok := msg.(*jsonrpc.Request); ok && req.IsCall() {
            calls[req.ID] = struct{}{}
        }
    }

    // 3. If only notifications, return 202 Accepted immediately
    if len(calls) == 0 {
        for _, msg := range incoming {
            c.incoming <- msg
        }
        w.WriteHeader(http.StatusAccepted)
        return
    }

    // 4. Create stream to track responses
    stream := c.newStream(req.Context(), calls, randText())

    // 5. Set up response headers and publish messages
    // 6. Hang until all responses sent
}
```

---

## Response Delivery

**Location:** `mcp/streamable.go:1252-1364`

```go
func (c *streamableServerConn) Write(ctx context.Context, msg jsonrpc.Message) error {
    // 1. Encode message
    data, _ := jsonrpc2.EncodeMessage(msg)

    // 2. Find the right stream
    var relatedRequest jsonrpc.ID
    if resp, ok := msg.(*jsonrpc.Response); ok {
        relatedRequest = resp.ID
    } else {
        // Check context for related incoming request
        if v := ctx.Value(idContextKey{}); v != nil {
            relatedRequest = v.(jsonrpc.ID)
        }
    }

    // 3. Look up stream
    stream := c.streams[c.requestStreams[relatedRequest]]
    if stream == nil {
        stream = c.streams[""]  // Fall back to standalone SSE
    }

    // 4. Deliver to stream (SSE or JSON)
    stream.deliverLocked(data, eventID, responseTo)

    // 5. Clean up if stream is done
}
```

---

## GET Handling (Standalone SSE)

**Location:** `mcp/streamable.go:823-862`

```go
func (c *streamableServerConn) serveGET(w http.ResponseWriter, req *http.Request) {
    streamID := ""
    lastIdx := -1

    // Handle Last-Event-ID for resumption
    if eid := req.Header.Get("Last-Event-ID"); eid != "" {
        streamID, lastIdx, _ = parseEventID(eid)
    }

    // Acquire the stream (may replay events)
    stream, done := c.acquireStream(ctx, w, streamID, lastIdx, protocolVersion)
    defer stream.release()

    // Hang until closed
    c.hangResponse(ctx, done)
}

func (c *streamableServerConn) hangResponse(ctx context.Context, done <-chan struct{}) {
    select {
    case <-ctx.Done():      // Client disconnected
    case <-done:            // Stream complete
    case <-c.done:          // Session closed
    }
}
```

---

## Stream Resumption

**EventStore Interface:**
```go
type EventStore interface {
    Open(ctx context.Context, sessionID, streamID string) error
    Append(ctx context.Context, sessionID, streamID string, data []byte) error
    After(ctx context.Context, sessionID, streamID string, lastIdx int) iter.Seq2[[]byte, error]
    SessionClosed(ctx context.Context, sessionID string) error
}
```

*V2 issue: `EventStore.Open` is unnecessary (artifact of earlier design). Can be a no-op.*

**Event ID Format:** `{streamID}_{index}`
- Example: `abc123_5` = stream "abc123", event index 5

**Resumption Flow:**
1. Client sends GET with `Last-Event-ID: abc123_5`
2. Server parses to streamID="abc123", lastIdx=5
3. Server replays events 6, 7, 8... from EventStore
4. Server continues delivering new events

---

## Client-Side Architecture

**Location:** `mcp/streamable.go:1385-2079`

```go
type StreamableClientTransport struct {
    Endpoint   string
    HTTPClient *http.Client
    MaxRetries int              // Default 5
    DisableStandaloneSSE bool   // Skip GET stream
}

type streamableClientConn struct {
    url        string
    client     *http.Client
    ctx        context.Context     // Detached from Connect
    cancel     context.CancelFunc
    incoming   chan jsonrpc.Message
    maxRetries int

    mu                sync.Mutex
    initializedResult *InitializeResult
    sessionID         string
}
```

---

## Client: Sending Messages

**Location:** `mcp/streamable.go:1660-1764`

```go
func (c *streamableClientConn) Write(ctx context.Context, msg jsonrpc.Message) error {
    // 1. Encode message
    data, _ := jsonrpc.EncodeMessage(msg)

    // 2. Create POST request
    req, _ := http.NewRequestWithContext(ctx, http.MethodPost, c.url, bytes.NewReader(data))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Accept", "application/json, text/event-stream")
    c.setMCPHeaders(req)  // Session ID, protocol version

    // 3. Execute request
    resp, err := c.client.Do(req)

    // 4. Handle response based on Content-Type
    contentType := resp.Header.Get("Content-Type")
    switch contentType {
    case "application/json":
        go c.handleJSON(requestSummary, resp)
    case "text/event-stream":
        go c.handleSSE(ctx, requestSummary, resp, forCall)
    }
}
```

---

## Client: Reconnection Strategy

**Location:** `mcp/streamable.go:1804-1851, 1971-2021`

```go
func (c *streamableClientConn) handleSSE(ctx context.Context, summary string,
    resp *http.Response, forCall *jsonrpc.Request) {
    for {
        // Process current stream
        lastEventID, reconnectDelay, clientClosed := c.processStream(ctx, summary, resp, forCall)

        if clientClosed {
            return
        }

        // Attempt reconnection with exponential backoff
        newResp, err := c.connectSSE(ctx, lastEventID, reconnectDelay, false)
        if err != nil {
            c.fail(err)
            return
        }
        resp = newResp
    }
}

func calculateReconnectDelay(attempt int) time.Duration {
    if attempt == 0 {
        return 0
    }
    // Exponential backoff with jitter
    backoff := time.Duration(float64(1*time.Second) * math.Pow(1.5, float64(attempt-1)))
    backoff = min(backoff, 30*time.Second)  // Cap at 30s
    jitter := rand.N(backoff)
    return backoff + jitter
}
```

---

## Stateless Mode

**For simple request/response without session state:**

```go
handler := mcp.NewStreamableHTTPHandler(
    func(req *http.Request) *mcp.Server { return server },
    &mcp.StreamableHTTPOptions{
        Stateless: true,
    },
)
```

**Stateless Behavior:**
- No `Mcp-Session-Id` validation
- Each request treated independently
- Server→Client requests rejected
- Notifications sent in request context only

**Use Cases:**
- Load-balanced deployments
- Serverless functions
- Simple tools without server-initiated messages

---

## JSON vs SSE Response Modes

| | SSE (default) | JSON |
|---|---|---|
| Content-Type | `text/event-stream` | `application/json` |
| Streaming | Yes | No |
| Server push | Yes | No |
| Compatibility | Needs EventSource | Universal |

**To enable JSON mode:**
```go
handler := mcp.NewStreamableHTTPHandler(getServer, &mcp.StreamableHTTPOptions{
    JSONResponse: true,
})
```

Use JSON mode for simpler clients that don't need streaming.

---

## Protocol Version Negotiation

**Location:** `mcp/streamable.go:310-324`

```go
// From spec §2.7:
// Client MUST include Mcp-Protocol-Version header on all requests
// Server validates against supported versions

var supportedProtocolVersions = []string{
    "2025-11-25",  // Latest
    "2025-06-18",
    "2025-03-26",
    "2024-11-05",  // Legacy
}

protocolVersion := req.Header.Get("Mcp-Protocol-Version")
if protocolVersion == "" {
    protocolVersion = "2025-03-26"  // Default for backwards compat
}
if !slices.Contains(supportedProtocolVersions, protocolVersion) {
    http.Error(w, "Unsupported protocol version", http.StatusBadRequest)
    return
}
```

---

## Session Security

**Session hijacking prevention:**

```go
type sessionInfo struct {
    // ...
    userID string  // From TokenInfo at session creation
}

// In ServeHTTP:
if sessInfo.userID != "" {
    tokenInfo := auth.TokenInfoFromContext(req.Context())
    if tokenInfo == nil || tokenInfo.UserID != sessInfo.userID {
        http.Error(w, "session user mismatch", http.StatusForbidden)
        return
    }
}
```

- Session ID alone is not enough
- Bind sessions to authenticated users
- Reject requests from different users

---

## Testing Streamable Transport

**Unit Tests with InMemoryTransport**
```go
func TestStreamable(t *testing.T) {
    server := mcp.NewServer(impl, nil)
    mcp.AddTool(server, tool, handler)

    cTransport, sTransport := mcp.NewInMemoryTransports()

    ss, _ := server.Connect(ctx, sTransport, nil)
    client := mcp.NewClient(impl, nil)
    cs, _ := client.Connect(ctx, cTransport, nil)

    result, _ := cs.CallTool(ctx, &mcp.CallToolParams{Name: "greet"})
}
```

**Integration Tests with Real HTTP**
```go
handler := mcp.NewStreamableHTTPHandler(...)
ts := httptest.NewServer(handler)
defer ts.Close()

transport := &mcp.StreamableClientTransport{Endpoint: ts.URL}
client := mcp.NewClient(impl, nil)
session, _ := client.Connect(ctx, transport, nil)
```

---

## Common Issues & Debugging

**Server Not Receiving Messages**
- Check `Accept` header includes both `application/json` and `text/event-stream`
- Verify `Content-Type: application/json` on POST body

**Session Not Found (404)**
- Session may have timed out (`SessionTimeout` option)
- Session may have been deleted by another request
- Check `Mcp-Session-Id` header is being sent

**Reconnection Failing**
- Check `Last-Event-ID` format matches `{streamID}_{index}`
- Verify EventStore is configured
- Check server hasn't garbage-collected events

**Debug Logging**
```go
handler := mcp.NewStreamableHTTPHandler(getServer, &mcp.StreamableHTTPOptions{
    Logger: slog.Default(),  // Enable logging
})
```

---

## Summary

**Tool Binding**
- Two-tier API: low-level `ToolHandler` vs generic `ToolHandlerFor`
- Schema inference from Go struct tags
- Validation and default values via jsonschema
- Tool errors vs protocol errors distinction

**Streamable Transport**
- HTTP endpoint with SSE for streaming
- Per-session transport management
- Multiple logical streams per session
- Reconnection and resumption support
- Stateful and stateless modes

---

## Resources

- **Tool Binding:** `mcp/tool.go`, `mcp/server.go:278-400`
- **Streamable Transport:** `mcp/streamable.go`
- **JSON Schema:** https://pkg.go.dev/github.com/google/jsonschema-go/jsonschema
- **MCP Streamable Spec:** https://modelcontextprotocol.io/specification/2025-11-25/basic/transports

---

## Q&A

Questions about tool binding implementation or streamable transport details?
