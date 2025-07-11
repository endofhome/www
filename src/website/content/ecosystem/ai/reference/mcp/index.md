---
category: Reference
type: ecosystem
tier: pro
ecosystem: http4k AI
title: "MCP SDK"
description: Feature overview of the http4k-ai-mcp-sdk module
aliases:
    - /ecosystem/http4k/reference/mcp/
---

### Installation (Gradle)

```kotlin
dependencies {
    { { < http4k_bom > } }

    // for general MCP server development
    implementation("org.http4k.pro:http4k-ai-mcp-sdk")

    // if you just want to connect to an MCP server
    implementation("org.http4k.pro:http4k-ai-mcp-client")
}
```

## About

The [Model Context Protocol](https://modelcontextprotocol.info/) is an open standard created by Anthropic that defines
how apps can feed information to AI language models. It creates a uniform way to link these models with various data
sources and tools, which streamlines the integration process. MCP services can be deployed in a Server or Serverless
environment.

MCP itself is based on the JSON RPC standard, which is used to communicate between the client and server. Messages are
sent from client to server and then asynchronously from server to client. MCP defines a set of standardised capabilities
which can be provided by the server or client. One use of these capabilities is to allow pre-trained models to have
access to both live data and APIs that can be used by the model to provide answers to user requests. Another use is to
provide agentic behaviour by providing standard communications between several MCP entities.

Currently the MCP standard supports the following transports:

- **HTTP Steaming:** Clients can interact with the MCP server by either a SSE connection (accepting
  `application/event-stream` or via plain HTTP (accepting `application/json`). Stream resumption and replay is supported
  on the SSE connection by calling GET with the `Last-Event-ID` header. All traffic is served by calling the `/mcp`
  endpoint.
- **Server Sent Events + HTTP:** Clients initiate an SSE connection to the server (on `/sse`) which is used to send
  messages to the
  client asynchronously at any time. Further client -> server requests and notifications are sent via HTTP `/messages`
  endpoint, with
  responses being sent back via the SSE.
- **Standard IO:** Clients start a process that communicates with the server via JSON RPC messages via standard input
  and output streams.

The MCP capabilities include:

- **Tools:** are exposed by the server and can be used by the client to perform tasks. The tool consists of a
  description and a JSON Schema definition for the inputs and outputs.
- **Prompts:** given a set of inputs by a client, the server can generate a prompt parameterised to those inputs. This
  allows servers to generate prompts that are tailored to the client's data.
- **Resources:** are exposed by the server and can be used by the client to access text or binary content an example of
  this is a browser tool that can access web pages.
- **Roots:** the client supplies a list of file roots to the server which can be used to resolve file paths.
- **Completions:** The server provides auto-completion of options for a particular Prompt or Resource.
- **Sampling:** An MCP server can request an LLN completion for text or binary content a connected Client.

## http4k ❤️ Model Context Protocol

http4k provides very good support for the **Model Context Protocol**, and has been designed to make it easy to build
your own MCP-compliant servers in Kotlin, using the familiar http4k methodology of simple and composable functional
protocols. Each of the capabilities is modelled as a **binding** between a capability description and a function that
exposes the capability. See [Capability Types](#capabilities) for more details.

The MCP support in http4k consists of two parts - the `http4k-ai-mcp-sdk` and
the [http4k-mcp-desktop](https://github.com/http4k/mcp-desktop) application which is used to connect the MCP server to
a desktop client such as **Claude Desktop**.

# SDK: http4k-ai-mcp-sdk

The core SDK for working with the Model Context Protocol. You can build your own MCP-compliant applications using this
module by plugging in capabilities into the server. The **http4k-ai-mcp-sdk module** provides a simple way to create
either
**HTTP Streaming**, **SSE**, **StdIo** or **Websocket** based servers. For StdIo-based servers, we recommend compiling
your server to GraalVM for ease of distribution.

## Capability Types

The MCP protocol is based on a set of capabilities that can be provided by the server or client. Each capability can be
installed separately into the server, and the client can interact with the server using these capabilities.

Additional, when using one of the Streaming protocols, a `Client` object is passed to the Capability handler through the
request, allowing the server to send messages back to the client. These calls can be blocking or non-blocking, depending
on the client capability in question. These work most effectively when the client sends a `Progress Token` to the
server, which can be used to identify the operation being progressed.

### Server Capability: Tools

Tools allow external MCP clients such as LLMs to request the server to perform bespoke functionality such as invoking an
API. The Tool capability is modelled as a function `typealias ToolHandler = (ToolRequest) -> ToolResponse`, filtered
with a `ToolFilter`, and can be
bound to a tool definition which describes it's arguments and outputs using the http4k Lens system:

{{< kotlin file="simple_tool_example.kt" >}}

#### Complex Tools request arguments

The http4k MCP SDK also supports handling of complex arguments in the request (and response - MCP draft). This can be
done by using the `auto()` extension function and passing an example argument instance in order that the complex JSON
schema can be rendered. Note that the Kotlin Reflection JAR also needs to be present on the classpath to take advantage
of this feature, or you can supply a custom instance of `ConfigurableMcpJson` (Moshi-based) to work without reflection (
we recommend the use of the [Kotshi](https://github.com/ansman/kotshi) compiler plugin to generate adapters for this
use-case).

{{< kotlin file="auto_tool_example.kt" >}}

### Server Capability: Completions

Completions give the server to standard autocomplete abilities based on partial input from a client. The Completion
capability is modelled as a
function `typealias CompletionHandler = (CompletionRequest) -> CompletionResponse`, filtered with a `CompletionFilter`,,
and can be bound to a prompt definition which describes it's arguments
using the http4k Lens system.

{{< kotlin file="completion_example.kt" >}}

### Server Capability: Prompts

Prompts allow the server to generate a prompt based on the client's inputs. The Prompt capability is modelled as a
function `typealias PromptHandler = (PromptRequest) -> PromptResponse`, filtered with a `PromptFilter`,, and can be
bound to a prompt definition which describes it's arguments
using the http4k Lens system.

{{< kotlin file="prompt_example.kt" >}}

[//]: # (### Capability: Sampling)

[//]: # ()

[//]: # (Sampling allows the server to invoke the client LLM model to generate some content. The Sampling capability is modelled)

[//]: # (as a function `&#40;SamplingRequest&#41; -> Sequence<SamplingResponse>`, and you can pass the contents of previous interactions)

[//]: # (as the)

[//]: # (context to the model.)

[//]: # ()

[//]: # ({{< kotlin file="sampling_example.kt" >}})

### Server Capability: Resources

Resources provide a way to interrogate the contents of data sources such as filesystem, database or website. The
Resource capability is modelled as a function `typealias ResourceHandler = (ResourceRequest) -> ResourceResponse`,
filtered with a `ResourceFilter`. Resources can be static or templated to provide bounds within which the client can
interact with the resource.

{{< kotlin file="static_resource_example.kt" >}}

### Server Capability: Reporting Progress

The Progress capability allows the server to report progress of a long-running operation to the client through the
`progress()` call.

{{< kotlin file="progress_example.kt" >}}

### Client Capability: Sampling

Sampling allow the server to request information about the client's request from it's connected LLM, passing context.
Note that in order for this to work, a `ProgressToken` must be present in the request, which is used to identify the
operation being progressed.

{{< kotlin file="sampling_example.kt" >}}
    
### Client Capability: Elicitation

Elicitation allow the server to request additional information from the user in order to complete a task, by rendering a
dynamic form based on a supplied schema. You can use the passed `Client` to send the elicitation request to the client
and then wait for a response. Note that in order for this to work, a `ProgressToken` must be present in the request,
which is used to identify the operation being progressed.

A user has the option to accept, decline or cancel the elicitation request, and the server can handle these responses
accordingly. .

{{< kotlin file="elicitation_example.kt" >}}

### Capability Packs: Composed MCP Capabilities

http4k MCP lets you combine any number of related capabilities into reusable collections using the `CapabilityPack` API.
This is perfect for organizing related tools, resources, or prompts that logically belong together and shipping them as
a module or library.

{{< kotlin file="capability_pack_example.kt" >}}

### MCP Servers

Servers are created by combining the configured MCP Protocol with a set of capabilities, an optional security, and a
binding to a Server or Serverless backend. The server can be started using any of the http4k server backends which
support SSE ( see [servers](/ecosystem/http4k/reference/servers)).

{{< kotlin file="server_streaming_example.kt" >}}

Alternatively you can use any non-SSE supporting server backend and forego the SSE support in lieu of request/response
via JSON:

{{< kotlin file="server_nonstreaming_example.kt" >}}

There are a number of different ways customise the MCP protocol server to suit your needs. Features that can be
configured are shown below. Note that the main SDK library is designed for simplicity - and you may have to drill down
one level to access some of these customisations:

- Security - Basic, Bearer, API Key or auto-discovered (or custom!)
  OAuth ([specification standard](https://modelcontextprotocol.info/specification/draft/basic/authorization))
- Session validation (via `SessionProvider`) - Ensure that the client is authenticated to access the contents of the
  session
- Event Store (via `SessionEventStore`) - Store and resume MCP event streams using the SSE last-event-id header
- Event Tracking (via `SessionEventTracking`) - Assign a unique ID to each event to track the progress of the event
  stream
- Origin validation (via `Filter` and `SseFilter`) - Protect against DNS rebinding attacks by configuring allowed
  origins

#### Important: Protecting Against DNS Rebinding Attacks

When deploying an MCP server that uses HTTP Streaming or SSE, you must implement `Origin` header validation to prevent
DNS rebinding attacks. These attacks can allow malicious websites to interact with your MCP server by changing IP
addresses after initial DNS
resolution, potentially bypassing same-origin policy protections. This can be done by implementing the HTTP (`Filter`)
and SSE specific (`SseFilter`) filter implementations and attaching them to the Polyhandler that is returned from the
`mcpXXX()` call.

The http4k-ai-mcp-sdk provides protection mechanisms that can be applied to your server:

{{< kotlin file="securing_against_sse_rebind.kt" >}}

#### Serverless Example

MCP capabilities can be bound to [http4k Serverless](/ecosystem/http4k/reference/serverless) functions using the HTTP
protocol in non-streaming mode. To activate this simply bind them into the non-streaming HTTP which is a simple
`HttpHandler`.

{{< kotlin file="serverless_example.kt" >}}

### MCP Client

http4k provides client classes to connect to your MCP servers via HTTP, SSE, JSONRPC or Websockets. The clients take
care of the
initial MCP handshake and provide a simple API to send and receive messages to the capabilities, or to register for
notifications with an MCP server.

{{< kotlin file="client_example.kt" >}}

# [http4k-mcp-desktop](https://github.com/http4k/mcp-desktop)

A desktop client that bridges StdIo-bound desktop clients such as **Claude Desktop** with your own MCP servers operating
over HTTP/SSE, either locally or remotely. The desktop client is a simple native application that can be downloaded from
the http4k GitHub, or built from the http4k source.

### To use mcp-desktop client with clients such as Claude Desktop or Cursor:

1. Download the `mcp-desktop` binary for your platform from: [https://github.com/http4k/mcp-desktop], or install it with
   brew:

```bash
brew tap http4k/tap
brew install http4k-mcp-desktop
```

2. Configure [Claude Desktop](https://claude.ai/download) to use the `mcp-desktop` binary as an MCP server with the
   following configuration. You can find the configuration file in `claude_desktop_config.json`, or by browsing through
   the
   developer settings menu. You can add as many MCP servers as you like. Note that [Cursor](https://www.cursor.com/)
   users should use the `--transport http-nonstream` or `--transport jsonrpc` option for correct integration:

```json
{
    "mcpServers": {
        "MyMcpServer": {
            "command": "http4k-mcp-desktop",
            // or path to the binary
            "args": [
                "--transport",
                "--http-stream",
                "--url",
                "http://localhost:3001/mcp"
            ]
        }
    }
}
```

### To build mcp-desktop from source:

1. Clone the [http4k MCP Desktop](https://github.com/http4k/mcp-desktop) repo
2. Install a GraalVM supporting JDK
3. Run `./gradlew :native-compile` to build the desktop client binary locally for your platform
