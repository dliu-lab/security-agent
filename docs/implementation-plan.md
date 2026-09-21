# Security Agent — Implementation Plan

**Status:** proposed work breakdown; no scanner, plugin, API, infrastructure or workflow has been implemented.

**Design baseline:** [HLD v0.7](high-level-design.md) and [assessment design](assessment-design.md), including the four intake paths and optional GHA remediation PR. The HLD remains the architecture authority; this document defines delivery order and acceptance gates.

**Selected hosted framework:** Strands Agents SDK, with AgentCore SDK Runtime integration. Load only the two trusted scanner skills through an explicit Strands `AgentSkills` configuration. Framework selection is settled; compatible dependency versions and loader behavior still require validation. The shared engine and local CLI remain independent of Strands.

Build the shared deterministic engine first and prove it through a local CLI. Add collectors and the trusted scanner plugin around that contract, then deploy the same engine through AgentCore. Deliver GHA reporting before enabling proposed patches and draft PRs. Tests accompany each component; the final pilot validates their integration and operating limits.

## 1. Delivery sequence

Step numbers describe an implementation sequence, not a requirement to finish every preceding step before starting the next. Explicit dependencies identify parallel work. Suggested owners are workstreams, not assigned people.

| Step | Deliverable | Depends on | Suggested owner |
| --- | --- | --- | --- |
| 1 | Repository foundations and decisions needed to begin | Existing design | Engineering, security and platform |
| 2 | Versioned contracts and synthetic fixture corpus | 1 | Engine and API engineering |
| 3 | Repository collector and frozen target inventory | 2 | Engine engineering |
| 4 | Deterministic checks, mappings, reports and working local CLI | 2, 3 | Engine and security engineering |
| 5 | Bounded MCP endpoint collector | 2, 4 | MCP integration engineering |
| 6 | Local CLI installation collector and portable export | 2, 4 | Local integration engineering |
| 7 | Bedrock remediation, trusted scanner plugin and local release | 4; relevant collectors before claiming their coverage | Agent and plugin engineering |
| 8 | Aurora state, S3 artifacts and validated ingestion | 2; storage interfaces from 4 | Platform and API engineering |
| 9 | Hosted AgentCore agent/API with explicit recovery | 4, 7, 8; integrate 5 and 6 for all intake paths | Agent and API engineering |
| 10 | GHA API client and report-only workflow | 9 | Developer platform |
| 11 | Optional patch generation, validation and draft PR | 7, 10 | Security and developer platform |
| 12 | Pilot, performance budgets and production release | 5–11 | All workstreams |

Steps 5, 6 and 8 can progress independently once their interfaces are stable. Step 7 can start with source assessments while the other collectors are built. Platform feasibility work starts in step 1; production configuration is completed before step 9. Lack of an AWS account or a corporate catalogue need not block a synthetic local prototype, but it does block the corresponding production acceptance gates.

## 2. Implementation steps and acceptance gates

### Step 1 — Establish repositories and resolve immediate decisions

Keep executable code, CLI, API, infrastructure and GHA integration in this application repository. Establish the separate scanner-plugin repository and its ownership; the existing marketplace distributes its releases. Agree private storage for corporate catalogue/mapping artifacts. Do not copy an editable plugin into the application repository.

Create the proposed Python package structure from the HLD, development dependency lock, build configuration and CI for formatting, linting, type checks, tests and package builds. Add dependency and release compatibility metadata. Use the same engine source for the local package and hosted image.

Record short architecture decisions for supported source languages/manifest patterns, skill dialects and local host versions. Run a bounded feasibility exercise to validate the pinned Strands/AgentCore SDK combination and trusted-skill loader, and verify the intended AWS region, Bedrock profile, Runtime identity/network path and Aurora Data API configuration. Select the corporate JWT machine integration or the IAM alternative; do not assume AWS credentials authenticate to a JWT-configured Runtime. Select the organisation's IaC and internal artifact-distribution tooling without adding a second deployment platform.

**Done when:** package CI builds an empty installable application; source ownership and compatibility strategy are documented; feasibility results identify which deployment inputs are available and which remain blockers. No cloud capability is declared supported solely because a mock passed.

### Step 2 — Define contracts before implementing adapters

Create typed models and generated JSON Schemas under `schemas/` for assessment requests/context, input selectors, frozen inventory, evidence/bundles, check results, findings, coverage, control mappings, reports, remediation and proposed patches. Define canonical serialization/digests, stable identities, schema compatibility and structured errors. Keep these dimensions distinct:

- `input_kind`: repository, MCP endpoint or CLI installation.
- `target_type`: skill or MCP; one input may yield many targets.
- Evidence set: source, installed bytes, configuration and endpoint observations, with provenance and correspondence.
- Backend and mode: local/hosted and fast/deep.
- Execution state, check state, policy result, patch status and GHA delivery status.

Formalise all seven hosted operations: `prepare_evidence_upload`, `complete_evidence_upload`, `start_assessment`, `get_assessment_status`, `get_assessment_report`, `cancel_assessment` and `resume_assessment`. Specify request limits, owner-derived authorization, idempotency, evidence references and the `targets`/`repository_scope` selector union. Local submission additionally supports `installation_scope`. Reserve and validate `remediation.generate_patch`; reject unsupported feature requests until that capability is implemented.

Build synthetic benign/vulnerable fixtures for individual skills, MCP packages, a mixed repository, a 20-skill repository, malformed archives, hostile instruction files, installed configurations and captured endpoint responses. Store expected findings and explicit unknown/error coverage. Use clearly labelled fixture-only controls until real reviewed mappings are available; never invent corporate control IDs.

**Done when:** valid examples round-trip through schemas; invalid selectors, arbitrary commands/roles/model profiles, body-supplied ownership and hosted workstation paths are rejected. Every fixture has an expected evidence scope and result, including limitations.

### Step 3 — Implement repository collection and target inventory

Build `engine/collectors/` and `engine/inventory/` around an immutable captured snapshot. Start with supplied local directories/archives; add approved read-only repository retrieval pinned to a revision. Record uncommitted content, file hashes, capture errors, exclusions and limits. Bound files, bytes, path depth, archive expansion, memory and elapsed time. Do not execute repository hooks, filters, builds or package installation; do not fetch referenced submodules/LFS objects or follow external paths without approved scope.

Discover selected `SKILL.md` packages and their relevant plugin/support files. Identify MCP source targets through explicit package paths and supported manifest/code patterns; preserve ambiguous candidates rather than silently claiming complete discovery. Build one shared file/dependency index and assign stable per-package target IDs. Link shared files to all affected targets and freeze the inventory for later resume.

**Done when:** a mixed repository is captured once and yields the expected skill/MCP inventory; a 20-skill fixture does not cause 20 clones; symlink/path escapes, concurrent edits and truncated collection produce bounded, explicit results. No target instructions, hooks or scripts activate.

### Step 4 — Deliver the first working local scanner

Implement the required pipeline in `engine/pipeline.py`: collection, inventory, checks, reviewed mappings/policy and reporting. Define a rule registry with applicability, required evidence, versions, severity/confidence basis, limitations, maintained remediation and verification guidance. Add shared, skill and MCP rule modules; select rules from evidence and permissions, independently of internal/external ownership.

Begin with a small reviewed rule set: secret indicators with redaction; dependency pinning/provenance; missing or escaping references; unsafe execution patterns in supported languages; skill capability/instruction indicators; and MCP schema/configuration issues. Heuristic indicators must retain their limitations. Implement `pass`, `fail`, `unknown`, `not_applicable`, `error` and `skipped`; missing implementation evidence cannot become a pass.

Add versioned many-to-many corporate mappings, applicability, exceptions and policy evaluation. Unmapped findings and mandatory unknowns remain visible. The corporate catalogue is authoritative; external categories are supplementary labels. Block a real corporate compliance verdict when required reviewed catalogue/policy inputs are missing.

Implement the local storage adapter and typed CLI entry points for assessment, status/report access, cancellation and explicit resume. Write `findings.json`, `coverage.json` and a readable report with evidence references, pinned versions and per-target limits. Use access-controlled run directories, atomic baseline publication, local locking/checkpoints and bounded workers. Resume validates the saved snapshot, pinned versions and current permissions; older interrupted attempts cannot overwrite a resumed attempt's output. Do not auto-upload results.

**Done when:** a standalone local fast scan completes against one mixed fixture and then the 20-skill batch, with expected findings, mappings and coverage. Identical captured inputs/versions yield equivalent deterministic results. The engine makes no model call, survives an interrupted run with honest state, rejects invalid resume/older-attempt publication, and completes with AgentCore/S3/database access denied. The initial technical prototype may use clearly labelled synthetic controls.

This is the first useful end-to-end milestone. It proves the central assessment contract before any assistant or cloud integration.

### Step 5 — Add endpoint assessment through the MCP SDK

Implement the `mcp_endpoint` collector using a pinned SDK and explicitly supported protocol revisions/transports. Resolve approved destinations and separately scoped credentials. Enforce a discovery/listing allowlist; reject tool execution, resource reads, prompt retrieval, target startup and unsupported server-to-client requests, including sampling and elicitation. Do not automatically connect to endpoints discovered in source/configuration.

Apply time/page/byte/depth/concurrency budgets, destination and redirect/DNS validation, TLS policy and secret redaction. Record observed transport/authentication metadata, advertised schemas, collection time/location, credential-view identity and partial failures. Keep source/deployment correspondence explicit and implementation-only checks unknown when evidence is absent.

**Done when:** approved synthetic HTTP servers cover successful, denied, paginated, malformed and hanging discovery; out-of-scope destinations and methods are rejected. Test servers are controlled fixtures started by the test harness, not target processes launched by the scanner. The same captured responses produce equivalent checks locally and through the eventual hosted adapter.

### Step 6 — Add installed CLI collection and export

Implement a versioned Claude Code reader under `engine/collectors/hosts/`. Read only selected user/project/managed/plugin records and approved package roots. Record configuration provenance, declared enablement, shadowed registrations, unresolved variables, actual installed bytes and unknown effective state for unsupported versions. Deduplicate parsing without merging distinct configured instances.

Read launch commands and helpers as data. Do not start an assistant, MCP process, package manager or target module to inspect it. Use a standalone collector or validated clean scanner profile; a neutral project directory alone does not disable user-wide components. Installed stdio MCPs remain static-only. Any existing HTTP endpoint requires its own permission and reachable collection location.

Add portable bundle export with hashes, target associations, redactions/exclusions, capture provenance and limits. Filter sensitive configuration before export; omit credential values, OAuth caches, full environments and session transcripts. Capture concurrent updates safely. Export remains explicit, and the collector does not interpret workstation loopback as a hosted endpoint.

**Done when:** fixtures cover installed skills, a mixed plugin, disabled/shadowed entries, cached older versions, missing source and unsupported hosts. Collection activates nothing and changes no user configuration. Re-imported bundles retain provenance/coverage gaps, and no upload happens during a local scan. Hosted upload is integrated after step 8.

### Step 7 — Add Bedrock advice and release the scanner plugin locally

Implement a framework-independent remediation adapter using the approved Bedrock inference profile. Send only bounded selected findings, relevant control excerpts and redacted evidence. The remediation model has no tools. Validate output schemas and every finding/evidence reference; preserve the deterministic baseline before advice starts. Failed or invalid advice marks the advisory stage incomplete and the lifecycle `Partial`, unless cancellation/interruption governs. Enforce deadlines, retry/token budgets and model/prompt metadata. Fast mode cannot invoke this adapter.

In the separate plugin repository, author `security-assessment` with only the trusted `mcp-assessment` and `skill-assessment` workflows and reviewed support resources. Both invoke the same high-level engine contract. Publish a pinned marketplace release and a compatible local engine package; record their independent revisions/digests. Keep the optional hosted-client plugin separate and defer it until the API exists.

Implement the local assistant adapter and hosted Strands `AgentSkills` loader against that same bundle. Validate that target content cannot register skills/tools or alter the manifest, checks, severity or mappings. Configure approved host Bedrock inference for both modes. A plain CLI fast scan remains independent of host inference.

**Done when:** local plugin fast/deep scans return the expected deterministic baseline; denied Bedrock access, malformed advice and injection fixtures cannot overwrite it, and advice failures produce the required incomplete/partial state. Setup detects incompatible plugin/engine versions and does not silently invoke AgentCore. Explicit loader inventory contains only the two approved scanner skills. Source support can ship internally first; advertise endpoint/installed support only after steps 5 and 6 pass.

### Step 8 — Build hosted persistence and evidence ingestion

Implement versioned migrations and the Aurora PostgreSQL storage adapter through RDS Data API. Model ownership, assessments, attempts, targets, finding/coverage relationships, artifact references, upload records and pinned versions. Add owner-scoped unique idempotency keys/input digests, session bindings, fencing tokens, leases and guarded transitions. Acceptance and initial-attempt creation commit together. Keep SQL parameterised and transactions short; reconcile uncertain commits before repeating writes.

Implement the shared verified-principal/authorization adapter from the selected identity design alongside storage and ingestion handlers. Step 9 wires it into Runtime and completes the hosted authentication acceptance gate. Never derive trusted ownership from request-body fields.

Build S3 evidence/checkpoint/report adapters and the two ingestion operation handlers using that authorization contract. Bind upload grants to owner, exact staging object, expiry, size and checksum; validate bounded archive structure before making evidence usable. Bind accepted evidence to a validated object version/digest or promote bytes outside the upload grant. Publish report references only after artifact upload and a guarded database commit. Implement expiry/orphan cleanup and coordinated retention.

Use a migration identity separate from the runtime database identity. Provision the approved private endpoints, encryption, secrets and IAM through IaC; keep cluster/secret/model selection in deployment configuration. AWS documents Data API as an HTTPS SQL interface without application-managed persistent database connections; validate the selected region/version and transaction/service limits in an actual development environment. [Aurora Data API](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/data-api.html).

**Done when:** concurrent duplicate requests converge, conflicting digests fail, cross-owner access fails, and stale/cancelled attempts cannot publish. Validate transaction expiry, throttling, uncertain commits and database unavailability against the real Data API. Upload tampering, mutable staging objects and oversized/escaping archives cannot become canonical evidence. Mocks and local PostgreSQL tests alone do not complete this gate.

### Step 9 — Deploy the agent and typed API on AgentCore

Compose the shared engine, Strands adapter, pinned two-skill bundle and rule/configuration identities into an immutable hosted image. Expose the seven application operations through the native invocation contract using the AgentCore SDK. Integrate the shared authentication/verified-ownership adapter, target entitlement, session binding, admission and durable acceptance before loading agent context or invoking models. Status/report/cancel operations use structured code without model calls.

Background work uses a bounded durable authorization record and service identity, never a persisted caller JWT. Check grant expiry/revocation at phase boundaries; interrupt expired work until reauthorized. Resume and report access verify current caller rights independently of session or assessment IDs.

Run one bounded assessment attempt per Runtime session initially. Track background activity through the SDK while keeping health responsive. Persist phase/target progress, renew/fence leases, handle cancellation and implement explicit resume with reauthorization and the same approved inventory/snapshot/versions. Detect accepted-but-not-started attempts. SDK task tracking reports activity; it does not replace durable recovery. [AgentCore asynchronous processing](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-long-run.html).

Integrate fast assessments first and prove recovery. Then enable the previously tested deep adapter, preserving baseline access during and after advice failures. Integrate endpoint discovery and uploaded installation bundles under the same permission and evidence rules. Reject raw workstation paths and hosted `installation_scope` requests.

**Done when:** an ordinary authenticated API client can submit, poll, retrieve, cancel and resume assessments without Claude Code. Kill sessions before work, during checks and during publication; verify new authorised control invocations can recover status/reports and stale attempts cannot overwrite results. Run all four intake scenarios and compare deterministic local/hosted results for equivalent captured evidence/context. Cloud acceptance also requires validated real catalogue policy, loader behavior, credentials and network configuration.

### Step 10 — Deliver the GHA report-only workflow

Implement a trusted workflow template/helper under `integrations/github/` with `workflow_dispatch` inputs `mode`, `target_types`, `base_branch` and `create_remediation_pr`. Keep the last input a boolean defaulting to `false`; while step 11 is unavailable, reject `true` explicitly. GitHub supports typed dispatch inputs and job-specific permissions. [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#onworkflow_dispatchinputs).

Resolve the approved base once, capture/upload its immutable snapshot and invoke the API. Persist the run manifest, evidence references, idempotency keys and assessment ID across retries and `run_attempt` changes. Use bounded polling and a documented timeout/cancel policy. Save restricted report references and a summary before applying policy-based workflow exit codes. Use the selected machine API identity and no repository write token.

**Done when:** a source-only MCP, skill and mixed batch each produces a report through GHA. Lost responses and workflow reruns reuse the original revision/assessment. Polling timeout does not falsely mark remote work stopped. Security findings can fail the policy gate without losing the report, and all report-only paths make no repository writes.

### Step 11 — Add optional source fixes and draft PR delivery

Implement the step-2 patch contract and server-side `remediation.generate_patch` gate: deep mode, explicit request, complete matching source and approved patch classes/paths. Generate finding-linked structured edits through the tool-less Bedrock adapter, with source identity, exact base commit, before-content hashes and a patch digest. Validate bounded edits of existing text files; reject protected workflow/policy/scanner files, escaping/symlink/submodule paths, binary/mode changes, absent/redacted original bytes and rule suppression. Return `generated`, `no_changes`, `manual_only` or `failed` explicitly.

Build a separate GHA validation job using a disposable exact-base checkout and trusted helpers. Verify the expected-run artifact identity, apply edits without fuzzy matching or model-provided commands, and rerun the pinned deterministic engine/static parsers. Retain original and candidate results. Require sufficient coverage, addressed-rule validation and no unacceptable regressions; do not execute target scripts, hooks, installs or tests in these jobs.

Enable a separate publishing job only when `create_remediation_pr: true`, the request is eligible and validation passed. Give only that job scoped repository content/PR write access. Recheck base SHA and bot branch/PR state, then push the validated change and create/reconcile one draft PR per assessment/base. Preserve human commits and closed-PR decisions. If the base changed, assess the new revision before regenerating fixes. Keep original assessment/policy state separate from patch and delivery status; report no-change/manual-only outcomes without empty PRs.

Arrange job dependencies so a failed security policy verdict does not skip eligible remediation. For example, let assessment emit execution/policy outcomes separately, run validation/publication only on eligible completed execution, then apply the overall policy exit in a final job. Assessment errors, cancellation and incomplete required coverage still block publication.

**Done when:** opt-out performs no writes; fast/endpoint-only/incomplete-source requests cannot publish; protected or invalid edits fail; retries do not duplicate PRs; human edits are not overwritten; and GitHub failures preserve findings. A completed assessment with a failed policy verdict can still reach eligible PR delivery. Validate repository CI trigger/isolation policy before enabling write-back, since draft status does not prevent downstream execution. No automatic merge occurs.

### Step 12 — Pilot and release the complete capability

Run benign/vulnerable and adversarial fixtures across all four scenarios, local/hosted backends and fast/deep modes. Measure collector latency, parsing/check time, report publication, model tokens/cost, polling/database load and recovery behavior separately. Exercise 20-target batches first, then the HLD's larger benchmark cases as capacity permits; set supported limits and admission budgets from results rather than promised worker counts.

Complete dependency/provenance review, release compatibility checks, IAM/secret denial and rotation tests, logging/redaction, retention/orphan cleanup, Aurora/S3 restore and complete-release rollback. Publish operating guidance for partial/interrupted reports, explicit resume, mandatory unknowns, unsupported targets and stale PRs. Promote the same tested engine/plugin/rule/configuration combination from development to production.

**Done when:** security and platform owners accept measured budgets, real corporate mappings, access boundaries, recovery/restore evidence and the supported capability matrix. Release local packages, the marketplace plugin, hosted image/API and GHA templates with compatible pinned identities. Keep the optional PR capability disabled until its repository-specific prerequisites pass.

## 3. What each milestone demonstrates

| Milestone | Required steps | Demonstration |
| --- | --- | --- |
| Local source prototype | 1–4 | One mixed repository and a 20-skill batch produce deterministic findings, coverage and reports without cloud/model dependencies |
| Local assessment capability | 5–7 plus prototype | Source, endpoint and installed-target paths work through approved local CLI/plugin configuration, with optional deep advice |
| Hosted assessment capability | 8–9 plus relevant collectors/plugin | Same evidence contracts and rules through authenticated API, durable records, cancellation and resume |
| GHA reporting | 10 | A manually dispatched repository assessment returns usable reports with reliable retry identity |
| Optional remediation PR | 11 | Explicit deep-mode opt-in produces validated source changes and a draft PR, or a clear no-PR reason |
| Production release | 12 | Supported workloads, policies, operating limits, restore and release procedures are verified |

These are delivery checkpoints, not independent products. A prototype milestone does not claim the complete v1 scope is finished.

## 4. Decisions and inputs needed by each gate

| Input/decision | Needed before | Work that can proceed while pending |
| --- | --- | --- |
| Representative corporate catalogue, reviewed mappings, severity and mandatory-unknown policy | Real corporate policy verdicts in step 4; hosted pilot | Synthetic fixtures, schemas and rule mechanics |
| Supported source languages, skill dialects and file-selection rules | Collector/check acceptance in steps 3–4 | Generic bounded snapshot handling |
| Supported MCP revisions/transports, target credentials and allowlists | Real endpoint validation in step 5 | Captured response fixtures and permission enforcement |
| Supported Claude Code versions/OS and clean scanner profile | Local installation/plugin acceptance in steps 6–7 | Host-reader interfaces and synthetic records |
| Scanner-plugin repository location, marketplace publication and internal engine distribution | Reproducible local/hosted packaging in step 7 | Engine implementation and plugin source planning |
| Approved Strands/AgentCore SDK versions, loader configuration and Bedrock profile/model/regions | Real inference in step 7 and image integration in step 9 | Tool contracts and simulated adapter failures |
| AWS account/region, Aurora configuration, IaC, private connectivity, identity and retention | Real storage/invocation gates in steps 8–9 | Migrations, lifecycle logic, mocks and local tests |
| GHA machine authentication, trusted workflow ref and approved repository/base refs | Workflow integration in step 10 | Typed client/helper and retry tests |
| Allowed patch classes, GitHub write credentials and downstream CI policy | Enabling PR delivery in step 11 | Patch validators and synthetic proposal fixtures |
| Numeric reliability, performance and cost targets | Production sign-off in step 12 | Benchmark instrumentation and prototype measurements |

## 5. First implementation backlog

Start with three small, reviewable application-repository changes. Each may become one or more PRs depending on size; these are proposed work items, not PRs already created.

| Order | Change | Completion evidence |
| --- | --- | --- |
| 1 | Package skeleton, CI, schema models, rule/storage interfaces and synthetic fixture conventions | Installable wheel; contract validation; invalid-input fixtures; explicit synthetic-control labelling |
| 2 | Bounded local snapshot collector, archive handling, mixed target inventory and shared file index | Stable inventory/hashes; scope and concurrent-edit cases; no target activation |
| 3 | First reviewed rules, mapping/policy adapter, JSON/readable reporting and local fast CLI | Known findings/unknowns on mixed and 20-skill fixtures; no model/cloud calls; reproducible baseline and interruption records |

Then parallelise endpoint, installed-host and hosted-storage work against those interfaces. Keep caching beyond shared parsing, distributed scan scheduling, automatic recovery, additional host/framework adapters, a browser UI and cross-repository PR creation outside the initial implementation unless measurements or a new requirement justify them.
