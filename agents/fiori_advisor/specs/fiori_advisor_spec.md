# Fiori Advisor — Agent Specification

Template: [agile-lab-dev/Agent-Specification](https://github.com/agile-lab-dev/Agent-Specification/blob/main/spec.md)

## Overview
**Fiori Advisor** is an agentic system that maps SAP **transaction codes (T-Codes)** to their
equivalent **SAP Fiori apps**, enriches each mapping with functional-module classification and
usage analytics, and delivers the result as a multi-sheet Excel workbook.

The agent accepts a list of T-Codes either pasted directly in the conversation or supplied as an
**Excel (`.xlsx`)** or **CSV (`.csv`)** file. It resolves each T-Code against the **SAP Fiori Apps
Reference Library** (its primary knowledge base), classifies the resulting app into a functional
hierarchy (Primary Module → Detailed Module → Submodule → Business Process), scores the
confidence of the mapping, and produces a workbook with a detailed mapping sheet, an executive
summary sheet, and a module-level roll-up sheet.

This specification aims to enable **consistent definition, governance, and observability** of the
Fiori Advisor agent in a structured format, following the Agent Specification standard above.

## Schema Structure
The agent descriptor follows an **OpenAPI 3.0-based** schema to enable easy documentation and
validation, consistent with the Agent Specification template.

## Key Attributes

### Identity & Ownership
- `id` *(string, required)* – `urn:agent:sap:fiori_advisor:1`
- `name` *(string, required)* – `Fiori Advisor`
- `fullyQualifiedName` *(string)* – `SAP T-Code to Fiori App Mapping Advisor`
- `description` *(string)* – Maps SAP T-Codes to Fiori apps, functional modules, and business
  processes, and exports the result as an annotated Excel workbook.
- `domain` *(string, required)* – `sap` (SAP S/4HANA transition & Fiori adoption)
- `version` *(string, required)* – `1.0.0`
- `environment` *(string)* – `production`
- `agentOwner` *(string)* – `sap-coe@yourcompany.com`

### Business Context & Goals
- `kind` *(string)* – `Single Agent`
- `agentGoal` *(string)* – "Given a list of SAP T-Codes, return a verified, module-classified
  mapping to the equivalent Fiori app(s), with confidence scoring and usage context, so SAP teams
  can plan Fiori adoption and S/4HANA transitions."
- `targetUser` *(string)* – `internal` (SAP functional/technical consultants, Basis/COE teams)
- `valueGeneration` *(array)*:
  - `ProcessAutomation` – automates a manual T-Code-to-Fiori research exercise
  - `InformationRetrieval` – retrieves authoritative data from the Fiori Apps Reference Library
  - `DecisionMaking` – supports Fiori rollout prioritization decisions
- `interactionMode` *(string)* – `RequestResponse`, upgraded to `HumanInTheLoop` whenever the
  input T-Code list or file cannot be determined automatically (the agent asks before guessing).
- `runMode` *(string)* – `Reactive` (triggered by a user request with pasted T-Codes or an
  uploaded file)
- `agencyLevel` *(string)* – `ModelDrivenWorkflow` (an LLM drives classification, confidence
  scoring, and sheet composition across a fixed set of tools)
- `toolsUse` *(string)* – `true`
- `learningCapability` *(string)* – `None`

### Technical Configuration
- `physicalEndpoint` *(object)*:
  - `apiUrl` – `local-cli://fiori-advisor` (invoked inside the aXet.code CLI session, no network
    endpoint of its own)
  - `authenticationMethod` – `N/A` (delegates authentication to the Fiori Apps Reference Library
    via SAP Passport when required)
  - `supportedMethods` – `["INVOKE"]`

### Dependencies & Components
- `dependencies` *(array)*:
  - SAP Fiori Apps Reference Library (external viewer)
  - Optional SAP usage-statistics extract (e.g. `ST03N`/transaction usage report) supplied by the
    user, used to populate `Usage Count`, `User Count`, `Last Used`
- `components` *(array)* – see **Agent Component Specification** below: Input Ingestion Tool,
  Fiori Library Lookup Tool, Module & Confidence Classifier (LLM), Workbook Export Tool, Evidence
  Guardrail, Mapping Workflow Orchestrator.

### Audit & Observability
- `audit` *(object)*:
  - `healthCheck` – `N/A` (stateless CLI invocation; health is the availability of the Fiori Apps
    Reference Library)
  - `decisionLogs` – each row's `Mapping Evidence / Rationale` column serves as the inline decision
    log for that mapping
  - `anomalyDetection` – rows with `Mapping Confidence = Validation Required` or `No Fiori App
    Found` surface automatically in the Summary and Module Summary sheets for review

### Knowledge Bases
- `knowledgeBases` *(array)*:
  - **SAP Fiori Apps Reference Library** *(primary)*
    - `sourceType` – `Structured`
    - `endpoint` – `https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/`
    - Provides: Fiori App ID, App Name, App Description, functional area/module tags, business
      catalog/role context used to derive the module hierarchy.
  - **SAP Transaction Usage Analytics** *(optional, secondary)*
    - `sourceType` – `Structured` (user-supplied extract, e.g. `ST03N` export, or an existing
      column in the input file)
    - `endpoint` – user-provided file path
    - Provides: `Usage Count`, `User Count`, `Last Used` when available; otherwise these fields
      are left blank/`N/A` rather than estimated.

### Security & Compliance
- `securityInfo` *(object)*:
  - `visibility` – `Internal`
  - `confidentiality` – `Medium` (T-Code usage data may reveal internal business processes)
  - `gdprSensitive` – `false`
  - No mapping is fabricated: every row not confirmed against the Fiori Apps Reference Library is
    explicitly labeled (`Candidate` / `Validation Required`) rather than presented as verified.

## Input & Output Contract

### Input
One of:
1. **Pasted T-Codes** — one or more SAP T-Codes typed directly in the request (free text, comma or
   newline separated).
2. **Uploaded file** — an `.xlsx` or `.csv` file containing one or more sheets/columns of T-Codes.
   Multiple sheets/columns are supported; each source location is preserved for traceability via
   `Source Sheet` and `Source Row Reference` in the output.

If no T-Code can be identified from either source, the agent asks the user for the T-Code list or
the correct file instead of guessing.

### Output — Workbook Structure
The deliverable is a single `.xlsx` workbook with three sheets:

#### Sheet 1 — `Fiori App Mapping` (detail sheet)
One row per T-Code (repeated if a T-Code maps to more than one Fiori app).

| Column | Description |
|---|---|
| T-Code | The SAP transaction code being mapped |
| T-Code Description | Short description of the T-Code's function |
| Fiori App ID | Identifier of the matched Fiori app (e.g. `F0862`) |
| Fiori App Name | Display name of the matched Fiori app |
| Fiori App Description | Description of the Fiori app's purpose |
| Primary Functional Module | Top-level SAP module (e.g. `Finance`, `Logistics`) |
| Detailed Functional Module | Sub-area within the primary module (e.g. `Accounts Payable`) |
| Functional Submodule | Finer-grained classification within the detailed module |
| Business Process | Business process the T-Code/app supports |
| Mapping Confidence | `Confirmed` \| `Candidate` \| `Validation Required` \| `No Fiori App Found` |
| Mapping Evidence / Rationale | Short justification citing the Fiori Apps Reference Library result (or why confidence is lower) |
| Usage Count | Number of recorded executions, if usage data is available |
| User Count | Number of distinct users, if usage data is available |
| Last Used | Most recent usage date, if usage data is available |
| Priority | Suggested Fiori-adoption priority (e.g. `High`/`Medium`/`Low`), derived from usage/confidence |
| Source Sheet | Name of the source sheet the T-Code was read from (or `Pasted` for typed input) |
| Source Row Reference | Row number/cell reference in the source file (or position in the pasted list) |

#### Sheet 2 — `Summary`
High-level roll-up of the run:

| Column | Description |
|---|---|
| Total T-Codes Processed | Count of distinct T-Codes submitted |
| Total Fiori Apps Mapped | Count of distinct Fiori App IDs returned |
| Confirmed Mappings | Count of rows with `Mapping Confidence = Confirmed` |
| Candidate Mappings | Count of rows with `Mapping Confidence = Candidate` |
| Validation Required | Count of rows with `Mapping Confidence = Validation Required` |
| No Fiori App Found | Count of T-Codes with no equivalent Fiori app |
| Source File(s) | Name(s) of the input file(s)/sheet(s) processed, or `Pasted input` |
| Generated On | Timestamp the workbook was generated |

#### Sheet 3 — `Module Summary`
Roll-up grouped by functional hierarchy:

| Column | Description |
|---|---|
| Primary Functional Module | Top-level SAP module |
| Detailed Functional Module | Sub-area within the primary module |
| Functional Submodule | Finer-grained classification |
| Business Process | Business process grouping |
| Mapped T-Code Count | Total T-Codes mapped under this grouping |
| Confirmed Count | T-Codes in this grouping with `Confirmed` mapping confidence |
| Candidate Count | T-Codes in this grouping with `Candidate` mapping confidence |
| Validation Required Count | T-Codes in this grouping with `Validation Required` mapping confidence |

## Example Agent Descriptor

```yaml
id: urn:agent:sap:fiori_advisor:1
name: Fiori Advisor
fullyQualifiedName: SAP T-Code to Fiori App Mapping Advisor
description: Maps SAP T-Codes to Fiori apps, functional modules, and business processes, exported as an annotated Excel workbook.
domain: sap
version: 1.0.0
environment: production
agentOwner: Sharath M Gangadhara
agentOwnerDisplayName: AXET Center of Excellence
email: mailto:sharath.gangadhara@nttdata.com
status: ACTIVE
kind: Single Agent
agentGoal: "Map a list of SAP T-Codes to verified, module-classified Fiori app equivalents with confidence scoring and usage context."
valueGeneration: ["ProcessAutomation", "InformationRetrieval", "DecisionMaking"]
interactionMode: "RequestResponse"
runMode: "Reactive"
agencyLevel: "ModelDrivenWorkflow"
toolsUse: true
learningCapability: "None"

physicalEndpoint:
  apiUrl: "local-cli://fiori-advisor"
  authenticationMethod: "N/A"
  supportedMethods: ["INVOKE"]

dependencies:
  - id: urn:data:sap:fiori_apps_library
    name: SAP Fiori Apps Reference Library
    description: External viewer providing authoritative T-Code-to-Fiori-app mappings.
  - id: urn:data:sap:usage_analytics
    name: SAP Transaction Usage Analytics
    description: Optional user-supplied usage extract (e.g. ST03N) for Usage Count/User Count/Last Used.

components:
  - id: urn:component:sap:tcode_input_tool:1
    name: T-Code Input Ingestion Tool
    kind: Tool
    version: 1.0.0
    description: Parses pasted T-Codes or reads them from an uploaded Excel/CSV file, preserving source sheet/row references.
  - id: urn:component:sap:fiori_library_lookup_tool:1
    name: Fiori Apps Reference Library Lookup Tool
    kind: Tool
    version: 1.0.0
    description: Queries the SAP Fiori Apps Reference Library external viewer for each T-Code.
  - id: urn:component:sap:module_classifier_llm:1
    name: Module & Confidence Classifier
    kind: LLM
    version: 1.0.0
    description: Classifies each matched Fiori app into Primary/Detailed Module, Submodule, and Business Process, and scores mapping confidence.
  - id: urn:component:sap:workbook_export_tool:1
    name: Workbook Export Tool
    kind: Tool
    version: 1.0.0
    description: Writes the Fiori App Mapping, Summary, and Module Summary sheets to an .xlsx workbook.
  - id: urn:component:sap:evidence_guardrail:1
    name: Evidence & Confidence Guardrail
    kind: Guardrail
    version: 1.0.0
    description: Blocks presenting an unverified mapping as Confirmed; forces explicit confidence labeling.
  - id: urn:component:sap:mapping_orchestrator:1
    name: Mapping Workflow Orchestrator
    kind: Orchestrator
    version: 1.0.0
    description: Sequences ingestion, lookup, classification, guardrail checks, and export.

audit:
  healthCheck: "N/A"
  decisionLogs: "inline:Fiori App Mapping!Mapping Evidence / Rationale"
  anomalyDetection: "inline:Module Summary!Validation Required Count"

knowledgeBases:
  - id: urn:kb:sap:fiori_apps_library
    name: SAP Fiori Apps Reference Library
    sourceType: "Structured"
    description: Authoritative catalog of Fiori apps, App IDs, descriptions, and functional/business context.
    endpoint: "https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/"
  - id: urn:kb:sap:usage_analytics
    name: SAP Transaction Usage Analytics
    sourceType: "Structured"
    description: Optional usage statistics (counts, users, last used) sourced from a user-supplied extract.
    endpoint: "user-provided-file"

securityInfo:
  visibility: "Internal"
  confidentiality: "Medium"
  gdprSensitive: false
```

# Agent Component Specification

## Overview
This section defines the attributes of the components used within the Fiori Advisor agentic
system, following the Agent Component Specification. Each component is modular and reusable
across the ingestion → lookup → classification → export workflow.

## Component Types

### 1️⃣ Tool — T-Code Input Ingestion Tool
Reads the T-Code list from either pasted text or an uploaded `.xlsx`/`.csv` file, deduplicates it
while preserving order, and records each T-Code's source sheet and row/position for traceability.

**Key Attributes:**
- Accepts free text or file input.
- Preserves `Source Sheet` / `Source Row Reference` metadata per T-Code.
- Falls back to asking the user when no T-Code can be identified.

```yaml
id: urn:component:sap:tcode_input_tool:1
name: T-Code Input Ingestion Tool
kind: Tool
version: 1.0.0
description: Parses pasted T-Codes or reads them from an uploaded Excel/CSV file.
inputFormat: "Text | XLSX | CSV"
outputFormat: "JSON (ordered, deduplicated T-Code list with source metadata)"
```

---

### 2️⃣ Tool — Fiori Apps Reference Library Lookup Tool
Queries the SAP Fiori Apps Reference Library external viewer for each T-Code to retrieve the
matching Fiori App ID, name, and description.

**Key Attributes:**
- Library-first, library-only policy: no cross-referencing of third-party sites unless the
  library is entirely unreachable.
- Returns `No Fiori App Found` explicitly when no match exists, instead of leaving ambiguity.

```yaml
id: urn:component:sap:fiori_library_lookup_tool:1
name: Fiori Apps Reference Library Lookup Tool
kind: Tool
version: 1.0.0
description: Queries the SAP Fiori Apps Reference Library for each T-Code.
apiEndpoint: "https://fioriappslibrary.hana.ondemand.com/sap/fix/externalViewer/"
inputFormat: "T-Code (string)"
outputFormat: "JSON (App ID, App Name, App Description, functional tags)"
```

---

### 3️⃣ LLM — Module & Confidence Classifier
Classifies each matched Fiori app into the functional hierarchy (Primary Functional Module →
Detailed Functional Module → Functional Submodule → Business Process), writes the mapping
rationale, and assigns a `Mapping Confidence` and `Priority`.

**Key Attributes:**
- Reasons over the library's returned metadata; never fabricates a module when evidence is
  insufficient — downgrades confidence to `Validation Required` instead.
- Produces the `Mapping Evidence / Rationale` text used for audit/decision logging.

```yaml
id: urn:component:sap:module_classifier_llm:1
name: Module & Confidence Classifier
kind: LLM
version: 1.0.0
description: Classifies Fiori apps into functional modules and scores mapping confidence.
modelName: "aXet-enabled LLM (OpenAI/Anthropic via aXet provider)"
fineTuned: false
apiProvider: "aXet provider/enabler"
```

---

### 4️⃣ Tool — Workbook Export Tool
Writes the final `.xlsx` workbook with the three required sheets: `Fiori App Mapping`, `Summary`,
and `Module Summary`.

**Key Attributes:**
- Aggregates detail rows into the Summary and Module Summary sheets.
- Deterministic column order matching the **Input & Output Contract** above.

```yaml
id: urn:component:sap:workbook_export_tool:1
name: Workbook Export Tool
kind: Tool
version: 1.0.0
description: Writes the Fiori App Mapping, Summary, and Module Summary sheets to an .xlsx workbook.
apiEndpoint: "local:write_xlsx"
inputFormat: "JSON (mapping rows)"
outputFormat: "XLSX"
```

---

### 5️⃣ Guardrail — Evidence & Confidence Guardrail
Enforces that no mapping is ever presented as `Confirmed` unless it is backed by a direct Fiori
Apps Reference Library match, and that low-evidence rows are visibly flagged.

**Key Attributes:**
- Hard enforcement: blocks the `Confirmed` label without library evidence.
- Ensures every row carries a non-empty `Mapping Evidence / Rationale`.

```yaml
id: urn:component:sap:evidence_guardrail:1
name: Evidence & Confidence Guardrail
kind: Guardrail
version: 1.0.0
description: Prevents unverified mappings from being labeled Confirmed.
enforcementLevel: "Hard"
condition: "Mapping Confidence = Confirmed requires a direct Fiori Apps Reference Library match"
action: "Downgrade to Candidate or Validation Required and record rationale"
```

---

### 6️⃣ Orchestrator — Mapping Workflow Orchestrator
Sequences the full workflow: ingest T-Codes → look up each against the Fiori Apps Reference
Library → classify module/confidence/priority → apply the evidence guardrail → export the
workbook.

**Key Attributes:**
- Deterministic execution order with a human-in-the-loop fallback at ingestion.
- Ensures Summary and Module Summary sheets are derived only after all detail rows are finalized.

```yaml
id: urn:component:sap:mapping_orchestrator:1
name: Mapping Workflow Orchestrator
kind: Orchestrator
version: 1.0.0
description: Controls the execution order of the Fiori Advisor components.
workflowManagement: "State-based workflow engine"
executionOrder: ["Tool:tcode_input", "Tool:fiori_library_lookup", "LLM:module_classifier", "Guardrail:evidence", "Tool:workbook_export"]
```
