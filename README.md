# Azure Data Factory to Microsoft Fabric — Transition Information Pack

**Audience:** Data & Analytics teams currently using Azure Data Factory (ADF)
**Purpose:** Provide a vendor-neutral, generic overview of the current Azure Data Factory (ADF) platform, how it compares with Data Factory in Microsoft Fabric (FDF), and the practical pros, cons, challenges, and recommended path for organisations considering a transition.

> This pack is written for a typical ADF customer. It describes common architectural patterns and decision points rather than any single organisation's environment. Use it as a starting framework and tailor the assessment, PoC, and cutover criteria to your own estate.

---

## 1. Current State — Azure Data Factory Today

Azure Data Factory is widely used as the **data movement and orchestration layer** of a modern data platform, sitting between source systems and the downstream processing/analytics estate.

### 1.1 How ADF is commonly used

| Capability | Typical usage |
|---|---|
| **Copy / ingestion** | Bulk and incremental copy from a wide range of source systems into cloud storage such as Azure Data Lake Storage (ADLS). |
| **ETL / transformation** | Pipeline-based ETL orchestration; transformation work is often performed downstream (e.g. Spark/Databricks, SQL, or Delta Lake processing layers). |
| **Source connectivity** | A significant proportion of sources are frequently **bespoke / line-of-business systems behind a corporate firewall**, accessed via Self-Hosted Integration Runtime (SHIR). |
| **Processing layer** | Notebooks/jobs (e.g. Spark/Databricks) operating on the curated data — the analytical "engine" of the platform. |
| **Orchestration** | ADF triggers and pipelines coordinate ingestion, then hand off to the processing layer. |

### 1.2 Strengths of a mature ADF estate

- Mature, well-understood platform with broad connector coverage.
- **Self-Hosted Integration Runtime (SHIR)** is well-suited to high-volume on-premises extraction and is battle-tested for bespoke systems.
- Clear separation of concerns: ADF moves and orchestrates; a dedicated engine transforms.
- Strong ALM story via Azure DevOps / Git integration and ARM-based deployment.
- Supports advanced networking (Managed VNet, private endpoints) where required for regulated data sources.

### 1.3 The strategic question

Microsoft is investing heavily in Microsoft Fabric, and within it, **Data Factory in Fabric (FDF)**. This naturally raises the question for any ADF customer:

> *"Should we keep investing in ADF, build new in FDF, or pursue a hybrid path?"*

The remainder of this document is intended to support that decision.

---

## 2. ADF vs. Data Factory in Fabric — Comparison

### 2.1 At a glance

| Dimension | Azure Data Factory (ADF) | Data Factory in Fabric (FDF) |
|---|---|---|
| **Service model** | PaaS, deployed per-factory in Azure | SaaS, consumed from a Fabric capacity (F SKU) |
| **Pipelines** | Mature, full activity catalogue | ~90% activity parity and growing; new SaaS-style activities (Teams, Outlook, semantic model refresh) |
| **Code-free transformation** | Mapping Data Flows (Spark) | **Dataflow Gen2** (Power Query + Spark engine) |
| **Connectivity abstraction** | Linked Services + Datasets | Simplified **Connections** (no Datasets concept) |
| **Cloud runtime** | Azure Integration Runtime (managed by you) | Managed by Fabric — no IR provisioning |
| **On-premises connectivity** | **Self-Hosted Integration Runtime (SHIR)** | **On-Premises Data Gateway (OPDG)** — the same gateway used by Power BI |
| **Triggers** | Schedule, Tumbling Window, Storage/Custom events | Schedule + Activator/Reflex events (no tumbling window today) |
| **Authoring workflow** | Author → Publish → Trigger | Author → Save → Run (no separate publish step) |
| **CI/CD** | Azure DevOps / GitHub + ARM templates | Native **Fabric deployment pipelines** + Git |
| **Monitoring** | ADF monitoring blade + Log Analytics | Unified **Fabric Monitoring Hub** |
| **AI assistance** | Limited | **Copilot** for pipeline and Dataflow Gen2 authoring |
| **Storage gravity** | Works with ADLS, Synapse, SQL, third-party | OneLake-centric; native integration with Lakehouse, Warehouse, Power BI |
| **Cost model** | Per-activity + DIU-hours + IR compute | Consumed from a **shared Fabric capacity** (F SKU) |
| **Governance** | Azure RBAC, Purview integration | Fabric workspace RBAC, sensitivity labels, Purview |

### 2.2 Where FDF clearly improves on ADF

- **Lower operational overhead** — no Integration Runtimes to size or patch for cloud-only workloads.
- **Faster authoring loop** — Save & Run, with Copilot assistance.
- **Unified analytics surface** — pipelines, lakehouse, warehouse, and BI live in the same workspace.
- **Native CI/CD** without ARM scaffolding.
- **OneLake shortcuts** allow Fabric to read existing ADLS / Delta data **in place**, without physical migration.

### 2.3 Where ADF currently retains an edge

- **Self-Hosted Integration Runtime** for high-volume on-premises extraction.
- **Tumbling-window triggers** and richer event-trigger options.
- **Managed VNet + private endpoint** patterns are more mature in ADF.
- **SSIS lift-and-shift** via SSIS-IR (FDF does not support this).
- **Customer-managed keys** and certain advanced security controls have richer parity in ADF today.
- **Predictable per-pipeline isolation** — FDF runs on a shared Fabric capacity that other workloads (Spark, BI) also consume.

### 2.4 Roadmap signal from Microsoft

- **ADF**: in steady-state — fully supported, but the majority of net-new investment is going into Fabric.
- **FDF**: actively evolving — expect ongoing improvements in connectors, triggers, gateway throughput, governance, and Copilot capabilities.
- A **"Mount existing ADF in Fabric"** capability already exists, allowing Fabric to surface and orchestrate existing ADF factories — explicitly designed to support a hybrid migration.
- Microsoft provides a **PowerShell migration module** (`Microsoft.FabricPipelineUpgrade`) that converts ADF pipeline JSON into Fabric pipeline definitions, accelerating bulk migration of straightforward pipelines.

---

## 3. Practical Pros, Cons, and Challenges

This section applies to the common pattern of **ADF used primarily for ingestion/orchestration, a separate processing/semantic layer (e.g. Spark/Databricks + Delta Lake), and a significant share of sources behind a firewall.** Adjust the weightings to match your own architecture.

### 3.1 Pros of moving ingestion to FDF

- **Connector parity** — the ADF connector catalogue is largely available in FDF, so most existing extractions are technically reproducible.
- **Reduced infrastructure management** — no cloud Integration Runtimes to maintain.
- **Modern authoring experience** — Copilot, simplified Connections, no publish step.
- **Native CI/CD and unified monitoring** reduce DevOps friction.
- **Optionality for the future** — even without adopting OneLake/Power BI day one, you position the platform to consume those capabilities later via shortcuts.
- **Single capacity-based bill** if/when Fabric becomes the broader platform.

### 3.2 Cons / where ADF currently serves you better

- **On-premises throughput**: OPDG is less proven than SHIR for very high-volume bulk ETL; per-node throughput and parallelism patterns differ.
- **Networking maturity**: ADF's Managed VNet + private endpoint story is currently richer.
- **Trigger gaps**: tumbling-window and certain event triggers used in mature ADF frameworks may need redesign.
- **Metadata-driven frameworks**: any "config-table-drives-N-pipelines" patterns will need refactoring because FDF removes the Datasets abstraction.
- **Processing-engine integration**: invoking external engines (e.g. Databricks) is functional but less first-class than in ADF — Microsoft is naturally promoting Fabric Spark/Lakehouse, and engine-specific features (e.g. Unity Catalog auth, job clusters, parameterisation) may lag slightly.
- **Capacity contention**: a busy Spark or BI workload on the same Fabric capacity can throttle pipelines, unlike ADF's PaaS isolation.
- **Cost shape**: ADF's per-use model can be cheaper for **bursty, small-volume** ingestion workloads than a reserved Fabric capacity.

### 3.3 Key challenges to plan for

1. **On-premises connectivity (highest-impact item)**
   - Move from SHIR to **On-Premises Data Gateway** clusters.
   - Validate throughput against your largest extracts; plan for HA and multi-node scaling.
   - Confirm support for any custom drivers (ODBC/JDBC/SAP) used by bespoke systems — some specialised drivers remain SHIR-only today.

2. **Processing / semantic layer**
   - Continue to invoke existing processing jobs (e.g. Databricks) from FDF pipelines; validate authentication, secret scopes, and cluster/compute policies in a PoC.
   - Keep curated data (e.g. Delta Lake in ADLS) as the source of truth; expose it to Fabric via **OneLake shortcuts** rather than physically migrating data.

3. **Refactoring metadata-driven pipelines**
   - The Microsoft `FabricPipelineUpgrade` tool handles simple pipelines well; complex parameterised frameworks will require targeted redesign around Connections + parameters.

4. **Networking, security and compliance review**
   - Re-validate the network path: OPDG outbound to Fabric service endpoints, identity model (workspace identity vs managed identity), key management, and data residency for the chosen Fabric region.
   - Especially important for regulated or sensitive data.

5. **Operational maturity**
   - ADF's monitoring, alerting, and Log Analytics integration are mature and well-understood by most operations teams.
   - Fabric Monitoring Hub and Activator-based alerting are improving rapidly but represent a new operational surface to learn and standardise on.

6. **Cost modelling**
   - Build a side-by-side comparison of current ADF DIU + SHIR VM costs against the smallest Fabric F SKU that meets throughput needs.
   - For a *pure ingestion* workload, the break-even rarely favours Fabric on its own — the business case strengthens significantly when OneLake, Warehouse, or Power BI are also consumed from the same capacity.

### 3.4 Recommended approach

For the common pattern — **ADF is "just the mover", a separate engine is the "brain", and many sources are on-premises** — the recommended path is **hybrid and phased**:

1. **Stabilise & assess** — inventory existing ADF pipelines, IRs, triggers, and dependencies; identify any features that currently lack FDF parity.
2. **Stand up a Fabric landing zone** — a non-production Fabric capacity, a workspace with appropriate governance, and an OPDG cluster connected to a representative bespoke source.
3. **Run a focused PoC (4–6 weeks)** — migrate one representative ingestion flow end-to-end (on-prem source → cloud storage/Delta → processing job → monitoring/alerting). This will surface OPDG throughput, processing-engine integration ergonomics, and operational gaps quickly.
4. **Adopt the hybrid bridge** — use the **Mount ADF in Fabric** capability so new ingestion can be authored in FDF while existing ADF pipelines continue to run unchanged.
5. **Migrate opportunistically** — start with simple Copy-heavy pipelines using the PowerShell upgrade tool; redesign complex frameworks as part of the move.
6. **Define cutover triggers** — e.g., OPDG meets throughput SLAs, Managed VNet/private endpoint parity lands, or a separate business case emerges to adopt OneLake / Power BI on Fabric.

> **Bottom line:** FDF is a strategic fit and the clear long-term direction, but it is **not yet an obvious tactical win** for an ingestion-only ADF workload feeding a separate processing layer. A hybrid approach lets you capture FDF's modern authoring and CI/CD benefits on new work, protect existing investments, and align fully with Fabric when the platform-level business case (OneLake, Power BI consolidation) is ready.

---

## 4. Suggested next steps

- Confirm the in-scope ADF pipelines and source systems for an inventory.
- Agree on a representative source for a PoC (ideally one bespoke, firewalled system).
- Gather the supporting references you need: Fabric capacity sizing guidance, OPDG sizing reference, and the latest FDF roadmap snapshot.
- Schedule a follow-up workshop to review PoC outcomes and define cutover triggers.

---

## References

- Compare Data Factory in Fabric vs Azure Data Factory — `learn.microsoft.com/fabric/data-factory/compare-fabric-data-factory-and-azure-data-factory`
- Migration planning for ADF to Fabric Data Factory — `learn.microsoft.com/fabric/data-factory/migrate-planning-azure-data-factory`
- Microsoft Fabric public roadmap (Data Factory) — `roadmap.fabric.microsoft.com/?product=datafactory`
- What's new in Fabric Data Factory — `blog.fabric.microsoft.com`

---

*This document is a generic information pack intended to support internal decision-making for organisations using Azure Data Factory. Roadmap items are directional and subject to change.*
