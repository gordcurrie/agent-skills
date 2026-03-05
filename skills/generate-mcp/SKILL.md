---
name: generate-mcp
description: >
  Step-by-step guide for building a production-quality Go MCP server that wraps
  an external API or service. Use this skill when asked to create, scaffold, or
  implement a new MCP server in Go — from initial API research through to CI and
  release automation.
license: MIT
---

# Skill: Generate a Go MCP Server

This skill walks through building a complete Go MCP server using the official
`modelcontextprotocol/go-sdk`. It is based on the patterns established in
[gordcurrie/unifi-mcp](https://github.com/gordcurrie/unifi-mcp) and covers every
phase from API research to GitHub release automation.

---

## Phase 0 — Understand the target API

Before writing any code, research the API or service you are wrapping.

1. **Authentication** — determine the auth mechanism (API key header, OAuth, basic auth, mTLS).  
   Prefer API keys over OAuth for MCP server use; they need no browser flow.

2. **Base URL shape** — identify whether endpoints follow a consistent prefix  
   (e.g. `/api/v1/`, `/proxy/network/integration/v1/`) and whether the site or  
   tenant is part of the path.

3. **Pagination** — check if list endpoints accept `offset`+`limit` or cursor-based  
   pagination. `offset`/`limit` maps nicely to the `Page[T]` generic pattern.

4. **Response envelopes** — note whether responses are bare arrays/objects or wrapped  
   (e.g. `{"data": [...], "totalCount": N}`). Design your `apiResponse[T]` decoder  
   around the actual shape.

5. **Write/mutation patterns** — identify POST/PUT/PATCH/DELETE semantics.  
   Check if the API returns the updated resource or just an HTTP status.

6. **Error format** — understand what a 4xx/5xx response body looks like so you  
   can surface useful error messages.

7. **TLS** — note whether the service uses self-signed certificates that users  
   may need to skip verification for.

8. **Grouping** — cluster related endpoints into logical tool groups  
   (sites, devices, clients, network, etc.). Each group becomes one file in  
   `internal/<pkg>/` and one file in `tools/`.

Document your findings in `PLAN.md` before proceeding.

---

## Phase 1 — Repository scaffolding

### 1.1 Directory layout

```
cmd/<server-name>/main.go      # entrypoint
internal/<pkg>/
    client.go                  # HTTP client + constructor
    types.go                   # all API response/request types
    <group>.go                 # one file per endpoint group
    <group>_test.go
tools/
    client_iface.go            # interface the tools layer depends on
    helpers.go                 # jsonResult / textResult / errorResult
    register.go                # RegisterAll wires every group
    <group>.go                 # one file per tool group
.github/
    workflows/
        ci.yml
        release.yml
    copilot-instructions.md
Makefile
.golangci.yml
go.mod
README.md
PLAN.md
.env.example
.gitignore
```

### 1.2 `go.mod`

```
module github.com/<owner>/<server-name>

go <version>

require (
    github.com/modelcontextprotocol/go-sdk <latest>
)
```

Fetch the latest `go-sdk` version: `go get github.com/modelcontextprotocol/go-sdk@latest`.

### 1.3 `.gitignore`

```
bin/
.env
```

### 1.4 `.env.example`

```
# Copy to .env and fill in values
<SERVICE>_BASE_URL=https://
<SERVICE>_API_KEY=
<SERVICE>_SITE_ID=          # omit if the API has no site concept
<SERVICE>_INSECURE=false
<SERVICE>_ALLOW_DESTRUCTIVE=false
```

---

## Phase 2 — Internal client package

### 2.1 `internal/<pkg>/client.go`

Follow this structure exactly:

```go
// Package <pkg> provides a client for the <ServiceName> API.
package <pkg>

import (
    "bytes"
    "context"
    "crypto/tls"
    "encoding/json"
    "errors"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "strconv"
    "strings"
    "time"
)

const maxResponseBytes = 10 << 20 // 10 MiB
const maxPageLimit = 1000

// Client is a <ServiceName> API client.
type Client struct {
    baseURL    string
    apiKey     string
    siteID     string       // omit field if the API has no site concept
    httpClient *http.Client
}

// NewClient creates a <ServiceName> API client.
// Set insecure to true to skip TLS verification for self-signed certificates.
func NewClient(baseURL, apiKey, siteID string, insecure bool) (*Client, error) {
    if baseURL == "" {
        return nil, errors.New("<SERVICE>_BASE_URL is required")
    }
    if apiKey == "" {
        return nil, errors.New("<SERVICE>_API_KEY is required")
    }

    transport := http.DefaultTransport
    if insecure {
        if base, ok := http.DefaultTransport.(*http.Transport); ok {
            t := base.Clone()
            if t.TLSClientConfig == nil {
                t.TLSClientConfig = &tls.Config{MinVersion: tls.VersionTLS12}
            }
            //nolint:gosec // InsecureSkipVerify is only set when <SERVICE>_INSECURE=true
            t.TLSClientConfig.InsecureSkipVerify = true // #nosec G402
            transport = t
        }
    }

    return &Client{
        baseURL: strings.TrimRight(baseURL, "/"),
        apiKey:  apiKey,
        siteID:  siteID,
        httpClient: &http.Client{
            Transport: transport,
            Timeout:   30 * time.Second,
        },
    }, nil
}

// site returns siteID if non-empty, otherwise the client default.
func (c *Client) site(siteID string) string {
    if siteID != "" {
        return siteID
    }
    return c.siteID
}

// get performs a GET request to the given path (relative to baseURL).
func (c *Client) get(ctx context.Context, path string) ([]byte, error) {
    return c.do(ctx, http.MethodGet, path, nil)
}

// getWithQuery performs a GET with optional offset/limit query parameters.
func (c *Client) getWithQuery(ctx context.Context, path string, offset, limit int) ([]byte, error) {
    if offset < 0 || limit < 0 {
        return nil, fmt.Errorf("getWithQuery: offset and limit must be >= 0")
    }
    if limit > maxPageLimit {
        return nil, fmt.Errorf("getWithQuery: limit must be <= %d (got %d)", maxPageLimit, limit)
    }
    if offset == 0 && limit == 0 {
        return c.get(ctx, path)
    }
    q := url.Values{}
    if offset > 0 {
        q.Set("offset", strconv.Itoa(offset))
    }
    if limit > 0 {
        q.Set("limit", strconv.Itoa(limit))
    }
    sep := "?"
    if strings.Contains(path, "?") {
        sep = "&"
    }
    return c.get(ctx, path+sep+q.Encode())
}

// post performs a POST with no body.
func (c *Client) post(ctx context.Context, path string) ([]byte, error) {
    return c.do(ctx, http.MethodPost, path, nil)
}

// postWithBody performs a POST with a JSON-encoded body.
func (c *Client) postWithBody(ctx context.Context, path string, body any) ([]byte, error) {
    b, err := json.Marshal(body)
    if err != nil {
        return nil, fmt.Errorf("marshal request: %w", err)
    }
    return c.do(ctx, http.MethodPost, path, bytes.NewReader(b))
}

// put performs a PUT with a JSON-encoded body.
func (c *Client) put(ctx context.Context, path string, body any) ([]byte, error) {
    b, err := json.Marshal(body)
    if err != nil {
        return nil, fmt.Errorf("marshal request: %w", err)
    }
    return c.do(ctx, http.MethodPut, path, bytes.NewReader(b))
}

// delete performs a DELETE request.
func (c *Client) delete(ctx context.Context, path string) ([]byte, error) {
    return c.do(ctx, http.MethodDelete, path, nil)
}

func (c *Client) do(ctx context.Context, method, path string, body io.Reader) ([]byte, error) {
    req, err := http.NewRequestWithContext(ctx, method, c.baseURL+path, body)
    if err != nil {
        return nil, fmt.Errorf("new request: %w", err)
    }
    req.Header.Set("X-API-Key", c.apiKey)
    if body != nil {
        req.Header.Set("Content-Type", "application/json")
    }

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("do request: %w", err)
    }
    defer resp.Body.Close() //nolint:errcheck

    data, err := io.ReadAll(io.LimitReader(resp.Body, maxResponseBytes))
    if err != nil {
        return nil, fmt.Errorf("read body: %w", err)
    }

    if resp.StatusCode < 200 || resp.StatusCode >= 300 {
        return nil, fmt.Errorf("HTTP %d: %s", resp.StatusCode, string(data))
    }
    return data, nil
}

// apiResponse is a generic wrapper for v1 REST responses of the form {"data": T}.
type apiResponse[T any] struct {
    Data T `json:"data"`
}

// decode unmarshals raw bytes into an apiResponse[T] and returns the inner Data.
func decode[T any](raw []byte) (T, error) {
    var envelope apiResponse[T]
    if err := json.Unmarshal(raw, &envelope); err != nil {
        var zero T
        return zero, fmt.Errorf("decode response: %w", err)
    }
    return envelope.Data, nil
}
```

### 2.2 `internal/<pkg>/types.go`

- Define `Page[T]` for all paginated list responses.
- One struct per distinct API resource.
- All fields use `json` struct tags.
- Unexported request body types (`lowerCamelRequest`) are fine; keep them in the file that uses them.

```go
// Page wraps a paginated list response.
type Page[T any] struct {
    Data       []T `json:"data"`
    TotalCount int `json:"totalCount"`
    Offset     int `json:"offset"`
    Limit      int `json:"limit"`
    Count      int `json:"count"`
}
```

### 2.3 Group files (`internal/<pkg>/<group>.go`)

One exported method per API operation. Follow the consistent error-wrapping pattern:

```go
// ListWidgets returns a page of widgets for the given site.
func (c *Client) ListWidgets(ctx context.Context, siteID string, offset, limit int) (Page[Widget], error) {
    path := fmt.Sprintf("/integration/v1/sites/%s/widgets", c.site(siteID))
    raw, err := c.getWithQuery(ctx, path, offset, limit)
    if err != nil {
        return Page[Widget]{}, fmt.Errorf("ListWidgets %s: %w", siteID, err)
    }
    return decode[Page[Widget]](raw)
}
```

---

## Phase 3 — Tools layer

### 3.1 `tools/client_iface.go`

Define an interface that exactly mirrors the methods the tools layer calls.
This enables testing with mocks without depending on the real HTTP client.

```go
package tools

import "context"
import "<module>/internal/<pkg>"

// serviceClient is the interface the tools layer requires.
// *<pkg>.Client satisfies this automatically.
type serviceClient interface {
    ListWidgets(ctx context.Context, siteID string, offset, limit int) (<pkg>.Page[<pkg>.Widget], error)
    GetWidget(ctx context.Context, siteID, widgetID string) (<pkg>.Widget, error)
    // ... one entry per method called by any tool
}
```

### 3.2 `tools/helpers.go`

```go
package tools

import (
    "encoding/json"
    "fmt"
    "github.com/modelcontextprotocol/go-sdk/mcp"
)

// jsonResult marshals v to a JSON TextContent result.
func jsonResult(v any) (*mcp.CallToolResult, any, error) {
    b, err := json.MarshalIndent(v, "", "  ")
    if err != nil {
        return errorResult(fmt.Errorf("marshal result: %w", err))
    }
    return &mcp.CallToolResult{
        Content: []mcp.Content{&mcp.TextContent{Text: string(b)}},
    }, nil, nil
}

// textResult wraps a plain string in a TextContent result.
func textResult(s string) (*mcp.CallToolResult, any, error) {
    return &mcp.CallToolResult{
        Content: []mcp.Content{&mcp.TextContent{Text: s}},
    }, nil, nil
}

// errorResult returns a tool-execution error as an isError:true result.
func errorResult(err error) (*mcp.CallToolResult, any, error) {
    msg := "unknown error"
    if err != nil {
        msg = err.Error()
    }
    return &mcp.CallToolResult{
        Content: []mcp.Content{&mcp.TextContent{Text: msg}},
        IsError: true,
    }, nil, nil
}
```

### 3.3 Tool registration pattern (`tools/<group>.go`)

```go
package tools

import (
    "context"
    "fmt"
    "github.com/modelcontextprotocol/go-sdk/mcp"
)

func registerWidgetTools(s *mcp.Server, client serviceClient) {
    // --- Input struct scoped tightly inside the register function ---
    type listInput struct {
        SiteID string `json:"site_id,omitempty" jsonschema:"site ID; omit to use default"`
        Offset int    `json:"offset,omitempty"  jsonschema:"pagination offset"`
        Limit  int    `json:"limit,omitempty"   jsonschema:"max results (≤ 1000)"`
    }

    mcp.AddTool(s, &mcp.Tool{
        Name:        "list_widgets",
        Description: "List widgets for a site.",
        Annotations: &mcp.ToolAnnotations{ReadOnlyHint: true},
    }, func(ctx context.Context, _ *mcp.CallToolRequest, input listInput) (*mcp.CallToolResult, any, error) {
        page, err := client.ListWidgets(ctx, input.SiteID, input.Offset, input.Limit)
        if err != nil {
            return nil, nil, fmt.Errorf("list_widgets: %w", err)
        }
        return jsonResult(page)
    })

    // --- Destructive tools (only call this sub-function when allowDestructive=true) ---
}

func registerDestructiveWidgetTools(s *mcp.Server, client serviceClient) {
    destructiveTrue := true

    type deleteInput struct {
        WidgetID  string `json:"widget_id"  jsonschema:"ID of the widget to delete"`
        Confirmed bool   `json:"confirmed"  jsonschema:"must be true to proceed"`
    }

    mcp.AddTool(s, &mcp.Tool{
        Name:        "delete_widget",
        Description: "Permanently delete a widget.",
        Annotations: &mcp.ToolAnnotations{DestructiveHint: &destructiveTrue},
    }, func(ctx context.Context, _ *mcp.CallToolRequest, input deleteInput) (*mcp.CallToolResult, any, error) {
        if !input.Confirmed {
            return errorResult(fmt.Errorf("delete_widget: set confirmed=true to proceed"))
        }
        if err := client.DeleteWidget(ctx, "", input.WidgetID); err != nil {
            return nil, nil, fmt.Errorf("delete_widget: %w", err)
        }
        return textResult("widget deleted")
    })
}
```

### 3.4 `tools/register.go`

```go
package tools

import "github.com/modelcontextprotocol/go-sdk/mcp"

// RegisterAll registers every enabled tool group with the MCP server.
func RegisterAll(s *mcp.Server, client serviceClient, allowDestructive bool) {
    registerSiteTools(s, client)
    registerWidgetTools(s, client)
    if allowDestructive {
        registerDestructiveWidgetTools(s, client)
    }
}
```

---

## Phase 4 — Entrypoint (`cmd/<server-name>/main.go`)

```go
// Command <server-name> is an MCP server for <ServiceName>.
package main

import (
    "context"
    "errors"
    "flag"
    "fmt"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/modelcontextprotocol/go-sdk/mcp"

    "<module>/internal/<pkg>"
    "<module>/tools"
)

var version = "dev"

func main() {
    if err := run(); err != nil {
        slog.Error("fatal", "err", err)
        os.Exit(1)
    }
}

func run() error {
    var transport, addr string
    flag.StringVar(&transport, "transport", "stdio", "stdio or http")
    flag.StringVar(&addr, "addr", "127.0.0.1:8080", "listen address for http transport")
    flag.Parse()

    baseURL          := os.Getenv("<SERVICE>_BASE_URL")
    apiKey           := os.Getenv("<SERVICE>_API_KEY")
    siteID           := os.Getenv("<SERVICE>_SITE_ID")
    insecure         := os.Getenv("<SERVICE>_INSECURE") == "true"
    allowDestructive := os.Getenv("<SERVICE>_ALLOW_DESTRUCTIVE") == "true"

    client, err := <pkg>.NewClient(baseURL, apiKey, siteID, insecure)
    if err != nil {
        return fmt.Errorf("client: %w", err)
    }

    s := mcp.NewServer(&mcp.Implementation{Name: "<server-name>", Version: version}, nil)
    tools.RegisterAll(s, client, allowDestructive)

    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    switch transport {
    case "stdio":
        if err := s.Run(ctx, &mcp.StdioTransport{}); err != nil && !errors.Is(err, context.Canceled) {
            return fmt.Errorf("stdio: %w", err)
        }
    case "http":
        httpServer := &http.Server{
            Addr:              addr,
            Handler:           http.MaxBytesHandler(mcp.NewStreamableHTTPHandler(func(_ *http.Request) *mcp.Server { return s }, nil), 4<<20),
            ReadHeaderTimeout: 10 * time.Second,
            ReadTimeout:       30 * time.Second,
            WriteTimeout:      30 * time.Second,
            IdleTimeout:       120 * time.Second,
            MaxHeaderBytes:    1 << 20,
        }
        go func() {
            <-ctx.Done()
            shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
            defer cancel()
            _ = httpServer.Shutdown(shutdownCtx)
        }()
        slog.Warn("HTTP transport has no authentication")
        slog.Info("listening", "addr", addr)
        if err := httpServer.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            return fmt.Errorf("http: %w", err)
        }
    default:
        return fmt.Errorf("unknown transport %q", transport)
    }
    return nil
}
```

---

## Phase 5 — Testing

### 5.1 Client tests (`internal/<pkg>/<group>_test.go`)

Use `httptest.NewTLSServer` to test the real HTTP client code paths without hitting live APIs.

```go
package <pkg>_test

import (
    "context"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "<module>/internal/<pkg>"
)

func TestListWidgets(t *testing.T) {
    want := <pkg>.Page[<pkg>.Widget]{
        Data:       []<pkg>.Widget{{ID: "w1", Name: "Foo"}},
        TotalCount: 1,
        Count:      1,
    }
    srv := httptest.NewTLSServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path != "/integration/v1/sites/site1/widgets" {
            http.NotFound(w, r)
            return
        }
        w.Header().Set("Content-Type", "application/json")
        _ = json.NewEncoder(w).Encode(map[string]any{"data": want})
    }))
    defer srv.Close()

    c, err := <pkg>.NewTestClient(srv)    // see note below
    if err != nil {
        t.Fatal(err)
    }

    got, err := c.ListWidgets(context.Background(), "site1", 0, 0)
    if err != nil {
        t.Fatal(err)
    }
    if len(got.Data) != 1 || got.Data[0].ID != "w1" {
        t.Fatalf("unexpected result: %+v", got)
    }
}
```

> **Test helper**: add an exported `NewTestClient(srv *httptest.Server) (*Client, error)` function in a `export_test.go` file (build tag `_test`) that creates a real client pointed at the test server with TLS verification disabled.

Use **table-driven tests** with `t.Run` for methods that have multiple cases (pagination, error paths, edge cases).

### 5.2 What to cover

- Happy path for every list/get method.
- Pagination: verify `offset` and `limit` query parameters are forwarded correctly.
- Error handling: HTTP 4xx/5xx → wrapped error returned to caller.
- Auth: verify the `X-API-Key` header is set on every request.
- Limit guard: verify requests with `limit > 1000` are rejected before the HTTP call.

---

## Phase 6 — Makefile

```makefile
BINARY := bin/<server-name>
CMD    := ./cmd/<server-name>

GOFUMPT_VERSION      := v0.7.0
GOSEC_VERSION        := v2.22.8
GOVULNCHECK_VERSION  := v1.1.4
GOLANGCILINT_VERSION := v2.10.1

.PHONY: all install-tools fix fmt vet lint sec vulncheck test build check clean

all: check

install-tools:
	go install mvdan.cc/gofumpt@$(GOFUMPT_VERSION)
	go install github.com/securego/gosec/v2/cmd/gosec@$(GOSEC_VERSION)
	go install golang.org/x/vuln/cmd/govulncheck@$(GOVULNCHECK_VERSION)
	go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@$(GOLANGCILINT_VERSION)

fix:   ; go fix ./...
fmt:   ; gofumpt -w .
vet:   ; go vet ./...
lint:  ; golangci-lint run ./...
sec:   ; gosec ./...
vulncheck: ; govulncheck ./...
test:  ; go test -race -count=1 ./...
build:
	@mkdir -p bin
	go build -ldflags "-X main.version=$$(git describe --tags --always --dirty 2>/dev/null || echo dev)" -o $(BINARY) $(CMD)

check: fix fmt vet lint sec vulncheck test build

clean: ; rm -f $(BINARY)
```

---

## Phase 6.5 — Linter configuration

### `.golangci.yml`

Commit a `.golangci.yml` at the repo root. Start from this baseline (derived from
[gordcurrie/unifi-mcp](https://github.com/gordcurrie/unifi-mcp)) and adjust as needed:

```yaml
version: "2"

run:
  timeout: 5m

linters:
  default: none
  enable:
    - bodyclose      # response body must be closed
    - errcheck       # all errors must be handled
    - gocritic       # opinionated style + performance checks
    - govet          # go vet passes
    - ineffassign    # no ineffectual assignments
    - misspell       # no misspelled comments/strings
    - noctx          # HTTP requests must carry a context
    - prealloc       # suggest slice pre-allocation
    - revive         # idiomatic Go style
    - staticcheck    # advanced static analysis
    - unconvert      # no unnecessary type conversions
    - unparam        # no parameters that are always the same value
    - gosec          # security checks

  settings:
    gocritic:
      enabled-tags:
        - diagnostic
        - performance
        - style
    revive:
      rules:
        - name: exported
          disabled: false
    gosec:
      excludes: []

formatters:
  enable:
    - gofumpt
  settings:
    gofumpt:
      extra-rules: true
```

> **Why `default: none` + explicit `enable`?** An allowlist means new linters added in
> future golangci-lint releases don't silently appear and break CI. You opt in
> deliberately.

### `//nolint` policy

`//nolint` directives are a last resort, not a convenience. Follow this policy:

**Always prefer fixing the issue over suppressing it.** The only broadly acceptable
suppressions are for *genuine false positives* or *intentional security trade-offs*
that the codebase documents explicitly.

**Rules:**

1. **Always include a comment explaining why.** A bare `//nolint:gosec` is not acceptable.
   The comment must explain the reasoning for a human reviewer, not just satisfy the tool.

   ```go
   // OK — explains why the flag is safe and when it is set
   //nolint:gosec // G402: InsecureSkipVerify is only set when <SERVICE>_INSECURE=true, explicit user opt-in
   t.TLSClientConfig.InsecureSkipVerify = true // #nosec G402

   // NOT OK — no explanation
   //nolint:gosec
   t.TLSClientConfig.InsecureSkipVerify = true
   ```

2. **Name the specific linter(s).** Use `//nolint:gosec` not `//nolint`.
   Suppressing all linters on a line hides future findings.

3. **Use line-level suppression, not file-level.** Never put `//nolint` at the top of
   a file unless the entire file is generated code.

4. **Never suppress `errcheck` without a deliberate reason.** If you are genuinely
   discarding an error (e.g. `defer resp.Body.Close()`), add a comment:

   ```go
   defer resp.Body.Close() //nolint:errcheck // response body close error on read path is non-actionable
   ```

5. **`gosec` suppressions need both annotations.** `gosec` uses its own `// #nosec GN`
   marker in addition to the golangci-lint `//nolint` directive:

   ```go
   //nolint:gosec // G402: skipping TLS verification is intentional, opt-in via env var
   t.TLSClientConfig.InsecureSkipVerify = true // #nosec G402
   ```

6. **Review all suppressions in PR.** Any `//nolint` line should be called out
   in the PR description so reviewers can evaluate the justification.

---

## Phase 7 — CI/CD

### 7.1 `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main, "feat/**"]
  pull_request:
    branches: [main]

jobs:
  check:
    name: make check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true
      - run: make install-tools
      - run: make check
      - name: Ensure working tree is clean
        run: |
          git diff --exit-code
          git diff --cached --exit-code
```

### 7.2 `.github/workflows/release.yml`

```yaml
name: Release

on:
  push:
    tags: ["v*"]

jobs:
  build:
    name: Build ${{ matrix.goos }}/${{ matrix.goarch }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - {goos: linux,   goarch: amd64}
          - {goos: linux,   goarch: arm64}
          - {goos: darwin,  goarch: amd64}
          - {goos: darwin,  goarch: arm64}
          - {goos: windows, goarch: amd64}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true
      - name: Build binary
        env:
          GOOS: ${{ matrix.goos }}
          GOARCH: ${{ matrix.goarch }}
          CGO_ENABLED: "0"
        run: |
          EXT=""
          [ "$GOOS" = "windows" ] && EXT=".exe"
          mkdir -p dist
          go build -trimpath -ldflags="-s -w" \
            -o "dist/<server-name>_${GOOS}_${GOARCH}${EXT}" \
            ./cmd/<server-name>
      - uses: actions/upload-artifact@v4
        with:
          name: <server-name>_${{ matrix.goos }}_${{ matrix.goarch }}
          path: dist/<server-name>_${{ matrix.goos }}_${{ matrix.goarch }}*

  release:
    name: Create GitHub Release
    runs-on: ubuntu-latest
    needs: build
    permissions:
      contents: write
      actions: read
    steps:
      - uses: actions/download-artifact@v4
        with:
          path: dist/
          merge-multiple: true
      - uses: softprops/action-gh-release@v2
        with:
          files: dist/*
          generate_release_notes: true
```

---

## Phase 8 — Documentation

### 8.1 `README.md` structure

Every README must include:

1. **One-line description** with links to the MCP spec and the wrapped service.
2. **Tools table** — one row per tool: `| tool_name | description | key parameters |`.
   - Group tools under `### <Group>` headings.
   - Destructive tools get their own `### Destructive (opt-in)` section with a warning block.
3. **Installation** — pre-built binary download table + `build from source` code block.
4. **Configuration** — env vars table with Required column.
5. **Running** — `stdio` and `http` transport examples.
6. **Client configuration** — VS Code, Claude Desktop, and OpenCode JSON snippets.
7. **Development** — `make` targets summary.

### 8.2 `PLAN.md` structure

Track implementation progress as a checklist. Mark phases `✅` when done.

```markdown
# Implementation Plan

## Phase 1 — Scaffolding ✅
- [x] Directory layout
- [x] go.mod + dependencies

## Phase 2 — Client: <group> ✅
- [x] types
- [x] client methods
- [x] tests

## Phase N — ...

---
**Total tools implemented: N**
```

---

## Phase 9 — Git workflow and commit conventions

### First commit

```bash
git init
git add .
python3 -c "open('/tmp/msg.txt','w').write('feat: initial scaffold\n\nAdd directory layout, go.mod, Makefile, CI, and release workflow.\n')"
git commit -F /tmp/msg.txt
```

### Ongoing commits

Always write multi-line commit messages via a temp file to avoid shell quoting issues:

```bash
python3 -c "open('/tmp/msg.txt','w').write('subject line\n\nbody line 1\nbody line 2\n')"
git add . && git commit -F /tmp/msg.txt
```

Never pass multi-line messages with `-m`.

### PR size guideline

Target **< 500 lines changed per PR** (excluding generated files). A good split:
- PR 1: scaffolding + client package (read-only methods)
- PR 2: tools layer (read-only tools)
- PR 3: write/mutation tools (non-destructive)
- PR 4: destructive tools + CI/release automation

### Quality gate before every commit

```bash
make check
```

This runs: `fix → fmt → vet → lint → sec → vulncheck → test → build`.

---

## Checklist — Definition of Done

Before marking any PR ready for review, confirm:

- [ ] `make check` passes with zero warnings
- [ ] Every exported type and function has a doc comment
- [ ] All `context.Context` is the first parameter on any I/O function
- [ ] No `interface{}` — use `any`
- [ ] No `init()` functions
- [ ] No global mutable state
- [ ] Error wrapping: `fmt.Errorf("operation name: %w", err)` everywhere
- [ ] Destructive tools require `confirmed: true` and are gated by `allowDestructive`
- [ ] Table-driven tests cover happy path + pagination + error paths
- [ ] `.golangci.yml` committed; `make lint` passes with zero findings
- [ ] Every `//nolint` directive names the specific linter and includes a reason comment
- [ ] README tools table is up to date
- [ ] PLAN.md phases are marked complete
