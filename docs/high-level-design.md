# Security Agent — High-Level Design

**Status:** proposed architecture for review; no agent, scanner, or deployment has been implemented.

**Version:** 0.2 — updated 21 September 2026

**Audience:** security engineering, application engineering, cloud platform, and architecture reviewers.

**Companion:** [Assessment design and control coverage](assessment-design.md).

## 1. Purpose and architectural direction

Security Agent is an internal service that assesses MCP servers and agent skill packages, records evidence, maps findings to the corporate control catalogue, and produces a report with explicit coverage limitations. It runs on Amazon Bedrock AgentCore Runtime and exposes an authenticated API. Claude Code users invoke the same service through a lightweight client skill.

One maintained agent application loads one trusted `security-assessment` plugin containing two assessment skills:

- `mcp-assessment`: source, configuration, and explicitly permitted endpoint discovery.
- `skill-assessment`: skill instructions, supporting scripts and resources, and relevant containing-plugin configuration, inspected as data.

Both skills use one deterministic assessment engine, evidence model, corporate-control mapping layer, and reporting pipeline. Deep mode adds contextual remediation through an approved Bedrock inference profile. Agent orchestration uses approved Bedrock inference in both modes; authoritative findings come from the deterministic engine.

The organisation already has a central plugin marketplace for skills and MCP packages or connection definitions. Use it as the distribution source for the trusted scanner plugin and the separately distributed Claude Code client. The proposed deployment pipeline selects an approved, pinned scanner-plugin release and includes it in the Security Agent image. Marketplace packages submitted for assessment remain untrusted evidence even when the same marketplace distributes the trusted scanner.

| Decision | Status |
| --- | --- |
| Product name: Security Agent | Confirmed |
| Production hosting: AgentCore Runtime; API exposure through `InvokeAgentRuntime` | Confirmed |
| One API and assessment lifecycle for MCP, skill, and mixed batches | Confirmed direction |
| One trusted scanner plugin with MCP and skill assessment skills | Confirmed direction |
| Existing corporate plugin marketplace | Confirmed organisational context |
| Selected marketplace scanner plugin included in an immutable container image | Proposed v1 delivery model |
| Corporate catalogue is the policy authority | Confirmed |
| Approved Bedrock inference; no external scanning SaaS or public model APIs | Confirmed |
| Python engine, AgentCore SDK, S3 artifacts, DynamoDB registry | Proposed implementation |
| Agent framework, authentication integration, AWS regions and service objectives | Open; recommendations and gates appear below |

The earlier EKS hosting proposal is superseded. One agent application may serve many isolated sessions; it does not mean one shared process or conversation for all users.

## 2. Scope and assumptions

Internal and external targets can both supply repository source or configuration. Target ownership, hosting, available evidence, and permitted operations are independent attributes. Source is pinned to an immutable snapshot or revision; endpoint conclusions record whether the source matches the deployed service.

V1 supports individual assessments and batches, including repositories containing both target types. Each target has an explicit ID, type, and evidence scope. A report describes what was assessed and what remains unknown; it is not a blanket guarantee of safety.

Initial assessment activities are static inspection and explicitly authorised MCP discovery. Target scripts, hooks, plugins, package installation, builds, local MCP startup, resource reads, prompt retrieval, and tool execution are outside default permission. Security Agent does not automatically apply remediation. References to other packages or endpoints do not authorise following them.

Approved open-source dependencies are packaged internally. Rules and vulnerability data use reviewed, versioned inputs; external scanning/enrichment services and unapproved model APIs are not runtime dependencies. Explicitly approved repository retrieval and MCP target discovery remain permitted. A representative corporate catalogue has not yet been supplied, so actual control IDs, mappings, and policy thresholds remain to be defined.

## 3. System architecture

![Security Agent architecture with AgentCore Runtime, two scanner skills, the deterministic engine, approved targets, storage, and Bedrock inference](diagrams/system-architecture.png)

[Open full-size diagram](diagrams/system-architecture.png) · [Editable diagram source](diagrams/system-architecture.mmd)

The Runtime box is a compute boundary. Its internal modules share the Runtime identity and network configuration; arrows between them do not imply separate IAM roles. S3 and DynamoDB preserve assessment state independently of the session. Approved release artifacts supply scanner skills, rule packs, mappings, and model configuration.

The native API invokes application code; it does not automatically create REST resources such as `/assessments`. AgentCore's HTTP application contract provides `/invocations` and `/ping`. The SDK supplies the serving integration. [AWS HTTP contract](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-http-protocol-contract.html).

### Component responsibilities

| Component | Responsibilities | Authoritative output |
| --- | --- | --- |
| Operation handler | Validate schema, authenticate application identity, authorise targets/results, bind sessions, enforce idempotency and admission | Accepted request and durable operation state |
| Agent and skill loader | Load only pinned scanner skills; interpret scoped requests, invoke registered tools, explain results | Explanations, never replacement findings |
| Two scanner skills | Provide target-specific workflow instructions and reference material | Versioned workflow guidance |
| Collectors and check engine | Retrieve approved evidence, parse bounded inputs, execute registered checks, enforce required phases | Evidence, check results, findings, coverage |
| Control and policy layer | Apply reviewed mappings, applicability, exceptions, and assessment policy | Corporate-control coverage and policy result |
| Remediation adapter | Submit selected evidence to Bedrock; validate structured suggestions and references | Advisory remediation linked to findings |
| Registry and artifact store | Preserve ownership, progress, attempts, immutable evidence and published reports | Recoverable assessment record |
| Claude Code client | Prepare a typed request and retrieve/present the hosted report | No independent scanner verdict |

The agent cannot override the manifest, skip a mandatory phase and publish success, change severity/mappings, or enable remediation in fast mode. Application tools and publication gates enforce those requirements.

## 4. Application, plugin, and skill packaging

### Source ownership and application package

Maintain the executable engine in the Security Agent application repository. Maintain the canonical scanner skills in the corporate marketplace's backing repository or versioned artifact source. The application consumes a pinned release; it does not maintain a second editable copy of those skills.

| Component | Source of truth | Deployment role |
| --- | --- | --- |
| API, agent adapter, registered tools, deterministic engine | Security Agent application repository | Executable application in the Runtime image |
| `security-assessment` plugin with both scanner skills | Corporate marketplace and its approved package source | Selected trusted workflow bundle included in the image |
| Claude Code client plugin | Corporate marketplace and its approved package source | Installed in the approved terminal client; excluded from hosted skill loading |
| Corporate catalogue, mapping data, rule metadata | Approved versioned internal configuration/artifact storage | Pinned inputs with recorded hashes and access controls |
| Packages or repositories under assessment | Supplied artifacts or explicitly approved retrieval | Evidence only; never part of trusted discovery |

Proposed application layout; the following packages have not yet been implemented:

```text
security-assessment/
├── src/security_agent/
│   ├── runtime/                 # API, identity, assessment lifecycle
│   ├── agent/                   # Framework adapter and registered tools
│   ├── engine/
│   │   ├── pipeline.py          # Required phases, budgets, scheduling
│   │   ├── collectors/          # Repository and permitted MCP evidence
│   │   ├── inventory/           # Targets, files, shared dependencies
│   │   ├── checks/
│   │   │   ├── shared/
│   │   │   ├── mcp/
│   │   │   └── skills/
│   │   ├── controls/            # Mapping and policy evaluation code
│   │   └── reporting/           # Findings, evidence, coverage
│   ├── storage/                 # Durable records and artifacts
│   └── remediation/             # Bounded Bedrock advisory adapter
├── deployment/                  # Image build, IaC, release lock data
└── control-mappings/            # Schemas and sanitised examples
```

The mapping code applies reviewed rule-to-control relationships supplied as versioned YAML/JSON or equivalent structured data. Confidential catalogue content stays in approved private storage. Record the catalogue, mappings, policy and rule versions used by every assessment; the LLM cannot create authoritative control IDs or replace mapping decisions.

### Deployed filesystem and trusted loading

The proposed image contains separate application and trusted-plugin paths:

```text
/app/security-agent/                         # Installed application and engine
/opt/security-agent/trusted-plugins/
└── security-assessment/
    └── skills/
        ├── mcp-assessment/SKILL.md
        └── skill-assessment/SKILL.md
/work/assessments/<assessment-id>/            # Target evidence and temporary work
```

Package each selected skill with its reviewed references and required support files. Configure trusted-plugin files as read-only to the runtime user and keep evidence paths outside all plugin/skill discovery roots. Separate folders support controlled loading and maintenance; they do not create separate IAM, process, or network security boundaries. Loaded scanner skills intentionally guide the agent. Application code enforces what their tools can do.

The plugin is a release package, not another agent or container. Select one framework adapter for v1. Strands with its `AgentSkills` integration is a candidate; a Claude Agent SDK adapter is another option when Claude plugin compatibility is required. Avoid maintaining both adapters initially.

The framework must explicitly load the two approved skill directories. Strands loads skill instructions and resource listings while the application provides resource-access tools; it does not interpret a Claude plugin manifest as a deployment. Use registered scanner tools rather than adopting unrestricted shell access from examples. Skill `allowed-tools` metadata is not relied on as an enforcement boundary. [Strands skills](https://strandsagents.com/docs/user-guide/concepts/plugins/skills/).

The Claude Agent SDK can load a plugin from an explicitly configured local directory after the deployment pipeline downloads it. Such a plugin can also contain hooks, subagents and MCP definitions. For the hosted scanner, validate a skills-only component allowlist and reject unapproved executable components, hooks, automatic MCP connections, and client helpers. A different folder does not suppress those components when a framework loads the whole plugin. Marketplace distribution and loading are implemented by the release pipeline and selected framework adapter; the design does not assume a native AgentCore marketplace-attachment feature. [Claude Agent SDK plugins](https://code.claude.com/docs/en/agent-sdk/plugins).

Keep target repositories outside trusted plugin/skill discovery and never inherit their `SKILL.md`, `AGENTS.md`, `CLAUDE.md`, hooks, or MCP configuration as agent instructions. These files are evidence. Distribute the Claude Code client in a separate marketplace plugin and exclude it from the hosted image's trusted skills, preventing recursive submission to Security Agent itself.

Marketplace MCP entries are also not automatic runtime connections. An approved remote MCP dependency remains separately hosted and requires explicit endpoint, tool, credential and network configuration. A package being assessed is only a target. V1 does not launch local stdio MCP packages or execute target tools as a consequence of reading marketplace metadata.

## 5. Assessment flow and modes

### Main execution flow

![Assessment sequence from authenticated submission through deterministic checks, optional deep remediation, and report retrieval](diagrams/assessment-flow.png)

[Open full-size diagram](diagrams/assessment-flow.png) · [Editable diagram source](diagrams/assessment-flow.mmd)

All model requests use server-configured approved profiles and bounded token/call budgets. A model-generated tool request still passes deterministic scope and phase checks. The remediation call is a separate logical context with no execution tools, even when it uses the same approved model as orchestration.

| Stage | Fast | Deep |
| --- | --- | --- |
| Hosted-agent orchestration | Approved Bedrock inference | Same |
| Evidence collection, checks, mappings and policy | Deterministic | Identical checks for identical captured evidence and versions |
| Remediation | Maintained rule guidance | Additional contextual Bedrock suggestions |
| Output | Findings, coverage, policy result, maintained guidance | Same authoritative results plus separate validated advice |
| Status/report/cancel operations | Structured application logic | Same; no model required |

Claude Code adds its own approved host inference when used as the client. Fast does not mean the full agent workflow is inference-free. Deep does not expand discovery permissions or silently add model-generated security findings.

### Deterministic engine stages

Both assessment skills invoke a registered high-level engine tool, such as `run_assessment`, using the validated assessment context. The engine enforces the required workflow in code and can process a whole batch without an LLM round trip for every file or check.

| Stage | Responsibility | Output |
| --- | --- | --- |
| Collection | Acquire a bounded approved snapshot, or permitted MCP discovery responses; record location, hash, time and retrieval errors | Traceable evidence |
| Inventory | Discover skills/MCP targets and associate their files, references and containing-plugin context within the approved scope | Frozen target inventory and shared file index |
| Checks | Run applicable deterministic rules against captured evidence; reuse common parsing and repository checks | Check results, findings and coverage gaps |
| Mapping and policy | Apply reviewed mappings, applicability, severity and policy rules | Corporate-control coverage and assessment policy result |
| Reporting | Preserve per-target evidence references and aggregate shared findings | Deterministic report and machine-readable results |

For skills, collected evidence includes `SKILL.md`, supporting scripts/resources, dependency manifests and relevant plugin configuration. For MCP source, it includes implementation, tool definitions and available configuration/deployment files. Permitted endpoint discovery can collect server information, advertised tool schemas and authentication metadata. Collection never activates target skills, executes scripts, installs dependencies, starts target MCPs or invokes target tools. Metadata is evidence of a declaration, not proof of runtime enforcement.

Example: collection captures a source file and its hash; a check detects a hard-coded credential; mapping associates that finding with an applicable corporate secret-management control; reporting cites the captured location while redacting the secret.

### Repository batch workflow

For a repository with 20 skills, accept one scoped repository assessment and initially run one attempt in one Runtime session. Acquire one immutable snapshot, walk it once within file/depth/byte limits, and discover the allowed skill packages. Record nested skills, supporting files and relevant shared plugin context; report exclusions, unresolved references and limit exhaustion as coverage limitations. Do not follow external references or paths outside the authorised snapshot without separate permission.

Build a shared file/dependency index, parse each relevant file once where practical, and run repository-wide checks once. A bounded worker pool performs target-specific checks and links common findings to affected skills. Checkpoint completed targets, publish deterministic results, then queue minimised finding bundles for deep remediation when selected. Status retrieval uses structured application logic without extra inference.

Bound target processing and Bedrock concurrency separately. Maintain separate evidence/context bundles per target to avoid mixing conclusions; produce one aggregate report with per-target coverage and failures. A target count of 20 does not cause 20 Runtime environments. On microVM compute, sessions are the execution-environment boundary, and live sessions can serve related invocations. [AWS session model](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-sessions.html).

## 6. API contract

Use one API service and shared typed operations inside the `InvokeAgentRuntime` payload for MCP, skill and mixed assessments. `target_type` selects the internal adapter; mode, lifecycle, report format and error contracts are shared. Authentication is common, while authorisation remains specific to the target, operation, owner and evidence. The following names and fields are proposed application contracts and will be formalised as schemas before implementation.

| Operation | Required intent | Behaviour |
| --- | --- | --- |
| `start_assessment` | Mode, explicit targets or repository discovery scope, evidence references, approved access, idempotency key | Persist acceptance, start bounded work, return assessment ID |
| `get_assessment_status` | Assessment ID | Return state, phase, target counts, incomplete work and errors |
| `get_assessment_report` | Assessment ID and report format | Return only results the caller can access |
| `cancel_assessment` | Assessment ID | Request cooperative cancellation; preserve completed evidence |
| `resume_assessment` | Interrupted assessment ID | Reauthorise access and create a new bounded attempt from valid checkpoints |

Illustrative submission; artifact IDs refer to previously ingested, authorised, immutable snapshots:

```json
{
  "schema_version": "1.0",
  "operation": "start_assessment",
  "idempotency_key": "example-request-001",
  "mode": "deep",
  "targets": [
    {
      "target_id": "mcp-01",
      "target_type": "mcp",
      "evidence_refs": ["artifact-mcp-source-001"]
    },
    {
      "target_id": "skill-01",
      "target_type": "skill",
      "evidence_refs": ["artifact-skill-source-001"]
    }
  ],
  "access": {
    "discovery": "disabled"
  }
}
```

An accepted response contains `assessment_id`, `attempt_id`, and `state: accepted`. HTTP success means the invocation succeeded; it is not a security verdict or proof of completed work. Use an explicit application error/state envelope for validation failures, conflicts, incomplete assessments, and model-stage errors. AWS transport/authentication errors remain distinguishable.

Resolve evidence IDs to owner-scoped storage records. Source ingestion accepts bounded uploaded snapshots or explicitly authorised repository retrieval pinned to a revision. Verify file hashes, extraction limits and path boundaries. A caller's local filesystem path is not a remote evidence reference. The manifest cannot select arbitrary roles, model profiles, commands, network destinations, or trusted scanner skills.

The submission schema must also support repository-scoped discovery: an immutable evidence reference, permitted target types and path selectors, and explicit discovery limits. Acceptance can precede inventory completion; target counts remain pending until discovery produces a durable inventory. Derive stable target identities from the snapshot and package paths, record the resolved inventory, and reuse it on resume. Discovery cannot broaden the caller's permissions. The example above shows the alternative of explicitly identified targets; the exact repository-selector fields remain a contract-design task.

Duplicate requests from the same owner with the same key and input digest return the existing assessment. Conflicting reuse is rejected. Client timeouts must be retried with the original key. Status/report operations remain available after the original compute has stopped by reading durable storage in an authorised invocation.

## 7. Information and control model

| Record | Principal contents |
| --- | --- |
| Assessment | ID, owner/tenant, canonical manifest and digest, mode, state, target counts, versions, budgets, artifact references |
| Attempt | Attempt ID, assessment ID, Runtime session ID, current phase, lease expiry, cancellation, fencing token, failure classification |
| Target | Target ID/type, origin, hosting, approved access, evidence scope, source-to-deployment correspondence |
| Evidence | Immutable artifact reference/hash, source revision, location, collection operation/time, observed/declared/inferred classification |
| Finding | Stable check identity/version, target links, evidence references, severity, confidence, corporate mappings and limitations |
| Coverage | Applicable controls/checks with `pass`, `fail`, `unknown`, `not_applicable`, `error`, or `skipped` and reasons |
| Remediation | Finding/evidence IDs, suggestion, assumptions, verification steps, model/profile/prompt metadata |

Store registry records in DynamoDB and encrypted artifacts in private S3. Separate raw evidence from ordinary report access. Reports reference restricted evidence without embedding credentials or unnecessary source. Define retention, deletion, encryption-key access and backup requirements before production; database TTL alone is not a complete artifact-deletion policy.

Corporate controls determine applicability and policy decisions. Maintain reviewed, versioned many-to-many mappings from checks to controls and relevant external categories. MCP guidance and OWASP MCP categories supplement MCP coverage; skill findings use applicable agentic/software-security categories. Missing mappings and mandatory unknowns remain explicit. No model creates authoritative corporate control IDs.

Capture the repository snapshot, scanner release/image, plugin/skill versions, rule pack, catalogue/mapping versions, prompts, and model settings. Resume uses the same approved snapshot and versions or creates a new assessment if equivalence cannot be preserved.

## 8. Identity and security boundaries

| Boundary | Design requirement |
| --- | --- |
| Caller to API | Validate configured authentication; authorise each operation, target and result |
| Caller to session | Atomically bind an execution session to verified owner, assessment and attempt; check on every invocation before loading agent state or invoking a model |
| Agent to tools | Registered operations only; fixed phase/mode gates, resource bounds and target-specific permissions in code |
| Trusted scanner to target evidence | Explicit trusted loader paths; target packages remain inert and cannot register tools, skills or hooks |
| Runtime to AWS/data stores | Least-privilege execution role, scoped storage/secret operations, encryption and audit |
| Runtime to targets | Explicit destinations/operations, TLS, redirect/DNS validation, bounded pagination and responses |
| Evidence to model | Selected redacted content, relevant control excerpts, bounded context; raw content is untrusted |
| Model to findings | Schema/reference validation; generated text cannot overwrite deterministic findings or policy |

Proposed default authentication is corporate JWT for both user clients and approved machine identities, subject to identity-provider support. Configure issuer, audiences, scopes and verified principal propagation. If the handler needs the bearer token, explicitly configure supported header forwarding and validation; do not assume identity claims appear automatically in the payload. An IAM/SigV4 alternative needs an equally explicit trusted ownership mechanism. One Runtime configuration uses the selected inbound authentication mode; do not assume IAM and JWT are interchangeable on the same configuration. [AWS inbound authentication](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html).

Session IDs, assessment IDs, body `owner_id` fields and caller-supplied user headers are not authorisation. Derive a user principal from verified issuer/subject and an approved tenant claim; define an equivalent machine-principal convention for CI. Bind execution sessions to `(owner, assessment_id, attempt_id)`. Reject a different start/resume in an already-bound execution session; idempotent replay of the same request returns the existing assessment. Authorised status/report access can use a fresh control invocation without loading unrelated agent context. Reject mismatched session ownership before any state reuse. Ordinary callers receive assessment invocation permissions, not Runtime shell/command access. The authenticated caller's target entitlement is checked independently of the scanner's ability to obtain target credentials.

Keep caller JWTs out of model context, logs, stored manifests and target requests. Accepted background work runs under an explicit, bounded assessment authorisation record and service identity rather than a persisted caller token. Define authorisation expiry/revocation checks at phase boundaries; expired grants interrupt work until reauthorised. Resume and result access always validate current caller rights. Target credentials have a separate scope and refresh policy.

### Authentication and authorisation of assessed MCPs

Assess authentication and authorisation separately. Authentication establishes the calling identity; authorisation controls access to tools, operations, resources and tenants. The requirement to protect a target depends on its exposed capabilities and corporate policy, not whether the target is labelled internal or external.

| Target/access case | Assessment requirement |
| --- | --- |
| Remote MCP exposing corporate data or privileged actions | Require protection under the applicable corporate policy; assess token validation and operation/resource permissions |
| Intentionally public MCP serving public information | Evaluate whether anonymous access is permitted and appropriately constrained; do not automatically fail it for lacking login |
| Local stdio MCP source/configuration | Assess process/OS permissions, environment credentials and downstream access; do not apply HTTP OAuth requirements blindly or launch the package |
| Source-only assessment | Require authorised source access; a live MCP credential is unnecessary |
| Protected endpoint discovery | Use a separate least-privilege target credential and only permitted discovery operations; listing tools does not authorise execution |

MCP makes protocol-level authorisation optional and defines its OAuth-based flow for HTTP transports. Validate applicable issuer, audience, expiry and permission handling from available evidence; never forward Security Agent's inbound token to a target MCP. A `401` response proves rejection of that request, not correct enforcement of every authentication or authorisation control. Record controls as `unknown` when evidence is insufficient. [MCP authorisation](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization), [token security](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/security-considerations).

### Execution and network boundaries

Collectors, parser subprocesses, orchestration and remediation inside one session share the Runtime role, network and filesystem. The microVM isolates compute sessions; it does not provide per-module or per-assessment S3 permissions under a shared role. Fast/deep separation is enforced in application code. If corporate policy requires stronger phase/tenant isolation, split execution across constrained Runtime deployments or an approved broker before production. This is an explicit security decision, not a property supplied by skill packaging.

Configure Runtime connectivity to approved private resources and AWS endpoints. Inbound private API connectivity and outbound access to internal targets are separate network decisions. External discovery uses controlled egress; reject unintended internal/link-local destinations and revalidate redirects and DNS results. VPC attachment is not a hostname allowlist. [AWS VPC connectivity](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-vpc.html).

Bedrock requests use a server-selected approved inference profile, including orchestration. Validate region routing, logging and data-retention settings against corporate requirements. The profile is used in the model invocation configuration; hosting in AgentCore does not automatically configure it. [Bedrock inference profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-use.html).

## 9. Lifecycle, resilience and concurrency

![Assessment lifecycle covering acceptance, execution, completion, partial results, interruption, recovery, and cancellation](diagrams/assessment-lifecycle.png)

[Open full-size diagram](diagrams/assessment-lifecycle.png) · [Editable diagram source](diagrams/assessment-lifecycle.mmd)

Lifecycle state is separate from security outcome: `Completed` can contain failed checks or unknown controls. A deep-mode inference failure leaves deterministic results accessible and marks the advisory stage incomplete, normally producing `Partial`. Each target also records its own phase and outcome. Cancellation and resume require current authorisation.

Persist acceptance before acknowledgement. Register background work and keep health responses responsive. The AgentCore SDK's asynchronous tracking communicates activity so work can continue after the initial response; it is not a durable queue or replay engine. Busy health prevents idle termination, not maximum-lifetime termination or crashes. [AWS asynchronous processing](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-long-run.html).

Checkpoint at phase/target boundaries and renew a conditional attempt lease. Only the current fencing token can publish canonical result references; stale attempts may leave orphan artifacts for cleanup but cannot replace published state. Cancellation and completion compete through conditional state transitions so a late result cannot overwrite an accepted cancellation. Timeouts and subprocess termination bound cancellation latency, although an external inference request may already be in flight.

V1 uses explicit recovery: status evaluation detects expired leases or accepted-but-not-started work, reports interruption, and permits authorised resume. Reauthorise current access and verify checkpoints before replay. Retries may repeat discovery or inference; exactly-once external effects are not promised. A scheduled reconciler may later automate this control-plane work while all scanning remains in AgentCore.

Enforce admission limits per owner, target and service, plus per-attempt parsing, bytes, pages, runtime and inference budgets. When capacity is unavailable, return a retryable admission error before accepting work. SQS/dispatcher integration is optional for future burst buffering. Set scan deadlines below the configured Runtime lifetime, and validate current regional quotas during deployment.

### Performance, caching and scale-out

Hosting a skill on AgentCore does not itself parallelise repository inspection. File discovery, parsing, deterministic checks and scheduling belong in the engine. Keep one bounded batch orchestration and separate limits for CPU work, I/O, model requests and aggregate model tokens. Both fast-mode orchestration and deep remediation consume the approved Bedrock budget. Avoid repeatedly submitting whole repositories or catalogues to the model.

Start by benchmarking one and two CPU-heavy workers, then tune I/O and inference concurrency independently. AWS currently lists a maximum allocation of 2 vCPU and 8 GB per Runtime session for the proposed microVM model; revalidate limits for the selected deployment before setting defaults. Worker count is not a CPU allocation. [AWS Runtime quotas](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/bedrock-agentcore-limits.html).

Separate reusable parsing from policy-result caches. Parsing cache identities include content hashes, parser versions and relevant shared scripts, references, plugin configuration and dependency manifests. Result reuse additionally requires compatible engine/rule, catalogue/mapping/policy and vulnerability-data versions, target/deployment attributes, authorised evidence/discovery scope, coverage inputs and current exception state. Recompute applicability, policy and coverage when those inputs differ; reuse of a parsed file alone never implies reuse of its verdict. Scope caches to authorised owners and revalidate access on reuse. Invalidate affected targets when shared context changes, and never treat cached endpoint observations as newly collected evidence.

No performance or batch-size guarantees have been measured. Benchmark representative 20-, 100- and 500-skill repositories with both small instruction packages and large script/dependency trees, including concurrent callers and cold/warm starts. Measure ingestion, inventory, checks, time to deterministic report, deep completion, peak memory/CPU, tokens, throttling, cache effectiveness and cost. Verify that concurrency and caching preserve deterministic findings and explicit coverage.

If measured resource use or deadlines require scale-out, add a dispatcher that assigns bounded target groups to distinct Runtime sessions. Reuse the original immutable evidence and shared observations, authorise each child attempt, and aggregate partial/complete results under one assessment. This is a future scheduling extension to the v1 single-session attempt model; it is not automatic one-session-per-skill fan-out.

## 10. Deployment and operations

| Layer | Proposed technology |
| --- | --- |
| Runtime application | Python, Pydantic contracts, AgentCore SDK; one pinned skill-capable agent framework |
| Scanner execution | Registered Python check modules and reviewed offline subprocess adapters where useful |
| MCP collection | Official MCP SDK adapter selected for supported protocol revisions |
| Model access | Approved Bedrock inference profile through the model adapter |
| Durable state | DynamoDB assessment/attempt registry |
| Artifacts | Private S3 with KMS encryption and controlled retention |
| Secrets | Approved AWS secret storage/identity integration, using references rather than embedded credentials |
| Release | Reviewed source and skills, immutable ECR image, pinned rule/catalogue artifacts, infrastructure as code |
| Telemetry | Structured operational logs, metrics and traces in approved AWS monitoring services |

### Marketplace-to-Runtime release flow

![Deployment flow from Security Agent source and selected marketplace scanner plugin through a verified ECR image to AgentCore, with separate Claude Code client distribution](diagrams/deployment-distribution.png)

[Open full-size diagram](diagrams/deployment-distribution.png) · [Editable diagram source](diagrams/deployment-distribution.mmd)

Use a container image as the proposed v1 artifact so the engine, parsers and approved scanner utilities share a reproducible dependency environment. AgentCore also supports ZIP deployment; an image is a project choice, not an AgentCore requirement. [AWS deployment options](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-get-started-cli.html).

1. Resolve the selected scanner plugin from the corporate marketplace to an approved immutable commit or artifact digest. Marketplace membership alone is not deployment approval, and a mutable branch/tag alone is not the release identity.
2. Verify package provenance/integrity, allowed components, referenced resources and compatibility with the engine tool contracts. Package approved dependencies during the build; do not install them while assessing targets.
3. Compose the application, selected scanner skill bundle and approved rule code into an image. Pin private catalogue/mapping artifacts by immutable reference and verified digest, and keep credentials out of image layers and manifests.
4. Record a release manifest containing the application revision, final image digest, plugin/skill identity, engine-tool contract version, rule and configuration identities. Validate actual loader inventory and run integration tests before promotion.
5. Publish the image to private ECR and deploy the same tested release through environment-specific configuration. At startup, verify pinned external configuration artifacts and explicitly load the two approved scanner skills. Fail readiness if a required artifact is missing, mismatched or incompatible.
6. Distribute the separate client plugin through the marketplace to approved Claude Code users. It authenticates to the shared API; it does not carry a second scanner engine or belong in the hosted skill loader.

| Skill delivery option | Position in this design | Consequence |
| --- | --- | --- |
| Selected pinned plugin included during image build | Proposed v1 default | Predictable startup and rollback; skill changes produce a new image release |
| Pinned plugin bundle retrieved from approved private storage at startup | Optional future alternative | Requires application-managed retrieval, integrity/compatibility checks, availability handling and rollback of the image/bundle pair |

Do not mount or automatically activate the entire marketplace, resolve `latest` during an assessment, or permit a request to choose arbitrary plugin paths. A startup-fetch alternative must pin the artifact in deployment configuration, verify it before readiness and hold that version for the attempt. It must not become per-request marketplace installation.

Use separate development, test and production configuration with least-privilege identities. Produce provenance/SBOM metadata and promote the same immutable application/plugin/rule release. Existing attempts retain their pinned release; rollbacks select a previously validated image and configuration combination through controlled Runtime version/endpoint selection. Separate source ownership does not create runtime privilege separation.

Log assessment/attempt IDs, phase transitions, elapsed time, error classes, artifact hashes, model usage and rule versions. Exclude credentials, raw source, target responses and full prompts by default. Record authorisation failures, budget exhaustion, stale attempts, model-stage failures, and report access. Restrict access to traces that may contain sensitive content.

| Operational objective | Verification approach |
| --- | --- |
| Reproducibility | Same captured evidence and pinned rules produce equivalent deterministic findings |
| Isolation | Cross-owner session/status/report access is rejected before context reuse |
| Reliability | Lost responses, duplicate submits and killed sessions preserve honest durable state |
| Bounded execution | Hostile archives, parsers, endpoints and model responses remain within configured limits |
| Performance/cost | Benchmark repository sizes and 20/100/500-target workloads, concurrent callers and cache states; cap inference and task concurrency separately |
| Availability and recovery | Agree numeric service objectives and recovery ownership before production |

MCP Inspector can remain an optional approved diagnostic adapter. It is not required for the production engine or used as the policy verdict. AgentCore Gateway, Memory, a vector database, a browser UI, and a separate scanner MCP server are not required by this HLD.

## 11. Delivery and validation

1. **Contracts and fixtures:** target/evidence/result schemas, representative corporate controls, versioned rule interfaces, benign/vulnerable MCP and skill fixtures.
2. **Assessment core:** static checks, permitted discovery, deterministic findings/coverage, shared control mapping, report output.
3. **Hosted agent and packaging:** marketplace scanner-plugin release, two explicitly loaded trusted skills, framework adapter, registered engine tools, approved model configuration and validated image composition.
4. **Service integration:** AgentCore invocation/authentication, storage, admission, leases, cancellation, resume and thin Claude Code client.
5. **Pilot:** validate mixed batches, malicious skill instructions, target permission boundaries, failure recovery, data handling, performance and cost.

Release criteria include verified skill loading from the pinned trusted package; rejection of unapproved plugin components and incompatible engine-tool contracts; no target activation/execution; equivalent deterministic results across clients, concurrency settings and valid cache reuse; no deep remediation in fast mode; faithful findings under prompt-injection attempts; correct coverage for missing evidence or discovery limits; rejected cross-owner access; safe retries and stale-attempt fencing; rollback of a complete release; and graceful partial reports when deep inference fails. These are planned tests, not claims of validation already performed.

## 12. Open decisions before production

| Decision | Required input |
| --- | --- |
| Corporate policy | Catalogue sample/format, control mappings, severity, mandatory unknowns, exceptions |
| Agent implementation | Framework/loader selection and approved dependency/runtime versions |
| Marketplace integration | Marketplace format/location, package source and approval process, immutable identity/signature scheme, engine compatibility contract and separate client publication |
| Identity | JWT or IAM integration, machine callers, session binding and verified ownership propagation |
| Evidence and targets | Approved ingestion and repository-selector schema, MCP revisions/transports, skill/plugin dialects, discovery bounds, credentials and allowlists |
| Isolation | Whether a shared execution role satisfies phase/tenant requirements |
| Bedrock | Approved model/profile, regions, residency, logging, token budgets and fallback policy |
| Operations | Measured batch/concurrency/runtime limits, service objectives, cache policy, scale-out trigger, support ownership, explicit versus automatic recovery |
| Data lifecycle | Classification, redaction, retention/deletion, backup/restore and report access |
| Supply chain | Internal vulnerability feed, update cadence, plugin/rule promotion and reassessment triggers |

The [detailed assessment design](assessment-design.md) records scanner coverage, rule contracts, and further implementation considerations. This HLD defines the overall Security Agent architecture and its review boundaries.
