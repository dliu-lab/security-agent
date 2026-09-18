# Security Assessment Agent

Design and research for an internal security assessment agent covering Model Context Protocol (MCP) servers and agent skill packages.

**Status:** proposed design. This repository currently contains research findings and design decisions; it does not yet contain an executable scanner.

## Objective

Build one security assessment agent deployed on Amazon Bedrock AgentCore Runtime and exposed through its authenticated invocation API. Package its assessment workflows in one trusted security plugin containing `mcp-assessment` and `skill-assessment` skills, backed by a shared deterministic engine, corporate-control mappings, evidence, and reporting. Assess internal and external MCPs and skill packages using available source/configuration and explicitly permitted endpoint evidence, with additional analysis through an approved AWS Bedrock inference profile for remediation in deep mode.

AgentCore Runtime is the required production hosting environment and replaces the earlier EKS deployment proposal. The two scanner skills run inside the hosted agent; a separate Claude Code terminal client skill invokes the same API. Application callers do not require a Claude Code session. A local CLI supports engine development and verification. Both fast and deep assessments run through the deployed agent.

## Start here

Read [the assessment design](docs/assessment-design.md) for the proposed architecture, control coverage, evidence model, discovery boundaries, Bedrock integration, and implementation milestones.

## Confirmed scope

- Assess both internal and external MCPs using available repository source code, configuration, deployment artifacts, and/or endpoint evidence.
- Assess agent skill packages, including instruction files, referenced scripts, resource paths, and relevant containing-plugin configuration. Keep separate target adapters/rule packs for MCPs and skills, with shared result contracts.
- Load only the trusted scanner plugin. Skill packages under review are evidence: do not install, activate, or execute them.
- Record ownership, hosting, and evidence availability separately. Select checks according to the evidence and applicable controls, rather than assuming external MCPs are endpoint-only.
- Inspect supplied files and make explicitly allowed discovery connections.
- Deliver the scanner as an agent on AgentCore Runtime, with typed submission, status, report, cancellation, and recovery operations exposed through `InvokeAgentRuntime`.
- Agent orchestration uses approved Bedrock inference in both modes; Claude Code adds client-host inference when used. Fast/deep distinguish assessment analysis, not whether the overall agent workflow uses inference.
- Fast mode runs deterministic assessment scripts and returns their findings; the assessment engine makes no model calls for detection or remediation.
- Deep mode runs the same deterministic checks, then makes an additional call through an approved AWS Bedrock inference profile for contextual remediation suggestions.
- Use the corporate control catalogue as the primary policy reference, with reviewed mappings to applicable MCP, agentic, and software-security categories.
- Runtime assessment must not depend on external scanning services or public inference APIs. Approved AWS infrastructure, explicitly approved repository retrieval, target discovery, and approved Bedrock inference are permitted network dependencies when enabled.

MCP communication itself does not require model inference: a scripted client can perform discovery. Client-host inference, hosted-agent orchestration, assessment-engine analysis, and any inference inside the target MCP are separate boundaries. Fast mode makes no claim about a target server's internal implementation. See [inference boundaries in the design](docs/assessment-design.md#inference-boundaries).

Batch size does not determine runtime count. Start with one bounded assessment batch per Runtime session and configurable internal concurrency. Persist assessment state and reports outside session memory. See [the API and lifecycle design](docs/assessment-design.md#agentcore-runtime-api-and-client-interfaces).

Repository assessments identify the inspected commit or snapshot. When a running endpoint is also assessed, record whether that source corresponds to its deployed version; available source alone does not verify the live deployment. See [evidence availability](docs/assessment-design.md#evidence-availability-and-deployment).

## Next implementation steps

1. Agree the input, evidence, finding, coverage, and remediation schemas.
2. Review a representative sample of corporate controls and define applicability and evidence requirements.
3. Implement the shared deterministic engine, separate MCP/skill rule packs, and bounded collectors.
4. Add rule fixtures and report generation.
5. Add the two scanner skills in one trusted plugin, the assessment agent, registered tools, and restricted Bedrock remediation adapter.
6. Deploy the agent on AgentCore Runtime with authenticated invocation, durable status, report access, cancellation, and interruption recovery.
7. Deliver the Claude Code terminal skill and client helper against that API.
8. Verify equivalent deterministic results across clients for the same evidence snapshots and rule versions, including agent scope enforcement and session interruption tests.

## Local data

The `local/`, `evidence/`, `reports/`, and `runs/` directories are excluded from Git by default for local catalogues, credentials, and assessment outputs. Only reviewed, sanitized examples should be committed. The current research document contains no supplied corporate catalogue or target evidence.
