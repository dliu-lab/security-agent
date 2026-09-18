# MCP Assessment Capability — Skill and API Application Design

Status: design proposal; assessment engine, Claude Code skill, API application, EKS deployment, rules, and integrations are not implemented.

This document records the initial design for an internal Model Context Protocol (MCP) assessment capability. Its purpose is to gather evidence, identify security findings, map them to the corporate control catalogue, and explain assessment limitations. A scan alone does not establish that an MCP is safe for every use.

## Objective

Build one reusable MCP security assessment engine delivered through both a skill runnable from the Claude Code terminal and an authenticated API-enabled application deployed on Amazon EKS. Both interfaces share assessment contracts, deterministic checks, corporate-control mappings, reports, and fast/deep mode definitions. Assess internal and external MCPs using available source/configuration and permitted endpoint evidence, with additional AWS Bedrock remediation analysis in deep mode.

## 1. Confirmed scope

- Both internal and external MCPs may provide repository source code, configuration, deployment artifacts, endpoint evidence, or a combination. Source availability is independent of ownership.
- Assessments use supplied files plus explicitly allowed discovery connections.
- Deliver a Claude Code terminal skill and an API-enabled application backed by the same engine. The skill supports local CLI execution or submission to the configured EKS service; API callers do not require a Claude Code session.
- Both modes use host-model inference when invoked as an LLM-hosted skill. Inference controlled by the deployed workflow, including host orchestration, must use approved AWS Bedrock configurations.
- Fast mode runs deterministic assessment scripts and produces findings, coverage, and maintained remediation guidance. The assessment engine makes no model calls for detection or remediation.
- Deep mode runs the same scripts first, then makes an additional call through an approved AWS Bedrock inference profile for contextual remediation suggestions.
- The assessment does not depend on external assessment products or public inference APIs. Explicitly approved repository retrieval, MCP endpoints, and AWS Bedrock are network dependencies when their respective capabilities are enabled.
- The corporate catalogue is the policy authority. External guidance supplements threat coverage and finding labels.

The deterministic engine should also run directly in a CLI or CI job without a host LLM. This is an optional execution path, not a claim that the full skill workflow is inference-free. Any assistant handling corporate controls or evidence must use an approved environment.

## 2. Architecture and mode boundaries

Use thin skill and API adapters around one reusable assessment engine:

```text
Claude Code skill → local CLI → shared assessment engine
Claude Code skill or API client → authenticated API → EKS Jobs
                                                         ↓
                                               Same assessment engine
```

The engine uses the same pipeline through either interface:

```text
Scope and policy → evidence collection → normalized evidence
    → deterministic rules → control mappings → findings and coverage
        ├── fast report
        └── minimized evidence → Bedrock → validated remediation → deep report
```

Keep these configuration dimensions separate:

| Dimension | Proposed values or content |
| --- | --- |
| Analysis mode | `fast`, `deep` |
| Invocation interface | Claude Code terminal skill, standalone CLI, application API |
| Execution backend | `local`, `eks`; independent of analysis mode |
| Ownership/origin | Internal, external; identify the owner or maintainer |
| Evidence available | Repository source, configuration, deployment artifacts, endpoint observations; any combination for either origin |
| Permitted access | Supplied files; explicitly approved repository retrieval and discovery; future separately authorised testing |
| Deployment context | Company-hosted or provider-hosted, transport, protocol version, data sensitivity |

### Delivery interfaces and EKS deployment

The Claude Code skill collects scope and mode, invokes the packaged CLI or configured service, and presents the resulting findings. Package the skill with `SKILL.md` and install it in a supported Claude Code skill location, such as `.claude/skills/mcp-assessment/SKILL.md` for a project. Keep assessment logic in the engine so the skill does not maintain a second set of checks. See the official [Claude Code skill documentation](https://code.claude.com/docs/en/skills).

The API application accepts validated manifests, authorises the caller's requested targets, creates asynchronous assessments, exposes status, and authorises report retrieval. Proposed routes are `POST /v1/assessments`, `GET /v1/assessments/{id}`, and `GET /v1/assessments/{id}/report`. The API does not execute arbitrary shell commands or accept caller-supplied Kubernetes manifests. API caller identity is separate from repository and MCP target credentials.

For remote execution, source inputs must identify an uploaded snapshot or explicitly approved repository revision; the service cannot assume access to the caller's local paths. Preserve input hashes, rules, mappings, and catalogue versions across interfaces. Both modes are available through both delivery interfaces. A direct fast API/CLI request requires no host-model inference; an LLM-hosted skill still uses host inference. Configure Claude Code host inference through approved Bedrock settings as well as the deep remediation stage. See [Claude Code on Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock).

The proposed implementation uses Python with Pydantic contracts, a CLI adapter, and a FastAPI application. Run assessment work in bounded Kubernetes Jobs on EKS. Separate repository collection, endpoint discovery, deterministic analysis, and Bedrock remediation into execution roles with their own network and credential scope. S3 stores evidence and reports; SQS and a DynamoDB scan registry support the shared service's asynchronous submissions, idempotency, and status. Pin dependencies and worker images, and package deployment configuration with Helm and the organisation's infrastructure tooling. This is an implementation recommendation, not deployed functionality.

The shared engine does not depend on Claude Code, and the application API need not be an MCP server. A browser UI or additional MCP service adapter remains an optional future interface; neither is required by the current API-enabled application objective.

### Evidence availability and deployment

Choose collectors and checks by available evidence and permitted access. Apply the same technical rule to comparable evidence regardless of whether the MCP is internal or external; corporate policy may add ownership-specific requirements. An externally maintained MCP may be hosted by the company or by its provider.

| Available evidence | Applies to | Assessment coverage |
| --- | --- | --- |
| Repository source and any supplied configuration | Internal or external MCPs | Static implementation, dependency, secret, and configuration checks; deployment-only controls may remain unknown |
| Endpoint observations only | Internal or external MCPs | Permitted discovery and observable protocol/configuration properties; inaccessible implementation controls remain unknown |
| Source/configuration plus endpoint observations | Internal or external MCPs | Combine static and observed evidence while recording whether the inspected source matches the running deployment |

Accept source as a pinned checkout or archive. Repository retrieval must be explicitly permitted, read-only, and bounded; a repository URL does not authorise executing its code, running hooks or builds, installing dependencies, following arbitrary links, or launching an MCP server. Record repository identity, commit or snapshot digest, local modifications and content hashes, relevant paths, and available deployment artifact/version identifiers. A branch name alone is insufficient to identify the assessed snapshot.

Keep repository findings scoped to the inspected revision. Source-to-deployment correspondence should be recorded as verified, unverified, or mismatched, with supporting evidence. Do not treat a clean repository assessment as verification of a provider's running service. Where correspondence is unverified, report source and endpoint conclusions separately.

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

The host must present the script-produced findings faithfully. Free-form host output must not replace deterministic evidence or introduce authoritative findings. The standalone CLI omits host inference; neither execution path can guarantee that a target with unverified implementation performs no inference. Record host and remediation model usage separately where observable, and identify target-side inference as declared, observed, or unknown.

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
├── skills/
│   └── mcp-assessment/
│       └── SKILL.md
├── src/mcp_assessment/
│   ├── contracts/
│   ├── collectors/
│   ├── checks/
│   ├── policy/
│   ├── reporting/
│   ├── remediation/
│   ├── storage/
│   ├── cli/
│   ├── api/
│   └── workers/
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
├── deploy/
│   ├── helm/
│   └── infrastructure/
└── tests/
    ├── fixtures/
    └── expected-results/
```

Keep `SKILL.md` focused on invocation, inputs, permissions, mode/backend selection, execution, and interpretation. Store control text, reference material, and executable checks separately. Package the shared engine so local CLI and EKS workers use the same implementations; API routing and skill instructions must not duplicate security logic. This structure is a proposal, not a list of existing files.

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
- Repository revision or snapshot identity where source is supplied, and source-to-deployment correspondence where an endpoint is also assessed.
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
5. Deliver the Claude Code terminal skill with local CLI execution and authenticated submission to the service.
6. Deliver the API application and EKS Jobs with asynchronous status, authorised report access, bounded execution, and retry/idempotency handling.
7. Pilot both interfaces against known benign and vulnerable fixtures, then internal and external MCPs across source-only, endpoint-only, and combined evidence configurations.

Acceptance tests should verify that the fast assessment engine makes no model calls; host orchestration uses approved inference configuration and preserves script-produced findings; discovery never becomes tool execution; malformed, hostile, or inaccessible targets yield bounded results; missing evidence remains visible; and deep remediation preserves deterministic findings. Target-side inference cannot be ruled out from endpoint discovery alone.

Include external MCP repository fixtures and internal MCP endpoint-only fixtures. Verify that evidence availability controls technical check selection, and that an unverified or mismatched source revision cannot produce an unsupported deployment-level pass.

Verify that local CLI and API/EKS execution produce equivalent deterministic findings and control mappings for the same captured evidence, engine, rules, and catalogue versions, excluding run metadata. Live observations and model suggestions may differ between runs. Test fast/deep selection through both interfaces, caller isolation, remote source-reference handling, and recovery of asynchronous jobs. The API execution path must work without Claude Code installed.

Decisions still needed: corporate catalogue format and sample controls; supported protocol versions and transports; discovery allowlists and authentication arrangements; severity and mandatory-unknown policy; evidence retention and redaction; internal vulnerability dataset; Bedrock profile/model/regions; Claude Code skill distribution; API identity integration and local-versus-EKS execution policy; and reassessment cadence.

The Git repository should initially contain design, schemas, reviewed rules, and synthetic fixtures. Runtime evidence, credentials, generated sensitive reports, and unapproved corporate material require explicit handling rules before they are committed.
