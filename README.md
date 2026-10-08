# Axolotl.Mesh

Axolotl.Mesh is the Elysium operation runtime for external systems.

It is not just an HTTP client. HTTP is the first adapter, but the product is a configurable, versioned, testable, and deployable operation engine for executing external operation contracts across protocols.

```text
input -> validate -> transform -> execute adapter -> validate -> transform -> output
```

Axolotl.Mesh owns operation execution. Protocols are adapters. Contracts, mappings, policies, auditing, and runtime behavior stay consistent across protocols.

## North Star

Axolotl.Mesh should become the Elysium runtime for external operation contracts.

```text
Contracts define the promise.
ButterMorph shapes the payload.
Axolotl.Mesh executes the operation.
Mule makes it durable.
SquirrelBox makes it safe.
Krackend orchestrates it.
KnOwl governs the contracts.
Pigeon carries async messages.
Charon routes traffic.
```

Design mantra:

```text
Protocols are adapters. The operation contract is the product.
```

## Position In Elysium

Axolotl.Mesh is designed to connect with the existing Elysium stack instead of rebuilding it.

| Elysium piece | Relationship |
| --- | --- |
| ButterMorph | Request/response transformation and validation engine |
| KnOwl | Contract catalog, operation versioning, promotion, release, and runtime artifact distribution patterns |
| Krackend | Orchestration layer that can dispatch HTTP and future task kinds through Axolotl.Mesh |
| Pigeon | Messaging adapter and event integration |
| Mule | Durable operation execution and deferred retries |
| SquirrelBox | Idempotency and safe operation execution for inbound/outbound flows |
| TurtlePath | Host pattern for Axolotl.Mesh services and APIs |
| Spider | Operation execution boundaries, traces, and diagnostics |
| Charon | Gateway and routing layer, not duplicated by Axolotl.Mesh |
| DataScorpio | Querying operation catalogs, runs, logs, and artifacts |

## Product Boundary

Axolotl.Mesh owns:

- Operation definitions and versions.
- Operation execution.
- Adapter registration and execution.
- Auth and secret resolution.
- Request and response shaping.
- Execution policies such as timeout, retry, backoff, and circuit breaker behavior.
- Result normalization.
- Audit trail and execution history.
- Dry run and test execution.
- YAML and database-backed operation storage abstractions.
- Runtime packages and artifacts for deployed operations.

Axolotl.Mesh does not own:

- General API gateway behavior; Charon owns gateway concerns.
- Workflow orchestration; Krackend owns orchestration.
- Broker implementation; Pigeon owns messaging transport.
- Durable action scheduling primitives; Mule owns durable execution.
- Inbox/outbox primitives; SquirrelBox owns idempotent entry protection.
- Dynamic schema and mapping DSL design from scratch; ButterMorph owns this layer.

## Core Concepts

### Operation Definition

A logical operation such as `customers.create`, `tax-regime.lookup`, or `invoice.submit`.

Typical fields include:

- `key`
- `name`
- `description`
- `kind`
- `owner`
- `tags`
- `currentVersion`
- `lifecycleStatus`

### Operation Version

An immutable versioned shape of an operation.

Typical fields include:

- `operationKey`
- `version`
- `status`
- `inputContract`
- `outputContract`
- `requestMapping`
- `responseMapping`
- `validation`
- `adapterConfiguration`
- `executionPolicy`
- `secretRequirements`
- `metadata`

Version lifecycle:

```text
Draft -> InReview -> Approved -> Deployed -> Deprecated -> Archived
```

### Operation Artifact

An immutable deployment payload generated from an approved or deployed operation version.

Artifacts contain all runtime data needed to execute the operation without querying design-time catalogs:

- Operation identity and version.
- Operation kind.
- Input and output schema snapshots.
- Validation DSL snapshots.
- Request and response mapping DSL snapshots.
- Adapter configuration.
- Execution policy.
- Required secrets and capabilities.
- Checksum and creation metadata.

### Runtime Operation

A deployed operation artifact materialized in runtime storage.

Runtime operations can be executed by:

- Direct library calls.
- HTTP API endpoints.
- Krackend task adapters.
- Mule durable actions.
- Pigeon consumers.
- Internal service code.

### Operation Run

One execution attempt from input to output.

Runs track:

- Run id.
- Operation key and version.
- Artifact id and checksum.
- Correlation id, causation id, and idempotency key.
- Trigger/source.
- Started and finished timestamps.
- Status and adapter status.
- Request/response metadata.
- Redacted diagnostics.
- Retry attempt history.

## Adapter Model

Every protocol or execution type implements a common adapter contract.

Conceptual shape:

```csharp
public interface IAxolotlMeshOperationAdapter
{
    string Kind { get; }

    ValueTask<AdapterValidationResult> ValidateAsync(
        OperationArtifact artifact,
        CancellationToken cancellationToken);

    ValueTask<AdapterExecutionResult> ExecuteAsync(
        AdapterExecutionRequest request,
        CancellationToken cancellationToken);
}
```

Initial operation kinds:

| Kind | Priority | Notes |
| --- | --- | --- |
| `http` | V1 | First production adapter |
| `graphql` | V2 | Can reuse HTTP execution with GraphQL-specific configuration |
| `grpc` | V2/V3 | Unary first; streaming later |
| `messaging` | V2 | Pigeon-backed publish and request/reply |
| `webhook.inbound` | V2 | Inbound trigger surface, likely SquirrelBox-protected |
| `sql` | V3 | Parameterized read/write operations |
| `file` | V3 | SFTP, blob, local, or S3-style operations |
| `websocket` | Later | Sessions and subscriptions need a separate runtime model |
| `soap` | Later | Useful for legacy systems, but not first-class early |
| `plugin` | Later | Controlled extension execution |

## Runtime Execution Pipeline

Recommended pipeline:

```text
1. Resolve operation artifact
2. Create run
3. Resolve input/context metadata
4. Validate input
5. Transform input into adapter request
6. Resolve secrets
7. Validate adapter request
8. Execute adapter through policy
9. Capture raw adapter result
10. Validate response
11. Transform response into output
12. Validate output
13. Complete run
14. Emit diagnostics/events
```

Pipeline extension points:

- Before input validation.
- After input validation.
- Before adapter execution.
- After adapter execution.
- Before output completion.
- On failure.
- On retry.

Spider can represent these boundaries and expose traces.

## Security And Secrets

Secret references should use explicit runtime-resolved expressions:

```text
${secret.provider.key}
${env.NAME}
${config:path}
```

Rules:

- Runtime artifacts may include secret references, never secret values.
- Design-time test execution resolves secrets through scoped providers.
- Execution logs redact configured headers and JSON paths.
- Secret providers are pluggable.
- DataProtection can protect stored secrets for early EF-based usage.
- Vault-style providers can come later.

## Observability

Run diagnostics should include:

- Operation key and version.
- Artifact checksum.
- Adapter kind.
- Destination with safe redaction.
- Status and duration.
- Retry attempts.
- Error code and classification.
- Correlation id.
- Idempotency key.
- Redacted request/response metadata.

Full sensitive payloads should not be stored by default.

## Error Taxonomy

Initial error groups:

- `ValidationFailed`
- `MappingFailed`
- `SecretResolutionFailed`
- `AdapterConfigurationInvalid`
- `AdapterExecutionFailed`
- `Timeout`
- `RetryExhausted`
- `CircuitOpen`
- `UnexpectedStatusCode`
- `ResponseParsingFailed`
- `OutputValidationFailed`
- `RuntimeCapabilityMissing`

Each error should expose a stable code, message, retryable flag, metadata, and redacted diagnostics.

## Proposed Package Direction

Core runtime:

```text
Axolotl.Mesh
Axolotl.Mesh.Abstractions
Axolotl.Mesh.DependencyInjection
```

Adapters and integrations:

```text
Axolotl.Mesh.Http
Axolotl.Mesh.ButterMorph
Axolotl.Mesh.Pigeon
Axolotl.Mesh.Krackend
Axolotl.Mesh.KnOwl
Axolotl.Mesh.TurtlePath
```

Storage:

```text
Axolotl.Mesh.Storage.Yaml
Axolotl.Mesh.Storage.EntityFramework
Axolotl.Mesh.Storage.InMemory
```

Durable and safe execution:

```text
Axolotl.Mesh.Mule
Axolotl.Mesh.SquirrelBox
```

Control plane and runtime hosts:

```text
Axolotl.Mesh.ControlPlane
Axolotl.Mesh.ControlPlane.Application
Axolotl.Mesh.ControlPlane.Storage.EntityFramework
Axolotl.Mesh.ControlPlane.WebUI
Axolotl.Mesh.Runtime
Axolotl.Mesh.Runtime.Storage.EntityFramework
Axolotl.Mesh.Runtime.WebUI
Axolotl.Mesh.Runtime.Api
```

## Recommended First Slice

Build the smallest impressive vertical slice:

1. YAML operation definition.
2. ButterMorph request mapping.
3. HTTP execution.
4. ButterMorph response mapping.
5. Normalized result.
6. Redacted diagnostics.
7. Tests against a local test server.

Sample scenario:

```text
Input customer payload -> transform -> POST /customers -> transform response -> output customer id
```

This proves the heart of Axolotl.Mesh without pulling in the control plane, EF storage, Mule, or Krackend too early.

## Roadmap

### Iteration 0: Product And Contract Spike

Define the smallest stable model before building the production runtime.

Deliverables:

- Operation model draft.
- Operation version model draft.
- Operation artifact model draft.
- Operation run result model draft.
- Adapter interface draft.
- HTTP artifact configuration draft.
- Secret reference model draft.
- Error taxonomy draft.

Acceptance criteria:

- A complete sample operation can be represented as YAML and as a C# model.
- The same sample can be converted into an immutable artifact.
- There is a clear line between design-time data and runtime artifact data.

### Iteration 1: Local Runtime MVP

Execute HTTP operations from local YAML with ButterMorph mapping and validation.

Initial scope:

- `Axolotl.Mesh.Abstractions`
- `Axolotl.Mesh`
- `Axolotl.Mesh.Http`
- `Axolotl.Mesh.ButterMorph`
- `Axolotl.Mesh.Storage.Yaml`
- `Axolotl.Mesh.Storage.InMemory`

Acceptance criteria:

- A YAML-defined operation can call a test HTTP endpoint.
- Request body can be generated from input using ButterMorph.
- Response can be normalized into output using ButterMorph.
- Failed validation returns structured diagnostics.
- Secrets never appear in execution logs.

### Iteration 2: Production HTTP Runtime

Harden HTTP execution for production usage.

Focus areas:

- `HttpClientFactory` integration.
- Retry policy with fixed/exponential backoff and jitter.
- Circuit breaker behavior.
- Per-operation timeout policy.
- Retryable status code and exception classification.
- Correlation, causation, and idempotency metadata.
- Structured run logs.
- Request/response size limits.
- Redaction policy for headers and body paths.
- OAuth2 client credentials.
- Secret resolver abstraction.

### Iteration 3: EF Storage And Operation Catalog

Move from local YAML-only definitions to durable, queryable operation storage.

Focus areas:

- EF Core storage for definitions, versions, artifacts, and runs.
- Version lifecycle.
- Artifact generation and deterministic checksums.
- Duplicate/conflict prevention.
- DataScorpio-backed catalog querying.
- Run history querying.

### Iteration 4: Mule And SquirrelBox Integration

Support durable and idempotent operation execution.

Focus areas:

- Mule durable action for operation execution.
- Deferred execution API.
- Mule lane selection per operation.
- SquirrelBox operation wrapper.
- Idempotency key resolution.
- Duplicate result replay.
- Retry metadata bridge.

### Iteration 5: Krackend HTTP Task Adapter

Make Axolotl.Mesh the runtime implementation for Krackend `TaskKind.Http`.

Focus areas:

- `Axolotl.Mesh.Krackend` package.
- `ITaskRuntimeAdapter` implementation for `TaskKind.Http`.
- Translation from Krackend HTTP task artifacts into Axolotl.Mesh operation execution.
- Sync response support where allowed.
- Callback/result mapping into orchestration runtime.
- Error mapping to Krackend retry/error policies.

### Iteration 6: Control Plane And Web UI

Provide a visual builder and management surface.

Focus areas:

- Browse operations.
- Create and edit operation versions.
- Configure HTTP adapters.
- Open ButterMorph designers for schemas, mappings, and validations.
- Test operations with sample input.
- Inspect request preview with redactions.
- Approve and deploy versions.
- View artifacts, checksums, and execution runs.

### Later Iterations

- Pigeon messaging adapter.
- GraphQL and gRPC adapters.
- Inbound webhook and evented operations.
- Advanced adapters for SOAP, SQL, files, blobs, WebSockets, AI calls, and custom plugins.

## YAML Draft

Example HTTP operation:

```yaml
key: customers.create
version: 1.0.0
kind: http
name: Create customer

input:
  schema: customers.create.input.v1
  validation: customers.create.input-validation.v1

output:
  schema: customers.create.output.v1
  validation: customers.create.output-validation.v1

mapping:
  request: customers.create.request-map.v1
  response: customers.create.response-map.v1

adapter:
  method: POST
  baseUrl: ${secret.crm.baseUrl}
  path: /customers
  headers:
    Content-Type: application/json
    Authorization: Bearer ${secret.crm.accessToken}
  expectedStatusCodes: [200, 201]

execution:
  timeoutMs: 10000
  retry:
    attempts: 3
    strategy: exponential
    initialDelayMs: 250
    maxDelayMs: 5000
    jitterRatio: 0.1
  circuitBreaker:
    enabled: true
    failureThreshold: 0.5
    minimumThroughput: 20
    breakDurationMs: 30000

redaction:
  headers:
    - Authorization
  bodyPaths:
    - $.password
    - $.token
```

Future gRPC operation:

```yaml
key: customers.create
version: 2.0.0
kind: grpc
name: Create customer through CRM gRPC

adapter:
  endpoint: ${secret.crm.grpcEndpoint}
  service: crm.Customers
  method: CreateCustomer
  deadlineMs: 10000
```

## Current Repository State

This repository currently contains the initial Axolotl library scaffold only:

```text
src/
  Axolotl/
    Axolotl.csproj
```

There is intentionally no implementation code yet. The first implementation step should be the Iteration 0 product and contract spike, followed by the local runtime MVP.

## Build

```bash
dotnet restore Axolotl.sln
dotnet build Axolotl.sln --configuration Release
```

## Release

1. Add release notes under a version heading in `CHANGELOG.md`.
2. Update `.release` with the matching tag, such as `v0.1.0`.
3. Merge the change to `main`.

The workflow creates a release branch, tags the commit, publishes the GitHub release, and pushes the NuGet package when trusted publishing is configured.
