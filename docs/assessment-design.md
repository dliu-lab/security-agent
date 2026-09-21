# Security Agent — MCP and Skill Assessment Design

Status: design proposal; assessment skills, local engine distribution, hosted agent, AgentCore deployment, rules, and integrations are not implemented.

**Aligned baseline:** HLD v0.6 — reviewed 21 September 2026.

For the system architecture and principal flows, start with the [Security Agent high-level design](high-level-design.md).

The HLD defines shared architecture, delivery, API, lifecycle and persistence decisions. This document expands assessment coverage, rules and evidence requirements under those decisions. Update both documents when a shared contract changes.

This document records the design for an internal security assessment capability covering Model Context Protocol (MCP) servers and agent skill packages. Its purpose is to gather evidence, identify security findings, map them to the corporate control catalogue, and explain assessment limitations. A scan alone does not establish that a target is safe for every use.

## Objective

Build Security Agent around one trusted security plugin containing `mcp-assessment` and `skill-assessment` skills, backed by a shared deterministic engine, corporate-control mappings, evidence, and reporting. Support direct execution in an approved local coding assistant and a hosted agent deployed on Amazon Bedrock AgentCore Runtime with an authenticated invocation API. Assess internal and external MCPs and skill packages using available source/configuration and explicitly permitted endpoint evidence, with additional analysis through an approved AWS Bedrock inference profile for remediation in deep mode. An optional separate client skill provides terminal access to the hosted API.

AgentCore Runtime replaces the earlier EKS application and worker deployment proposal for hosted execution. Local execution is also a supported path: the same scanner plugin invokes the installed shared engine directly, without an AgentCore call. The application repository will produce a hosted image and a reusable local engine package, provisionally a pinned Python wheel with a CLI/tool adapter. The engine remains one implementation; the packaging and adapters are proposed, not existing capabilities.

## 1. Confirmed scope

- Both internal and external MCPs may provide repository source code, configuration, deployment artifacts, endpoint evidence, or a combination. Source availability is independent of ownership.
- Skill targets are source packages, whether standalone or contained in a plugin/repository. Inspect relevant package instructions, scripts, dependencies, and containing-plugin configuration as evidence without activating the target.
- Assessments use supplied files plus explicitly allowed discovery connections.
- Support repositories of skills, MCP source repositories, MCP endpoints, and skills/MCPs installed in a supported CLI. A local installation collector supplies inert evidence to local checks or, after explicit export/upload, to hosted checks.
- Deliver one canonical marketplace scanner plugin containing two skills, usable in compatible local assistants and the hosted AgentCore agent. Each scanner has its own target adapter and rule pack; evidence, policy, and report contracts are shared.
- Maintain agent/engine and scanner-plugin source in separate repositories. The marketplace distributes approved plugin releases; the application consumes a pinned compatible release.
- Local assistants invoke the installed pinned engine directly. An optional separate remote client plugin submits to the hosted API and retrieves reports. API callers do not require Claude Code.
- Local-assistant and hosted-agent orchestration use approved Bedrock inference in both modes; an assistant acting as a remote client adds its own host inference. All workflow-controlled inference must use approved Bedrock configurations.
- Fast mode runs deterministic assessment scripts and produces findings, coverage, and maintained remediation guidance. The assessment engine makes no model calls for detection or remediation.
- Deep mode runs the same scripts first, then makes an additional call through an approved AWS Bedrock inference profile for contextual remediation suggestions.
- For source-repository assessments invoked from GitHub Actions, an optional `workflow_dispatch` parameter requests proposed code fixes and a draft remediation PR. Runtime generates a bounded advisory patch; a separate trusted GHA workflow validates and publishes it without merging.
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
                                          S3 artifacts + Aurora PostgreSQL registry
                                          (SQL through the RDS Data API)

Scanner plugin: mcp-assessment + skill-assessment
Both paths: same engine, pinned rules/mappings, optional deep Bedrock adapter
Local direct execution does not invoke AgentCore or automatically upload artifacts.
```

Application code enforces this workflow in either execution path. The deterministic engine owns collection through baseline reporting; the local or hosted controller invokes the separate remediation adapter only in deep mode:

```text
Scope and policy → evidence collection → frozen target inventory and normalized evidence
    → deterministic rules → control mappings → findings and coverage
        → publish deterministic baseline report
            ├── fast: finish
            └── deep: minimized evidence → Bedrock adapter → validated advice
                       → optional requested source patch → deep report

Optional GHA delivery: validated patch → disposable candidate checkout/static checks
    → separate publishing job → draft PR (original findings remain unchanged)
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

### Four scenarios and intake contract

Implement three evidence collectors: `repository`, `mcp_endpoint` and `cli_installation`. They produce a shared inventory/evidence contract consumed by the two target adapters, `skill` and `mcp`. Do not duplicate check engines for each entry point. A repository or installation can contain many targets; a target can have several evidence sources. Internal/external ownership, collection location, assessment backend and fast/deep mode remain independent fields.

The [HLD collector coverage table](high-level-design.md#collector-coverage) maps each scenario to its required component. Endpoint collection uses a bounded MCP SDK client; repository and installation collection feed the shared skill/MCP checks described below.

| Scenario | Resolution | Reused checks | Expected coverage boundary |
| --- | --- | --- | --- |
| Repository containing skills | Capture once; discover `SKILL.md` packages within approved paths and supported plugin layouts; attach relevant supporting files and containing-plugin context | Skill instructions, static scripts, dependencies, references, shared plugin configuration | Report excluded paths, missing references and unsupported dialects; repository content does not prove how a host loads it |
| MCP source repository | Capture once; use explicit package/entry-point selectors and supported manifest/code patterns to identify MCP candidates | MCP source/configuration, dependencies, auth implementation, execution/data-flow indicators | MCP has no universal source filename; ambiguous or unsupported candidates require explicit scope or an inventory limitation, not a whole-repo pass |
| MCP endpoint | Resolve approved endpoint and transport, credential reference and collection location; capture only permitted version-aware discovery/listing | Observable transport/auth metadata, advertised tool/resource/prompt schemas, description and capability indicators | No tool invocation, resource reads, prompt retrieval, load tests or inferred implementation pass |
| Skill/MCP installed in a CLI | A local host adapter reads selected installation records/configuration, resolves approved package paths and snapshots bytes | Skill/MCP rules plus installation/configuration checks and provenance comparisons | Configured, installed, enabled and observed-running are different states; unavailable source and unverified effective state remain unknown |

Freeze the inventory after bounded resolution. Repository target identity includes snapshot and relative package path; installed instance identity includes capture, host, configuration scope and registration. Keep duplicate or shadowed registrations linked, not collapsed by display name. Parse identical package bytes once, but evaluate each instance's configuration, permissions and associated endpoints separately. Preserve source-to-installed-package and source/installed-package-to-deployment correspondence independently as `verified`, `unverified` or `mismatched`, with supporting evidence. Package/version names alone cannot verify correspondence.

**Repository collection.** Accept a pinned remote revision or bounded local snapshot/archive, including an explicit record of local modifications. Build a shared file index and capture approved common dependency/plugin files once. Apply byte/file/depth/time limits and record truncation and exclusions. Do not execute Git hooks or content filters, initialise submodules, fetch LFS objects, install dependencies or follow symlinks/remote references outside scope merely because a repository names them. An MCP URL found in code is a candidate association, not discovery permission. Missing shared files produce coverage gaps rather than invented evidence.

**Endpoint collection.** Use the same bounded MCP SDK adapter locally or in Runtime. Resolve an `endpoint_ref` through the approved target registry/access configuration; authorise the actual destination, operations and credential context independently of its display name. Record collection time, network vantage and credential-view identity without tokens. Reject unsupported server-to-client requests, including sampling and elicitation, rather than satisfying them through the scanner's model or user credentials. A denied/incomplete listing produces explicit error/unknown coverage. Configured URLs and advertised metadata are not proof of authorisation enforcement. Source evidence can be attached later in a new assessment with an explicit correspondence record.

**Installed CLI collection.** Initially support a pinned Claude Code host/version range behind `engine/collectors/hosts/claude_code`; other CLIs need separate adapters. Select the project context and user/project/managed/plugin scopes to inspect. Read relevant configuration as data, retain its provenance and evaluate precedence only for validated host versions. Record declared enablement, disabled/shadowed entries, unresolved variables and unavailable managed settings. Do not label a configured entry as running or verified effective merely because it exists on disk.

For Claude Code, the adapter needs to account for personal/project skills and plugin-provided skills, rather than scanning only one `skills/` directory. Read relevant `.mcp.json` and `~/.claude.json` sections for project/local/user MCP definitions; managed configuration and plugin definitions can also affect the selected project. Pin scope/precedence interpretation to the supported release. [Skill locations](https://code.claude.com/docs/en/skills#choose-where-skills-load), [MCP scopes](https://code.claude.com/docs/en/mcp#mcp-installation-scopes).

Resolve the installed plugin's actual files from installation metadata: a marketplace plugin may be copied into a versioned cache or loaded from a local source directory. Do not assume the marketplace repository HEAD is what the user installed, or that every cache directory is active. Snapshot the resolved files without refreshing or reinstalling anything. [Plugin cache behaviour](https://code.claude.com/docs/en/plugins-reference#plugin-caching-and-file-resolution).

Treat `command`, `args`, environment declarations and credential-helper configuration as evidence. Do not run `npx`, `uvx`, Docker, interpreters, package managers, helper scripts or host listing commands that may initialise servers. Never import target modules to inspect them. An already installed stdio MCP remains a static package/configuration assessment; do not start it or attach to another client's stdio session. The MCP stdio transport normally has the client launch a subprocess, which would exceed v1 permissions. An already available HTTP endpoint may be assessed separately if explicitly authorised. [MCP transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports).

Use a standalone collector or a clean scanner host process/profile that allows only approved scanner components. Reading another profile does not load it. Do not modify the user's installation to disable targets, and do not rely solely on a neutral working directory to suppress user-wide plugins, hooks or MCP connections. If a supported host cannot establish this boundary, require the standalone CLI for collection. Existing target activity in the user's other sessions is outside the assessment guarantee.

### Portable evidence bundle and execution location

The same local engine package provides collection/export and assessment entry points. The optional remote client can invoke that approved collector when installed; it contains no second collector implementation. API-only callers can instead supply already prepared authorised artifacts. Collection/export does not require agent inference or an AgentCore session.

1. Resolve the permitted inventory and capture selected files/configuration locally into an access-controlled run directory. Detect concurrent edits or installation updates; retry within a bound or mark the snapshot inconsistent instead of mixing versions silently.
2. For a local assessment, evaluate the snapshot with the installed engine. Keep reports local by default.
3. For an explicitly selected hosted assessment, prepare a sanitised portable bundle, call `prepare_evidence_upload`, upload to the scoped S3 destination, then call `complete_evidence_upload` to obtain an immutable owner-scoped evidence reference. Only then call `start_assessment` with explicit target/evidence associations. Upload completion is not an assessment result.
4. Hosted checks validate the bundle and re-evaluate its captured contents under pinned rules. Treat client-collected observations and configuration as supplied evidence with known provenance, not independently verified live state. A hash detects changed bytes; it does not attest completeness or trustworthiness of the workstation.

The proposed bundle manifest contains schema/collector/host-adapter versions; capture time and location; repository revision or installation-capture ID; target/registration identities; file paths relative to approved roots and exported-file digests; configuration provenance; declared/effective-state confidence; endpoint observations with access context and timestamps where authorised; exclusions/redactions; and collection errors/limits. Use portable root identifiers rather than exposing unnecessary user home paths. Validate archive paths, symlinks, expansion ratio, file count/size and total processing bounds before accepting evidence. Freeze validated contents under an immutable artifact reference.

Select relevant config fields instead of exporting whole home/config directories. Never export credential values, OAuth caches, private keys, full process environments or active session transcripts. Inspect secret presence locally, redact values before export, and record how transformation limits hosted checks; a local secret-detector assertion is declared evidence, not a hosted-verified finding. Hash the exported bytes; retain any original-byte evidence only under approved local access/retention. Runtime obtains its own separately authorised target credential reference if hosted discovery is requested. Neither an upload nor a bundle's endpoint list grants that permission.

An endpoint accessible only on the workstation, including loopback or a workstation-only VPN, must be collected locally unless an approved Runtime network path already exists. Export only explicitly permitted observations when central reporting is selected. Runtime must not reinterpret `localhost`, open a tunnel or start a target to regain access. Evidence collection location and assessment backend are both recorded; stale observations may be inadequate for a current-control assessment.

### Submission examples

The following are proposed selector fragments, not implemented commands. Use the shared mode/version/idempotency envelope. Hosted `start_assessment` accepts exactly one of `repository_scope` or `targets`; direct local submission additionally accepts `installation_scope`. `input_kind` is recorded in the resolved inventory, not used as a substitute for `target_type`.

| Entry point | Submission selector | Resolution |
| --- | --- | --- |
| Skills repository | `repository_scope`: snapshot evidence reference, `target_types: [skill]`, approved paths and inventory limits | One batch with a stable target ID per discovered skill |
| MCP source repository | `repository_scope`: snapshot evidence reference, `target_types: [mcp]`, package paths and inventory limits | One or more explicitly resolved MCP package targets; ambiguous candidates remain visible |
| MCP endpoint | `targets`: explicit `target_type: mcp`, registered `endpoint_ref`, approved discovery grant/credential context | One MCP target with bounded endpoint observations; source is optional |
| Installed skill/MCP, local | `installation_scope`: supported host, project context, selected scopes/registrations, permitted read roots and limits | Local inventory, then skill/MCP checks using the same engine |
| Installed skill/MCP, hosted | `targets`: resolved skill/MCP IDs and immutable exported-bundle `evidence_refs` | Hosted static assessment of supplied installation evidence; additional endpoint access requires a separate grant |

For a repository containing both kinds, select both target types in one `repository_scope`. For combined source/installation/endpoint evidence or a batch spanning inputs, use explicit `targets` after collection/inventory, preserving each association and permission. Reject local paths and `installation_scope` at the hosted boundary. These additions use the same API, lifecycle, policy and report contracts; they do not create four agents or four scanner services.

Collection and parsing dominate many batches, so snapshot a repository once, share an immutable file index, and run common dependency/plugin checks once with affected-target links. Schedule remaining target checks with the existing bounded pools. Keep installed instance-specific configuration checks separate. Cache only reusable observations under content, collector/rule and relevant configuration versions; endpoint/auth-context observations need separate freshness and scope handling. Twenty skills produce twenty target records within a batch, not twenty clones, model agents or Runtime sessions. Deep mode sends bounded finding/evidence groups rather than entire repositories to Bedrock.

### AgentCore Runtime API and client interfaces

For hosted execution, use the managed [InvokeAgentRuntime API](https://docs.aws.amazon.com/bedrock-agentcore/latest/APIReference/API_InvokeAgentRuntime.html). The application interprets a validated operation envelope; these are proposed application operations, not additional native AgentCore REST routes:

| Operation | Contract |
| --- | --- |
| `prepare_evidence_upload` | Authorise an owner-scoped staging object and short-lived upload grant bound to declared size/digest/retention |
| `complete_evidence_upload` | Validate owner, uploaded object integrity and bounded bundle structure; return an immutable evidence reference or reject |
| `start_assessment` | Authorise the manifest, bind an idempotency key to the caller and input digest, persist the assessment, then start bounded work and return its ID |
| `get_assessment_status` | Authorise access and return durable phase, per-target progress, and errors |
| `get_assessment_report` | Authorise access and return published baseline/deep results with lifecycle and advisory status, or explicitly report that no report is available yet |
| `cancel_assessment` | Persist a cooperative cancellation request; assessment tasks check it at bounded intervals |
| `resume_assessment` | Authorise a new attempt for interrupted work, preserving the approved manifest, evidence, and version constraints |

Include a schema version, operation, and validated operation-specific arguments. A start request contains mode, evidence references, permitted discovery and an idempotency key, plus either explicitly typed targets or an authorised repository scope. For repository scope, require an immutable snapshot reference, permitted target types/path selectors and discovery limits. Counts remain pending until bounded inventory produces durable stable target IDs and their evidence scopes; reuse that frozen inventory on resume. This is source-package discovery, separate from permission to connect to an MCP endpoint. Route resolved targets using validated `target_type`; the model cannot select arbitrary skills or enlarge the approved scope. The [HLD API contract](high-level-design.md#6-api-contract) defines both submission forms; exact selector fields remain to be formalised.

Upload bytes directly to the authorised S3 object; never send archives through model context. Staged data cannot be assessed before completion validation. Bind grants to an exact object/owner/expiry/checksum, enforce content/size limits, and expire abandoned uploads. Completion retries reconcile the existing record. Evidence references are re-authorised on every submission. These are ingestion operations in the same service; they do not import a trusted local verdict. Optional `remediation.generate_patch` is rejected outside deep source assessments and is subject to server-side patch policy.

Completion binds the evidence reference to the validated S3 VersionId/digest or promotes validated bytes to a service-owned immutable key outside the upload grant. Do not read a mutable staging key after validation. Return explicit ingestion state/errors and withhold usable evidence references until bounded validation completes.

Do not accept arbitrary shell commands, cloud roles, model profiles, or executable prompts as operation definitions. Return an explicit assessment state in the JSON payload; an accepted assessment is not a completed assessment. Native HTTP success can be 200 with an `accepted` application state. A separate REST facade could later expose `/v1/assessments`, but is not required for API delivery.

Implement the Runtime HTTP contract through the AgentCore SDK: `/invocations` for requests and `/ping` for health. Corporate JWT is the HLD's proposed default, subject to identity-provider support; IAM/SigV4 remains an alternative. The selected Runtime configuration uses one authentication mode. AWS SDK invocation fits IAM; JWT callers use authenticated HTTPS. Integrate the organisation's identity system and authorise every assessment, target, and report operation. Derive ownership from verified identity or a trusted identity propagation layer, never request-body owner fields or a session ID. Verify identity propagation as part of deployment; transport authentication alone does not provide per-report ownership. See [HTTP contract](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-http-protocol-contract.html) and [inbound authentication](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html).

The optional remote client skill collects scope and mode, invokes a packaged API helper, and presents the report. Keep credentials outside skill files. Hosted source inputs identify an uploaded immutable snapshot or an explicitly approved repository revision; the hosted agent cannot read a caller's local path. Both hosted analysis modes use the same agent. Configure Claude Code's own inference through approved [Bedrock settings](https://code.claude.com/docs/en/amazon-bedrock).

### Local direct execution

Install the reviewed marketplace scanner plugin and a compatible, pinned shared engine package in an approved local environment before assessment. The host adapter exposes the engine's typed CLI/tool contract; plugin instructions alone do not supply the executable engine. Select local execution explicitly and validate the plugin, engine, rule and configuration release identities. Missing or incompatible components fail clearly; do not silently switch to the hosted API or download executable dependencies during a scan.

The local assistant invokes the engine against an authorised local snapshot or explicitly permitted repository retrieval. Capture a stable evidence snapshot and hashes before evaluating checks so concurrent edits cannot produce an unidentified mixture of file versions. Local source access does not authorise endpoint connections. Protected MCP discovery uses separate, least-privilege target credentials, not the assistant's model credentials or the hosted API token.

Store manifests, evidence references, phase/attempt records and reports in an approved local location with access controls, retention and secret redaction. Local execution requires neither S3 nor a hosted database/Data API, and does not automatically submit evidence or reports to AgentCore or upload them elsewhere. Define local cancellation, interrupted-run reporting and explicit resume using those run records; do not imply cloud lease or recovery guarantees. Local deep mode uses the same remediation adapter with approved AWS credentials and inference profile. Both the assistant's own Bedrock orchestration and deep remediation can send selected content to Bedrock; local execution is not synonymous with offline execution.

Start the coding assistant in a neutral trusted workspace and pass the target snapshot as an evidence path. Disable automatic target-project instruction, plugin, hook and MCP loading, including `AGENTS.md`, `CLAUDE.md` and `.mcp.json`, through the chosen host adapter. Loading the trusted scanner plugin is permitted; activating the package under assessment is not. Local assessment does not run target scripts, hooks, builds or local stdio MCP servers. Local filesystem, environment credentials, sandbox and network permissions differ from AgentCore and require their own approved configuration; the two paths do not provide equivalent isolation.

### Skill packaging and agent integration

Use one canonical `security-assessment` plugin containing `mcp-assessment` and `skill-assessment`, distributed through the existing corporate marketplace. Each skill supplies its own procedure and references and invokes the common engine through a validated CLI/tool adapter. Approved local assistants install that scanner plugin and a compatible pinned engine package. The hosted build includes an approved, pinned plugin release in the application image. The optional `assessment-client` skill lives in a separate marketplace plugin for remote API access; exclude it from hosted skill loading to prevent recursive submission. The [HLD packaging and release flow](high-level-design.md#10-deployment-and-operations) defines source ownership, compatibility checks, pinned release identities and the optional future hosted startup-fetch alternative.

A plugin groups versioned skills and supporting resources; it is not another deployed agent or container. The runtime framework must explicitly load the two trusted skills. Strands is a proposed option: its `AgentSkills` integration loads skill instructions, while the application supplies resource-access tools. Alternatively, the Claude Agent SDK can load a Claude Code plugin explicitly. A Claude `.claude-plugin/plugin.json` manifest is not automatically interpreted by AgentCore or by Strands. Choose and pin the framework adapter during implementation. See [Strands skills](https://strandsagents.com/docs/user-guide/concepts/plugins/skills/) and [Claude Agent SDK plugins](https://code.claude.com/docs/en/agent-sdk/plugins).

In a local compatible coding assistant, use the canonical scanner plugin for direct execution, or select the separate remote client for hosted execution. The two roles must be explicit; installing the local scanner does not require using the hosted API. The separate scanner-plugin repository is canonical for skill source and the marketplace distributes its approved releases. The Security Agent repository owns the executable engine and emits both a local package and hosted image. Share versioned API/tool contracts and avoid duplicate engine implementations or a second editable copy of scanner skills. Select one hosted framework for v1; the local assistant adapter remains a separate supported interface. Pin host adapters and plugin/engine versions, and verify that they do not activate unapproved plugin components. If a Claude plugin format is used, its manifest and namespacing serve compatible hosts; a selected Strands adapter explicitly registers only the two skill directories. See [Claude Code skills](https://code.claude.com/docs/en/skills), [plugin packaging](https://code.claude.com/docs/en/plugins), and the [Agent Skills format](https://agentskills.io/specification).

Keep skills concise; load reviewed control excerpts and detailed references as needed. Record skill and instruction versions with each assessment. Separate trusted scanner skills from target evidence paths. Do not load target skills, plugins, hooks, `AGENTS.md`, `CLAUDE.md` or MCP settings as active host configuration. Static inventory of these files is permitted evidence collection. Referenced target scripts are evidence and must not execute. Plugin hooks and unrestricted shell tools are not required for the scanner.

### Sessions, batches, and durable execution

For hosted execution, use one bounded assessment attempt per Runtime session initially, with a fresh session ID scoped to the authenticated owner. One repository batch can contain multiple MCP or skill targets. A batch of 20 items does not automatically create 20 Runtime environments. On the proposed microVM compute type, new session IDs receive separate execution environments; calls to the same active session reuse its environment. Keep assessment ID, attempt ID, Runtime session ID, and SDK task ID distinct. Local batches use engine run/attempt identifiers and local checkpoints without creating Runtime sessions. See [Runtime sessions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-sessions.html).

Choose application concurrency explicitly. As in the HLD, benchmark one and two CPU-heavy workers first, then tune I/O discovery and Bedrock concurrency independently for the selected environment; these are benchmark cases, not promised defaults. Set separate bounds for parsing, discovery, Bedrock calls, and total in-flight assessments. Reuse repository-wide observations while preserving per-target evidence and coverage. Internal tasks/subagents share the environment; separate Runtime sessions are an explicit deployment and scheduling choice. One security agent means one maintained agent application, not one globally shared session for all users.

For hosted asynchronous work, register and complete SDK background tasks, keep health responses responsive, and report busy status while work continues. Do not block the invocation/health loop with scanner work. Busy status prevents idle expiry; it does not override maximum lifetime or recover lost work. See [AWS asynchronous processing guidance](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-long-run.html).

For the hosted service, persist owner, canonical input digest, pinned versions, state, per-target progress, artifact hashes, current attempt, lease expiry, and cancellation in Amazon Aurora PostgreSQL, accessed through the RDS Data API over HTTPS; store evidence and reports in private encrypted S3. Commit acceptance in a short explicit Data API transaction before acknowledging submission. Enforce an owner-scoped unique idempotency key, store the canonical input digest with it, return the existing assessment for matching reuse, and reject a changed digest under the same key.

Acquire and renew leases through short PostgreSQL transactions, using database time and updates guarded by the expected record version, current attempt token, state and lease expiry as applicable. Use explicit Data API begin/commit/rollback operations and the returned transaction ID for statements belonging to the same atomic change. Guard result publication and cancellation similarly so an expired or superseded attempt cannot replace current results. Write immutable S3 artifacts before transactionally publishing their references; S3 writes and database commits are not one atomic operation, so unreferenced artifacts require bounded cleanup. Do not hold a transaction or database lock while collecting evidence, running checks or awaiting Bedrock. After an uncertain commit, verify persisted state using the original idempotency key or attempt identity before repeating a transition.

Reuse the AWS SDK's HTTPS client and bound Data API request concurrency, retries and statement/result sizes. The application does not manage a PostgreSQL connection pool; Data API manages database connections, while cluster capacity and service quotas still limit throughput. Use approved IAM permissions for the configured cluster and database credential secret in Secrets Manager, with a least-privilege database role; keep secret values and transaction IDs out of model context and reports. Confirm Data API support in the selected Aurora region/version and choose provisioned capacity or Serverless v2 during deployment design. [Aurora Data API](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/data-api.html).

Preserve completed phases; mark missing work as interrupted or partial. An authenticated `resume_assessment` starts a new attempt after lease expiry with bounded retries. Reports remain available independently of session memory. The local storage adapter preserves applicable manifest, phase, attempt and artifact records in approved local storage, with local locking/atomic publication; it does not depend on Aurora, the RDS Data API or S3. The [HLD lifecycle design](high-level-design.md#9-lifecycle-resilience-and-concurrency) defines the hosted persistence boundary.

V1 recovery is explicit in either path: interrupted work is reported and the caller can request resume after authorisation and checkpoint validation. For hosted work, automatic recovery requires a separately deployed reconciler that detects stale or accepted-but-not-started assessments and invokes a new Runtime attempt. A scheduled AWS function can provide that control-plane role without moving hosted assessment execution out of AgentCore. SQS is optional for hosted admission/backpressure; neither a queue nor SDK background-task tracking alone guarantees recovery. Local process termination likewise does not imply automatic restart. Retries may repeat discovery or inference, so do not promise exactly-once execution. Bound deadlines to the selected environment and checkpoint before exhausting budgets.

### Runtime implementation and security boundary

Proposed shared components are Python, Pydantic contracts, registered deterministic checks, an MCP SDK adapter, an approved Bedrock remediation adapter, and local/hosted storage adapters. The hosted state adapter uses the AWS SDK RDS Data API client for Amazon Aurora PostgreSQL, with versioned schema migrations; its artifact adapter uses S3. Keep the engine independent of the agent framework. Package a pinned local wheel/CLI and the hosted Runtime image from the same engine source. The hosted layer adds the AgentCore SDK and selected agent framework; compatible local hosts use an explicitly configured plugin/tool adapter and do not require the AgentCore serving layer or cloud database. Pin dependencies, skills and rules in both distributions, publish approved local packages internally and hosted images in ECR, and use infrastructure as code for Runtime configuration. A FastAPI service, Kubernetes deployment, MCP service adapter, or browser UI is not required to expose the native invocation API.

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

Deep mode does not expand discovery permissions, invoke MCP tools or automatically increase detection coverage. It improves advice using the same evidence and can generate proposed source edits when explicitly requested. Runtime does not modify the target; only the separately opted-in GHA delivery workflow may create a remediation branch/PR. Behavioural testing can be added later as a separate capability with an explicit scope.

Neither the local assistant, hosted agent, remote client host, nor remediation model may change authoritative deterministic findings, severity, exceptions, or policy decisions. Publish the deterministic baseline before starting deep remediation: in the hosted path commit its authorised report references and results after S3 upload; locally publish them atomically. That baseline remains retrievable while advice runs and after an advice failure or interruption. A failed or invalid remediation response marks advice incomplete and produces lifecycle state `Partial`, unless cancellation or compute interruption determines the terminal state. Lifecycle state, check outcomes and the corporate policy verdict are separate fields, as defined in the [HLD lifecycle](high-level-design.md#9-lifecycle-resilience-and-concurrency).

### Inference boundaries

MCP is a communication protocol. The bounded MCP SDK client can perform permitted discovery through scripted requests without invoking a model. The engine enforces discovery scope and limits independently of agent inference.

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

The [HLD application and plugin layout](high-level-design.md#4-application-plugin-and-skill-packaging) defines the deployment paths and engine modules. Agent and scanner-plugin code have separate source repositories; the marketplace distributes plugin releases:

```text
Security Agent application repository
├── src/security_agent/             # Shared engine, local CLI and hosted adapters
├── packaging/                     # Proposed local wheel/CLI distribution
├── deployment/                    # Hosted container, IaC and release lock data
├── integrations/github/           # Trusted workflow templates and delivery helpers
├── control-mappings/              # Schemas and sanitised examples
├── schemas/                       # Assessment, finding, remediation contracts
├── assets/                        # Report templates
└── tests/                         # Fixtures and expected results

Separate scanner-plugin source repository
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

The marketplace catalogues approved releases from the scanner-plugin repository. The hosted image build and approved local installation consume the selected immutable scanner bundle; neither creates a second editable skill source. Pin agent and plugin revisions independently and record their tested combination in the release manifest. The application build emits both a hosted image and a compatible local engine package from the same engine code. Keep scanner `SKILL.md` files focused on target-specific procedures and references, and the optional remote client focused on API invocation and report retrieval. Provision authorised, versioned corporate-control snapshots through the appropriate local or hosted configuration adapter; keep credentials out of packages. Reuse executable checks across both scanners and execution paths, and enforce scope and mode selection in code. This structure is a proposal, not a list of existing files; the separate plugin repository's location, local package delivery and marketplace configuration remain to be confirmed.

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
- Reject resource reads, prompt retrieval and tool execution in v1; a future testing capability would require separate explicit permissions and contracts.
- Validate destinations, redirects, DNS resolution, and discovered authorization URLs before connecting.
- Permit explicitly configured internal destinations while rejecting unintended destinations.
- Bound pages, bytes, parsing depth, schema resolution, and total execution time.
- Reject target package installation and local MCP process startup in v1.
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

### GitHub Actions remediation delivery

Add an optional source-remediation delivery path to the same assessment API. The dispatch parameter is `create_remediation_pr`, default `false`. With it disabled, both analysis modes remain report-only. With it enabled, require deep mode, the approved workflow repository/base branch and complete source evidence pinned to the resolved base commit. Reject endpoint-only or installation-only requests and unsupported fork/cross-repository writes. Source ownership and API entitlement still require verification; the input flag alone is not a privilege grant. Initially create at most one draft PR per assessment/base commit, grouping compatible fixes across selected MCP/skill packages.

Illustrative workflow input fragment only; a runnable workflow and jobs have not been implemented:

```yaml
on:
  workflow_dispatch:
    inputs:
      mode:
        description: Assessment analysis mode
        type: choice
        options: [fast, deep]
        default: fast
        required: true
      target_types:
        description: Repository package types to assess
        type: choice
        options: [skill, mcp, both]
        default: both
        required: true
      base_branch:
        description: Approved branch to assess and target with the PR
        type: string
        default: main
        required: true
      create_remediation_pr:
        description: Generate fixes and open a draft PR; requires deep mode
        type: boolean
        default: false
        required: true
```

Use the typed `inputs.create_remediation_pr` boolean in conditions, validate enum/ref inputs, and pass inputs through structured helper arguments rather than interpolating them into shell code. The workflow definition/helper comes from a trusted approved revision, while `base_branch` selects only evidence and PR destination. The organisation's protected workflow/ref policy must prevent an arbitrary dispatch ref from obtaining write credentials. [GitHub dispatch input types](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#onworkflow_dispatchinputs).

**Assessment job:** resolve and snapshot the base commit once, submit a repository-scoped assessment, and set `remediation.generate_patch` from the authorised dispatch choice. Use a stable idempotency key derived from repository, workflow run, commit and normalised inputs; retain it across retries of that run. Poll the shared status/report operations with bounded backoff. A GHA wait timeout need not mean the remote scan stopped: retain the assessment ID and explicitly resume polling or cancel according to workflow policy. Save the report and patch references before evaluating policy exit codes so findings are available even when the scan detects violations. No repository write token is required for this job.

Persist a run manifest containing resolved base SHA, evidence references, canonical inputs, idempotency key and assessment ID as they become available. A new `run_attempt` for the same workflow run reloads these values instead of resolving a possibly moved branch. Use the stable ingestion/submission keys to reconcile a lost response. If the manifest is unavailable, stop for reconciliation; use a new logical run to assess a changed revision.

**Patch generation in Runtime:** the tool-less Bedrock remediation adapter returns structured edits, not shell commands. The proposed patch contract includes schema version, assessment ID, repository identity, base commit, finding/evidence IDs, relative file paths, expected before-content hashes, replacement content, rationale, assumptions and limitations. Application code validates references and budgets and publishes an immutable patch artifact/digest with `patch_status`. Limit v1 to bounded edits of existing text files in approved source/skill paths. Reject binary changes, renames, deletions, symlink/submodule paths, executable-mode changes, missing original bytes and edits outside scope. Redacted source that prevents exact patching yields `manual_only`. Do not invent missing secret values.

Apply trusted patch policy independently of the model. Exclude workflow/actions, scanner rules/configuration, control mappings, policy/permission files and secret stores from automatic edits by default. Fixes cannot suppress rules, lower severity, hide findings or weaken permissions to make a scan pass. Relevant protected-file findings still receive manual guidance. Rule-linked patch classes, maximum files/bytes and required coverage are approved configuration; confidence wording from the model cannot bypass them. Suggestions may remain useful when required evidence is missing, but incomplete required coverage blocks PR publication.

**Validation job:** use a fresh disposable checkout of the exact assessed commit, separate from the trusted workflow/helper. Download only the authorised patch artifact, verify digest and before-content hashes, reject paths that escape the checkout, and apply edits deterministically without fuzzy patching. Run the same pinned engine and applicable bounded static parsers against the candidate snapshot, with target configuration/hook loading disabled. Record addressed rules, new findings, coverage and a new snapshot digest. Require configured patch-validation gates and no unacceptable regressions; preserve both original and candidate results. Static success does not prove functional correctness. Do not run model-suggested commands, target builds/tests, package installs, local target actions or Git hooks in this job.

**Publishing job:** consume only the validated candidate/patch identity from the expected run, recheck base SHA and permitted repository, and create a bot-owned branch. A separate job gets the minimum repository content/PR write credentials; disable persisted checkout credentials and keep them out of artifacts, models and Runtime. Use a repository `GITHUB_TOKEN` only where its permissions and repository/organisation settings permit PR creation; otherwise use an approved narrowly scoped GitHub App installation token. Prefer short-lived credentials; future cross-repository operation needs separately designed authorisation. Use a stable branch plus PR marker tied to assessment/base/patch digest, reconcile uncertain responses, and never force-push over human edits or silently reopen a closed PR. A changed base yields `stale_base` and requires assessment of the new base commit followed by patch regeneration and validation, rather than an automatic rebase.

The PR is a review proposal: include findings/control IDs, change rationale, actual static validation, remaining manual findings and test limitations. Link to restricted reports instead of embedding sensitive evidence. Do not merge or edit the base branch. Report functional tests as pending/unavailable unless separately approved isolated CI ran them. Draft PRs can still trigger or queue CI; enable publication only for repositories with an approved CI policy for generated branches and their token's event behaviour. Keep target code away from privileged jobs, prevent recursive bot runs, and explicitly handle required workflows that need manual approval or another approved trigger. [GitHub PR creation](https://docs.github.com/en/rest/pulls/pulls#create-a-pull-request), [workflow-trigger behaviour](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow#triggering-a-workflow-from-a-workflow).

Use the selected machine authentication for Security Agent. AWS OIDC role credentials suit an IAM-configured Runtime, while the proposed corporate-JWT deployment requires its approved machine JWT flow. Do not assume a GitHub OIDC token or AWS credentials are accepted by that JWT endpoint. Keep assessment API, AWS and GitHub write credentials separate. [OIDC with AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws).

`assessment_state` and `policy_result` describe assessment execution/security outcomes. `patch_status` describes the requested artifact (`not_requested`, `generated`, `no_changes`, `manual_only`, `failed`); it never marks original findings fixed. A failed requested generation produces an incomplete advisory deliverable and `Partial` unless cancellation/interruption governs; an honest `no_changes` or `manual_only` outcome can be complete. GHA owns independent `delivery_status`, such as `not_requested`, `created`, `existing_pr`, `no_changes`, `manual_only`, `blocked_policy`, `stale_base`, `validation_failed` or `permission_denied`. A delivery error fails the requested delivery job while retaining the assessment/report; a no-change/manual-only result creates no empty PR. Record assessment/patch IDs, branch/PR URL and failure reason in the durable workflow summary/artifacts without exposing credentials. A failed security policy verdict must not itself prevent the optional remediation job from processing an otherwise complete, eligible assessment.

## 9. Delivery milestones and open decisions

Follow the [implementation plan](implementation-plan.md) for execution order, parallel workstreams, acceptance gates and the first implementation backlog. The milestones below summarise scanner coverage and integration requirements.

1. Define repository/endpoint/installation selectors, portable bundle and typed MCP/skill evidence, finding, coverage, and mapping schemas using representative corporate controls.
2. Implement the shared engine and separate rule packs with fixtures and a small set of high-confidence checks for each target type.
3. Add bounded, explicitly permitted MCP discovery and inert skill/plugin source inspection with honest coverage reporting.
4. Author the two scanner skills in one trusted marketplace plugin, implement compatible local and hosted tool adapters, and enforce the pipeline outside model instructions.
5. Package the shared engine for approved local installation and the hosted image; add approved Bedrock orchestration and shared deep remediation, preserving deterministic results.
6. Deploy the security agent on AgentCore Runtime with authenticated typed invocation, ownership checks, durable status/artifacts, bounded batch execution, cancellation, and explicit interruption recovery.
7. Support direct use of the scanner plugin in compatible local assistants, with pinned engine installation, neutral workspaces and approved local records; optionally deliver the separate remote client plugin against the API.
8. Pilot local and hosted execution against known benign and vulnerable fixtures, including source-only, endpoint-only, combined MCP evidence, skill packages, and mixed repositories.
9. Add the trusted GHA API client, optional structured patch output, isolated static validation and opt-in draft-PR publishing; validate credentials, base-commit binding, retries and repository CI policy before enabling write-back.

Acceptance tests should verify that the deterministic engine makes no model calls; fast workflows in either path cannot invoke the dedicated remediation stage; local/hosted/client orchestration uses approved inference configuration and preserves script-produced findings; discovery never becomes target tool execution or local stdio startup; malformed, hostile, or inaccessible targets yield bounded results; missing evidence remains visible; and deep remediation preserves deterministic findings. Target-side inference cannot be ruled out from endpoint discovery alone.

Include external MCP repository fixtures and internal MCP endpoint-only fixtures. Verify that evidence availability controls technical check selection, and that an unverified or mismatched source revision cannot produce an unsupported deployment-level pass.

Verify equivalent deterministic findings and control mappings across local direct execution, remote client and direct API for the same captured evidence, engine/plugin/rule/catalogue/mapping versions and assessment context, excluding run metadata. Live observations, model suggestions and environment-dependent coverage may differ and must be reported. Test local/hosted selection independently of fast/deep mode, target-type selection, unknown skill names, incompatible releases, required phase completion, authorised evidence references and report access. Ordinary API consumers must work without a local Claude Code installation; local direct assessment must work without AgentCore invocation, Aurora, the RDS Data API or S3.

For local execution, verify no automatic evidence/report upload, local access and retention controls, stable snapshots during concurrent edits, bounded workers, interruption/cancellation, local locking and explicit resume. Exercise both valid and unavailable Bedrock credentials for local deep mode. Confirm a local run does not silently become a remote submission, and that target permissions remain separate from model/API credentials. Test the local host from a neutral workspace with a malicious target `AGENTS.md`, `CLAUDE.md`, `.mcp.json`, hooks and plugin files, proving these are not automatically loaded or activated.

For hosted execution, test concurrent duplicate submissions and idempotency-key conflicts against PostgreSQL uniqueness constraints, lost responses, accepted-but-not-started work, session termination, stale leases, explicit resume, cancellation, hung checks, and partial deep-mode failure. Verify Data API transaction-ID handling, rollback, transaction expiry, throttling, request limits, uncertain commit reconciliation, database/API unavailability and guarded publication after lease expiry or cancellation. A failed or lost Data API response must not turn incomplete work into success. Verify per-batch and inference concurrency limits in both environments. Exercise a 20-target batch locally and remotely without assuming one process/session per skill. Confirm interrupted attempts cannot overwrite newer results in either storage adapter.

Add skill fixtures containing malicious instructions, misleading `allowed-tools`, escaping paths, bundled download-and-execute scripts, plugin hooks, and linked MCP configuration. Confirm the scanner never installs or activates target skills/plugins, reads target repository instructions as trusted policy, or invokes its own client skill recursively. Test the trusted scanner plugin itself before promotion and pin the release under test.

Cover all four intake scenarios and their combinations. For installed targets, test supported/unsupported host versions, user/project/managed precedence, disabled/shadowed registrations, cached versus active packages, concurrent updates, unavailable binaries/source, symlink boundaries and credential redaction. Verify clean scanner processes never activate user-wide target hooks/MCPs and export/upload remains explicit. Test bounded ingestion, cross-owner evidence references, tampered bundles and workstation-only endpoints. For optional GHA delivery, test false/fast/endpoint-only input combinations, exact before hashes, protected paths, malicious edits, incomplete coverage, failed static validation, empty/manual-only changes, stale base branches, uncertain push/PR results, existing human commits and GitHub/API permission failures. Confirm reports survive delivery failures and required downstream CI is neither assumed to run nor reported as passed.

Decisions still needed: corporate catalogue format and sample controls; supported MCP revisions/transports and skill/plugin dialects; discovery allowlists and credentials; severity and mandatory-unknown policy; local/hosted evidence retention/redaction; internal vulnerability dataset; approved Bedrock profile/model/regions; hosted framework and approved local host adapters; plugin/local engine/optional client distribution and compatibility; API authentication and verified identity propagation; Aurora region/version, provisioned or Serverless v2 capacity, Data API availability/limits and secret management; local filesystem/sandbox and cloud isolation requirements; measured batch/runtime budgets; whether automatic recovery is required beyond explicit resume; and reassessment cadence.

The Git repository should initially contain design, schemas, reviewed rules, and synthetic fixtures. Runtime evidence, credentials, generated sensitive reports, and unapproved corporate material require explicit handling rules before they are committed.
