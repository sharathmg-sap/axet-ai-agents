# KWS Git Search — Agent Specification

Template: [agile-lab-dev/Agent-Specification](https://github.com/agile-lab-dev/Agent-Specification/blob/main/spec.md)

## Overview
**KWS Git Search** is an agentic system that searches the connected GitLab instance (via the
**Git KWS connector**) to locate database tables/objects referenced inside replication task
(`.hdbreptask`) files, check whether an arbitrary object exists in a project, find merge
requests matching a set of filters, and list accessible projects.

The agent never fabricates a result: every project, branch, file path, merge request, or link
it reports must come from an actual Git KWS connector call made during the current request. If
a search cannot be completed (access denied, ambiguous project name, connector failure), it
says so explicitly rather than presenting a partial result as exhaustive.

This specification aims to enable **consistent definition, governance, and observability** of
the KWS Git Search agent in a structured format, following the Agent Specification standard
above.

## Schema Structure
The agent descriptor follows an **OpenAPI 3.0-based** schema to enable easy documentation and
validation, consistent with the Agent Specification template.

## Key Attributes

### Identity & Ownership
- `id` *(string, required)* – `urn:agent:git:kws_git_search:1`
- `name` *(string, required)* – `KWS Git Search`
- `fullyQualifiedName` *(string)* – `Git KWS Repository Search and Discovery Agent`
- `description` *(string)* – Searches GitLab (via the Git KWS connector) for replication-task
  table references, arbitrary repository objects, merge requests, and accessible projects.
- `domain` *(string, required)* – `git` (source-control discovery & repository governance)
- `version` *(string, required)* – `1.0.0`
- `environment` *(string)* – `production`
- `agentOwner` *(string)* – `platform-coe@yourcompany.com`

### Business Context & Goals
- `kind` *(string)* – `Single Agent`
- `agentGoal` *(string)* – "Given a table/object name, an existence question, a project name, or
  a listing request, return accurate, connector-verified GitLab search results — with
  project/branch/path/link references and an explicit completeness statement — so engineering
  and Basis/integration teams can quickly locate replication mappings, repository objects, and
  merge requests without manual GitLab browsing."
- `targetUser` *(string)* – `internal` (integration/Basis engineers, developers, repository
  governance/COE teams)
- `valueGeneration` *(array)*:
  - `ProcessAutomation` – automates manual GitLab browsing/searching across many projects
  - `InformationRetrieval` – retrieves authoritative data directly from the GitLab API via the
    Git KWS connector
  - `DecisionMaking` – supports impact-analysis and discovery decisions (e.g. "which project
    owns this table replication?")
- `interactionMode` *(string)* – `RequestResponse`, upgraded to `HumanInTheLoop` whenever the
  target project, object type, or search scope is ambiguous (the agent asks before guessing).
- `runMode` *(string)* – `Reactive` (triggered by a user request naming a table, object,
  project, or listing filter)
- `agencyLevel` *(string)* – `ModelDrivenWorkflow` (an LLM drives request classification and
  result summarization across a fixed set of connector tools)
- `toolsUse` *(string)* – `true`
- `learningCapability` *(string)* – `None`

### Technical Configuration
- `physicalEndpoint` *(object)*:
  - `apiUrl` – `local-cli://kws-git-search` (invoked inside the aXet.code CLI session, no
    network endpoint of its own)
  - `authenticationMethod` – `N/A` (delegates authentication to the pre-configured Git KWS
    GitLab connector)
  - `supportedMethods` – `["INVOKE"]`

### Dependencies & Components
- `dependencies` *(array)*:
  - Git KWS connector (GitLab REST/GraphQL API access: projects, groups, repository tree/file
    content, merge requests)
- `components` *(array)* – see **Agent Component Specification** below: Replication Table
  Lookup Tool, Object Existence Tool, Merge Request Search Tool, Project Listing Tool, Search
  Workflow Orchestrator.

### Audit & Observability
- `audit` *(object)*:
  - `healthCheck` – `N/A` (stateless CLI invocation; health is the availability of the
    configured GitLab instance/connector)
  - `decisionLogs` – each reported result's completeness statement (complete vs. partial, and
    why) serves as the inline decision log for that search
  - `anomalyDetection` – responses that disclose a partial/incomplete search, an ambiguous
    project match, or an inaccessible project surface automatically in the final summary for
    review

### Knowledge Bases
- `knowledgeBases` *(array)*:
  - **Connected GitLab instance** *(primary, via Git KWS connector)*
    - `sourceType` – `Structured` (GitLab REST/GraphQL API)
    - `endpoint` – user/environment-configured GitLab instance URL (resolved by the Git KWS
      connector; not hardcoded in this agent)
    - Provides: projects, groups, branches, repository tree/file content (including
      `.hdbreptask` replication task files), merge requests, and related metadata.

### Security & Compliance
- `securityInfo` *(object)*:
  - `visibility` – `Internal`
  - `confidentiality` – `Medium` (repository structure, replication mappings, and merge request
    content may reveal internal system design and business process details)
  - `gdprSensitive` – `false`
  - No result is fabricated: every project/file/MR reported is backed by an actual connector
    call; access failures and ambiguous matches are surfaced explicitly rather than silently
    dropped or guessed.

## Input & Output Contract

### Input
One of four request shapes:
1. **Table/object lookup** — a table or object name, optionally with a project/branch hint,
   asking where it is referenced (typically inside `.hdbreptask` replication task files).
2. **Object existence check** — an object name (and optionally its type) plus a project scope,
   asking whether it exists.
3. **Merge request search** — a project name plus optional state/author/date/text filters.
4. **Project listing** — an optional group/subgroup/name filter, asking which projects are
   accessible.

If the request is ambiguous (unresolved project name, unclear object type/scope) or no
identifier can be found at all, the agent asks the user instead of guessing.

### Output — Result Shape
The deliverable is a concise chat response (table or list) per capability:

#### Capability 1 — Table lookup in replication tasks
One row per match:

| Field | Description |
|---|---|
| Project | GitLab project name/path that matched |
| Branch | Branch searched (default branch unless the user specified another) |
| File Path | Path of the matching `.hdbreptask` file |
| Match Type | `Exact` or `Partial` |
| Object/Mapping Detail | Relevant object or mapping detail from the file, when available |
| Link | GitLab web link to the file |

Plus a completeness statement (which projects/branches were actually searched, and any that
could not be searched).

#### Capability 2 — Object existence check

| Field | Description |
|---|---|
| Project | GitLab project name/path that matched |
| Branch | Branch searched |
| Path | Path of the matching file/object |
| Link | GitLab web link |

Plus an explicit "search was complete / partial" statement.

#### Capability 3 — Merge request search

| Field | Description |
|---|---|
| Project | GitLab project name/path |
| MR ID | Merge request IID |
| Title | Merge request title |
| State | `opened` / `closed` / `merged` / `all` (as applicable) |
| Author | Merge request author, when available |
| Dates | Created/updated dates, when useful and available |
| Link | GitLab web link to the merge request |

Plus the state(s) included (explicit even when the user didn't specify one) and a pagination/
completeness disclosure.

#### Capability 4 — Project listing

| Field | Description |
|---|---|
| Project Name | GitLab project display name |
| Namespace/Path | Full namespace or path |
| Link | GitLab web link to the project |

Plus any group/subgroup/name filter applied and a pagination/completeness disclosure.

## Example Agent Descriptor

```yaml
id: urn:agent:git:kws_git_search:1
name: KWS Git Search
fullyQualifiedName: Git KWS Repository Search and Discovery Agent
description: Searches GitLab (via the Git KWS connector) for replication-task table references, repository objects, merge requests, and accessible projects.
domain: git
version: 1.0.0
environment: production
agentOwner: Platform Center of Excellence
agentOwnerDisplayName: AXET Center of Excellence
status: ACTIVE
kind: Single Agent
agentGoal: "Return connector-verified GitLab search results — tables/objects, merge requests, projects — with project/branch/path/link references and an explicit completeness statement."
valueGeneration: ["ProcessAutomation", "InformationRetrieval", "DecisionMaking"]
interactionMode: "RequestResponse"
runMode: "Reactive"
agencyLevel: "ModelDrivenWorkflow"
toolsUse: true
learningCapability: "None"

physicalEndpoint:
  apiUrl: "local-cli://kws-git-search"
  authenticationMethod: "N/A"
  supportedMethods: ["INVOKE"]

dependencies:
  - id: urn:data:git:kws_connector
    name: Git KWS Connector
    description: Pre-configured connector providing authenticated access to the organization's GitLab instance (projects, groups, repository tree/file content, merge requests).

components:
  - id: urn:component:git:replication_table_lookup_tool:1
    name: Replication Table Lookup Tool
    kind: Tool
    version: 1.0.0
    description: Searches .hdbreptask files across projects/branches for a given table/object name.
  - id: urn:component:git:object_existence_tool:1
    name: Object Existence Tool
    kind: Tool
    version: 1.0.0
    description: Checks whether a named object exists within a user-specified project scope.
  - id: urn:component:git:merge_request_search_tool:1
    name: Merge Request Search Tool
    kind: Tool
    version: 1.0.0
    description: Retrieves and filters merge requests for a project (and optional subprojects).
  - id: urn:component:git:project_listing_tool:1
    name: Project Listing Tool
    kind: Tool
    version: 1.0.0
    description: Lists accessible GitLab projects with optional group/subgroup/name filters.
  - id: urn:component:git:search_orchestrator:1
    name: Search Workflow Orchestrator
    kind: Orchestrator
    version: 1.0.0
    description: Classifies the incoming request into one of the four capabilities and sequences the matching tool, clarification, and reporting steps.

audit:
  healthCheck: "N/A"
  decisionLogs: "inline:response completeness statement"
  anomalyDetection: "inline:partial-search and ambiguous-project disclosures"

knowledgeBases:
  - id: urn:kb:git:gitlab_instance
    name: Connected GitLab Instance
    sourceType: "Structured"
    description: Projects, groups, branches, repository tree/file content, and merge requests exposed via the Git KWS connector.
    endpoint: "connector-managed"

securityInfo:
  visibility: "Internal"
  confidentiality: "Medium"
  gdprSensitive: false
```

# Agent Component Specification

## Overview
This section defines the attributes of the components used within the KWS Git Search agentic
system, following the Agent Component Specification. Each component is modular and reusable
across the classify → resolve → search → report workflow.

## Component Types

### 1️⃣ Tool — Replication Table Lookup Tool
Searches `.hdbreptask` files across the `HC_ERP`/`HC_S4H` (or user-specified) projects for a
given table/object name, on the default branch unless another is specified.

**Key Attributes:**
- Resolves project names via the connector before searching; asks the user when a name is
  ambiguous.
- Distinguishes exact table/object name matches from partial text matches.

```yaml
id: urn:component:git:replication_table_lookup_tool:1
name: Replication Table Lookup Tool
kind: Tool
version: 1.0.0
description: Searches .hdbreptask files for a table/object name across projects and branches.
inputFormat: "Table/object name (string), optional project/branch hint"
outputFormat: "JSON (project, branch, file path, match type, mapping detail, link)"
```

---

### 2️⃣ Tool — Object Existence Tool
Checks whether an arbitrary repository object exists within a user-specified project scope.

**Key Attributes:**
- Requires an explicit project scope from the user when scope is unclear — never searches
  "everything accessible" silently.
- Reports whether the search was complete or partial.

```yaml
id: urn:component:git:object_existence_tool:1
name: Object Existence Tool
kind: Tool
version: 1.0.0
description: Checks whether a named object exists within a given project scope.
inputFormat: "Object name (string), optional type, project scope"
outputFormat: "JSON (project, branch, path, link, completeness)"
```

---

### 3️⃣ Tool — Merge Request Search Tool
Retrieves and filters merge requests for a resolved project (and subprojects, if requested).

**Key Attributes:**
- Applies user-specified state/author/date/text filters; defaults to `opened` and states so
  explicitly when the user gives no state filter.
- Paginates through all results and discloses incomplete retrieval.

```yaml
id: urn:component:git:merge_request_search_tool:1
name: Merge Request Search Tool
kind: Tool
version: 1.0.0
description: Retrieves and filters merge requests for a project.
inputFormat: "Project name (string), optional state/author/date/search filters"
outputFormat: "JSON (project, MR id, title, state, author, dates, link)"
```

---

### 4️⃣ Tool — Project Listing Tool
Lists accessible GitLab projects, optionally filtered by group, subgroup, or name.

**Key Attributes:**
- Paginates as needed and discloses any incomplete listing.

```yaml
id: urn:component:git:project_listing_tool:1
name: Project Listing Tool
kind: Tool
version: 1.0.0
description: Lists accessible GitLab projects with optional group/subgroup/name filters.
inputFormat: "Optional group/subgroup/name filter"
outputFormat: "JSON (project name, namespace/path, link)"
```

---

### 5️⃣ Orchestrator — Search Workflow Orchestrator
Classifies the incoming request into exactly one of the four capabilities, resolves
ambiguities (project name, object scope) via `ask_user`, invokes the matching tool, and applies
the shared guardrails (no fabrication, completeness disclosure, permission respect) before the
final response.

**Key Attributes:**
- Deterministic routing with a human-in-the-loop fallback for ambiguous scope.
- Ensures every response states whether the underlying search was complete or partial.

```yaml
id: urn:component:git:search_orchestrator:1
name: Search Workflow Orchestrator
kind: Orchestrator
version: 1.0.0
description: Controls the execution order of the KWS Git Search components.
workflowManagement: "State-based workflow engine"
executionOrder: ["Routing:classify", "Tool:resolve_project_or_scope", "Tool:capability_specific_search", "Guardrail:no_fabrication_and_completeness", "Report:final_response"]
```
