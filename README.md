# MCP Assessment

Design and research for an internal Model Context Protocol security assessment capability.

**Status:** proposed design. This repository currently contains research findings and design decisions; it does not yet contain an executable scanner.

## Objective

Build one reusable MCP security assessment engine delivered through both a skill runnable from the Claude Code terminal and an authenticated API-enabled application deployed on Amazon EKS. Both interfaces share assessment contracts, deterministic checks, corporate-control mappings, reports, and fast/deep mode definitions. Assess internal and external MCPs using available source/configuration and permitted endpoint evidence, with additional AWS Bedrock remediation analysis in deep mode.

The Claude Code skill can invoke the local CLI or submit assessments to the EKS service. Execution location (`local` or `eks`) is independent of analysis mode (`fast` or `deep`). The application API supports programmatic callers without requiring a Claude Code session.

## Start here

Read [the assessment design](docs/assessment-design.md) for the proposed architecture, control coverage, evidence model, discovery boundaries, Bedrock integration, and implementation milestones.

## Confirmed scope

- Assess both internal and external MCPs using available repository source code, configuration, deployment artifacts, and/or endpoint evidence.
- Record ownership, hosting, and evidence availability separately. Select checks according to the evidence and applicable controls, rather than assuming external MCPs are endpoint-only.
- Inspect supplied files and make explicitly allowed discovery connections.
- Both modes use host-model inference when invoked as an LLM-hosted skill. Inference controlled by the deployed workflow, including host orchestration, must use approved AWS Bedrock configurations.
- Fast mode runs deterministic assessment scripts and returns their findings; the assessment engine makes no model calls for detection or remediation.
- Deep mode runs the same deterministic checks, then makes an additional call through an approved AWS Bedrock inference profile for contextual remediation suggestions.
- Use the corporate control catalogue as the primary policy reference, with reviewed mappings to external MCP threat categories.
- Runtime assessment must not depend on external scanning services or public inference APIs. Explicitly approved repository retrieval, target discovery, and the approved Bedrock service are permitted network dependencies when enabled.

MCP communication itself does not require model inference: a scripted client can perform discovery. Host orchestration, assessment-engine analysis, and any inference inside the target MCP are separate boundaries. Fast mode makes no claim about a target server's internal implementation. See [inference boundaries in the design](docs/assessment-design.md#inference-boundaries).

Repository assessments identify the inspected commit or snapshot. When a running endpoint is also assessed, record whether that source corresponds to its deployed version; available source alone does not verify the live deployment. See [evidence availability](docs/assessment-design.md#evidence-availability-and-deployment).

## Next implementation steps

1. Agree the input, evidence, finding, coverage, and remediation schemas.
2. Review a representative sample of corporate controls and define applicability and evidence requirements.
3. Implement the deterministic engine and bounded discovery collectors.
4. Add rule fixtures and report generation.
5. Add the restricted Bedrock remediation adapter and Claude Code terminal skill around the shared CLI.
6. Deliver the authenticated API application and EKS worker deployment, with submission, status, and report retrieval.
7. Verify equivalent deterministic results across interfaces for the same evidence snapshots and rule versions.

## Local data

The `local/`, `evidence/`, `reports/`, and `runs/` directories are excluded from Git by default for local catalogues, credentials, and assessment outputs. Only reviewed, sanitized examples should be committed. The current research document contains no supplied corporate catalogue or target evidence.
