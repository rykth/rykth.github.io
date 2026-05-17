---
title: "Model Context Protocol (MCP)"
date: 2025-05-17T12:19:58+05:30
description: "MCP is an open protocol that standardizes how applications provide context to LLMs."
tags: [LLM, MCP, golang]
---

["MCP is an open protocol that standardizes how applications provide context to LLMs. Think of MCP like a USB-C port for AI applications. Just as USB-C provides a standardized way to connect your devices to various peripherals and accessories, MCP provides a standardized way to connect AI models to different data sources and tools."](https://modelcontextprotocol.io/introduction)

## Overview

The Model Context Protocol (MCP) is an open standard that makes it easy for
developers to connect their data with AI tools in a secure, two-way setup. It
works in a simple way: you can either share your data by running an MCP server,
or create AI tools (called MCP clients) that connect to those servers to use the
data.

At the core of today's AI breakthroughs is a powerful tool called a Large
Language Model (LLM). These models work in a surprisingly simple yet effective
way: they take in a piece of text—called a prompt—and generate a response based
on patterns they've learned from vast amounts of data. Think of it as a smart
text generator: you give it input, and it predicts the most logical output.
Despite their complexity under the hood, LLMs essentially operate on a
straightforward principle—text in, text out.

Large Language Models (LLMs) don't have access to live or external data
sources—they only know what they were trained on. That means they don't
automatically know things like your name or current events. If you want the LLM
to know your name, you have to tell it directly, for example by saying, "What's
my name? My name is Alex." The model will then use that context to answer
correctly. Similarly, if you ask, "What are the trending repositories this
week?", the LLM won't be able to answer on its own. Instead, it may recognize
that it needs help and ask to use a tool or connect to a service to find that
information.

The Model Context Protocol (MCP) helps standardize how AI tools and data sources
communicate. It makes it easy to build clients and servers that can talk to each
other in a consistent way. Since MCP is an open specification, developers can
create tools—like SDKs, clients, and servers—in any programming language they
prefer. The best part is that these tools are interchangeable. No matter which
LLM provider or application you use, as long as they follow the MCP standard,
everything will work together smoothly.

## Architecture

Host is the main app or environment where the AI runs and interacts with the
user. Think of it like the "home base" for the AI assistant. This could be
something like a code editor, a desktop AI app, or even a chat interface. The
host is responsible for managing the conversation with the AI, sending and
receiving messages, and connecting to different MCP clients or servers when the
AI needs extra information—like files, tools, or data. It's the central place
where everything comes together.

In MCP, a host is any app that can work with an MCP server—like a code editor,
chat interface, or AI assistant platform. The host can manage multiple clients,
and each client connects directly to one MCP server. These servers might link to
external APIs, databases, or other data sources. What's powerful is that each
MCP server provides a list of tools it supports, which the client can share with
the host. For example, if you want to find trending GitHub repositories, you
could connect a "trending repositories" MCP server. The host would then know
about this tool and could use it anytime the AI gets a question it can't answer
on its own. It simply checks the list of tools and sees if one of them can help
based on the tool's description—making the AI smarter and more resourceful.

In the Model Context Protocol (MCP), communication between components happens
through two main methods: stdio and SSE (Server-Sent Events). These are just
different ways for MCP clients and servers to talk to each other. Stdio is a
simple, stream-based method where messages are sent through standard input and
output—great for local tools or CLI-based servers. On the other hand, SSE is
used for real-time, one-way communication over HTTP, allowing servers to push
updates to clients as they happen. This is especially useful for web-based tools
or when working with long-running tasks. By supporting both stdio and SSE, MCP
gives developers the flexibility to build and connect tools in the way that best
fits their environment—whether it's a lightweight local script or a cloud-based
API.


![screenshot](/images/mcp-arch.png "screenshot")


### Protocol layer

In the Model Context Protocol (MCP), the protocol layer is the set of rules that
defines how clients and servers communicate and understand each other. It
ensures that messages, requests, and responses follow a clear, consistent format
so both sides know what to expect. This layer handles things like how tools are
described, how data is requested, and how results are returned. By having a
standardized protocol layer, different MCP clients and servers—no matter who
builds them—can work together smoothly and reliably. This makes it easier for
developers to build AI tools that connect to a variety of data sources without
needing custom solutions for each one.

From the MCP documentation:
```ts
class Protocol<Request, Notification, Result> {
    // Handle incoming requests
    setRequestHandler<T>(schema: T, handler: (request: T, extra: RequestHandlerExtra) => Promise<Result>): void

    // Handle incoming notifications
    setNotificationHandler<T>(schema: T, handler: (notification: T) => Promise<void>): void

    // Send requests and await responses
    request<T>(request: Request, schema: T, options?: RequestOptions): Promise<T>

    // Send one-way notifications
    notification(notification: Notification): Promise<void>
}
```

### Transport layer

The transport layer in the Model Context Protocol (MCP) is responsible for how
data actually moves between the clients and servers. It defines the technical
methods used to send and receive messages so both sides stay connected and
understand each other. MCP supports different transport options, like using
standard input/output (stdio) for simple, local communication, or HTTP combined
with Server-Sent Events (SSE) for web-based, real-time interactions. This
flexibility allows MCP to work in many environments—from lightweight local tools
to complex cloud services—making it easier to build and scale AI-powered
applications.

### Message types

In the Model Context Protocol (MCP), message types define the different kinds of
communication that happen between clients and servers. These messages help
organize the conversation, so both sides know what's being asked or shared.
Common message types include requests for information, responses with data,
notifications about changes, and error messages. By clearly defining these
types, MCP ensures that interactions are smooth and predictable, allowing AI
tools to request the right data, understand replies correctly, and handle any
issues gracefully. This structured messaging helps make AI assistants more
reliable and effective when working with external data and services.

From the MCP documentation:
```ts
interface Request {
  method: string;
  params?: { ... };
}
```

```ts
interface Result {
  [key: string]: unknown;
}
```

```ts
interface Error {
  code: number;
  message: string;
  data?: unknown;
}
```

```ts
interface Notification {
  method: string;
  params?: { ... };
}
```

### Connection lifecycle

The connection lifecycle in the Model Context Protocol (MCP) is the process that
manages how a client and server start, communicate, and end their interaction.
It begins when the client opens a connection to the server and they introduce
themselves by sharing information about the tools and capabilities available.
During this time, the client can send requests and receive responses, keeping
the conversation going smoothly. When the task is complete or no longer needed,
the connection is closed properly to free up resources. This careful management
of connections helps keep AI applications running efficiently and ensures
reliable communication between different parts of the system.

## MCP Server Example in Golang: Hacker News Integration

This example demonstrates how to create a Model Context Protocol (MCP) server in
Golang that provides tools for fetching stories from Hacker News. The server
implements two main tools:
- Fetching top stories
- Fetching new stories

Will use only a `tool` as a specific capability or service that an MCP server
offers to help the AI assistant perform tasks. Think of tools as specialized
helpers—like searching files, accessing databases, or calling external APIs—that
the AI can use when it needs information or wants to take action beyond its own
knowledge. The server provides a list of these tools to the client, which then
shares them with the host. When the AI encounters a question it can't answer on
its own, it looks through these tools and decides if one of them can help. This
makes the AI smarter and more useful by letting it tap into real-time data and
powerful resources.

We'll use [mcp-go](https://github.com/mark3labs/mcp-go) library from
[mark3labs](https://github.com/mark3labs). Please give a _star_ to that
repository.

### Architecture Overview

The application is structured into several packages:

```
pkg/
  ├── hn/          # Hacker News client implementation
  └── server/      # MCP server implementation and tools
cmd/
  └── main.go      # Server entry point
```

### Implementation Details

#### 1. Hacker News Client (`pkg/hn/client.go`)

The client package provides functionality to interact with the Hacker News API. It implements:

- `TopStories(number int)` - Fetches the top N stories from Hacker News
- `NewStories(number int)` - Fetches the latest N stories from Hacker News

Each story contains:
```go
type Story struct {
    By          string
    Descendants int
    Id          int
    Kids        []int
    Score       int
    Time        int
    Title       string
    Url         string
}
```

#### 2. MCP Server Implementation (`pkg/server/`)

The server package implements two MCP tools:

1. `get_hn_top_stories` - Tool for fetching top stories
2. `get_hn_new_stories` - Tool for fetching new stories

Each tool is implemented with:
- A tool definition (using `mcp.NewTool`)
- A handler function that processes requests
- Parameter validation
- Error handling

Example tool implementation:
```go
func GetTopStories(client topStoriesClient) (mcp.Tool, server.ToolHandlerFunc) {
	readOnlyHint := true
	tool := mcp.NewTool("get_hn_top_stories", mcp.WithDescription("Fetch the top stories ..."),
		mcp.WithToolAnnotation(mcp.ToolAnnotation{
			Title:        "Get top Hacker News stories",
			ReadOnlyHint: &readOnlyHint,
		}),
		mcp.WithNumber("number", mcp.Required(), mcp.Description("The number of top stories to retrieve from Hacker News (maximum 500).")),
	)

	handler := func(ctx context.Context, request mcp.CallToolRequest) (*mcp.CallToolResult, error) {
		...
	}

	return tool, handler
}
```

```go
tool := mcp.NewTool("get_hn_top_stories", 
    mcp.WithDescription("Fetch the top stories from Hacker News..."),
```

- Creates a new MCP tool with a unique identifier "get_hn_top_stories"
- Provides a human-readable description of the tool's purpose

The `description` of an MCP tool plays a key role in helping the AI assistant
understand what the tool does and when to use it. When an MCP server shares its
tools with the client, each tool comes with a clear description explaining its
purpose and capabilities. The client then passes this information to the host,
which uses it to decide which tool might help answer a user's question. For
example, if the AI doesn't know the answer, it can look through the tool
descriptions to find one that matches the topic—like a tool for searching
trending GitHub repositories or accessing a company's database. By reading these
descriptions, the AI can choose the right tool to get accurate, relevant
information and provide better responses.


```go
mcp.WithToolAnnotation(mcp.ToolAnnotation{
    Title:        "Get top Hacker News stories",
    ReadOnlyHint: &readOnlyHint,
}),
```
- Adds metadata annotations to the tool
- Sets a display title for the tool
- Applies the read-only hint for security purposes

In MCP, `annotations` are extra pieces of information added to messages or data
that help AI models better understand the context or meaning. They act like
helpful notes or labels that clarify what a certain part of the text or data
represents. For example, an annotation might highlight that a specific word is a
file name, a command, or a variable. By using annotations, MCP enables AI
assistants to interpret information more accurately, respond smarter, and
perform tasks more effectively—especially when working with complex data or
code. This makes the interaction between AI and external tools clearer and more
precise.


```go
mcp.WithNumber("number", 
    mcp.Required(), 
    mcp.Description("The number of top stories to retrieve...")),
```
- Defines a required numeric parameter named "number"
- Marks the parameter as required using `mcp.Required()`
- Provides a description of what the parameter does

In the Model Context Protocol (MCP), `parameters` are the specific pieces of
information or settings that are passed along with requests and commands between
clients and servers. They help define exactly what the AI or the tool needs to
do. For example, when asking a tool to search files, parameters might include
the search keywords, file types, or folders to look in. These parameters ensure
that requests are clear and precise, so the tool can return the right data or
perform the correct action. By using well-defined parameters, MCP makes
communication between AI assistants and external tools more effective and
reliable.

```go
handler := func(ctx context.Context, request mcp.CallToolRequest) (*mcp.CallToolResult, error) {
```
- Defines the handler function that processes tool requests
- Takes a context and the tool request as parameters
- Returns either a result or an error

In MCP, a `handler` is a part of the system—usually within the client or
server—that processes specific types of messages or requests. Think of a handler
like a specialized worker who knows exactly how to deal with certain tasks, such
as fetching data, running a tool, or managing connections. When a message
arrives, the handler reads it, understands what needs to be done, and then
carries out the appropriate action. Handlers help organize the communication in
MCP, making sure each request is dealt with quickly and correctly so the AI
assistant can get the right information or perform the right task.

## Running the Server

Please check out [go-mcp](https://github.com/rykth/go-mcp) repository for
full implementation and documentation.

![mcp example](/images/go-mcp3.gif "mcp example")

### References

* ["Model Context Protocol"](https://modelcontextprotocol.io/introduction)    
* ["mcp-go repository"](https://github.com/mark3labs/mcp-go)    
* ["Use MCP servers in VS Code (Preview)"](https://code.visualstudio.com/docs/copilot/chat/mcp-servers)