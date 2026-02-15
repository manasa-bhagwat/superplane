# SuperPlane Knowledge Transfer - AWS CodePipeline Integration

**Date:** February 15, 2026  
**For:** AWS CodePipeline Integration Task  
**Status:** Complete Knowledge Transfer Session

---

## Table of Contents

1. [What is SuperPlane?](#what-is-superplane)
2. [Architecture Overview](#architecture-overview)
3. [Technology Stack](#technology-stack)
4. [Core Concepts](#core-concepts)
5. [Directory Structure](#directory-structure)
6. [Data Flow & System Design](#data-flow--system-design)
7. [Integration System Deep Dive](#integration-system-deep-dive)
8. [AWS Integration Architecture](#aws-integration-architecture)
9. [How to Build AWS CodePipeline Integration](#how-to-build-aws-codepipeline-integration)
10. [Development Workflow](#development-workflow)
11. [Testing & Debugging](#testing--debugging)
12. [Troubleshooting Guide](#troubleshooting-guide)

---

## What is SuperPlane?

**Category:** DevOps Control Plane / Workflow Orchestration Platform / Infrastructure Automation Tool

SuperPlane is an **open-source DevOps control plane** for defining and running **event-based workflows**. Think of it as:

- **Zapier/n8n for DevOps** - Connects all your DevOps tools
- **Airflow for Infrastructure** - Orchestrates multi-step operational workflows
- **Temporal for Platform Engineering** - Event-driven automation across CI/CD, infra, observability, incidents

### Key Capabilities

1. **Workflow Orchestration**: Model multi-step operational workflows as directed graphs
2. **Event-Driven Automation**: Trigger workflows from pushes, deploys, alerts, schedules, webhooks
3. **Visual Canvas Builder**: Design and manage DevOps processes in a UI
4. **Integration Hub**: 75+ components connecting GitHub, AWS, Slack, PagerDuty, Datadog, etc.

### Use Cases

- **Policy-gated deploys**: CI green → hold outside hours → require approvals → trigger deploy
- **Progressive delivery**: Deploy 10% → 30% → 60% → 100% with verification gates
- **Incident triage**: Alert fires → fetch context → generate evidence pack → create ticket
- **Release trains**: Wait for tags from multiple services → fan-in → coordinated deploy

---

## Architecture Overview

SuperPlane is built as a **modular monolith** with independent, scalable modules.

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         SuperPlane                               │
│                                                                   │
│  ┌──────────────┐                                                │
│  │   Frontend   │  React + TypeScript + Vite                     │
│  │   (web_src)  │  ← Auto-generated TypeScript client           │
│  └──────┬───────┘                                                │
│         │ REST + WebSocket                                       │
│  ┌──────▼────────────────────────────────────────────────────┐  │
│  │              Public API (port 8000)                        │  │
│  │  • Serves UI (static files)                                │  │
│  │  • REST endpoints (gRPC-Gateway)                           │  │
│  │  • WebSocket hub (real-time updates)                       │  │
│  │  • JWT authentication                                      │  │
│  └──────┬────────────────────────────────────────────────────┘  │
│         │ gRPC                                                   │
│  ┌──────▼────────────────────────────────────────────────────┐  │
│  │             Internal API (port 50051)                      │  │
│  │  • gRPC server (protobuf)                                  │  │
│  │  • Business logic                                          │  │
│  │  • Authorization (Casbin RBAC)                             │  │
│  │  • Database access (GORM)                                  │  │
│  └──────┬────────────────────────────────────────────────────┘  │
│         │ RabbitMQ Messages                                      │
│  ┌──────▼────────────────────────────────────────────────────┐  │
│  │                    Workers (Background)                    │  │
│  │  • EventRouter: Routes events through workflow graphs      │  │
│  │  • NodeExecutor: Executes component logic                  │  │
│  │  • NodeQueueWorker: Processes queue items                  │  │
│  │  • WebhookProvisioner: Sets up webhooks in external systems│  │
│  │  • IntegrationRequestWorker: Syncs integration configs     │  │
│  │  • EmailConsumer: Sends invitation/notification emails     │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────┐        ┌─────────────────────┐          │
│  │    PostgreSQL      │        │      RabbitMQ       │          │
│  │  (Data Storage)    │        │  (Message Queue)    │          │
│  └────────────────────┘        └─────────────────────┘          │
└───────────────────────────────────────────────────────────────────┘
                        │
                        ▼
          ┌────────────────────────────┐
          │   External Integrations    │
          │  • GitHub, GitLab          │
          │  • AWS (ECR, Lambda, etc.) │
          │  • Slack, Discord          │
          │  • PagerDuty, Datadog      │
          │  • Semaphore, CircleCI     │
          └────────────────────────────┘
```

### Component Interaction Flow

```
User Action → Public API → Internal API → Workers → External Services
     ↓                                       ↓
Database ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
```

---

## Technology Stack

### Backend (Go)

- **Language**: Go 1.21+
- **Web Framework**: Custom (gRPC + gRPC-Gateway)
- **API Definition**: Protocol Buffers (protobuf)
- **Database**: PostgreSQL 17+ with GORM (ORM)
- **Message Queue**: RabbitMQ
- **Authentication**: JWT tokens, OIDC
- **Authorization**: Casbin (RBAC)
- **Encryption**: AES-GCM (for secrets)
- **Testing**: Go testing, testify, gotestsum

### Frontend (TypeScript)

- **Language**: TypeScript
- **Framework**: React 18
- **Build Tool**: Vite
- **API Client**: Auto-generated from OpenAPI spec
- **State Management**: React hooks, context
- **UI Components**: Custom + Radix UI primitives
- **Testing**: Playwright (E2E)

### Infrastructure

- **Container**: Docker + Docker Compose
- **Orchestration**: Kubernetes (optional)
- **CI/CD**: Semaphore CI
- **Deployment**: Single host or K8s

### AWS SDKs (Relevant for Your Task)

- `github.com/aws/aws-sdk-go-v2` - AWS SDK for Go v2
- Services used: STS, IAM, EventBridge, ECR, Lambda, CodeArtifact

---

## Core Concepts

### 1. Organizations

- **Top-level tenant boundary** providing complete data isolation
- All resources belong to an organization
- Multi-tenancy is organization-scoped

### 2. Accounts & Users

- **Account**: A person's identity (email, name)
- **User**: A person's membership in a specific organization
- One account can have multiple users (one per organization)

### 3. Canvases

- **Workspace for building workflows**
- Contains workflow graph: nodes + edges
- Belongs to an organization

### 4. Workflows

- **Directed graph of nodes and edges**
- Each node is a component or trigger
- Edges define data flow and dependencies

### 5. Components

- **Actions you can run** (e.g., "Run Lambda", "Send Slack message")
- Defined by integrations or built-in
- Have configuration fields, execute logic, emit events

### 6. Triggers

- **Start workflows** based on events
- Types: webhooks, schedules, manual starts
- Can be integration-specific (e.g., "On GitHub Push")

### 7. Integrations

- **Connect SuperPlane to external services**
- Store encrypted credentials
- Expose components and triggers
- Examples: GitHub, AWS, Slack, PagerDuty

### 8. Events

- **Data flowing through workflow graphs**
- Created when triggers fire or components complete
- Routed through edges to downstream nodes

### 9. Executions

- **Instance of a component running**
- Has state: pending, running, completed, failed, waiting
- Stores input data, output data, logs

### 10. Queue Items

- **Work items for components**
- Created by EventRouter
- Processed by NodeQueueWorker → NodeExecutor

---

## Directory Structure

```
/workspace
├── cmd/                          # Application entry points
│   ├── server/                   # Main server (starts API + workers)
│   │   └── main.go               # Entry point → calls pkg/server.Start()
│   └── cli/                      # CLI tool
│
├── pkg/                          # Core Go packages
│   ├── server/                   # Server startup logic
│   │   └── server.go             # Starts Public API, Internal API, Workers
│   │
│   ├── grpc/                     # Internal gRPC API implementation
│   │   ├── actions/              # gRPC service handlers (business logic)
│   │   └── server.go             # gRPC server setup
│   │
│   ├── public/                   # Public API (HTTP + WebSocket)
│   │   └── server.go             # gRPC-Gateway, static file serving
│   │
│   ├── workers/                  # Background workers
│   │   ├── event_router.go       # Routes events through workflows
│   │   ├── node_executor.go      # Executes component logic
│   │   ├── node_queue_worker.go  # Processes queue items
│   │   ├── webhook_provisioner.go# Sets up webhooks
│   │   └── integration_request_worker.go # Syncs integrations
│   │
│   ├── models/                   # Database models (GORM)
│   │   ├── account.go
│   │   ├── organization.go
│   │   ├── integration.go
│   │   ├── workflow.go
│   │   ├── execution.go
│   │   └── ...
│   │
│   ├── core/                     # Core interfaces
│   │   ├── component.go          # Component interface
│   │   ├── integration.go        # Integration interface
│   │   └── trigger.go            # Trigger interface
│   │
│   ├── integrations/             # Integration implementations
│   │   ├── aws/                  # AWS integration
│   │   │   ├── aws.go            # Main integration file
│   │   │   ├── ecr/              # ECR components/triggers
│   │   │   ├── lambda/           # Lambda components
│   │   │   ├── codeartifact/     # CodeArtifact components/triggers
│   │   │   ├── eventbridge/      # EventBridge client
│   │   │   ├── iam/              # IAM client
│   │   │   └── common/           # Shared utilities
│   │   ├── github/               # GitHub integration
│   │   ├── slack/                # Slack integration
│   │   └── ...
│   │
│   ├── components/               # Built-in components (no integration)
│   │   ├── approval/             # Manual approval gate
│   │   ├── filter/               # Filter events
│   │   ├── http/                 # HTTP request
│   │   └── ...
│   │
│   ├── triggers/                 # Built-in triggers
│   │   ├── webhook/              # Generic webhook
│   │   ├── schedule/             # Cron schedule
│   │   └── start/                # Manual start
│   │
│   ├── registry/                 # Component/Integration registry
│   │   └── registry.go           # Global registry singleton
│   │
│   ├── configuration/            # Configuration field types
│   │   └── field.go              # Field definitions (string, number, list, etc.)
│   │
│   ├── database/                 # Database connection
│   │   └── database.go           # GORM setup
│   │
│   ├── authorization/            # RBAC (Casbin)
│   │   ├── authorization.go      # Authorization service
│   │   └── interceptor.go        # gRPC interceptor
│   │
│   ├── authentication/           # Authentication
│   │   └── authentication.go     # JWT validation
│   │
│   ├── crypto/                   # Encryption
│   │   └── crypto.go             # AES-GCM encryptor
│   │
│   └── ...
│
├── web_src/                      # Frontend React app
│   ├── src/
│   │   ├── api/                  # Auto-generated TypeScript client
│   │   ├── pages/                # Page components
│   │   │   └── workflowv2/       # Workflow canvas UI
│   │   │       ├── mappers/      # Integration/component mappers
│   │   │       └── renderers/    # Field renderers (for config)
│   │   ├── components/           # Reusable UI components
│   │   └── hooks/                # React hooks
│   └── ...
│
├── protos/                       # Protocol Buffer definitions
│   ├── canvases.proto            # Workflow canvas APIs
│   ├── integrations.proto        # Integration APIs
│   ├── components.proto          # Component APIs
│   ├── organizations.proto       # Organization APIs
│   └── ...
│
├── db/                           # Database
│   ├── migrations/               # Schema migrations (*.up.sql, *.down.sql)
│   └── structure.sql             # Current schema snapshot
│
├── test/                         # Tests
│   ├── e2e/                      # End-to-end tests (Playwright)
│   └── support/                  # Test utilities
│
├── docs/                         # Documentation
│   └── contributing/             # Contributor guides
│       ├── architecture.md       # Architecture overview
│       ├── building-an-integration.md
│       └── component-implementations.md
│
├── scripts/                      # Build/deployment scripts
├── templates/                    # Email/canvas templates
└── Makefile                      # Common dev tasks
```

---

## Data Flow & System Design

### Event-Driven Workflow Execution

SuperPlane uses an **event-driven architecture** where events flow through workflow graphs.

#### Event Lifecycle

```
1. Event Creation
   ├─ Trigger fires (webhook, schedule, manual)
   ├─ Component completes execution
   └─ Event record created in DB with state=pending

2. Event Routing (EventRouter worker)
   ├─ Polls DB for pending events every 500ms
   ├─ Finds downstream nodes via edges
   ├─ Creates QueueItems for each node
   └─ Publishes QueueItems to RabbitMQ

3. Queue Processing (NodeQueueWorker)
   ├─ Consumes QueueItems from RabbitMQ
   ├─ Creates Execution record (state=pending)
   └─ Calls component.ProcessQueueItem()

4. Component Execution (NodeExecutor worker)
   ├─ Component.Execute() runs business logic
   ├─ Calls external APIs (GitHub, AWS, Slack, etc.)
   ├─ Updates Execution state (completed/failed)
   └─ Creates new Event with output data

5. Event Propagation
   └─ New event flows to downstream nodes → repeat from step 2
```

#### Sequence Diagram

```
Trigger         EventRouter     NodeQueueWorker    NodeExecutor     Component
  │                  │                  │               │               │
  │ Fire event       │                  │               │               │
  │─────────────────>│                  │               │               │
  │                  │                  │               │               │
  │                  │ Poll pending     │               │               │
  │                  │ events           │               │               │
  │                  │─────────────────>│               │               │
  │                  │                  │               │               │
  │                  │ Create QueueItem │               │               │
  │                  │─────────────────>│               │               │
  │                  │                  │               │               │
  │                  │                  │ Process item  │               │
  │                  │                  │──────────────>│               │
  │                  │                  │               │               │
  │                  │                  │               │ Execute       │
  │                  │                  │               │──────────────>│
  │                  │                  │               │               │
  │                  │                  │               │               │ Call AWS API
  │                  │                  │               │               │─────────────>
  │                  │                  │               │               │
  │                  │                  │               │               │<─────────────
  │                  │                  │               │<──────────────│
  │                  │                  │<──────────────│               │
  │                  │<─────────────────│               │               │
  │                  │                  │               │               │
  │                  │ New event        │               │               │
  │                  │─────────────────>│               │               │
  │                  │ (repeat)         │               │               │
```

### Database Schema (Simplified)

```
accounts (id, email, name)
  │
  └─> account_providers (account_id, provider, provider_id, access_token)
  │
  └─> users (account_id, organization_id, ...)
        │
        └─> organizations (id, name, ...)
              │
              ├─> workflows (organization_id, canvas_id, name, nodes, edges)
              │     │
              │     ├─> workflow_events (workflow_id, state, data)
              │     │
              │     ├─> workflow_node_executions (workflow_id, node_id, state, data)
              │     │
              │     └─> workflow_node_queue_items (workflow_id, node_id, event_id)
              │
              ├─> app_installations (organization_id, app_name, configuration, metadata)
              │     │
              │     ├─> app_installation_secrets (installation_id, name, value)
              │     │
              │     ├─> app_installation_subscriptions (installation_id, node_id)
              │     │
              │     └─> webhooks (installation_id, url, secret, configuration)
              │
              └─> secrets (organization_id, name, value)
```

---

## Integration System Deep Dive

### What is an Integration?

An **Integration** is a connection to an external service (GitHub, AWS, Slack, etc.). It:

1. **Stores configuration** (e.g., AWS Role ARN, GitHub App ID)
2. **Manages credentials** (encrypted secrets, OAuth tokens)
3. **Exposes components** (actions like "Run Lambda", "Send Slack message")
4. **Exposes triggers** (events like "On GitHub Push", "On ECR Image Push")
5. **Sets up webhooks** (registers webhooks in external systems)

### Integration Interface

Every integration must implement `core.Integration`:

```go
type Integration interface {
    Name() string                                    // Unique identifier (e.g., "aws")
    Label() string                                   // Display name (e.g., "AWS")
    Icon() string                                    // Icon name
    Description() string                             // User-facing description
    Instructions() string                            // Markdown setup instructions
    Configuration() []configuration.Field            // Config fields (e.g., region, role ARN)
    Components() []Component                         // Actions exposed by integration
    Triggers() []Trigger                             // Events exposed by integration
    Sync(ctx SyncContext) error                      // Called when config changes
    Cleanup(ctx IntegrationCleanupContext) error     // Called when integration deleted
    Actions() []Action                               // Custom actions (e.g., "test connection")
    HandleAction(ctx IntegrationActionContext) error // Handle custom actions
    ListResources(resourceType string, ...) ([]IntegrationResource, error) // List resources
    HandleRequest(ctx HTTPRequestContext)            // HTTP request handler
}
```

### Integration Lifecycle

```
1. User creates integration
   └─> API creates app_installations record (state=pending)

2. Integration.Sync() called (IntegrationRequestWorker)
   ├─> Validates configuration
   ├─> Sets up credentials (OAuth, API keys, AssumeRole)
   ├─> Registers webhooks (via EventBridge, GitHub App, etc.)
   ├─> Stores metadata (integration-specific data)
   └─> Calls integration.Ready() → state=ready

3. Integration in use
   ├─> Components call integration APIs
   ├─> Triggers receive webhook events
   └─> Webhooks managed by WebhookProvisioner/WebhookCleanupWorker

4. User updates configuration
   └─> Integration.Sync() called again

5. User deletes integration
   ├─> Integration.Cleanup() called
   │   └─> Deregisters webhooks, revokes credentials
   └─> Record soft-deleted (deleted_at set)
```

---

## AWS Integration Architecture

### Overview

The AWS integration uses **AssumeRoleWithWebIdentity (IRSA-style)** for authentication:

1. SuperPlane provides an **OIDC identity provider** endpoint
2. User creates an **IAM OIDC Identity Provider** pointing to SuperPlane
3. User creates an **IAM Role** with trust policy allowing SuperPlane to assume it
4. SuperPlane issues **OIDC tokens** signed with its private key
5. SuperPlane calls **STS AssumeRoleWithWebIdentity** to get temporary AWS credentials
6. Components use these credentials to call AWS APIs

### Directory Structure

```
pkg/integrations/aws/
├── aws.go                    # Main integration (implements core.Integration)
├── sts.go                    # STS AssumeRoleWithWebIdentity logic
├── resources.go              # Resource listing (regions, etc.)
│
├── common/                   # Shared utilities
│   ├── common.go             # Tag normalization, account ID parsing
│   ├── error.go              # AWS error handling
│   └── time.go               # Time parsing utilities
│
├── ecr/                      # ECR integration
│   ├── client.go             # ECR API client
│   ├── get_image.go          # Component: Get image details
│   ├── scan_image.go         # Component: Scan image for vulnerabilities
│   ├── get_image_scan_findings.go # Component: Get scan results
│   ├── on_image_push.go      # Trigger: On ECR image push (EventBridge)
│   ├── on_image_scan.go      # Trigger: On ECR scan complete (EventBridge)
│   └── resources.go          # List ECR repositories
│
├── lambda/                   # Lambda integration
│   ├── client.go             # Lambda API client
│   ├── run_function.go       # Component: Run Lambda function
│   └── resources.go          # List Lambda functions
│
├── codeartifact/             # CodeArtifact integration
│   ├── client.go             # CodeArtifact API client
│   ├── get_package_version.go        # Component: Get package version
│   ├── copy_package_versions.go      # Component: Copy package versions
│   ├── create_repository.go          # Component: Create repository
│   ├── delete_repository.go          # Component: Delete repository
│   ├── on_package_version.go         # Trigger: On package version (EventBridge)
│   └── resources.go                  # List domains, repositories
│
├── eventbridge/              # EventBridge client
│   ├── client.go             # EventBridge API wrapper
│   └── common.go             # EventBridge rule patterns
│
└── iam/                      # IAM client
    └── client.go             # IAM policy/role operations
```

### AWS Integration Flow

#### Setup Flow

```
1. User creates AWS integration (config: roleArn, region, tags)

2. If roleArn empty → Show browser action
   └─> Guide user to create IAM OIDC provider + role

3. If roleArn provided → Sync()
   ├─> Call STS AssumeRoleWithWebIdentity
   │   ├─> Generate OIDC token (JWT signed with SuperPlane's private key)
   │   └─> Get temporary credentials (AccessKeyId, SecretAccessKey, SessionToken)
   │
   ├─> Configure IAM role (if needed)
   │   └─> Attach policies for EventBridge, ECR, Lambda, etc.
   │
   ├─> Configure EventBridge
   │   ├─> Create EventBridge connection (API destination)
   │   │   └─> Points to SuperPlane webhook endpoint
   │   └─> Create EventBridge rules for triggers
   │       ├─> ECR image push
   │       ├─> ECR scan complete
   │       └─> CodeArtifact package version
   │
   └─> Store metadata (accountID, eventBridgeConnectionArn, etc.)
```

#### Component Execution Flow

```
Component.Execute() called
  │
  ├─> Get AWS config
  │   ├─> Get integration metadata (roleArn, region)
  │   ├─> Generate OIDC token
  │   ├─> Call STS AssumeRoleWithWebIdentity
  │   └─> Get temporary credentials
  │
  ├─> Create AWS service client (ECR, Lambda, CodeArtifact)
  │   └─> Use temporary credentials
  │
  ├─> Call AWS API
  │   └─> DescribeImages, InvokeFunction, GetPackageVersion, etc.
  │
  └─> Return output data
```

#### Trigger Flow

```
AWS EventBridge → SuperPlane Webhook
  │
  ├─> Webhook endpoint: /webhooks/:id
  │   └─> Verifies signature (HMAC with webhook secret)
  │
  ├─> Trigger.HandleWebhook() called
  │   ├─> Parses EventBridge event
  │   ├─> Extracts data (image tag, scan results, package version)
  │   └─> Creates workflow event
  │
  └─> Event flows through workflow graph
```

### AWS Credentials Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          SuperPlane                              │
│                                                                   │
│  1. Generate OIDC Token                                           │
│     ┌──────────────────────────────────────────────────────┐    │
│     │ JWT (signed with SuperPlane's private RSA key)       │    │
│     │ {                                                     │    │
│     │   "iss": "https://app.superplane.com",               │    │
│     │   "sub": "org:12345:integration:67890",              │    │
│     │   "aud": "sts.amazonaws.com",                        │    │
│     │   "exp": now + 15min                                 │    │
│     │ }                                                     │    │
│     └──────────────────────────────────────────────────────┘    │
│                              │                                    │
└──────────────────────────────┼────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                         AWS STS                                  │
│                                                                   │
│  2. AssumeRoleWithWebIdentity                                    │
│     ├─> Verify JWT signature (using SuperPlane's public key)    │
│     ├─> Check trust policy on IAM role                          │
│     │   └─> Allows "sts.amazonaws.com" from SuperPlane OIDC     │
│     └─> Return temporary credentials                            │
│         ├─> AccessKeyId                                          │
│         ├─> SecretAccessKey                                      │
│         ├─> SessionToken                                         │
│         └─> Expiration (15 min - 12 hours)                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Service APIs                              │
│                                                                   │
│  3. Call AWS APIs with temporary credentials                     │
│     ├─> ECR: DescribeImages, StartImageScan, etc.               │
│     ├─> Lambda: InvokeFunction                                   │
│     └─> CodeArtifact: GetPackageVersion, etc.                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## How to Build AWS CodePipeline Integration

Now let's apply everything you've learned to build the **AWS CodePipeline integration**.

### Step 1: Understand AWS CodePipeline

#### Key Concepts

- **Pipeline**: Workflow that defines stages and actions
- **Stage**: Group of actions (e.g., Source, Build, Deploy)
- **Action**: Unit of work (e.g., S3 source, CodeBuild, Lambda)
- **Execution**: Instance of a pipeline run
- **Artifact**: Output from an action (stored in S3)

#### Useful AWS APIs

**CodePipeline Client** (`github.com/aws/aws-sdk-go-v2/service/codepipeline`):

- `GetPipeline`: Get pipeline definition
- `StartPipelineExecution`: Trigger pipeline
- `GetPipelineExecution`: Get execution status
- `GetPipelineState`: Get current state
- `ListPipelines`: List pipelines
- `ListPipelineExecutions`: List recent executions

**EventBridge Events**:

- `codepipeline.amazonaws.com` source
- Event types:
  - `CodePipeline Pipeline Execution State Change`
  - `CodePipeline Stage Execution State Change`
  - `CodePipeline Action Execution State Change`

### Step 2: Plan Components and Triggers

#### Components to Build

1. **Start Pipeline Execution**
   - Config: pipeline name, client request token (optional)
   - Action: Call `StartPipelineExecution`
   - Output: execution ID, pipeline ARN

2. **Get Pipeline Execution**
   - Config: pipeline name, execution ID
   - Action: Call `GetPipelineExecution`
   - Output: status, start time, end time, artifact revisions

3. **Get Pipeline State**
   - Config: pipeline name
   - Action: Call `GetPipelineState`
   - Output: stages, actions, latest execution per stage

#### Triggers to Build

1. **On Pipeline Execution State Change**
   - EventBridge pattern: pipeline execution started/succeeded/failed
   - Config: pipeline names (filter), states to watch
   - Output: pipeline name, execution ID, state

2. **On Pipeline Stage State Change**
   - EventBridge pattern: stage started/succeeded/failed
   - Config: pipeline names, stage names, states
   - Output: pipeline name, stage name, execution ID, state

3. **On Pipeline Action State Change**
   - EventBridge pattern: action started/succeeded/failed
   - Config: pipeline names, stage names, action names, states
   - Output: pipeline name, stage, action, execution ID, state

### Step 3: Create Directory and Files

```bash
mkdir -p /workspace/pkg/integrations/aws/codepipeline
cd /workspace/pkg/integrations/aws/codepipeline
```

#### File Structure

```
codepipeline/
├── client.go                         # CodePipeline API client
├── common.go                         # Shared types and utilities
├── resources.go                      # List pipelines
├── start_pipeline_execution.go       # Component
├── start_pipeline_execution_test.go  # Unit tests
├── get_pipeline_execution.go         # Component
├── get_pipeline_execution_test.go    # Unit tests
├── get_pipeline_state.go             # Component
├── get_pipeline_state_test.go        # Unit tests
├── on_pipeline_execution_state.go    # Trigger
├── on_pipeline_execution_state_test.go # Unit tests
├── example.go                        # Example output data
└── example_output_*.json             # Example outputs
```

### Step 4: Implement CodePipeline Client

**File**: `pkg/integrations/aws/codepipeline/client.go`

```go
package codepipeline

import (
    "context"

    awsConfig "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/credentials"
    "github.com/aws/aws-sdk-go-v2/service/codepipeline"
)

type Client struct {
    client *codepipeline.Client
}

func NewClient(ctx context.Context, region string, creds credentials.StaticCredentialsProvider) (*Client, error) {
    cfg, err := awsConfig.LoadDefaultConfig(ctx,
        awsConfig.WithRegion(region),
        awsConfig.WithCredentialsProvider(creds),
    )
    if err != nil {
        return nil, err
    }

    return &Client{
        client: codepipeline.NewFromConfig(cfg),
    }, nil
}

func (c *Client) StartPipelineExecution(ctx context.Context, pipelineName string, clientRequestToken *string) (*codepipeline.StartPipelineExecutionOutput, error) {
    return c.client.StartPipelineExecution(ctx, &codepipeline.StartPipelineExecutionInput{
        Name:               &pipelineName,
        ClientRequestToken: clientRequestToken,
    })
}

func (c *Client) GetPipelineExecution(ctx context.Context, pipelineName, executionID string) (*codepipeline.GetPipelineExecutionOutput, error) {
    return c.client.GetPipelineExecution(ctx, &codepipeline.GetPipelineExecutionInput{
        PipelineName:        &pipelineName,
        PipelineExecutionId: &executionID,
    })
}

func (c *Client) GetPipelineState(ctx context.Context, pipelineName string) (*codepipeline.GetPipelineStateOutput, error) {
    return c.client.GetPipelineState(ctx, &codepipeline.GetPipelineStateInput{
        Name: &pipelineName,
    })
}

func (c *Client) ListPipelines(ctx context.Context) ([]string, error) {
    output, err := c.client.ListPipelines(ctx, &codepipeline.ListPipelinesInput{})
    if err != nil {
        return nil, err
    }

    pipelines := []string{}
    for _, pipeline := range output.Pipelines {
        if pipeline.Name != nil {
            pipelines = append(pipelines, *pipeline.Name)
        }
    }

    return pipelines, nil
}
```

### Step 5: Implement "Start Pipeline Execution" Component

**File**: `pkg/integrations/aws/codepipeline/start_pipeline_execution.go`

```go
package codepipeline

import (
    "context"
    "fmt"

    "github.com/mitchellh/mapstructure"
    "github.com/superplanehq/superplane/pkg/configuration"
    "github.com/superplanehq/superplane/pkg/core"
    awsIntegration "github.com/superplanehq/superplane/pkg/integrations/aws"
)

type StartPipelineExecution struct{}

type StartPipelineExecutionSpec struct {
    PipelineName       string  `json:"pipelineName" mapstructure:"pipelineName"`
    ClientRequestToken *string `json:"clientRequestToken,omitempty" mapstructure:"clientRequestToken"`
}

func (c *StartPipelineExecution) Name() string {
    return "aws.codepipeline.start-pipeline-execution"
}

func (c *StartPipelineExecution) Label() string {
    return "Start Pipeline Execution"
}

func (c *StartPipelineExecution) Description() string {
    return "Start a new execution of an AWS CodePipeline pipeline"
}

func (c *StartPipelineExecution) Documentation() string {
    return `Triggers a pipeline execution in AWS CodePipeline.

## Configuration

- **Pipeline Name**: The name of the pipeline to execute
- **Client Request Token**: Optional idempotency token

## Output

- **executionId**: The ID of the pipeline execution
- **pipelineArn**: The ARN of the pipeline
`
}

func (c *StartPipelineExecution) Icon() string {
    return "aws.codepipeline"
}

func (c *StartPipelineExecution) Color() string {
    return "#FF9900" // AWS orange
}

func (c *StartPipelineExecution) ExampleOutput() map[string]any {
    return map[string]any{
        "executionId":  "abc123def456",
        "pipelineArn":  "arn:aws:codepipeline:us-east-1:123456789012:my-pipeline",
    }
}

func (c *StartPipelineExecution) OutputChannels(configuration any) []core.OutputChannel {
    return []core.OutputChannel{core.DefaultOutputChannel}
}

func (c *StartPipelineExecution) Configuration() []configuration.Field {
    return []configuration.Field{
        {
            Name:        "pipelineName",
            Label:       "Pipeline Name",
            Type:        configuration.FieldTypeString,
            Required:    true,
            Description: "The name of the pipeline to execute",
        },
        {
            Name:        "clientRequestToken",
            Label:       "Client Request Token",
            Type:        configuration.FieldTypeString,
            Required:    false,
            Description: "Optional idempotency token",
        },
    }
}

func (c *StartPipelineExecution) Setup(ctx core.SetupContext) error {
    spec := StartPipelineExecutionSpec{}
    if err := mapstructure.Decode(ctx.Configuration, &spec); err != nil {
        return fmt.Errorf("failed to decode configuration: %w", err)
    }

    if spec.PipelineName == "" {
        return fmt.Errorf("pipelineName is required")
    }

    return nil
}

func (c *StartPipelineExecution) ProcessQueueItem(ctx core.ProcessQueueContext) (*uuid.UUID, error) {
    return nil, nil // Synchronous execution
}

func (c *StartPipelineExecution) Execute(ctx core.ExecutionContext) error {
    spec := StartPipelineExecutionSpec{}
    if err := mapstructure.Decode(ctx.Configuration, &spec); err != nil {
        return ctx.ExecutionState.Failed(fmt.Errorf("failed to decode configuration: %w", err))
    }

    // Get AWS config and credentials
    awsConfig, err := awsIntegration.GetAWSConfig(ctx)
    if err != nil {
        return ctx.ExecutionState.Failed(fmt.Errorf("failed to get AWS config: %w", err))
    }

    // Create CodePipeline client
    client, err := NewClient(context.Background(), awsConfig.Region, awsConfig.Credentials)
    if err != nil {
        return ctx.ExecutionState.Failed(fmt.Errorf("failed to create CodePipeline client: %w", err))
    }

    // Start pipeline execution
    output, err := client.StartPipelineExecution(context.Background(), spec.PipelineName, spec.ClientRequestToken)
    if err != nil {
        return ctx.ExecutionState.Failed(fmt.Errorf("failed to start pipeline execution: %w", err))
    }

    // Return output
    return ctx.ExecutionState.Completed(map[string]any{
        "executionId": *output.PipelineExecutionId,
        "pipelineArn": fmt.Sprintf("arn:aws:codepipeline:%s:%s:%s", awsConfig.Region, awsConfig.AccountID, spec.PipelineName),
    })
}

func (c *StartPipelineExecution) Actions() []core.Action {
    return nil
}

func (c *StartPipelineExecution) HandleAction(ctx core.ActionContext) error {
    return fmt.Errorf("no actions defined")
}

func (c *StartPipelineExecution) HandleWebhook(ctx core.WebhookRequestContext) (int, error) {
    return 404, fmt.Errorf("webhooks not supported")
}

func (c *StartPipelineExecution) Cancel(ctx core.ExecutionContext) error {
    return nil
}

func (c *StartPipelineExecution) Cleanup(ctx core.SetupContext) error {
    return nil
}
```

### Step 6: Register Components in AWS Integration

**File**: `pkg/integrations/aws/aws.go`

Add import:

```go
import (
    // ... existing imports
    "github.com/superplanehq/superplane/pkg/integrations/aws/codepipeline"
)
```

Add components to `Components()`:

```go
func (a *AWS) Components() []core.Component {
    return []core.Component{
        // ... existing components
        &codepipeline.StartPipelineExecution{},
        &codepipeline.GetPipelineExecution{},
        &codepipeline.GetPipelineState{},
    }
}
```

Add triggers to `Triggers()`:

```go
func (a *AWS) Triggers() []core.Trigger {
    return []core.Trigger{
        // ... existing triggers
        &codepipeline.OnPipelineExecutionState{},
    }
}
```

### Step 7: Implement EventBridge Trigger

**File**: `pkg/integrations/aws/codepipeline/on_pipeline_execution_state.go`

```go
package codepipeline

import (
    "encoding/json"
    "fmt"

    "github.com/mitchellh/mapstructure"
    "github.com/superplanehq/superplane/pkg/configuration"
    "github.com/superplanehq/superplane/pkg/core"
    "github.com/superplanehq/superplane/pkg/integrations/aws/eventbridge"
)

type OnPipelineExecutionState struct{}

type OnPipelineExecutionStateSpec struct {
    PipelineNames []string `json:"pipelineNames,omitempty" mapstructure:"pipelineNames"`
    States        []string `json:"states,omitempty" mapstructure:"states"`
}

type PipelineExecutionEvent struct {
    Version    string `json:"version"`
    ID         string `json:"id"`
    DetailType string `json:"detail-type"`
    Source     string `json:"source"`
    Account    string `json:"account"`
    Time       string `json:"time"`
    Region     string `json:"region"`
    Detail     struct {
        Pipeline      string `json:"pipeline"`
        ExecutionID   string `json:"execution-id"`
        State         string `json:"state"`
        Version       int    `json:"version"`
    } `json:"detail"`
}

func (t *OnPipelineExecutionState) Name() string {
    return "aws.codepipeline.on-pipeline-execution-state"
}

func (t *OnPipelineExecutionState) Label() string {
    return "On Pipeline Execution State Change"
}

func (t *OnPipelineExecutionState) Description() string {
    return "Triggers when a CodePipeline pipeline execution state changes"
}

func (t *OnPipelineExecutionState) Documentation() string {
    return `Triggers when a CodePipeline pipeline execution changes state (e.g., STARTED, SUCCEEDED, FAILED).

## Configuration

- **Pipeline Names**: Filter by pipeline names (optional)
- **States**: Filter by execution states (optional): STARTED, SUCCEEDED, FAILED, CANCELED, RESUMED, SUPERSEDED

## Output

- **pipeline**: Pipeline name
- **executionId**: Execution ID
- **state**: Execution state
- **version**: Pipeline version
`
}

func (t *OnPipelineExecutionState) Icon() string {
    return "aws.codepipeline"
}

func (t *OnPipelineExecutionState) Color() string {
    return "#FF9900"
}

func (t *OnPipelineExecutionState) ExampleOutput() map[string]any {
    return map[string]any{
        "pipeline":    "my-pipeline",
        "executionId": "abc123def456",
        "state":       "SUCCEEDED",
        "version":     1,
    }
}

func (t *OnPipelineExecutionState) OutputChannels(configuration any) []core.OutputChannel {
    return []core.OutputChannel{core.DefaultOutputChannel}
}

func (t *OnPipelineExecutionState) Configuration() []configuration.Field {
    return []configuration.Field{
        {
            Name:        "pipelineNames",
            Label:       "Pipeline Names",
            Type:        configuration.FieldTypeList,
            Required:    false,
            Description: "Filter by pipeline names (leave empty for all)",
            TypeOptions: &configuration.TypeOptions{
                List: &configuration.ListTypeOptions{
                    ItemLabel: "Pipeline",
                    ItemDefinition: &configuration.ListItemDefinition{
                        Type: configuration.FieldTypeString,
                    },
                },
            },
        },
        {
            Name:        "states",
            Label:       "States",
            Type:        configuration.FieldTypeList,
            Required:    false,
            Description: "Filter by execution states",
            TypeOptions: &configuration.TypeOptions{
                List: &configuration.ListTypeOptions{
                    ItemLabel: "State",
                    ItemDefinition: &configuration.ListItemDefinition{
                        Type: configuration.FieldTypeString,
                        Enum: []string{"STARTED", "SUCCEEDED", "FAILED", "CANCELED", "RESUMED", "SUPERSEDED"},
                    },
                },
            },
        },
    }
}

func (t *OnPipelineExecutionState) Setup(ctx core.SetupContext) error {
    spec := OnPipelineExecutionStateSpec{}
    if err := mapstructure.Decode(ctx.Configuration, &spec); err != nil {
        return fmt.Errorf("failed to decode configuration: %w", err)
    }

    // Build EventBridge pattern
    pattern := eventbridge.EventPattern{
        Source:     []string{"aws.codepipeline"},
        DetailType: []string{"CodePipeline Pipeline Execution State Change"},
    }

    if len(spec.PipelineNames) > 0 {
        pattern.Detail = map[string]any{
            "pipeline": spec.PipelineNames,
        }
    }

    if len(spec.States) > 0 {
        if pattern.Detail == nil {
            pattern.Detail = make(map[string]any)
        }
        pattern.Detail["state"] = spec.States
    }

    // Request webhook with pattern
    return ctx.Integration.RequestWebhook(pattern)
}

func (t *OnPipelineExecutionState) HandleWebhook(ctx core.WebhookRequestContext) (int, error) {
    var event PipelineExecutionEvent
    if err := json.NewDecoder(ctx.Request.Body).Decode(&event); err != nil {
        return 400, fmt.Errorf("failed to decode webhook payload: %w", err)
    }

    // Emit event
    err := ctx.Events.Emit(core.DefaultOutputChannel.Name, map[string]any{
        "pipeline":    event.Detail.Pipeline,
        "executionId": event.Detail.ExecutionID,
        "state":       event.Detail.State,
        "version":     event.Detail.Version,
    })

    if err != nil {
        return 500, fmt.Errorf("failed to emit event: %w", err)
    }

    return 200, nil
}

func (t *OnPipelineExecutionState) Cleanup(ctx core.SetupContext) error {
    return nil
}

func (t *OnPipelineExecutionState) ProcessQueueItem(ctx core.ProcessQueueContext) (*uuid.UUID, error) {
    return nil, nil
}

func (t *OnPipelineExecutionState) Actions() []core.Action {
    return nil
}

func (t *OnPipelineExecutionState) HandleAction(ctx core.ActionContext) error {
    return fmt.Errorf("no actions defined")
}
```

### Step 8: Add EventBridge Configuration

The AWS integration's `Sync()` method already handles EventBridge setup. When a trigger calls `ctx.Integration.RequestWebhook(pattern)`, it:

1. Creates a webhook record in DB with the pattern
2. WebhookProvisioner worker picks it up
3. Calls AWS EventBridge API to create:
   - EventBridge Connection (API destination pointing to SuperPlane)
   - EventBridge Rule (with the pattern)
   - EventBridge Target (pointing to the Connection)

You may need to update `configureEventBridge()` in `aws.go` to handle CodePipeline patterns.

### Step 9: Add Frontend Mappers

**File**: `web_src/src/pages/workflowv2/mappers/aws.tsx`

Add CodePipeline icon and component mappings:

```typescript
// Add to icons
export const awsCodePipelineIcon = () => (
  <img src="/assets/icons/integrations/aws.codepipeline.svg" alt="AWS CodePipeline" />
);

// Add to component mappers
componentMappers["aws.codepipeline.start-pipeline-execution"] = {
  icon: awsCodePipelineIcon,
  label: "Start Pipeline",
};

componentMappers["aws.codepipeline.get-pipeline-execution"] = {
  icon: awsCodePipelineIcon,
  label: "Get Execution",
};

// Add to trigger mappers
triggerMappers["aws.codepipeline.on-pipeline-execution-state"] = {
  icon: awsCodePipelineIcon,
  label: "Pipeline State Change",
};
```

Add icon file: `web_src/src/assets/icons/integrations/aws.codepipeline.svg`

### Step 10: Write Tests

**File**: `pkg/integrations/aws/codepipeline/start_pipeline_execution_test.go`

```go
package codepipeline

import (
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/superplanehq/superplane/pkg/core"
)

func TestStartPipelineExecution_Setup(t *testing.T) {
    component := &StartPipelineExecution{}

    t.Run("valid configuration", func(t *testing.T) {
        ctx := core.SetupContext{
            Configuration: map[string]any{
                "pipelineName": "my-pipeline",
            },
        }

        err := component.Setup(ctx)
        assert.NoError(t, err)
    })

    t.Run("missing pipeline name", func(t *testing.T) {
        ctx := core.SetupContext{
            Configuration: map[string]any{},
        }

        err := component.Setup(ctx)
        assert.Error(t, err)
        assert.Contains(t, err.Error(), "pipelineName is required")
    })
}

// Add more tests for Execute() using mock clients
```

### Step 11: Build, Test, Lint

```bash
# Format Go code
make format.go

# Run linter
make lint

# Run tests
make test PKG_TEST_PACKAGES=./pkg/integrations/aws/codepipeline/...

# Build
make check.build.app

# Regenerate protobuf (if you added new proto messages)
make pb.gen

# Regenerate OpenAPI spec
make openapi.spec.gen
make openapi.client.gen
make openapi.web.client.gen
```

### Step 12: Test E2E

Create an E2E test in `test/e2e/integrations/aws_codepipeline_test.go`:

```go
package integrations_test

import (
    "testing"

    "github.com/superplanehq/superplane/test/support"
)

func TestAWSCodePipeline(t *testing.T) {
    ctx := support.NewE2EContext(t)

    // Create AWS integration
    integration := ctx.CreateAWSIntegration(support.AWSIntegrationConfig{
        RoleArn: "arn:aws:iam::123456789012:role/superplane-test",
        Region:  "us-east-1",
    })

    // Create canvas with CodePipeline trigger
    canvas := ctx.CreateCanvas()
    trigger := ctx.AddNode(canvas, "aws.codepipeline.on-pipeline-execution-state", map[string]any{
        "pipelineNames": []string{"test-pipeline"},
        "states":        []string{"SUCCEEDED"},
    })

    // Add component
    component := ctx.AddNode(canvas, "http", map[string]any{
        "method": "POST",
        "url":    ctx.WebhookURL("/test-webhook"),
    })

    ctx.AddEdge(canvas, trigger, component)

    // Simulate EventBridge webhook
    ctx.SendWebhook(trigger, map[string]any{
        "detail": map[string]any{
            "pipeline":     "test-pipeline",
            "execution-id": "exec-123",
            "state":        "SUCCEEDED",
        },
    })

    // Assert component executed
    ctx.AssertExecutionCompleted(component)
}
```

### Step 13: Documentation

Add documentation to `docs/components/aws.md`:

```markdown
## CodePipeline

### Start Pipeline Execution

Triggers a pipeline execution in AWS CodePipeline.

**Configuration:**
- Pipeline Name: The name of the pipeline to execute

**Output:**
- executionId: The ID of the pipeline execution
- pipelineArn: The ARN of the pipeline

### On Pipeline Execution State Change

Triggers when a CodePipeline pipeline execution state changes.

**Configuration:**
- Pipeline Names: Filter by pipeline names (optional)
- States: Filter by execution states (optional)

**Output:**
- pipeline: Pipeline name
- executionId: Execution ID
- state: Execution state
```

Generate component docs:

```bash
make gen.components.docs
```

---

## Development Workflow

### Daily Development Loop

1. **Start dev environment**:
   ```bash
   make dev.setup   # First time only
   make dev.start   # Starts all services
   ```

2. **Make changes** to Go code or frontend

3. **Format code**:
   ```bash
   make format.go   # Format Go
   make format.js   # Format TypeScript
   ```

4. **Run tests**:
   ```bash
   make test PKG_TEST_PACKAGES=./pkg/integrations/aws/codepipeline/...
   ```

5. **Check build**:
   ```bash
   make lint
   make check.build.app
   make check.build.ui
   ```

6. **Commit and push**:
   ```bash
   git add .
   git commit -m "feat: Add AWS CodePipeline integration"
   git push -u origin cursor/aws-codepipeline-integration-5396
   ```

### Docker Commands (when Go not installed locally)

```bash
# Run Go commands in Docker
docker compose -f docker-compose.dev.yml exec app go test ./pkg/integrations/aws/codepipeline/...

# Run make commands
docker compose -f docker-compose.dev.yml exec app make lint
```

### Environment Variables

Key environment variables (set in `.env` or `docker-compose.dev.yml`):

```bash
# Database
DATABASE_URL=postgres://superplane:password@postgres:5432/superplane_dev
DB_NAME=superplane_dev

# RabbitMQ
RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/

# Encryption
ENCRYPTION_KEY=your-32-byte-encryption-key

# JWT
JWT_SECRET=your-jwt-secret

# OIDC (for AWS integration)
OIDC_KEYS_PATH=/app/keys/oidc

# Base URL
BASE_URL=http://localhost:8000
PUBLIC_API_BASE_PATH=/api/v1

# Workers (enable all in dev)
START_PUBLIC_API=yes
START_INTERNAL_API=yes
START_GRPC_GATEWAY=yes
START_WEB_SERVER=yes
START_CONSUMERS=yes
START_EVENT_ROUTER=yes
START_NODE_EXECUTOR=yes
START_NODE_QUEUE_WORKER=yes
START_WEBHOOK_PROVISIONER=yes
START_WEBHOOK_CLEANUP_WORKER=yes
START_INTEGRATION_REQUEST_WORKER=yes
START_EVENT_DISTRIBUTER=yes
```

---

## Testing & Debugging

### Unit Tests

```bash
# Run all tests
make test

# Run specific package
make test PKG_TEST_PACKAGES=./pkg/integrations/aws/codepipeline/...

# Run with verbose output
docker compose -f docker-compose.dev.yml exec app go test -v ./pkg/integrations/aws/codepipeline/...

# Run single test
docker compose -f docker-compose.dev.yml exec app go test -v ./pkg/integrations/aws/codepipeline/... -run TestStartPipelineExecution_Setup
```

### E2E Tests

```bash
# Setup E2E environment (first time)
make test.e2e.setup

# Run E2E tests
make e2e E2E_TEST_PACKAGES=./test/e2e/integrations/...

# Run single E2E test
make test.e2e.single FILE=test/e2e/integrations/aws_codepipeline_test.go LINE=10
```

### Debugging

1. **Add log statements**:
   ```go
   ctx.Logger.Info("Starting pipeline execution", "pipeline", spec.PipelineName)
   ctx.Logger.Error("Failed to start pipeline", "error", err)
   ```

2. **Check logs**:
   ```bash
   docker compose -f docker-compose.dev.yml logs -f app
   ```

3. **Inspect database**:
   ```bash
   docker compose -f docker-compose.dev.yml exec postgres psql -U superplane -d superplane_dev
   ```

4. **Check RabbitMQ**:
   - UI: http://localhost:15672 (guest/guest)

5. **Use debugger**:
   - Add `import "github.com/davecgh/go-spew/spew"`
   - Use `spew.Dump(variable)` to inspect data structures

---

## Troubleshooting Guide

### Common Issues

#### 1. "failed to get AWS config"

**Cause**: AWS integration not synced or credentials expired.

**Solution**:
- Check integration state in DB: `SELECT state FROM app_installations WHERE app_name = 'aws'`
- Check metadata: `SELECT metadata FROM app_installations WHERE app_name = 'aws'`
- Re-sync integration by updating configuration in UI

#### 2. "webhook not triggering"

**Cause**: EventBridge rule not created or webhook secret mismatch.

**Solution**:
- Check webhooks table: `SELECT * FROM webhooks WHERE installation_id = '...'`
- Check EventBridge rule in AWS console
- Check webhook logs: `SELECT * FROM webhook_deliveries`

#### 3. "component not found"

**Cause**: Component not registered or integration not imported.

**Solution**:
- Check `init()` in integration file: `registry.RegisterIntegration("aws", &AWS{})`
- Check import in `cmd/server/main.go`: `_ "github.com/superplanehq/superplane/pkg/integrations/aws"`
- Rebuild: `make check.build.app`

#### 4. "transaction isolation broken"

**Cause**: Called `database.Conn()` inside a function that receives `tx *gorm.DB`.

**Solution**:
- Always pass `tx` through the call chain
- Use `*InTransaction()` variants of model methods
- Example:
  ```go
  // Bad
  func process(tx *gorm.DB) {
      user, _ := models.FindUser(id) // Uses database.Conn() internally
  }

  // Good
  func process(tx *gorm.DB) {
      user, _ := models.FindUserInTransaction(tx, id)
  }
  ```

#### 5. "migration failed"

**Cause**: Migration syntax error or duplicate migration.

**Solution**:
- **NEVER manually create migration files**
- Use `make db.migration.create NAME=add-codepipeline-support`
- Run migration: `make db.migrate DB_NAME=superplane_dev`
- Rollback if needed: Edit migration file and re-run

#### 6. "linter errors"

**Cause**: Code doesn't follow style guidelines.

**Solution**:
- Run formatter: `make format.go`
- Fix imports: `goimports -w file.go`
- Check errors: `make lint`

#### 7. "protobuf changes not reflected"

**Cause**: Protobuf not regenerated.

**Solution**:
```bash
make pb.gen
make openapi.spec.gen
make openapi.client.gen
make openapi.web.client.gen
```

### Debugging Workflow Execution

1. **Find execution**:
   ```sql
   SELECT * FROM workflow_node_executions WHERE node_id = 'your-node-id' ORDER BY created_at DESC LIMIT 10;
   ```

2. **Check execution state**:
   ```sql
   SELECT id, state, error_message, output_data FROM workflow_node_executions WHERE id = '...';
   ```

3. **Check events**:
   ```sql
   SELECT * FROM workflow_events WHERE workflow_id = '...' ORDER BY created_at DESC LIMIT 10;
   ```

4. **Check queue items**:
   ```sql
   SELECT * FROM workflow_node_queue_items WHERE node_id = '...' ORDER BY created_at DESC LIMIT 10;
   ```

### Performance Issues

1. **Slow event routing**:
   - Check EventRouter logs
   - Ensure indexes on `workflow_events` table
   - Check for large workflows (>100 nodes)

2. **Worker backlog**:
   - Check RabbitMQ queue depth
   - Scale workers horizontally (increase replica count)

3. **Database slow queries**:
   - Enable query logging: `SET log_statement = 'all';`
   - Use `EXPLAIN ANALYZE` on slow queries
   - Add indexes as needed

---

## System Design Diagrams

### Component Execution Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     Component Execution                          │
│                                                                   │
│  User triggers workflow                                           │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────┐                                                │
│  │   Trigger    │ (webhook, schedule, manual)                    │
│  │  fires       │                                                │
│  └──────┬───────┘                                                │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────────────────────┐        │
│  │  Create Event (state=pending)                        │        │
│  │  ├─> workflow_events table                           │        │
│  │  └─> data = trigger output                           │        │
│  └──────┬───────────────────────────────────────────────┘        │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────────────────────┐        │
│  │  EventRouter (polls every 500ms)                     │        │
│  │  ├─> Find pending events                             │        │
│  │  ├─> Find downstream nodes via edges                 │        │
│  │  ├─> Create QueueItems                               │        │
│  │  └─> Publish to RabbitMQ                             │        │
│  └──────┬───────────────────────────────────────────────┘        │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────────────────────┐        │
│  │  NodeQueueWorker (consumes from RabbitMQ)            │        │
│  │  ├─> Create Execution (state=pending)                │        │
│  │  ├─> Call component.ProcessQueueItem()               │        │
│  │  └─> If sync, call component.Execute() immediately   │        │
│  └──────┬───────────────────────────────────────────────┘        │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────────────────────┐        │
│  │  Component.Execute()                                  │        │
│  │  ├─> Get AWS credentials (STS AssumeRoleWithWebIdentity) │   │
│  │  ├─> Create AWS service client (CodePipeline)        │        │
│  │  ├─> Call AWS API (StartPipelineExecution)           │        │
│  │  ├─> Update execution (state=running)                │        │
│  │  └─> Return output data                              │        │
│  └──────┬───────────────────────────────────────────────┘        │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────────────────────┐        │
│  │  Execution.Completed()                                │        │
│  │  ├─> Update execution (state=completed)              │        │
│  │  ├─> Create new Event with output data               │        │
│  │  └─> Event flows to downstream nodes (repeat)        │        │
│  └──────────────────────────────────────────────────────┘        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### AWS Credentials Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   AWS Credentials Flow                           │
│                                                                   │
│  Component needs AWS credentials                                 │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────────────────────┐        │
│  │  GetAWSConfig(ctx)                                    │        │
│  │  ├─> Get integration from ctx                         │        │
│  │  ├─> Get integration.Configuration (roleArn, region)  │        │
│  │  └─> Get integration.Metadata (accountID, etc.)      │        │
│  └──────┬───────────────────────────────────────────────┘        │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────────────────────┐        │
│  │  GenerateOIDCToken()                                  │        │
│  │  ├─> Load OIDC private key                            │        │
│  │  ├─> Create JWT with claims:                         │        │
│  │  │   • iss: https://app.superplane.com               │        │
│  │  │   • sub: org:123:integration:456                  │        │
│  │  │   • aud: sts.amazonaws.com                        │        │
│  │  │   • exp: now + 15min                              │        │
│  │  └─> Sign with RSA private key                       │        │
│  └──────┬───────────────────────────────────────────────┘        │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────────────────────┐        │
│  │  STS AssumeRoleWithWebIdentity                        │        │
│  │  ├─> RoleArn: arn:aws:iam::123:role/superplane       │        │
│  │  ├─> WebIdentityToken: <OIDC JWT>                    │        │
│  │  ├─> RoleSessionName: superplane-org-123-int-456     │        │
│  │  └─> DurationSeconds: 3600                           │        │
│  └──────┬───────────────────────────────────────────────┘        │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────────────────────┐        │
│  │  AWS STS Returns Credentials                          │        │
│  │  ├─> AccessKeyId: ASIA...                            │        │
│  │  ├─> SecretAccessKey: ...                            │        │
│  │  ├─> SessionToken: ...                               │        │
│  │  └─> Expiration: 1 hour from now                     │        │
│  └──────┬───────────────────────────────────────────────┘        │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────────────────────┐        │
│  │  Create AWS Service Client                            │        │
│  │  ├─> CodePipeline client                             │        │
│  │  ├─> ECR client                                       │        │
│  │  ├─> Lambda client                                    │        │
│  │  └─> Use temporary credentials                       │        │
│  └──────┬───────────────────────────────────────────────┘        │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────────────────────┐        │
│  │  Call AWS API                                         │        │
│  │  ├─> StartPipelineExecution                          │        │
│  │  ├─> GetPipelineExecution                            │        │
│  │  └─> GetPipelineState                                │        │
│  └──────────────────────────────────────────────────────┘        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Conclusion

You now have a complete understanding of SuperPlane's architecture, codebase structure, and how to build the AWS CodePipeline integration. Key takeaways:

1. **SuperPlane is a DevOps control plane** - Think Zapier for DevOps, orchestrating workflows across GitHub, AWS, Slack, PagerDuty, etc.

2. **Event-driven architecture** - Events flow through workflow graphs, triggering component execution via RabbitMQ queues.

3. **Modular monolith** - Public API, Internal API, and Workers are separate but run in the same process (can be scaled independently).

4. **AWS integration uses OIDC** - AssumeRoleWithWebIdentity for secure, keyless authentication.

5. **Components are reusable actions** - Define configuration, execute logic, emit events.

6. **Triggers start workflows** - Webhooks, schedules, or manual starts.

7. **Integrations tie it all together** - Manage credentials, expose components/triggers, set up webhooks.

8. **Testing is critical** - Unit tests for Setup/Execute, E2E tests for full workflows.

9. **Development workflow** - Format → Test → Lint → Build → Commit → Push.

10. **Troubleshooting** - Check logs, inspect DB, verify EventBridge rules, trace executions.

### Next Steps

1. Read through existing AWS components (ECR, Lambda, CodeArtifact) to understand patterns
2. Review the integration guide: `docs/contributing/building-an-integration.md`
3. Set up your dev environment: `make dev.setup && make dev.start`
4. Build the CodePipeline integration following the steps above
5. Write comprehensive tests (unit + E2E)
6. Submit PR with video demo

You're now ready to be a major contributor to SuperPlane! Good luck with the AWS CodePipeline integration. 🚀

---

**Questions? Issues?**

- Check the docs: `docs/contributing/`
- Search the codebase: `grep -r "pattern" pkg/`
- Ask in Discord: https://discord.superplane.com
- Open an issue: https://github.com/superplanehq/superplane/issues

**Remember**: You know this codebase inside and out now. Trust your understanding, follow the patterns, and build something awesome!
