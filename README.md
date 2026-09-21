# Security Agent

Design and research for an internal security assessment agent covering Model Context Protocol (MCP) servers and agent skill packages.

**Status:** proposed design. This repository currently contains research findings and design decisions; it does not yet contain an executable scanner.

**Repository:** [dliu-lab/security-agent](https://github.com/dliu-lab/security-agent).

## Objective

Build Security Agent around one trusted scanner plugin containing `mcp-assessment` and `skill-assessment` skills, backed by a shared deterministic engine, corporate-control mappings, evidence, and reporting. Support direct execution in a compatible local coding assistant and a hosted agent on Amazon Bedrock AgentCore Runtime. Assess internal and external MCPs and skill packages using available source/configuration and explicitly permitted endpoint evidence, with additional analysis through an approved AWS Bedrock inference profile for remediation in deep mode.

AgentCore Runtime remains the hosting environment for the remote service and replaces the earlier EKS deployment proposal. Locally, the same marketplace scanner plugin invokes an installed, pinned engine CLI/tool adapter without an AgentCore call. The application repository will produce both the hosted image and a reusable local engine package; neither is implemented yet. An optional, separate client plugin invokes the hosted API when remote execution is preferred. Both execution paths support fast and deep mode.

## Start here

Read the [Security Agent high-level design](docs/high-level-design.md) for architecture diagrams, component responsibilities, API and execution flows, data/security boundaries, deployment, resilience, and open decisions.

Read [the detailed assessment design](docs/assessment-design.md) for control coverage, rule contracts, evidence requirements, and implementation considerations.

The [four assessment entry points](docs/high-level-design.md#four-assessment-entry-points) cover skill repositories, MCP source repositories, MCP endpoints and installed CLI skills/MCPs. Repository, endpoint and local-installation collectors feed the same two scanners. Installed evidence can be assessed locally or explicitly exported for hosted assessment.

For repository scans from GitHub Actions, the proposed [remediation workflow](docs/high-level-design.md#github-actions-assessment-and-optional-remediation-pr) adds a `workflow_dispatch` boolean, `create_remediation_pr`, defaulting to `false`. Enabling it requires deep mode: the API returns findings plus a proposed patch, and trusted GHA jobs validate the changes and open a draft PR. Original findings remain intact and no automatic merge occurs. This workflow is designed but not implemented.

## Confirmed scope

- Assess both internal and external MCPs using available repository source code, configuration, deployment artifacts, and/or endpoint evidence.
- Assess agent skill packages, including instruction files, referenced scripts, resource paths, and relevant containing-plugin configuration. Keep separate target adapters/rule packs for MCPs and skills, with shared result contracts.
- Load only the trusted scanner plugin. Skill packages under review are evidence: do not install, activate, or execute them.
- For local assessment, start the assistant in a neutral trusted workspace and pass the target as evidence; prevent automatic loading of target instructions, hooks, plugins, or MCP configuration.
- Record ownership, hosting, and evidence availability separately. Select checks according to the evidence and applicable controls, rather than assuming external MCPs are endpoint-only.
- Inspect supplied files and make explicitly allowed discovery connections.
- Inspect selected installed CLI packages/configuration through a local read-only collector; do not launch configured stdio servers or implicitly upload workstation evidence.
- Deliver a local scanner plugin/engine integration and a hosted AgentCore agent, with typed submission, status, report, cancellation, and recovery operations for the remote API.
- Local-assistant and hosted-agent orchestration use approved Bedrock inference in both modes. A remote client assistant adds its own host inference when used. Fast/deep distinguish assessment analysis, independently of local/hosted execution.
- Fast mode runs deterministic assessment scripts and returns their findings; the assessment engine makes no model calls for detection or remediation.
- Deep mode runs the same deterministic checks, then makes an additional call through an approved AWS Bedrock inference profile for contextual remediation suggestions.
- Use the corporate control catalogue as the primary policy reference, with reviewed mappings to applicable MCP, agentic, and software-security categories.
- Runtime assessment must not depend on external scanning services or public inference APIs. Approved AWS infrastructure, explicitly approved repository retrieval, target discovery, and approved Bedrock inference are permitted network dependencies when enabled.

MCP communication itself does not require model inference: a scripted client can perform discovery. Client-host inference, hosted-agent orchestration, assessment-engine analysis, and any inference inside the target MCP are separate boundaries. Fast mode makes no claim about a target server's internal implementation. See [inference boundaries in the design](docs/assessment-design.md#inference-boundaries).

Batch size does not determine runtime count. Hosted execution starts with one bounded assessment batch per Runtime session, with state in Amazon Aurora PostgreSQL accessed through the RDS Data API over HTTPS, and artifacts in private S3. Local execution uses the same bounded engine with approved local run records, evidence and reports; it does not require AgentCore, Aurora, the RDS Data API or S3 and does not upload artifacts automatically. Local execution is not necessarily offline: approved host inference, deep remediation, repository retrieval and target discovery may use the network. See [local direct execution](docs/assessment-design.md#local-direct-execution) and [the hosted API](docs/assessment-design.md#agentcore-runtime-api-and-client-interfaces).

The agent/engine and scanner plugin have separate source repositories. The existing corporate marketplace will distribute approved releases from the scanner-plugin repository to local assistants and the hosted image build. The optional remote client has a separate plugin package. Pin compatible plugin, engine and rule releases in both paths, with explicit skill loading and separate evidence paths. Local filesystem, credentials and sandbox controls differ from AgentCore isolation and must be configured explicitly. See [marketplace and deployment packaging in the HLD](docs/high-level-design.md#10-deployment-and-operations).

Repository assessments identify the inspected commit or snapshot. When a running endpoint is also assessed, record whether that source corresponds to its deployed version; available source alone does not verify the live deployment. See [evidence availability](docs/assessment-design.md#evidence-availability-and-deployment).

## Next implementation steps

1. Agree the input, evidence, finding, coverage, and remediation schemas.
2. Review a representative sample of corporate controls and define applicability and evidence requirements.
3. Implement the shared deterministic engine, separate MCP/skill rule packs, and bounded collectors.
4. Add rule fixtures and report generation.
5. Add the two scanner skills, shared engine CLI/tool adapters, compatible pinned local package and hosted image, and restricted Bedrock remediation adapter.
6. Deploy the agent on AgentCore Runtime with authenticated invocation, durable status, report access, cancellation, and interruption recovery.
7. Support the canonical scanner plugin directly in approved local assistants; optionally deliver the separate remote client plugin against the API.
8. Verify equivalent deterministic results across local and hosted execution for the same captured evidence, engine/plugin/rule/configuration versions and assessment context. Report environment-dependent coverage differences and test scope enforcement and interruption in each path.
9. Add the optional GHA remediation-PR workflow with commit-bound patches, isolated static validation, scoped publishing permissions and explicit delivery outcomes.

## Local data

The `local/`, `evidence/`, `reports/`, and `runs/` directories are excluded from Git by default for local catalogues, credentials, and assessment outputs. Only reviewed, sanitized examples should be committed. The current research document contains no supplied corporate catalogue or target evidence.
