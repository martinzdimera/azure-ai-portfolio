# Martin Ždimera — Azure & AI Delivery Portfolio

Azure platform architect for regulated environments. This repository holds sanitized public editions of documents from my work.

The architecture documents are full references, with system views, operating mechanisms, validation evidence and limitations; case sheets record particular deliveries and their outcomes. The summaries below explain what each item is and what its documents contain, so readers can choose which reference to open. Sections move from Azure platform architecture and internal project tooling to the delivery framework and its products, followed by the research apparatus and operational references.

### Azure platform architecture

**MCP-Azure Gateway (design)** — MCP-Azure Gateway is an architecture design for an MCP (Model Context Protocol) gateway through which company AI assistants access an organization's internal systems. The design places Azure Front Door Premium with Private Link in front of Azure Container Apps, uses Microsoft Entra ID as the authorization server, Cosmos DB for the skill catalog, Managed Redis for cache, session and rate-limit data, and OpenTelemetry with Application Insights for observability. The documentation consists of 13 text documents, 22 diagrams (Mermaid source with SVG rendering) and 3 machine-readable contracts (one OpenAPI file and two JSON Schemas), and includes 23 architecture decision records (ADR-001 to ADR-023). According to its README, the documentation is version 1.0, dated 2026-09-17, status "Draft" ("Návrh"), ready for comments and approval, not for production deployment without further review. The design has not been implemented. The documents are in Czech.

Documents: [Design documentation (Czech)](mcp-azure-gateway-design/00-README.md) · [Diagram atlas (HTML — download the folder and open locally)](mcp-azure-gateway-design/index.html) · [ADRs](mcp-azure-gateway-design/09-adr-a-zdroje.md) · [Contracts](mcp-azure-gateway-design/kontrakty/README.md)

<details><summary>What the documentation covers</summary>

- Index — This index of the documentation set.
- Business context and use cases — Non-technical context: purpose, use cases, benefits, stakeholders, shadow-IT consequence.
- Task analysis and plan — Analysis of the brief, scope, assumptions, and work plan.
- Solution architecture — Context, containers, Azure topology, network flow.
- Code architecture — Gateway and ingestor components, modules, boundaries, testability.
- Data architecture — Cosmos DB containers, entity schemas, ERD, retention policies.
- API and protocols — Gateway HTTP surface, MCP methods, error taxonomy, JSON request/response examples.
- State logic and workflows — State machines: catalog, connections, tool invocation, circuit breaker, rollout gates.
- Security and authorization — STRIDE threat model, OAuth 2.1 authentication, authorization model, secrets, compliance.
- Operations, observability and rollout — SLO/SLI, telemetry, alerting, runbooks, CI/CD, phased rollout, IT handover.
- ADRs and sources — All 23 architecture decision records, sources, open questions.
- Design comparison — Independent comparison with alternative design: agreements, differences, factual errors, recommendations.
- Design review — Second-view review of the finished design, seeking unnecessary complexity.

</details>

Reference architecture document of a production AKS platform — in preparation.

### Internal tooling used on real Azure projects

**UDF2** — An infrastructure-as-code configuration and automated deployment tool for Azure environments. It is an internal tool used on real large Azure projects, connecting infrastructure planning with repeatable deployment of hub-and-spoke landing zones. A configurator turns snapshots, landing-zone artifacts or design sheets into reconciled intent and a draft inventory. Planning performs no Azure writes; a human approves the draft before a bridge emits deployment configuration. The deployment pipeline then reads declarative inventory and provisions resources in dependency order. The reference documents architecture profiles, safety boundaries, deployment checks and a live end-to-end exercise that verified the source environment remained unchanged.

Documents: [Architecture reference](UDF2_ARCHITECTURE_v3.1-public.pdf)

<details><summary>What the document covers</summary>

- System Architecture (C4) — Context, components, deployment boundaries and runtime interactions.
- Layer 1: The Configurator — Intake, reconciliation, draft inventories and planning workbench.
- Layer 2: The Deploy Framework — Inventory contracts, dependency ordering and profile-aware deployment stages.
- The Bridge: emit-deploy-config — How planning artifacts become deployment configuration.
- Governance and Safety Model — Human approvals, sandbox boundaries, redaction and audit records.
- Testing and Verification — Configurator tests, negative safety checks and deployment verification.
- Architecture Decision Records — Reasons behind inventory, approval and planning contract choices.
- Limitations and Failure Modes — Known constraints, failure scenarios and the recovery model.

</details>

**AutoDoc** — An Azure infrastructure documentation and diagramming tool that generates workbooks and architecture views from collected inventory. This internal tool was used on real large Azure projects to keep documentation connected to the deployed infrastructure. It collects a snapshot, then generates workbooks and diagrams offline from that shared input. The diagram pipeline separates normalization, routing, layout and rendering, retaining intermediate artifacts for inspection and reruns. Workbook output includes resource inventories, compliance evaluations and collection warnings. The reference describes topology classification, architecture profiles, geometric verification and cross-stage consistency checks, with reproducibility bounded by the rendering toolchain and fonts.

Documents: [Architecture reference](AutoDoc_FW_ARCHITECTURE_v3-public.pdf)

<details><summary>What the document covers</summary>

- System Architecture (C4) — System context, processing components, deployment and runtime flows.
- The Four-Stage Diagram Pipeline — Normalization, edge classification, geometry and final rendering.
- Data Contracts — Snapshot structure, intermediate artifacts and lineage between stages.
- Key Design Patterns — Hub detection, routing corridors, configuration and controlled reruns.
- Architecture Profile Integration — Centralized inspection and distributed network topology handling.
- Testing and Verification — Snapshot tests, geometric checks and cross-stage consistency rules.
- Architecture Decision Records — Recorded choices for execution, rendering and verification boundaries.
- Limitations and Failure Modes — Unsupported cases, phase failures and safe rerun conditions.

</details>

### Delivery framework and what it has built

**Factory FW** — Control framework for analysing, designing and delivering software products with AI agents. A human approves the design at four gates; delivery then runs as an explicit state machine in which separate agents write, check and independently review each stage, with all state recorded in one file. Its purpose is delivery quality without hands-on supervision: once the owner relaxes pre-flight and records an autonomy approval, stages run unattended within budget and iteration limits, and security-critical stages stay human-gated. Implemented as a Node CLI. FleetGuard is the measured pilot documented in the reference.

Documents: [As-built reference](FactoryFW_ASBUILT_v1-public.pdf)

<details><summary>What the document covers</summary>

- Two-layer architecture as-built — Design approvals, delivery transitions and shared lifecycle state.
- Safety and quality mechanisms — Scope checks, autonomy approval, recovery and structured verdicts.
- From design to implementation — Architecture changes and dispositions of pre-implementation review findings.
- Validation — Framework test records, process exercises and measured pilot evidence.
- Deployment model — Pinned framework installation, version records and retained project state.
- Lessons Learned — Operational lessons from delivery, review and framework hardening.

</details>

**FleetGuard** — A multi-tenant SaaS product, developed with Factory FW as the framework's first measured pilot. The product documents vehicle-fleet damage through an offline-capable web application. The case sheet describes a stalled AI-prototyped project taken over from its previous author and delivered with the agentic framework: ~5 weeks, 93 commits, 22 merged PRs, application code not written by hand; human approvals at every gate. It also covers verification (a 385-test suite, two production walkthrough rounds) and a pre-production readiness assessment of my own delivery.

Documents: [Case sheet v2.4](fleetguard-case-sheet-v2.4.pdf) · [Delivery mechanics](fleetguard-delivery-mechanics-v1.0.pdf)

<details><summary>What the document covers</summary>

- Case sheet: Context — Product purpose, inherited condition and recovered requirements.
- Case sheet: Approach — Design gates, staged delivery, independent review and resumable state.
- Case sheet: Delivery, measured from the repository — Implemented capabilities, development history and recorded human interventions.
- Case sheet: Verification, measured after deployment — Walkthroughs, authorization checks, test evidence and unresolved defects.
- Case sheet: Pre-production readiness assessment — Historical operational gaps, evidence requirements and hardening priorities.
- Case sheet: Why it matters — Delivery controls and requirements examined against recorded evidence.
- Delivery mechanics: Design phase G0–G3; Stage-by-stage account S1–S7 — Design approvals, stage outcomes, amendments and review evidence.
- Delivery mechanics: What broke and how it fed back; Numbers — Failures, corrections carried forward and repository measurements.

</details>

**pts (in development)** — A tool for work measurement, reporting and estimates. It is being built with Factory FW. A target-state document will be added; full documentation follows on completion.

### Research apparatus

**Deep-Research** — A research workflow with executable checks around AI-authored reports, claim verification and independent review. Its full profile combines preflight, twelve phases and three validation gates. Mandatory claim classes receive census verification, while ordinary assertions receive a deterministic sample. A fresh reviewer examines a bounded evidence bundle without the authoring conversation. SHA-256 digests bind report output to verification and review artifacts, and changes require affected checks to be repeated. The reference also inventories Python control modules and test files, distinguishing inspected test assertions from executed evidence; it reports no current test pass count.

Documents: [Technical reference v3.0](Deep-Research-apparatus-reference-v3.0-public-EN.pdf) · [English overview](Deep-Research-Martins_v2.2-public-EN.pdf) · [Czech overview](Deep-Research-Martins_v2.2-public.pdf)

<details><summary>What the document covers</summary>

- Architecture — Research artifacts, control components and boundaries between their responsibilities.
- Pipeline — Research phases, validation gates, restart points and delivery decisions.
- Source verification — Registration, accessibility, appraisal and evidence lineage as separate checks.
- Claim verification — Claim–source pairs, census coverage, deterministic sampling and resolution records.
- Blind adversarial review — Bounded inputs, reviewer independence, findings and confirmation rules.
- Integrity and attestation — Output digests, generated QA summaries and external reference boundaries.
- Implementation and test evidence — Inspected modules, test assertions and limits of execution evidence.
- Empirical evidence — The public run, its coverage indicator and the blocked finalization path.
- Limitations and lessons — Sampling limits, semantic uncertainty and constraints on review independence.

</details>

### Other references

**Hyper-V** — Recovery and follow-up case sheets about repairing virtual disk chains under restricted access to the affected hosts. The recovery separated read-only diagnosis from intervention, required verified copies before repair, and used independently reviewed scripts with explicit abort conditions. AI assisted with analysis and script preparation; the human approved hand-offs. Production machines were recovered with no recorded data loss. The follow-up describes a script built around pinned inspection facts, declarative probes, a single guarded write and returned evidence. It records safe stops when assumptions failed and leaves the remaining repair in progress at the document's reporting date.

Documents: [Recovery case sheet](hyperv-recovery-case-sheet-v1.1.pdf) · [Phase 2 case sheet](hyperv-phase2-case-sheet-v1.0.pdf)

<details><summary>What the document covers</summary>

- Recovery: The situation; The constraint — Broken disk chains and restricted execution access.
- Recovery: How it was done — Diagnosis, verified copies, reviewed scripts and abort criteria.
- Recovery: How AI assistance actually worked here — Assisted analysis and script preparation under human approval.
- Recovery: Result; The root cause, and what it was not — Recorded recovery and policy-driven failure of checkpoint merging.
- Phase 2: How the repair script is built — Pinned facts, probe tables, a single guarded write, evidence packaging.
- Phase 2: How it is validated before hand-off — Specification review, test vectors and rehearsed failure paths.
- Phase 2: The day the design was tested — Runtime mismatches, platform refusal and recorded safe stops.
- Phase 2: What comes next; Status — Successor repair approach and explicitly unfinished follow-up work.

</details>

### Principles behind all of it

Fail-closed controls (nothing passes silently), measured claims instead of adjectives, human design authority at every gate, independent cross-model review, and determinism where possible.

Versions, page counts and SHA-256 checksums for every file are in [MANIFEST.md](MANIFEST.md); each PDF document passes an automated pre-publication check (format, empty pages, sanitization, diagram rendering) before it is published here. Full internal versions and additional references are available on request.

**Contact:** martin@snehurka.net · [LinkedIn](https://www.linkedin.com/in/zdimera)

---

*Česky: Sanitizované veřejné edice referenčních dokumentů z mé praxe — architektura, mechanismy a měřené výsledky zachovány, klientské údaje odstraněny. Plné interní verze na vyžádání.*
