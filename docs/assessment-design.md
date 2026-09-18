# MCP Assessment Skill — Proposed Design

Status: design proposal; assessment engine, skill, rules, and integrations are not implemented.

This document records the initial design for an internal Model Context Protocol (MCP) assessment capability. Its purpose is to gather evidence, identify security findings, map them to the corporate control catalogue, and explain assessment limitations. A scan alone does not establish that an MCP is safe for every use.

## 1. Confirmed scope

- Internal MCPs usually provide source code and configuration; external MCPs may provide only an endpoint.
- Assessments use supplied files plus explicitly allowed discovery connections.
- Both modes use host-model inference when invoked as an LLM-hosted skill. Inference controlled by the deployed workflow, including host orchestration, must use approved AWS Bedrock configurations.
- Fast mode runs deterministic assessment scripts and produces findings, coverage, and maintained remediation guidance. The assessment engine makes no model calls for detection or remediation.
- Deep mode runs the same scripts first, then makes an additional call through an approved AWS Bedrock inference profile for contextual remediation suggestions.
- The assessment does not depend on external assessment products or public inference APIs. Approved MCP endpoints and AWS Bedrock are explicit network dependencies when their respective capabilities are enabled.
- The corporate catalogue is the policy authority. External guidance supplements threat coverage and finding labels.

The deterministic engine should also run directly in a CLI or CI job without a host LLM. This is an optional execution path, not a claim that the full skill workflow is inference-free. Any assistant handling corporate controls or evidence must use an approved environment.

## 2. Architecture and mode boundaries

Use a thin skill wrapper around a deterministic assessment engine:

```text
Scope and policy → evidence collection → normalized evidence
    → deterministic rules → control mappings → findings and coverage
        ├── fast report
        └── minimized evidence → Bedrock → validated remediation → deep report
```

Keep three independent configuration dimensions:

| Dimension | Proposed values or content |
| --- | --- |
| Analysis mode | `fast`, `deep` |
| Evidence access | Supplied files; explicitly permitted discovery; future separately authorised testing |
| Target context | Ownership, transport, protocol version, deployment, sensitivity, available evidence |

Deep mode does not expand permissions, invoke MCP tools, modify a target, or automatically increase detection coverage. It improves advice using the same evidence. Behavioural testing can be added later as a separate capability with an explicit scope.

Neither the host model nor the remediation model may silently change deterministic findings, severity, exceptions, or policy decisions. If remediation inference fails, retain the deterministic report and mark the advisory stage unavailable or partial.

### Inference boundaries

MCP is a communication protocol. A scripted client can connect to a server and request discovery results without invoking a model; the official [MCP Inspector CLI documentation](https://modelcontextprotocol.io/docs/2026-07-28/tools/inspector/cli) demonstrates scripted MCP requests.

| Boundary | Fast mode | Deep mode |
| --- | --- | --- |
| Skill host/orchestrator | Uses model inference to interpret the request, orchestrate the fixed workflow, and present results | Uses host inference for the same purposes |
| Deterministic assessment engine | No model calls; scripts establish findings, mappings, and policy results | Identical deterministic checks and results |
| Remediation analysis | Maintained guidance returned by scripts; no dedicated model analysis | Additional Bedrock inference produces contextual suggestions |
| Target MCP implementation | May itself use inference; discovery alone cannot establish its internals | Same uncertainty; deep mode does not expand discovery permissions |

The host must present the script-produced findings faithfully. Free-form host output must not replace deterministic evidence or introduce authoritative findings. The standalone CLI omits host inference; neither execution path can guarantee that an external target performs no inference. Record host and remediation model usage separately where observable, and identify target-side inference as declared, observed, or unknown.

## 3. Control model

Separate policy requirements, technical checks, threat categories, and evidence:

```text
Evidence → assessment rule → finding → corporate control(s)
                                   → external category/categories
```

Corporate controls define applicability, required evidence, acceptance criteria, ownership, and exception handling. MCP specifications provide version-specific requirements. OWASP MCP Top 10 provides a threat taxonomy and a coverage cross-check; broader agentic guidance is useful where host behaviour or cross-tool actions affect risk.

Mappings should be reviewed, versioned, and many-to-many. Report a catalogue coverage gap when a relevant finding has no corporate mapping. Do not invent control identifiers or let a model create authoritative mappings.

Pin external reference versions or commits. OWASP’s MCP project is evolving, so framework updates should trigger mapping review instead of silently changing assessment results. See [OWASP MCP Top 10](https://owasp.org/projects/mcp-top-10) and [OWASP Top 10 for Agentic Applications](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/).

## 4. Proposed package structure

```text
mcp-assessment/
├── SKILL.md
├── scripts/
│   ├── assess
│   ├── collectors/
│   ├── checks/
│   ├── reporting/
│   └── bedrock/
├── rules/
├── references/
│   ├── corporate-controls.yaml
│   ├── control-mappings.yaml
│   ├── source-versions.yaml
│   ├── assessment-boundaries.md
│   └── deep-mode.md
├── schemas/
│   ├── assessment-input.schema.json
│   ├── finding.schema.json
│   └── remediation.schema.json
├── assets/report-template.md
└── tests/
    ├── fixtures/
    └── expected-results/
```

Keep `SKILL.md` focused on invocation, inputs, permissions, mode selection, execution, and interpretation. Store control text, reference material, and executable checks separately. This structure is a proposal, not a list of existing files.

## 5. Assessment coverage

| Area | Initial checks and limitations |
| --- | --- |
| Inventory and ownership | Owner, approved use, endpoint or package identity, version, baseline changes; shadow-MCP discovery needs inventory evidence |
| Identity and tokens | Required authentication, issuer/audience validation, expiry, secret handling, token passthrough; metadata does not prove enforcement |
| Authorisation | Tool/resource permissions, least privilege, tenant isolation, downstream identity; normally needs implementation or authorised test evidence |
| Transport and protocol | TLS, exposure, origin handling, message validation, feature and version compatibility |
| Tools, prompts, resources | Schema quality, excessive capabilities, suspicious instructions, unexpected destinations, description drift, sensitive content |
| Execution and filesystem | Shell construction, unsafe evaluation, traversal, unrestricted file access, excessive process privileges |
| Network and data movement | SSRF protections, redirects, destination restrictions, secret exposure, excessive returned data |
| Supply chain | Pinned dependencies, lockfiles, package/image provenance, internal vulnerability data and its freshness |
| State and isolation | Applicable session/state ownership, replay or tampering protection, cache separation |
| Availability | Timeouts, response limits, parsing depth, schema complexity, pagination, concurrency; discovery excludes load testing |
| Audit and lifecycle | Identity/action logging, redaction, retention, exceptions, reassessment triggers |

The official [MCP security guidance](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices) addresses token passthrough, confused-deputy risks, SSRF, state-related attacks, and local server compromise.

Select checks using the target’s declared and observed protocol version and features. Do not assume a single initialization, discovery, or session model across revisions. Review the [MCP revision changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog) when implementing collectors.

HTTP and stdio need different authentication checks. Public access is a finding only where the exposed capability and corporate policy require protection. See [MCP authorization](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization).

## 6. Discovery permissions and scanner threat model

Treat the MCP server, supplied source, manifests, descriptions, schemas, and returned content as untrusted input. They may try to influence the scanner, exhaust resources, access internal destinations, or place malicious instructions in the remediation prompt.

The assessment manifest should identify allowed targets, permitted operations, credential references, timeouts, response limits, and retention requirements. Credentials should be obtained through approved secret handling rather than embedded in manifests or reports.

Discovery must:

- Use version-appropriate discovery/listing operations within configured permissions.
- Treat resource reads, prompt retrieval, and tool execution as separate permissions.
- Validate destinations, redirects, DNS resolution, and discovered authorization URLs before connecting.
- Permit explicitly configured internal destinations while rejecting unintended destinations.
- Bound pages, bytes, parsing depth, schema resolution, and total execution time.
- Avoid automatic package installation or local MCP process startup.
- Record attempted operations, failures, and incomplete collections.

Tool annotations such as `readOnlyHint` do not grant execution permission. They are claims from a potentially untrusted server; see the [MCP specification and security principles](https://modelcontextprotocol.io/specification/2026-07-28).

Deterministic pattern matching can flag injection indicators, but cannot prove arbitrary natural-language content safe. Clearly distinguish rule certainty from confidence in the underlying security conclusion.

## 7. Rules, evidence, and result contracts

Every rule should declare its stable identifier and version, applicability, required evidence, evaluation method, severity rationale, confidence basis, reviewed mappings, limitations, remediation template, and verification procedure. Use an explicit result for missing evidence.

Recommended check states are `pass`, `fail`, `unknown`, `not_applicable`, `error`, and `skipped`. Missing implementation evidence is usually `unknown`. A failed collector is an `error`, not a pass. A skipped check should record why it did not run.

Each finding should retain:

- Finding ID, rule/version, target identity, context, and timestamp.
- Evidence references, hashes, locations, and redacted excerpts.
- Observed, declared, or inferred evidence classification.
- Corporate mappings and external categories.
- Severity, confidence, limitations, remediation, and verification steps.

Store raw evidence separately with restricted access. Reports should reference it without unnecessarily copying secrets or sensitive source content. Record scanner, rule-pack, mapping, and reference versions for reproducibility.

Generate `findings.json`, `coverage.json`, and a readable report first. Add SARIF when needed by internal developer workflows. Report unresolved mandatory checks alongside any policy result. Endpoint-only assessments have narrower assurance: no observed weakness is not evidence that inaccessible controls work.

## 8. Bedrock remediation stage

Send only selected findings, relevant control text, and necessary redacted excerpts. Retrieve control text locally by reviewed IDs; a vector database is unnecessary for the initial design.

Use an approved inference profile with least-privilege IAM permissions. Record profile, model, prompt version, and inference settings. The remediation model receives no execution tools and cannot fetch additional evidence or act on target instructions. The skill host may launch the assessment engine within the configured scope; that orchestration permission does not extend to the remediation model.

Require structured suggestions containing finding ID, proposed fix, evidence IDs, assumptions, and verification steps. Reject malformed output and unknown references. Keep advice visibly separate from deterministic results.

Where appropriate, use [Bedrock private connectivity](https://docs.aws.amazon.com/bedrock/latest/userguide/usingVPC.html). Validate actual processing destinations against residency requirements when using [cross-region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html). Review model/API-specific [Bedrock retention behaviour](https://docs.aws.amazon.com/bedrock/latest/userguide/data-retention.html), invocation logging, report retention, and access controls. No particular profile, region, or model has been selected.

## 9. Delivery milestones and open decisions

1. Define input, evidence, result, and mapping schemas using representative corporate controls.
2. Implement the deterministic CLI with fixtures and a small set of high-confidence checks.
3. Add bounded, explicitly permitted discovery and honest coverage reporting.
4. Add the Bedrock adapter and validate advisory output isolation.
5. Pilot against known benign and vulnerable fixtures, then representative internal and external MCPs.

Acceptance tests should verify that the fast assessment engine makes no model calls; host orchestration uses approved inference configuration and preserves script-produced findings; discovery never becomes tool execution; malformed, hostile, or inaccessible targets yield bounded results; missing evidence remains visible; and deep remediation preserves deterministic findings. Target-side inference cannot be ruled out from endpoint discovery alone.

Decisions still needed: corporate catalogue format and sample controls; supported protocol versions and transports; discovery allowlists and authentication arrangements; severity and mandatory-unknown policy; evidence retention and redaction; internal vulnerability dataset; Bedrock profile/model/regions; and reassessment cadence.

The Git repository should initially contain design, schemas, reviewed rules, and synthetic fixtures. Runtime evidence, credentials, generated sensitive reports, and unapproved corporate material require explicit handling rules before they are committed.
