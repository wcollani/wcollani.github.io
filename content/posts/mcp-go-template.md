---
title: "A Go Template for MCP Servers: Patterns from 18 Integrations"
date: 2026-06-15
tags: ["go", "mcp", "ai", "homelab", "agents"]
description: "After writing 18 MCP tool integrations in Go, I extracted the pattern into a template. Here's what I learned about structuring a Go MCP server for multiple integrations."
showToc: true
---

MCP (Model Context Protocol) is how AI agents get access to tools and data beyond what's in the context window. You define a server that exposes tools and resources; the AI client connects to it and can call those tools during inference. The Python ecosystem has tutorials and official examples. Go? Much sparser.

When I started wiring my homelab for MCP — connecting Grafana, Prometheus, Plex, Unraid, UniFi, and a dozen other services — I had to figure out the patterns myself. After 18 providers and roughly 4000 lines of Go, I extracted the skeleton into [mcp-go-template](https://github.com/wcollani/mcp-go-template) so you can skip straight to writing the integration.

This post walks through what's in the template and why the design ended up where it did.

## The Problem with Naive Go MCP Servers

The official Go SDK (`github.com/modelcontextprotocol/go-sdk`) is excellent but intentionally low-level. The tutorials show you how to register a single tool directly on a `*mcp.Server`. That works fine for one integration:

```go
s.AddTool(&mcp.Tool{Name: "list_dashboards", ...}, func(ctx context.Context, req *mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    // handler inline
})
```

The problem shows up around integration three. You end up with a `main.go` that's half tool registration boilerplate and half actual logic, the HTTP client and config for each service is scattered across handlers, and there's no clear place to put tests. By integration five you're copying patterns inconsistently.

What you want is a boundary between "wire up this integration" and "call into it."

## The Provider Interface

The core of the template is a single interface:

```go
type Provider interface {
    Name() string

    // Resources (read-only state)
    GetResources() ([]mcp.Resource, error)
    GetResourceContent(uri string) (string, error)
    GetResourceTemplates() ([]mcp.ResourceTemplate, error)

    // Prompts
    GetPrompts() ([]mcp.Prompt, error)
    GetPrompt(name string, arguments map[string]string) (*mcp.GetPromptResult, error)

    // Tools (actions / mutations)
    GetTools() ([]mcp.Tool, error)
    CallTool(name string, arguments map[string]interface{}) (*mcp.CallToolResult, error)
}
```

Every integration — Grafana, Prometheus, Plex, whatever — implements this interface. The server loop calls `GetTools()` and `GetResources()` once at startup to register everything, then delegates `CallTool` and `GetResourceContent` at request time. Adding a new integration is `s.AddProvider(myservice.NewProvider(...))` in main and nothing else.

The interface draws a hard line between **resources** (read-only state — dashboards, status, config) and **tools** (actions that change things). Some MCP clients present these differently, and some restrict tool execution while allowing resource reads. Keeping them separate now saves you from a refactor later.

## A Minimal Provider

The `hello` package in the template is the smallest possible working implementation:

```go
type Provider struct{}

func (h *Provider) Name() string { return "hello" }

func (h *Provider) GetResources() ([]mcp.Resource, error) {
    return []mcp.Resource{
        {
            URI:         "hello://world",
            Name:        "Hello World",
            MIMEType:    "text/plain",
        },
    }, nil
}

func (h *Provider) GetResourceContent(uri string) (string, error) {
    if uri == "hello://world" {
        return "Hello from your MCP server!", nil
    }
    return "", fmt.Errorf("resource not found: %s", uri)
}

func (h *Provider) GetTools() ([]mcp.Tool, error) {
    return []mcp.Tool{
        *mcphelper.NewTool(
            "greet",
            "Return a greeting for the given name.",
            map[string]interface{}{
                "name": map[string]interface{}{
                    "type":     "string",
                    "required": true,
                },
            },
        ),
    }, nil
}

func (h *Provider) CallTool(name string, args map[string]interface{}) (*mcp.CallToolResult, error) {
    if name == "greet" {
        who, _ := args["name"].(string)
        return mcphelper.TextResult(fmt.Sprintf("Hello, %s!", who)), nil
    }
    return mcphelper.ErrorResult(fmt.Errorf("tool not found: %s", name)), nil
}
```

Copy this package, rename it, and you have a working scaffold that compiles, has tests, and satisfies the interface. The only things you add are an HTTP client, the service's configuration, and your actual tool/resource logic.

## Tool Construction

Defining MCP tools requires a JSON Schema object for the input parameters. Writing this by hand in `map[string]interface{}` is verbose and easy to get wrong. The template includes a `NewTool` helper that keeps it readable:

```go
mcphelper.NewTool(
    "list_grafana_dashboards",
    "List Grafana dashboards. Optionally filter by title keyword.",
    map[string]interface{}{
        "query": map[string]interface{}{
            "type":        "string",
            "description": "Filter by dashboard title (optional)",
        },
    },
)
```

Required fields are marked with `"required": true` at the property level — the helper promotes them to the top-level `required` array in the emitted schema automatically. This is a non-standard shorthand but it makes the call sites significantly cleaner, especially for tools with four or five parameters.

Return helpers keep the call sites clean too:

```go
return mcphelper.TextResult("Operation completed successfully.")
return mcphelper.ErrorResult(fmt.Errorf("service unavailable: %w", err))
```

The `ErrorResult` distinction matters. MCP has two levels of error: protocol errors (the session terminates) and tool errors (`IsError: true` inside the result). Tool errors are what you want in almost every case — they're visible to the LLM, which can read the message and decide to retry or tell the user what went wrong. A `return nil, err` kills the session.

## Transport Selection

The server supports both stdio and SSE via a single env var:

```bash
# stdio — works with Claude Desktop, MCP Inspector, local agents
go run ./cmd/server

# SSE — works with remote agents connecting over HTTP
MCP_TRANSPORT=sse PORT=8080 go run ./cmd/server
```

In practice: use stdio for local tooling and development, SSE for anything containerized or accessed remotely. The selection happens in `server.Run()` — the providers don't need to know which transport is active.

The SSE handler includes CORS headers that cover the common cases — Claude's web interface, MCP Inspector, and most self-hosted clients.

## What I'd Do Differently

A few things I wish I'd decided earlier:

**One provider per service, not one per capability.** I initially split Grafana into separate packages for dashboards and alerts. This meant two `AddProvider` calls for one service, two sets of config, and confusion about which package handled cross-cutting concerns. Grouping everything for a service into one package is cleaner.

**Put config in the provider, not in main.** Early versions had `main.go` pulling env vars and passing them as a dozen parameters. Better to give each provider a `NewProviderFromEnv()` constructor that reads its own config, so `main.go` stays readable as the provider count grows.

**Test `CallTool` dispatch, not just individual methods.** The switch statement in `CallTool` is where bugs accumulate — missing cases, wrong argument type assertions. Table-driven tests over `CallTool` with different tool names catch these before the LLM does.

## The Template

Clone it, rename the module, copy `hello/` as your first provider:

```bash
git clone https://github.com/wcollani/mcp-go-template
cd mcp-go-template
# update module name in go.mod, then:
cp -r internal/provider/hello internal/provider/myservice
go test ./...
```

The README walks through the full workflow: adding a provider, registering tools, running locally, and deploying via Docker. CI is wired for lint and tests on every push.

The Go MCP ecosystem is still early. If you're building something with it, this should save you the first few days of pattern-finding.
