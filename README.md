# MCP Assessment

Design and research for an internal Model Context Protocol security assessment skill.

**Status:** proposed design. This repository currently contains research findings and design decisions; it does not yet contain an executable scanner.

## Start here

Read [the assessment design](docs/assessment-design.md) for the proposed architecture, control coverage, evidence model, discovery boundaries, Bedrock integration, and implementation milestones.

## Confirmed scope

- Assess internal MCPs using available source code and configuration.
- Assess external MCPs using available endpoint evidence.
- Inspect supplied files and make explicitly allowed discovery connections.
- Both modes use host-model inference when invoked as an LLM-hosted skill. Inference controlled by the deployed workflow, including host orchestration, must use approved AWS Bedrock configurations.
- Fast mode runs deterministic assessment scripts and returns their findings; the assessment engine makes no model calls for detection or remediation.
- Deep mode runs the same deterministic checks, then makes an additional call through an approved AWS Bedrock inference profile for contextual remediation suggestions.
- Use the corporate control catalogue as the primary policy reference, with reviewed mappings to external MCP threat categories.
- Runtime assessment must not depend on external scanning services or public inference APIs. Allowed target discovery and the approved Bedrock service are explicit network exceptions.

MCP communication itself does not require model inference: a scripted client can perform discovery. Host orchestration, assessment-engine analysis, and any inference inside the target MCP are separate boundaries. Fast mode makes no claim about a target server's internal implementation. See [inference boundaries in the design](docs/assessment-design.md#inference-boundaries).

## Next implementation steps

1. Agree the input, evidence, finding, coverage, and remediation schemas.
2. Review a representative sample of corporate controls and define applicability and evidence requirements.
3. Implement the deterministic engine and bounded discovery collectors.
4. Add rule fixtures and report generation.
5. Add the restricted Bedrock remediation adapter and skill wrapper.

## Local data

The `local/`, `evidence/`, `reports/`, and `runs/` directories are excluded from Git by default for local catalogues, credentials, and assessment outputs. Only reviewed, sanitized examples should be committed. The current research document contains no supplied corporate catalogue or target evidence.
