# Security Agent — MCP and Skill Assessment Design

Status: design proposal; assessment skills, local engine distribution, hosted agent, AgentCore deployment, rules, and integrations are not implemented.

For the system architecture and principal flows, start with the [Security Agent high-level design](high-level-design.md).

This document records the design for an internal security assessment capability covering Model Context Protocol (MCP) servers and agent skill packages. Its purpose is to gather evidence, identify security findings, map them to the corporate control catalogue, and explain assessment limitations. A scan alone does not establish that a target is safe for every use.

## Objective

Build Security Agent around one trusted security plugin containing `mcp-assessment` and `skill-assessment` skills, backed by a shared deterministic engine, corporate-control mappings, evidence, and reporting. Support direct execution in an approved local coding assistant and a hosted agent deployed on Amazon Bedrock AgentCore Runtime with an authenticated invocation API. Assess internal and external MCPs and skill packages using available source/configuration and explicitly permitted endpoint evidence, with additional analysis through an approved AWS Bedrock inference profile for remediation in deep mode. An optional separate client skill provides terminal access to the hosted API.

AgentCore Runtime replaces the earlier EKS application and worker deployment proposal for hosted execution. Local execution is also a supported path: the same scanner plugin invokes the installed shared engine directly, without an AgentCore call. The application repository will produce a hosted image and a reusable local engine package, provisionally a pinned Python wheel with a CLI/tool adapter. The engine remains one implementation; the packaging and adapters are proposed, not existing capabilities.

## 1. Confirmed scope

- Both internal and external MCPs may provide repository source code, configuration, deployment artifacts, endpoint evidence, or a combination. Source availability is independent of ownership.
- Skill targets are source packages, whether standalone or contained in a plugin/repository. Inspect relevant package instructions, scripts, dependencies, and containing-plugin configuration as evidence without activating the target.
- Assessments use supplied files plus explicitly allowed discovery connections.
- Deliver one canonical marketplace scanner plugin containing two skills, usable in compatible local assistants and the hosted AgentCore agent. Each scanner has its own target adapter and rule pack; evidence, policy, and report contracts are shared.
- Local assistants invoke the installed pinned engine directly. An optional separate remote client plugin submits to the hosted API and retrieves reports. API callers do not require Claude Code.
- Local-assistant and hosted-agent orchestration use approved Bedrock inference in both modes; an assistant acting as a remote client adds its own host inference. All workflow-controlled inference must use approved Bedrock configurations.
- Fast mode runs deterministic assessment scripts and produces findings, coverage, and maintained remediation guidance. The assessment engine makes no model calls for detection or remediation.
- Deep mode runs the same scripts first, then makes an additional call through an approved AWS Bedrock inference profile for contextual remediation suggestions.
- The assessment does not depend on external scanning services or public inference APIs. Approved AWS infrastructure, explicitly approved repository retrieval, target discovery, and approved Bedrock inference are permitted network dependencies when enabled. Neither local execution nor AgentCore hosting authorises other model providers.
- The corporate catalogue is the policy authority. External guidance supplements threat coverage and finding labels.

The engine CLI can also run directly without host inference; when invoked through a coding assistant or hosted agent, fast mode still involves that host's inference. Status, report retrieval, and cancellation use structured application logic and need no model call. Local/hosted execution and fast/deep analysis are independent choices. Any assistant handling corporate controls or evidence must use an approved environment.

## 2. Architecture and mode boundaries

Use the same trusted scanner plugin and engine through two execution adapters:

```text
LOCAL DIRECT                              HOSTED
Approved coding assistant                 Optional API client / application / CI
  + trusted scanner plugin                             ↓
              ↓                           Authenticated InvokeAgentRuntime API
Pinned local engine CLI/tool adapter                    ↓
              ↓                           Hosted agent + same scanner plugin
Shared deterministic engine                             ↓
              ↓                           Registered engine tools + same engine
Approved local run records/reports                       ↓
                                          S3 artifacts + DynamoDB registry

Scanner plugin: mcp-assessment + skill-assessment
Both paths: same engine, pinned rules/mappings, optional deep Bedrock adapter
Local direct execution does not invoke AgentCore or automatically upload artifacts.
```

The engine enforces this pipeline in either execution path:

```text
Scope and policy → evidence collection → normalized evidence
    → deterministic rules → control mappings → findings and coverage
        ├── fast report
        └── minimized evidence → Bedrock → validated remediation → deep report
```

| Component | Responsibility |
| --- | --- |
| Two assessment skills | Target-specific workflow instructions, evidence requirements, interpretation guidance, and references |
| Local assistant / hosted agent | Interpret requests within authorised scope, invoke the installed engine adapter, and explain results using approved Bedrock inference |
| Deterministic engine | Establish findings, mappings, coverage, and policy results; enforce required phases and mode restrictions in code |
| Remediation stage | Generate validated advisory suggestions from existing findings without execution tools |
| Local CLI/tool adapter | Validate local requests and scope; invoke the shared engine; maintain approved local run records and reports |
| Runtime entry point | Validate hosted typed operations and caller/target access; manage durable hosted assessment state |
| Optional remote client skill | Prepare requests, invoke the hosted API, retrieve reports, and present results |

Skill instructions guide the agent but do not enforce access control or guarantee execution order. Required checks and publication gates are enforced by code, even if the agent skips a tool or receives malicious target instructions.

Keep these configuration dimensions separate:

| Dimension | Proposed values or content |
| --- | --- |
| Analysis mode | `fast`, `deep` |
| Target type | `mcp`, `skill`; a mixed batch declares the type on each target |
| Invocation interface | Local scanner plugin with engine adapter; hosted API directly or through the optional remote client |
| Execution location | Local process or AgentCore Runtime; fast and deep supported in either |
| Ownership/origin | Internal, external; identify the owner or maintainer |
| Evidence available | Repository source, configuration, deployment artifacts, endpoint observations; any combination for either origin |
| Permitted access | Supplied files; explicitly approved repository retrieval and discovery; future separately authorised testing |
| Deployment context | Company-hosted or provider-hosted, transport, protocol version, data sensitivity |

### AgentCore Runtime API and client interfaces

For hosted execution, use the managed [InvokeAgentRuntime API](https://docs.aws.amazon.com/bedrock-agentcore/latest/APIReference/API_InvokeAgentRuntime.html). The application interprets a validated operation envelope; these are proposed application operations, not additional native AgentCore REST routes:

| Operation | Contract |
| --- | --- |
| `start_assessment` | Authorise the manifest, bind an idempotency key to the caller and input digest, persist the assessment, then start bounded work and return its ID |
| `get_assessment_status` | Authorise access and return durable phase, per-target progress, and errors |
| `get_assessment_report` | Authorise access and return the report or scoped artifact references |
| `cancel_assessment` | Persist a cooperative cancellation request; assessment tasks check it at bounded intervals |
| `resume_assessment` | Authorise a new attempt for interrupted work, preserving the approved manifest, evidence, and version constraints |

Include a schema version, operation, and validated operation-specific arguments. A start request contains mode, typed target/evidence references, permitted discovery, and an idempotency key. Route each target to the corresponding trusted scanner skill using validated `target_type`; the model cannot switch to an arbitrary skill or enlarge the target set. Do not accept arbitrary shell commands, cloud roles, model profiles, or executable prompts as operation definitions. Return an explicit assessment state in the JSON payload; an accepted assessment is not a completed assessment. Native HTTP success can be 200 with an `accepted` application state. A separate REST facade could later expose `/v1/assessments`, but is not required for API delivery.

Implement the Runtime HTTP contract through the AgentCore SDK: `/invocations` for requests and `/ping` for health. Choose IAM/SigV4 or configured JWT inbound authentication; the selected Runtime configuration uses one authentication mode. AWS SDK invocation fits IAM; JWT callers use authenticated HTTPS. Integrate the organisation's identity system and authorise every assessment, target, and report operation. Derive ownership from verified identity or a trusted identity propagation layer, never request-body owner fields or a session ID. Verify identity propagation as part of deployment; transport authentication alone does not provide per-report ownership. See [HTTP contract](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-http-protocol-contract.html) and [inbound authentication](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html).

The optional remote client skill collects scope and mode, invokes a packaged API helper, and presents the report. Keep credentials outside skill files. Hosted source inputs identify an uploaded immutable snapshot or an explicitly approved repository revision; the hosted agent cannot read a caller's local path. Both hosted analysis modes use the same agent. Configure Claude Code's own inference through approved [Bedrock settings](https://code.claude.com/docs/en/amazon-bedrock).

### Local direct execution

Install the reviewed marketplace scanner plugin and a compatible, pinned shared engine package in an approved local environment before assessment. The host adapter exposes the engine's typed CLI/tool contract; plugin instructions alone do not supply the executable engine. Select local execution explicitly and validate the plugin, engine, rule and configuration release identities. Missing or incompatible components fail clearly; do not silently switch to the hosted API or download executable dependencies during a scan.

The local assistant invokes the engine against an authorised local snapshot or explicitly permitted repository retrieval. Capture a stable evidence snapshot and hashes before evaluating checks so concurrent edits cannot produce an unidentified mixture of file versions. Local source access does not authorise endpoint connections. Protected MCP discovery uses separate, least-privilege target credentials, not the assistant's model credentials or the hosted API token.

Store manifests, evidence references, phase/attempt records and reports in an approved local location with access controls, retention and secret redaction. Local execution requires neither S3 nor DynamoDB, and does not automatically submit evidence or reports to AgentCore or upload them elsewhere. Define local cancellation, interrupted-run reporting and explicit resume using those run records; do not imply cloud lease or recovery guarantees. Local deep mode uses the same remediation adapter with approved AWS credentials and inference profile. Both the assistant's own Bedrock orchestration and deep remediation can send selected content to Bedrock; local execution is not synonymous with offline execution.

Start the coding assistant in a neutral trusted workspace and pass the target snapshot as an evidence path. Disable automatic target-project instruction, plugin, hook and MCP loading, including `AGENTS.md`, `CLAUDE.md` and `.mcp.json`, through the chosen host adapter. Loading the trusted scanner plugin is permitted; activating the package under assessment is not. Local assessment does not run target scripts, hooks, builds or local stdio MCP servers. Local filesystem, environment credentials, sandbox and network permissions differ from AgentCore and require their own approved configuration; the two paths do not provide equivalent isolation.

### Skill packaging and agent integration

Use one canonical `security-assessment` plugin containing `mcp-assessment` and `skill-assessment`, distributed through the existing corporate marketplace. Each skill supplies its own procedure and references and invokes the common engine through a validated CLI/tool adapter. Approved local assistants install that scanner plugin and a compatible pinned engine package. The hosted build includes an approved, pinned plugin release in the application image. The optional `assessment-client` skill lives in a separate marketplace plugin for remote API access; exclude it from hosted skill loading to prevent recursive submission. The [HLD packaging and release flow](high-level-design.md#10-deployment-and-operations) defines source ownership, compatibility checks, pinned release identities and the optional future hosted startup-fetch alternative.

A plugin groups versioned skills and supporting resources; it is not another deployed agent or container. The runtime framework must explicitly load the two trusted skills. Strands is a proposed option: its `AgentSkills` integration loads skill instructions, while the application supplies resource-access tools. Alternatively, the Claude Agent SDK can load a Claude Code plugin explicitly. A Claude `.claude-plugin/plugin.json` manifest is not automatically interpreted by AgentCore or by Strands. Choose and pin the framework adapter during implementation. See [Strands skills](https://strandsagents.com/docs/user-guide/concepts/plugins/skills/) and [Claude Agent SDK plugins](https://code.claude.com/docs/en/agent-sdk/plugins).

In a local compatible coding assistant, use the canonical scanner plugin for direct execution, or select the separate remote client for hosted execution. The two roles must be explicit; installing the local scanner does not require using the hosted API. The marketplace-backed package sources remain canonical while the Security Agent repository owns the executable engine and emits both a local package and hosted image. Share versioned API/tool contracts and avoid duplicate engine implementations or a second editable copy of scanner skills. Pin the selected host adapter as well as plugin/engine versions, and verify that it does not activate unapproved plugin components. If a Claude plugin format is used, its manifest and namespacing serve compatible hosts; the Strands adapter explicitly registers only its two skill directories. See [Claude Code skills](https://code.claude.com/docs/en/skills), [plugin packaging](https://code.claude.com/docs/en/plugins), and the [Agent Skills format](https://agentskills.io/specification).

Keep skills concise; load reviewed control excerpts and detailed references as needed. Record skill and instruction versions with each assessment. Separate trusted scanner skills from target evidence paths. Do not load target skills, plugins, hooks, `AGENTS.md`, `CLAUDE.md` or MCP settings as active host configuration. Static inventory of these files is permitted evidence collection. Referenced target scripts are evidence and must not execute. Plugin hooks and unrestricted shell tools are not required for the scanner.

### Sessions, batches, and durable execution

For hosted execution, use one bounded assessment attempt per Runtime session initially, with a fresh session ID scoped to the authenticated owner. One repository batch can contain multiple MCP or skill targets. A batch of 20 items does not automatically create 20 Runtime environments. On the proposed microVM compute type, new session IDs receive separate execution environments; calls to the same active session reuse its environment. Keep assessment ID, attempt ID, Runtime session ID, and SDK task ID distinct. Local batches use engine run/attempt identifiers and local checkpoints without creating Runtime sessions. See [Runtime sessions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-sessions.html).

Choose application concurrency explicitly: for example, four work items within one batch, tuned after measurement. Set separate bounds for parsing, discovery, Bedrock calls, and total in-flight assessments. Reuse repository-wide observations while preserving per-target evidence and coverage. Internal tasks/subagents share the environment; separate Runtime sessions are an explicit deployment and scheduling choice. One security agent means one maintained agent application, not one globally shared session for all users.

For hosted asynchronous work, register and complete SDK background tasks, keep health responses responsive, and report busy status while work continues. Do not block the invocation/health loop with scanner work. Busy status prevents idle expiry; it does not override maximum lifetime or recover lost work. See [AWS asynchronous processing guidance](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-long-run.html).

For the hosted service, persist owner, canonical input digest, pinned versions, state, per-target progress, artifact hashes, current attempt, lease expiry, and cancellation in DynamoDB; store evidence and reports in private encrypted S3. Write acceptance before acknowledging submission. Bind idempotency keys to owner and input digest, rejecting conflicting reuse. Use conditional lease acquisition and an attempt token to prevent stale attempts publishing results. Preserve completed phases; mark missing work as interrupted or partial. An authenticated `resume_assessment` starts a new attempt after lease expiry with bounded retries. Reports remain available independently of session memory. The local storage adapter preserves applicable manifest, phase, attempt and artifact records in approved local storage, with local locking/atomic publication; it does not depend on these AWS stores.

V1 recovery is explicit in either path: interrupted work is reported and the caller can request resume after authorisation and checkpoint validation. For hosted work, automatic recovery requires a separately deployed reconciler that detects stale or accepted-but-not-started assessments and invokes a new Runtime attempt. A scheduled AWS function can provide that control-plane role without moving hosted assessment execution out of AgentCore. SQS is optional for hosted admission/backpressure; neither a queue nor SDK background-task tracking alone guarantees recovery. Local process termination likewise does not imply automatic restart. Retries may repeat discovery or inference, so do not promise exactly-once execution. Bound deadlines to the selected environment and checkpoint before exhausting budgets.

### Runtime implementation and security boundary

Proposed shared components are Python, Pydantic contracts, registered deterministic checks, an MCP SDK adapter, an approved Bedrock remediation adapter, and local/hosted storage adapters. Keep the engine independent of the agent framework. Package a pinned local wheel/CLI and the hosted Runtime image from the same engine source. The hosted layer adds the AgentCore SDK and selected agent framework; compatible local hosts use an explicitly configured plugin/tool adapter and do not require the AgentCore serving layer. Pin dependencies, skills and rules in both distributions, publish approved local packages internally and hosted images in ECR, and use infrastructure as code for Runtime configuration. A FastAPI service, Kubernetes deployment, MCP service adapter, or browser UI is not required to expose the native invocation API.

Within one Runtime environment, collection, checks, and remediation have logical module boundaries but share the execution role, filesystem, and network configuration. Fast mode's lack of detection/remediation model calls is enforced and tested in code; agent orchestration still needs Bedrock access. Session compute isolation does not automatically restrict a shared role's S3 or secret permissions per assessment. Scope IAM and artifact access, check ownership in trusted code, and keep raw evidence out of ordinary model context. If stronger phase-level isolation is mandatory, use separately constrained Runtime deployments/roles or approved services. Do not claim that Python modules or subprocesses create IAM isolation.

Hosted assessment callers receive invocation access, not Runtime shell/command permissions. Expose only registered assessment operations to the agent. Validate egress destinations, redirects, and DNS as part of collection in either path; VPC attachment or local network access alone is not a target allowlist. Configure approved connectivity for internal services and AWS APIs where required. See [Runtime security guidance](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-security-best-practices.html) and [VPC connectivity](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-vpc.html).

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

Neither the local assistant, hosted agent, remote client host, nor remediation model may change authoritative deterministic findings, severity, exceptions, or policy decisions. If remediation inference fails in either path, retain the deterministic report and mark the advisory stage unavailable or partial.

### Inference boundaries

MCP is a communication protocol. A scripted client can connect to a server and request discovery results without invoking a model; the official [MCP Inspector CLI documentation](https://modelcontextprotocol.io/docs/2026-07-28/tools/inspector/cli) demonstrates scripted MCP requests.

| Boundary | Fast mode | Deep mode |
| --- | --- | --- |
| Local coding assistant, on the direct path | Uses approved Bedrock host inference to follow the scanner skill and invoke the local engine | Same local orchestration; no AgentCore invocation |
| Optional remote client assistant | Uses approved host inference to invoke the API and present results | Same client-host inference |
| Hosted security agent, on the remote path | Uses approved Bedrock inference to follow the scanner skill within the enforced workflow and explain findings | Same bounded orchestration |
| Deterministic assessment engine | No model calls; scripts establish findings, mappings, and policy results | Identical deterministic checks and results |
| Remediation analysis, either path | Maintained guidance returned by scripts; no dedicated model analysis | Shared adapter calls the approved Bedrock profile for contextual suggestions |
| Target MCP implementation | May itself use inference; discovery alone cannot establish its internals | Same uncertainty; deep mode does not expand discovery permissions |

The local assistant, hosted agent and remote client must present script-produced findings faithfully. Free-form output must not replace deterministic evidence or introduce authoritative findings. A standalone engine CLI invocation omits host inference, but fast mode invoked through either assistant path is not inference-free. Record execution location, local/client-host, hosted-agent and remediation model usage separately where observable, and identify target-side inference as declared, observed, or unknown. Skill source assessment cannot establish how an unseen host actually executes that skill.

## 3. Control model

Separate policy requirements, technical checks, threat categories, and evidence:

```text
Evidence → assessment rule → finding → corporate control(s)
                                   → external category/categories
```

Corporate controls define applicability, required evidence, acceptance criteria, ownership, and exception handling. MCP specifications provide version-specific requirements for MCP targets. OWASP MCP Top 10 provides a threat taxonomy and coverage cross-check for applicable MCP risks; skill findings use reviewed corporate mappings and relevant agentic/software-security categories. Do not force every skill finding into an MCP category. Broader agentic guidance is useful where host behaviour or cross-tool actions affect risk.

Mappings should be reviewed, versioned, and many-to-many. Report a catalogue coverage gap when a relevant finding has no corporate mapping. Do not invent control identifiers or let a model create authoritative mappings.

Pin external reference versions or commits. OWASP’s MCP project is evolving, so framework updates should trigger mapping review instead of silently changing assessment results. See [OWASP MCP Top 10](https://owasp.org/projects/mcp-top-10) and [OWASP Top 10 for Agentic Applications](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/).

## 4. Proposed package structure

The [HLD application and plugin layout](high-level-design.md#4-application-plugin-and-skill-packaging) defines the deployment paths and engine modules. Keep these sources independently maintained:

```text
Security Agent application repository
├── src/security_agent/             # Shared engine, local CLI and hosted adapters
├── packaging/                     # Proposed local wheel/CLI distribution
├── deployment/                    # Hosted container, IaC and release lock data
├── control-mappings/              # Schemas and sanitised examples
├── schemas/                       # Assessment, finding, remediation contracts
├── assets/                        # Report templates
└── tests/                         # Fixtures and expected results

Corporate marketplace backing package sources
└── plugins/
    ├── security-assessment/
    │   ├── .claude-plugin/plugin.json   # If Claude-compatible
    │   └── skills/
    │       ├── mcp-assessment/
    │       └── skill-assessment/
    └── assessment-client/              # Optional remote-access plugin
        └── skills/assessment-client/  # SKILL.md and hosted API helper

Approved private configuration storage
└── versioned catalogue, mappings, rule metadata and policy artifacts
```

The hosted image build and approved local installation consume the selected pinned scanner bundle from its marketplace source; neither creates a second editable skill source. The application build emits both a hosted image and a compatible local engine package from the same code. Keep scanner `SKILL.md` files focused on target-specific procedures and references, and the optional remote client focused on API invocation and report retrieval. Provision authorised, versioned corporate-control snapshots through the appropriate local or hosted configuration adapter; keep credentials out of packages. Reuse executable checks across both scanners and execution paths, and enforce scope and mode selection in code. This structure is a proposal, not a list of existing files; local package delivery and the marketplace's concrete format and locations remain to be confirmed.

## 5. Assessment coverage

### MCP scanner

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

### Skill scanner

| Area | Initial checks and limitations |
| --- | --- |
| Identity and format | Applicable `SKILL.md` metadata, name/path consistency, owner, declared purpose, missing referenced resources; format compliance alone is not security assurance |
| Instructions and authority | Indicators of host-policy bypass, hidden actions, unrelated secret requests, excessive delegation, or persistent permission changes; semantic indicators remain heuristic |
| Declared capabilities | Broad tool, filesystem, and network access compared with intended use; declarations do not prove enforcement by the host |
| Bundled scripts | Unsafe command construction, dynamic execution, credential access, download-and-execute patterns, destructive operations, and unrestricted reads/writes, inspected statically |
| Dependencies and provenance | Version pinning, lockfiles, package identity, internal vulnerability data and freshness, unverifiable binaries |
| Data handling | Sensitive collection, outbound destinations, logging/report exposure, retention instructions |
| References and paths | Traversal, symlink escape, absolute paths, missing helpers, and remote references outside approved evidence scope |
| Containing-plugin context | Relevant hooks, MCP configuration, agents, commands, and settings that affect the selected skill; inspect as data without installing the plugin |

Skills legitimately contain instructions. Imperative wording alone is not a finding. Assess instructions against intended purpose, corporate policy, and available evidence. Validate the [Agent Skills format](https://agentskills.io/specification) while treating host-specific fields separately; do not assume a field such as `allowed-tools` is a security boundary across hosts.

Link cross-file evidence, such as a skill instruction invoking a script that reads credentials and sends them to a destination. A plugin hook or configured MCP server may affect several skills; record shared findings once with explicit affected-target links. A reference to an MCP does not authorise connecting to it. Include missing related artifacts in coverage; source findings do not prove runtime behaviour. A mixed batch preserves separate target-type coverage and does not apply MCP-only controls to skill files.

## 6. Discovery permissions and scanner threat model

Treat MCP servers, skill packages, supplied source, manifests, descriptions, schemas, and returned content as untrusted input. They may try to influence the scanner, exhaust resources, access internal destinations, or place malicious instructions in the agent or remediation prompt. Keep target skills and plugin files outside trusted discovery paths; inspecting them never installs or activates them.

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

- Finding ID, rule/version, target ID/type, affected-target links, context, and timestamp.
- Evidence references, hashes, locations, and redacted excerpts.
- Repository revision or snapshot identity where source is supplied, and source-to-deployment correspondence where an endpoint is also assessed.
- Observed, declared, or inferred evidence classification.
- Corporate mappings and external categories.
- Severity, confidence, limitations, remediation, and verification steps.

Store raw evidence separately with restricted access in the approved local or hosted store. Reports should reference it without unnecessarily copying secrets or sensitive source content. Record execution location, local package or hosted image identity, scanner, plugin/skill, host configuration, rule-pack, mapping, and reference versions for reproducibility. Include environment-dependent limits, unavailable dependencies, source/endpoint reachability, and access restrictions in coverage rather than claiming the same operational assurance for local and hosted runs. Use typed target references so the same repository can contain MCP and skill targets without mixing their coverage.

Generate `findings.json`, `coverage.json`, and a readable report first. Add SARIF when needed by internal developer workflows. Report unresolved mandatory checks alongside any policy result. Endpoint-only assessments have narrower assurance: no observed weakness is not evidence that inaccessible controls work.

## 8. Bedrock remediation stage

Send only selected findings, relevant control text, and necessary redacted excerpts. Retrieve control text locally by reviewed IDs; a vector database is unnecessary for the initial design.

Use an approved inference profile with least-privilege IAM permissions. Record profile, model, prompt version, and inference settings. The remediation model receives no execution tools and cannot fetch additional evidence or act on target instructions. The local assistant or hosted agent can invoke registered engine operations within the configured scope; that orchestration permission does not extend to the remediation model. Both scanner skills and execution paths share this advisory adapter. Locally, use an approved AWS credential provider and configured inference profile without invoking AgentCore; missing credentials or denied inference must preserve the deterministic report and mark advice unavailable. Neither local plugin installation nor AgentCore hosting selects the approved model/profile automatically.

Require structured suggestions containing finding ID, proposed fix, evidence IDs, assumptions, and verification steps. Reject malformed output and unknown references. Keep advice visibly separate from deterministic results.

Where appropriate, use [Bedrock private connectivity](https://docs.aws.amazon.com/bedrock/latest/userguide/usingVPC.html). Validate actual processing destinations against residency requirements when using [cross-region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html). Review model/API-specific [Bedrock retention behaviour](https://docs.aws.amazon.com/bedrock/latest/userguide/data-retention.html), invocation logging, report retention, and access controls. No particular profile, region, or model has been selected.

## 9. Delivery milestones and open decisions

1. Define typed MCP/skill input, evidence, finding, coverage, and mapping schemas using representative corporate controls.
2. Implement the shared engine and separate rule packs with fixtures and a small set of high-confidence checks for each target type.
3. Add bounded, explicitly permitted MCP discovery and inert skill/plugin source inspection with honest coverage reporting.
4. Author the two scanner skills in one trusted marketplace plugin, implement compatible local and hosted tool adapters, and enforce the pipeline outside model instructions.
5. Package the shared engine for approved local installation and the hosted image; add approved Bedrock orchestration and shared deep remediation, preserving deterministic results.
6. Deploy the security agent on AgentCore Runtime with authenticated typed invocation, ownership checks, durable status/artifacts, bounded batch execution, cancellation, and explicit interruption recovery.
7. Support direct use of the scanner plugin in compatible local assistants, with pinned engine installation, neutral workspaces and approved local records; optionally deliver the separate remote client plugin against the API.
8. Pilot local and hosted execution against known benign and vulnerable fixtures, including source-only, endpoint-only, combined MCP evidence, skill packages, and mixed repositories.

Acceptance tests should verify that the deterministic engine makes no model calls; fast workflows in either path cannot invoke the dedicated remediation stage; local/hosted/client orchestration uses approved inference configuration and preserves script-produced findings; discovery never becomes target tool execution or local stdio startup; malformed, hostile, or inaccessible targets yield bounded results; missing evidence remains visible; and deep remediation preserves deterministic findings. Target-side inference cannot be ruled out from endpoint discovery alone.

Include external MCP repository fixtures and internal MCP endpoint-only fixtures. Verify that evidence availability controls technical check selection, and that an unverified or mismatched source revision cannot produce an unsupported deployment-level pass.

Verify equivalent deterministic findings and control mappings across local direct execution, remote client and direct API for the same captured evidence, engine/plugin/rule/catalogue/mapping versions and assessment context, excluding run metadata. Live observations, model suggestions and environment-dependent coverage may differ and must be reported. Test local/hosted selection independently of fast/deep mode, target-type selection, unknown skill names, incompatible releases, required phase completion, authorised evidence references and report access. Ordinary API consumers must work without a local Claude Code installation; local direct assessment must work without AgentCore invocation, S3 or DynamoDB.

For local execution, verify no automatic evidence/report upload, local access and retention controls, stable snapshots during concurrent edits, bounded workers, interruption/cancellation, local locking and explicit resume. Exercise both valid and unavailable Bedrock credentials for local deep mode. Confirm a local run does not silently become a remote submission, and that target permissions remain separate from model/API credentials. Test the local host from a neutral workspace with a malicious target `AGENTS.md`, `CLAUDE.md`, `.mcp.json`, hooks and plugin files, proving these are not automatically loaded or activated.

For hosted execution, test duplicate submissions, idempotency-key conflicts, lost responses, accepted-but-not-started work, session termination, stale leases, explicit resume, cancellation, hung checks, and partial deep-mode failure. Verify per-batch and inference concurrency limits in both environments. Exercise a 20-target batch locally and remotely without assuming one process/session per skill. Confirm interrupted attempts cannot overwrite newer results in either storage adapter.

Add skill fixtures containing malicious instructions, misleading `allowed-tools`, escaping paths, bundled download-and-execute scripts, plugin hooks, and linked MCP configuration. Confirm the scanner never installs or activates target skills/plugins, reads target repository instructions as trusted policy, or invokes its own client skill recursively. Test the trusted scanner plugin itself before promotion and pin the release under test.

Decisions still needed: corporate catalogue format and sample controls; supported MCP revisions/transports and skill/plugin dialects; discovery allowlists and credentials; severity and mandatory-unknown policy; local/hosted evidence retention/redaction; internal vulnerability dataset; approved Bedrock profile/model/regions; hosted framework and approved local host adapters; plugin/local engine/optional client distribution and compatibility; API authentication and verified identity propagation; local filesystem/sandbox and cloud isolation requirements; measured batch/runtime budgets; whether automatic recovery is required beyond explicit resume; and reassessment cadence.

The Git repository should initially contain design, schemas, reviewed rules, and synthetic fixtures. Runtime evidence, credentials, generated sensitive reports, and unapproved corporate material require explicit handling rules before they are committed.
