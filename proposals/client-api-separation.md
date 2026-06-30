# Proposal: Normative A2A Client API Specification

## Problem

The A2A specification currently defines a single `A2AService` in `a2a.proto`. This service
describes operations that an individual agent *server* exposes. Client libraries are generated
directly from that service definition — meaning the "client API" is by definition identical to
the server API, constrained to operations that can be modelled as a direct RPC call to a single
server, and carrying server-level routing concerns (e.g. `tenant`) that have no meaning in
application code.

This conflation creates several problems:

- **No normative client interface.** There is no specification for what an A2A client library
  must expose to application code. SDK authors in all six languages independently decide the
  shape of their client, producing fragmented surfaces with no basis for cross-SDK
  interoperability testing.

- **Routing is a client concern with no client home.** The `tenant` field must be set correctly
  on every request, but it is derived from the chosen `AgentInterface` in an `AgentCard` — a
  resolution step that belongs in the client library, not in application code.

- **No path to network-layer operations.** Operations that are not server RPCs — fan-out to
  multiple agents, pub/sub topics — have no normative place to land. Every attempt to add them
  (see #1029, #1593, #1995) runs into the fact that the only available hook is the server service
  definition, which is the wrong place.

- **No tool schema derivable from the client API.** As AI agents increasingly use A2A operations
  as tools in their reasoning loops, a normative client API becomes the natural source for
  generating tool definitions. Without one, each SDK and framework defines its own tool surface
  ad hoc.

## Design Principle: Logical vs Physical Interface

The core idea is to separate the **logical A2A interface** — the set of operations application
code and agents program against — from the **physical interface** exposed by individual A2A
servers.

The physical interface (`A2AService`) provides the primitives that *enable* the logical interface.
An A2A server only needs to implement the operations that make sense for a single agent endpoint.
The logical client API may go beyond that: some operations are composed by the client library from
multiple physical calls, and others are fulfilled entirely by the transport layer. The server never
needs to know.

```
Application / Agent
        │
        │  Logical Client API  (what you program to)
        ▼
  A2A Client Library
        │
        ├──────────────────────────────┐
        │                              │
        ▼                              ▼
  A2AService (server)         Transport / Broker
  Physical primitives         Network-layer operations
  (per-agent RPC)             (fan-out, pub/sub, etc.)
```

A concrete example: `SendMessage` in the logical API accepts one or more `AgentCard` targets. When
there is a single target the client library maps it directly to `A2AService.SendMessage` on that
server — the physical and logical operations are identical. When there are multiple targets, the
client library fans out across them using whatever primitive the transport provides (parallel HTTP
calls, multicast, a pub/sub publish). Each individual server still only sees a standard
`SendMessage` from `A2AService`. The multi-agent behaviour lives entirely in the logical layer.

This means the physical server spec (`A2AService`) remains stable and minimal — servers implement
only what an individual agent endpoint needs to expose — while the logical client API can evolve
independently to express richer network-level operations without burdening server authors.

## Proposed Solution

Introduce a normative A2A Client API specification defined in a machine-readable IDL that is
separate from the existing server API definition. The client API covers exactly the same
operations as `A2AService` today, but is shaped for application code: each operation carries an
`AgentCard` (or repeated `AgentCard`) as its explicit routing target, and all server-level routing
details are resolved internally by the transport binding.

The key design points:

- **`SendMessage` is the single send operation**, accepting one or more `AgentCard` targets. A
  single target is a point-to-point call; multiple targets trigger transport-level fan-out
  (parallel calls, multicast, or pub/sub publish). There is no separate multi-agent variant.
- **All task operations carry an `AgentCard`** identifying the server that owns the task, replacing
  the `tenant` field which is an implementation detail resolved from the card.
- **`ClientSendMessageResponse` is always a list of per-agent results**, keeping the return type
  uniform whether one or many agents were targeted, and allowing partial failures to be reported.

### Logical Client API

The following describes the client-facing operations in transport-agnostic terms. The concrete IDL
used to express this normatively is discussed in the [IDL Options](#idl-options) section below.

```
SendMessage(agents: AgentCard[], message, configuration?) → AgentSendResult[]
SendStreamingMessage(agents: AgentCard[], message, configuration?) → stream StreamResponse

GetTask(agent: AgentCard, id) → Task
ListTasks(agent: AgentCard, filter?) → Task[]
CancelTask(agent: AgentCard, id) → Task
SubscribeToTask(agent: AgentCard, id) → stream StreamResponse

CreateTaskPushNotificationConfig(agent: AgentCard, config) → TaskPushNotificationConfig
GetTaskPushNotificationConfig(agent: AgentCard, task_id, config_id) → TaskPushNotificationConfig
ListTaskPushNotificationConfigs(agent: AgentCard, task_id) → TaskPushNotificationConfig[]
DeleteTaskPushNotificationConfig(agent: AgentCard, task_id, config_id)

GetExtendedAgentCard(agent: AgentCard) → AgentCard
```

Where `AgentSendResult` is a per-agent outcome containing either a `SendMessageResponse` or an
error — a failed delivery to one agent does not preclude success for others.

### IDL Options

The client API should be expressed in a machine-readable IDL to enable code scaffolding generation
across all supported SDK languages. Two options are discussed here; the choice of IDL is an open
question for community input.

#### Option A: Protocol Buffers (`a2a_client.proto`)

Define the client API as a new proto service alongside the existing `a2a.proto`, feeding directly
into the existing `buf`-based code generation pipeline.

```proto
syntax = "proto3";
package lf.a2a.v1;

import "a2a.proto";

// A2AClientService defines the canonical interface that A2A client libraries
// expose to application code.
//
// Unlike A2AService — which describes operations an individual agent server
// exposes over a specific protocol binding — A2AClientService describes what
// a client library presents to its caller. Operations take an explicit AgentCard
// (or repeated AgentCard) as their routing target; the transport binding is
// responsible for resolving the AgentCard's supportedInterfaces, selecting a
// protocol, and handling any tenant routing internally.
service A2AClientService {

  // Send a message to one or more agents.
  // A single target maps to A2AService.SendMessage; multiple targets trigger
  // transport-level fan-out.
  rpc SendMessage(ClientSendMessageRequest) returns (ClientSendMessageResponse);

  rpc SendStreamingMessage(ClientSendMessageRequest)
      returns (stream StreamResponse);

  rpc GetTask(ClientGetTaskRequest) returns (Task);
  rpc ListTasks(ClientListTasksRequest) returns (ListTasksResponse);
  rpc CancelTask(ClientCancelTaskRequest) returns (Task);
  rpc SubscribeToTask(ClientSubscribeToTaskRequest)
      returns (stream StreamResponse);

  rpc CreateTaskPushNotificationConfig(
      ClientCreateTaskPushNotificationConfigRequest)
      returns (TaskPushNotificationConfig);
  rpc GetTaskPushNotificationConfig(
      ClientGetTaskPushNotificationConfigRequest)
      returns (TaskPushNotificationConfig);
  rpc ListTaskPushNotificationConfigs(
      ClientListTaskPushNotificationConfigsRequest)
      returns (ListTaskPushNotificationConfigsResponse);
  rpc DeleteTaskPushNotificationConfig(
      ClientDeleteTaskPushNotificationConfigRequest)
      returns (google.protobuf.Empty);

  rpc GetExtendedAgentCard(ClientGetExtendedAgentCardRequest)
      returns (AgentCard);
}

message ClientSendMessageRequest {
  repeated AgentCard agents = 1 [(google.api.field_behavior) = REQUIRED];
  Message message = 2 [(google.api.field_behavior) = REQUIRED];
  SendMessageConfiguration configuration = 3;
  google.protobuf.Struct metadata = 4;
}

message ClientSendMessageResponse {
  repeated AgentSendResult results = 1;
}

message AgentSendResult {
  AgentCard agent = 1;
  oneof outcome {
    SendMessageResponse response = 2;
    google.rpc.Status error = 3;
  }
}

message ClientGetTaskRequest {
  AgentCard agent = 1 [(google.api.field_behavior) = REQUIRED];
  string id = 2 [(google.api.field_behavior) = REQUIRED];
  optional int32 history_length = 3;
}

// Remaining request messages follow the same pattern — each carries an
// AgentCard as the routing target alongside the fields from the corresponding
// A2AService request.
```

**Advantages:** reuses the existing `buf` toolchain and generates stubs in all six SDK languages
with no additional infrastructure. Proto is already the normative IDL for `A2AService`, keeping
both APIs in one ecosystem.

**Considerations:** proto is primarily designed for server-side RPC service definitions. The
client API has different semantics (multi-target, transport-level fan-out) that are not naturally
expressed in proto's service model and would rely on comments rather than the type system to
convey intent.

#### Option B: TypeSpec

[TypeSpec](https://typespec.io) is a language developed by Microsoft for describing APIs and data
models. It compiles to multiple output formats (OpenAPI, JSON Schema, Protobuf, and client SDK
scaffolding) from a single source, and is used by Azure as the canonical definition language for
its entire SDK surface.

Where proto models a *service* — a set of RPCs exposed by a server — TypeSpec's `interface`
construct models a set of *operations* that a client exposes to its callers, which maps more
directly to what the client API represents. TypeSpec also has a richer type system (unions,
templates, decorators) and explicit support for generating client library scaffolding rather than
server stubs.

A sketch of the client API expressed in TypeSpec:

```typespec
import "@typespec/http";

using TypeSpec.Http;

@doc("The canonical interface A2A client libraries expose to application code.")
interface A2AClient {

  @doc("""
    Send a message to one or more agents.
    A single target is point-to-point; multiple targets trigger transport-level fan-out.
  """)
  sendMessage(agents: AgentCard[], message: Message, configuration?: SendMessageConfiguration):
    AgentSendResult[];

  @doc("Streaming variant of sendMessage.")
  sendStreamingMessage(agents: AgentCard[], message: Message, configuration?: SendMessageConfiguration):
    StreamResponse[];

  getTask(agent: AgentCard, id: string, historyLength?: int32): Task;

  listTasks(agent: AgentCard, filter?: ListTasksFilter): ListTasksResponse;

  cancelTask(agent: AgentCard, id: string): Task;

  subscribeToTask(agent: AgentCard, id: string): StreamResponse[];

  createTaskPushNotificationConfig(agent: AgentCard, config: TaskPushNotificationConfig):
    TaskPushNotificationConfig;

  getTaskPushNotificationConfig(agent: AgentCard, taskId: string, configId: string):
    TaskPushNotificationConfig;

  listTaskPushNotificationConfigs(agent: AgentCard, taskId: string):
    TaskPushNotificationConfig[];

  deleteTaskPushNotificationConfig(agent: AgentCard, taskId: string, configId: string): void;

  getExtendedAgentCard(agent: AgentCard): AgentCard;
}

model AgentSendResult {
  agent: AgentCard;
  response?: SendMessageResponse;
  error?: ErrorResponse;
}
```

**Advantages:** TypeSpec's `interface` construct directly expresses a client library's operation
set rather than a server's RPC surface. A single TypeSpec definition can emit OpenAPI for
documentation, JSON Schema for validation, and client SDK scaffolding for all supported languages,
without requiring the output formats to be kept in sync manually. The language is actively
maintained and used at scale across the Azure SDK.

**Considerations:** TypeSpec is less established in the agent protocol space than proto and would
introduce a new toolchain dependency. The existing A2A data model defined in `a2a.proto` would
need to be either re-expressed in TypeSpec or referenced via the TypeSpec protobuf emitter.

### Code Scaffolding

Regardless of IDL choice, the normative client API definition should drive generation of:

- **Client interface scaffolding** — abstract base classes or interfaces in each SDK language that
  SDK authors implement.
- **Default transport stub** — a base implementation that returns `UNIMPLEMENTED` for any
  operation not supported by the transport binding, which richer bindings override.
- **Transport support matrix** — a generated documentation table showing which transport bindings
  support which operations.

```
Client API IDL
       │
       ├── SDK scaffolding — Python, Go, Java, TS, .NET, Rust
       │
       └── DefaultClient base (unsupported ops → UNIMPLEMENTED)
               ▲                  ▲
       JsonRpcClient         MqttClient
       (inherits defaults)   (overrides pub/sub operations)
```

## Alternatives Considered

**Leave the client API as an SDK convention.** Each of the six language SDKs independently models
the client surface. Already happening today; produces fragmented interfaces with no cross-SDK
interoperability baseline.

**Extend `A2AService` with client-only operations.** Adding fan-out or topic operations to the
*server* service forces every A2A server to stub out operations that are semantically the
transport's or broker's responsibility, burdening server authors and weakening conformance testing.

**Per-binding extension definitions only.** Each Custom Protocol Binding defines its own client
extension independently. Avoids a shared client definition but means there is no unified client
interface to test against and application code must be transport-aware.

## Future Possibilities

This separation is a prerequisite for several open proposals, not a solution to any single one.
Once a normative `A2AClientService` exists as a separately-versioned surface, the following
classes of operation have a well-defined home — they can be added to the client API without
touching `A2AService` or requiring every server implementation to change:

**Multi-agent fan-out.** `SendMessage` already accepts multiple `AgentCard` targets in the design
above. Transport bindings map this to parallel point-to-point calls, multicast, or a pub/sub
publish according to their capabilities, with no changes to `A2AService`. (Relevant: #1029, #1593)

**Pub/sub and topic operations.** Operations such as `ListTopics` and `SubscribeToTopic` are
resolved by the broker, not by any individual agent server. They can be added to the client API
and declared as supported by transport bindings that expose a brokered network, without any impact
on the server spec.

## Client Operations as Agent Tools

A normative client API is the natural source for tool definitions surfaced to AI agents. An agent
that needs to delegate work can be given `SendMessage`, `GetTask`, or pub/sub operations as
structured tools derived directly from the client API definition, with consistent schemas across
all SDK languages and frameworks.

This means the same pipeline that generates SDK scaffolding can also generate tool definitions for
agent frameworks — MCP tool schemas, OpenAI function definitions, or equivalent — with no
per-framework hand-authoring. Without a normative client spec, each integration generates its own
ad-hoc tool surface.

## CLI Design

A normative client API also provides a direct foundation for the official A2A CLI (#1929). Each
client operation maps naturally to a CLI sub-command, with the operation's input fields becoming
flags and positional arguments. Because the CLI and the SDK share the same generated types, there
is no separate command design to maintain — the CLI becomes a thin shell over the same client
interface scaffolding.

For example:

```
a2a send-message --agent <agent-card-url> --message "Summarise this document" [file]
a2a get-task     --agent <agent-card-url> <task-id>
a2a list-tasks   --agent <agent-card-url> --status working
a2a cancel-task  --agent <agent-card-url> <task-id>
```

As new operations are added to the client API they automatically become candidates for CLI
sub-commands, keeping the CLI in sync with the client spec without additional design work.

## Related Issues

- #1029 — Support publish/subscribe methods for async communications
- #1593 — Built-in pub/sub support in the A2A protocol
- #1929 — Official A2A CLI
- #1995 — Bidirectional streaming and improved stream semantics
