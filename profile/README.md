# Solomon

Decisions made in a meeting live in someone's notes, if they're written down at all, and
follow-ups fall through because no system tracks whether they actually happened. Solomon is an
AI-native decision workspace for teams: it turns meetings, documents, and incoming signals into
tracked decisions, follow-up tasks, and reports, with AI personas assisting review instead of
people manually chasing every follow-up.

## Core loop

```mermaid
flowchart LR
    a["Connect sources<br/><sub>docs, meetings, chat</sub>"] --> b["Collect signals"]
    b --> c["Review<br/><sub>AI-assisted</sub>"]
    c --> d["Decide / assign tasks"]
    d --> e["Track"]
    e --> f["Report"]
    f --> g["Automate"]
    g -.->|feeds back| a
```

Everything under "Decide / assign" produces a **decision** or a **directive** (task) that stays
attached to the meeting/document it came from, so reports are always traceable back to source.

## Data model

A decision or directive always keeps a pointer back to the meeting or document that produced it,
and a report is assembled from that trail — not from someone's memory of what was said.

```mermaid
flowchart LR
    subgraph Sources
        m["Meeting"]
        doc["Document"]
    end
    subgraph Outputs
        dec["Decision<br/><sub>framework + rationale</sub>"]
        dir["Directive<br/><sub>owner + due date</sub>"]
    end
    rep["Report"]

    m --> dec
    m --> dir
    doc --> dec
    doc --> dir
    dec --> rep
    dir --> rep
    rep -.->|traceable to source| m
    rep -.->|traceable to source| doc
```

## System architecture

Client and server are both organized as layered/Clean Architecture, so the domain rules on each
side stay independent of frameworks (Flutter, Axum, Postgres).

```mermaid
flowchart TB
    subgraph Client["Flutter client — iOS · Android · macOS · Windows · Linux"]
        direction TB
        cpres["presentation<br/><sub>widgets, Riverpod providers</sub>"]
        cdomain["domain<br/><sub>pure Dart, no framework deps</sub>"]
        cdata["data<br/><sub>repositories, API client</sub>"]
        cpres --> cdomain
        cdata --> cdomain
    end

    subgraph Server["Rust Cloud API — Cargo workspace"]
        direction TB
        api["api<br/><sub>axum HTTP layer</sub>"]
        app["application<br/><sub>use cases, orchestration</sub>"]
        domain["domain<br/><sub>pure business rules</sub>"]
        pg["postgres<br/><sub>sqlx repositories</sub>"]
        worker["worker<br/><sub>background jobs</sub>"]
        api --> app --> domain
        pg --> domain
        worker --> app
    end

    auth[("Supabase Auth")]
    db[("Postgres")]
    llm[("LLM providers")]

    cdata -->|REST, bearer token| api
    cdata -->|session / PKCE| auth
    api -->|verifies JWT| auth
    pg --> db
    app --> llm
```

**Authority stays server-side.** Meeting-state transitions, report generation rules, RAG
selection, cost limits, and permission checks live only in the Rust `domain`/`application`
crates — the Flutter client never re-implements them, it renders whatever the API returns. A
client-supplied `workspace_id`, `user_id`, role, or cost estimate is never treated as an
authorization source by the server.

## AI review engine (conceptual)

None of this code is public — this diagrams the *shape* of the system, not the implementation.
The model never writes state directly: it proposes a transition, and a total guard function
either accepts it and persists the next state, or rejects it outright.

```mermaid
stateDiagram-v2
    [*] --> Intake
    Intake --> Analysis : sources attached
    Analysis --> Review : ready for AI review
    Review --> Gate : structured findings validated
    Gate --> Decided : guard accepts transition
    Gate --> Rejected : guard rejects transition
    Analysis --> BlockedEvidence : insufficient evidence
    BlockedEvidence --> Analysis : evidence resumed
    Review --> BlockedModel : model backend unavailable
    BlockedModel --> Review : backend resumed
    Decided --> [*]
    Rejected --> [*]

    note right of Gate
        Every transition is either
        accepted and persisted, or
        rejected — no partial writes,
        no trusting model output as-is.
    end note
```

Reviews themselves are structured, multi-axis findings validated against a fixed schema before
they're allowed near the guard — malformed or out-of-range output is a hard rejection, not a
best-effort parse. Job processing is safe under concurrency, and every accepted transition is
content-addressed so what the model saw and what it decided are both reconstructable later:

```mermaid
sequenceDiagram
    participant Worker
    participant Queue as Postgres job queue
    participant LLM as LLM provider
    participant Guard as State guard
    participant DB as Postgres (state of record)

    Worker->>Queue: claim job (row-level lock, skip already-claimed)
    Queue-->>Worker: job leased
    loop heartbeat while processing
        Worker->>Queue: extend lease
    end
    Worker->>LLM: request structured review (schema-constrained)
    LLM-->>Worker: multi-axis findings (typed, not free text)
    Worker->>Worker: validate against policy schema
    alt malformed or out-of-range
        Worker->>Queue: checkpoint: reject, no state write
    else valid
        Worker->>Guard: propose transition(current_state, findings)
        Guard-->>Worker: accept(next_state) or reject(reason)
        Worker->>DB: canonicalize (RFC 8785 JCS) + SHA-256 hash the evidence
        Worker->>DB: persist next_state under optimistic concurrency control
    end
    Worker->>Queue: release lease
```

The interesting engineering problem isn't "call an LLM" — it's constraining what an LLM's output
is allowed to do to a system of record, and making every step of that auditable. The specific
review axes, approval-policy thresholds, and prompts are the actual product and stay closed.

## Security & control-plane architecture

An analysis of Palantir's publicly documented enterprise LLM architecture (Foundry / AIP /
Ontology / Apollo / Gotham), reframed as Solomon's target control plane. Every claim below is
graded by evidence level — Verified Fact (directly stated in official docs), Likely Inference
(structurally derived from combining several official docs), or Unknown (not confirmed by public
material) — and only Verified Fact is treated as a product claim.

### Bottom line

1. **An LLM is not a security boundary.** Authorization has to be enforced by the authenticated
   session, Ontology lookups, object/property policy, and function/Action submission criteria —
   not by the model.
2. **An Ontology is not a knowledge graph.** It's an operating contract that binds data meaning,
   relationships, permissions, logic, and the allowed actions together.
3. **No-retention and no-transmission are different things.** Using an external model means the
   selected prompt and context are sent to an external inference runtime, whatever the retention
   policy says afterward.
4. **Fully internal processing is a deployment choice.** Banning external egress outright
   requires a customer-hosted, on-prem, or air-gapped path.
5. **Writes must be constrained to formalized actions.** Free-text model output is never executed
   directly — it goes through typed Actions, approval, re-validation, audit, and rollback.
6. **Solomon should copy the control plane before the platform.** Gateway, Policy, Permission,
   Context, Evidence, Audit, Risk, and Human Approval come first — not a Foundry-scale data
   platform.

```mermaid
flowchart LR
    U["User"] --> APP["Application"]
    APP --> IAM["Identity / Session"]
    IAM --> ONT["Ontology"]
    ONT --> POL["Policy + Permission"]
    POL --> CTX["Authorized Context"]
    CTX --> GW["LLM Gateway"]
    GW --> MODEL["LLM"]
    MODEL --> TOOL["Tool / Function"]
    TOOL --> POL
    TOOL --> ACT["Typed Action"]
    ACT --> HITL["Human Approval"]
    HITL --> WRITE["Governed Write"]
    GW --> AUDIT["Audit + Evidence"]
    ACT --> AUDIT

    classDef trusted fill:#e8f7f5,stroke:#168b86,color:#102a2a;
    classDef model fill:#fff4df,stroke:#d88a00,color:#3b2600;
    classDef control fill:#eaf1fb,stroke:#3d70b2,color:#12233b;
    class IAM,ONT,POL,CTX,TOOL,ACT,HITL,WRITE,AUDIT trusted;
    class MODEL model;
    class APP,GW control;
```

> The point isn't "the LLM understands permissions" — it's that **only information and tools that
> already passed authorization are ever shown to the LLM.**

### Research approach and evidence levels

The guiding question isn't "which model does Palantir use" — it's: what data is the model shown,
under whose authority are lookups/search/tool calls performed, where are security decisions
enforced, what gets validated before model output becomes a real action, what crosses the trust
boundary when an external model is used, and can an execution be reproduced and audited later.

```mermaid
flowchart TD
    CLAIM["Analytical claim"] --> Q1{"Does an official doc\nstate this directly?"}
    Q1 -->|"Yes"| VF["Verified Fact"]
    Q1 -->|"No"| Q2{"Strongly derived by\ncombining several official docs?"}
    Q2 -->|"Yes"| LI["Likely Inference"]
    Q2 -->|"No"| UNK["Unknown"]
```

| Label | Meaning | Usage rule |
|---|---|---|
| Verified Fact | Directly confirmed in an official doc | Safe to describe as a product capability |
| Likely Inference | Structural interpretation combining multiple sources | Use as a design hypothesis, needs verification |
| Unknown | Not confirmed by public material | Never state as a product capability |

### Palantir's platform roles

| Platform | Primary role | Position in LLM integration |
|---|---|---|
| **Foundry** | Data connections, pipelines, datasets, Ontology, functions, apps, security, audit | The data/permission/business-meaning foundation |
| **AIP** | Model access, Logic, Chatbot/Agent, Evals, observability | The generative-AI application layer |
| **Apollo** | Cloud, on-prem, edge, and disconnected deployment + updates | The runtime and software-delivery layer |
| **Gotham** | Defense/intelligence/operations domain applications | A mission-app layer that composes with Foundry/Ontology |

```mermaid
flowchart TB
    subgraph Mission["Domain / mission applications"]
        GOTHAM["Gotham"]
        CUSTOM["Custom Applications"]
    end
    subgraph AI["AI application layer"]
        AIP["AIP"]
        AGENT["Agents / Logic / Evals"]
        AIP --> AGENT
    end
    subgraph OPS["Operations layer"]
        FOUNDRY["Foundry"]
        ONTOLOGY["Ontology"]
        DATA["Data + Logic + Actions + Security"]
        FOUNDRY --> ONTOLOGY --> DATA
    end
    subgraph DELIVERY["Delivery layer"]
        APOLLO["Apollo"]
        ENV["Cloud / On-prem / Edge / Air-gap"]
        APOLLO --> ENV
    end
    Mission --> AI --> OPS
    DELIVERY -. "deploy · update · observe" .-> Mission
    DELIVERY -. "deploy · update · observe" .-> AI
    DELIVERY -. "deploy · update · observe" .-> OPS
```

Why an Ontology isn't just a knowledge graph:

```mermaid
flowchart LR
    OBJ["Objects"] --> ONT["Ontology"]
    PROP["Properties"] --> ONT
    LINK["Links"] --> ONT
    FUNC["Functions"] --> ONT
    ACTION["Actions"] --> ONT
    POLICY["Dynamic Security"] --> ONT
    LOGIC["Business Logic"] --> ONT
    ONT --> READ["Allowed nouns\nwhat can be seen"]
    ONT --> DO["Allowed verbs\nwhat can be done"]
```

### Enterprise LLM request lifecycle

```mermaid
sequenceDiagram
    actor U as User
    participant APP as Application
    participant IAM as Identity / Session
    participant ONT as Ontology
    participant POL as Policy Engine
    participant CTX as Context Builder
    participant GW as LLM Gateway
    participant M as Model
    participant TOOL as Tool / Function
    participant ACT as Action Engine
    participant AUD as Audit

    U->>APP: question or task request
    APP->>IAM: resolve user · group · project · purpose
    IAM-->>APP: authenticated execution context
    APP->>ONT: query objects/documents under user permissions
    ONT->>POL: evaluate object/property/row/purpose policy
    POL-->>ONT: allowed scope and obligations
    ONT-->>CTX: only permission-applied results
    CTX->>CTX: minimize · redact · cite · token-budget
    CTX->>GW: model request envelope
    GW->>M: selected prompt and context
    M-->>GW: answer or proposed tool call
    GW->>TOOL: execute under the same user context
    TOOL->>POL: server-side re-authorization
    POL-->>TOOL: allow or deny
    TOOL-->>GW: structured execution result
    GW->>ACT: propose a typed Action if a change is needed
    ACT->>POL: re-check submission criteria and risk
    ACT-->>APP: approval request or execution result
    APP-->>U: response linked to evidence
    APP->>AUD: session · context · model · tool · action events
```

Design invariants:

- Retrieval and tool calls always pass a server-side permission check.
- Describing a user's permissions in the prompt is not enforcement.
- Tool arguments the model proposes are never trusted as-is — re-validated against schema and
  policy.
- Writes are expressed as typed Actions or constrained functions, not free text.
- Every material decision links back to user, session, policy version, model version, and
  evidence.

### Trust boundaries and enforcement points

```mermaid
flowchart LR
    subgraph T1["Trust zone: enterprise control plane"]
        ID["Identity"]
        POLICY["Policy"]
        RETRIEVAL["Authorized Retrieval"]
        MIN["Minimization / Redaction"]
        TOOLS["Tool Broker"]
        ACTIONS["Action Validation"]
        AUDIT["Audit"]
    end
    subgraph T2["Non-deterministic zone"]
        PROMPT["Prompt"]
        LLM["LLM"]
        OUTPUT["Generated Output"]
    end
    subgraph T3["External provider zone (optional)"]
        API["Provider Endpoint"]
        RUNTIME["Inference Runtime"]
    end
    ID --> POLICY --> RETRIEVAL --> MIN --> PROMPT
    PROMPT --> LLM --> OUTPUT --> TOOLS
    TOOLS --> POLICY
    TOOLS --> ACTIONS --> AUDIT
    LLM -. "if an external model is chosen" .-> API --> RUNTIME

    classDef untrusted fill:#fff0e6,stroke:#cf5b20,color:#3c1c0d;
    classDef trusted fill:#e9f7f4,stroke:#168b86,color:#102a2a;
    class LLM,OUTPUT,API,RUNTIME untrusted;
    class ID,POLICY,RETRIEVAL,MIN,TOOLS,ACTIONS,AUDIT trusted;
```

| Enforcement point | What it must decide |
|---|---|
| Identity / Session | Who, in which project, for what purpose |
| Retrieval / Ontology | Which objects, rows, properties, document chunks are visible |
| Policy Engine | Role, attribute, classification, purpose, time, region, risk conditions |
| Context Builder | Minimum necessary data, redaction, token budget, citations |
| LLM Gateway | Allowed model, region, cost, log policy, egress |
| Tool Broker | Allowed tools and arguments, re-authorization, timeout, side-effect limits |
| Action Engine | Submission criteria, approval, idempotency, rollback |
| Audit Service | Reproducible linkage from request to result |

### Permission propagation

"The LLM inherits the user's permissions" is shorthand for the outcome, not the mechanism —
every server component re-authorizes against the user or project context independently.

```mermaid
flowchart TD
    SESSION["Authenticated Session"] --> CLAIMS["User / Group / Project / Purpose"]
    CLAIMS --> QUERY["Ontology Query"]
    CLAIMS --> RETRIEVER["Document Retriever"]
    CLAIMS --> FUNCTION["Function Call"]
    CLAIMS --> ACTION["Action Submission"]
    QUERY --> P1["Object + Property Policies"]
    RETRIEVER --> P2["Document / Row / Marking Policies"]
    FUNCTION --> P3["Function Authorization"]
    ACTION --> P4["Submission Criteria + Approval"]
    P1 --> VISIBLE["Allowed results"]
    P2 --> VISIBLE
    P3 --> VISIBLE
    P4 --> EFFECT["Allowed side effects"]
    VISIBLE --> LLM["LLM's effective visibility"]
```

| Model | Example | Solomon application |
|---|---|---|
| RBAC | Analyst, admin, approver | Feature and default-role limits |
| ABAC | Department, project, purpose, region, data owner | Dynamic context policy |
| MAC / classification | Sensitivity, marking, compartment | Forced separation of data and model paths |

### RAG and context construction

```mermaid
flowchart LR
    Q["User question"] --> AUTH["Principal Context"]
    AUTH --> PLAN["Retrieval Plan"]
    PLAN --> OR["Ontology Retrieval"]
    PLAN --> DR["Document / Chunk Retrieval"]
    PLAN --> FR["Function Retrieval"]
    OR --> POL["Policy Filter"]
    DR --> POL
    FR --> POL
    POL --> RANK["Rank + Deduplicate"]
    RANK --> MIN["Field Selection + Redaction"]
    MIN --> CITE["Citation + Provenance"]
    CITE --> BUDGET["Token Budget"]
    BUDGET --> PROMPT["Grounded Prompt"]
    PROMPT --> MODEL["Model"]
    MODEL --> ANSWER["Answer + Evidence Links"]
```

Security-critical points in RAG:

- The existence of an embedding or index must never bypass source-document permissions.
- User scope must be preserved before and after candidate generation.
- Chunks must stay linked to their source document, object, and policy version.
- Deletion and access revocation must propagate to search results and caches.
- Never assume tool and function results are auto-cited.
- The exact internal vector DB product and physical placement are not confirmed by public
  material.

```mermaid
stateDiagram-v2
    [*] --> Discovered: candidate search
    Discovered --> Authorized: permission filter
    Authorized --> Minimized: field/chunk minimization
    Minimized --> Redacted: sensitive-data handling
    Redacted --> Cited: source linkage
    Cited --> Prompted: model input
    Prompted --> Logged: metadata recorded per policy
    Logged --> Expired: TTL / retention expiry
    Authorized --> Revoked: access revoked
    Prompted --> Revoked: cache invalidated
    Revoked --> Expired
```

### Actions and write control

Model output is a proposed action, not an execution command. A real change has to pass schema,
permission, submission criteria, risk, approval, and idempotency.

```mermaid
stateDiagram-v2
    [*] --> Proposed: LLM or user proposal
    Proposed --> SchemaValidated: type/required-field check
    SchemaValidated --> Authorized: re-authorized as the same user
    Authorized --> RiskChecked: impact/amount/scope evaluation
    RiskChecked --> AwaitingApproval: approval required
    RiskChecked --> Executing: auto-execution allowed
    AwaitingApproval --> Executing: approved
    AwaitingApproval --> Rejected: denied / expired
    Executing --> Committed: success
    Executing --> Failed: failure
    Committed --> Audited: result, evidence, version recorded
    Failed --> RolledBack: compensating action or rollback
    Rejected --> Audited
    RolledBack --> Audited
    Audited --> [*]
```

Required controls on the write path: allowed fields/value ranges per Action type; separation of
user vs. system authority; confirmation/approval/dual-approval by risk tier; request-ID-based
idempotency; before/after object state and diff logging; partial-failure compensation
transactions; replay protection on reused approvals; and linkage to policy/prompt/model version.

### Data exposure and external egress

```mermaid
flowchart TD
    DATA["Enterprise data"] --> SELECT["Permission-applied + minimal context"]
    SELECT --> EXT{"External model?"}
    EXT -->|"Yes"| SEND["Selected data sent to an external endpoint"]
    SEND --> INFER["External inference runtime processes the meaning"]
    INFER --> DELETE["No-retention policy after completion"]
    DELETE --> NOTRAIN["Not used for training"]
    EXT -->|"No"| INTERNAL["Processed in a customer-controlled environment"]
    NOTE["No-retention/no-training ≠ no-transmission"] -.-> SEND
```

For an external model service to compute an output, the inference runtime has to process the
plaintext meaning of what was sent — transport encryption protects the network, it doesn't remove
the external processing itself.

| Stage | Control |
|---|---|
| Before retrieval | Data classification, restricted views, object/property policy, row filters |
| Context construction | Minimum fields, chunk limits, sensitive-data scanning, redaction, pseudonymization |
| Model selection | Allow-lists keyed on sensitivity, region, contract, cost |
| Network | Private link, egress allow-list, proxy, region pinning |
| Output | Schema validation, policy filter, sensitive-data check, citation check |
| Tools / actions | Server-side re-authorization, submission criteria, approval, rate/scope limits |
| Logs | Secret exclusion, separate permissions, TTL, encryption, audit |

What public material alone cannot guarantee: a general-purpose output DLP applied automatically
across every AIP path; complete prevention of prompt injection and jailbreaks; identical marking
behavior across every BYOM path; or evidence that homomorphic encryption, MPC, or differential
privacy are applied to the default LLM inference path.

### Deployment topology selection

```mermaid
flowchart TD
    START["Classify the workload"] --> Q1{"Is external egress banned?"}
    Q1 -->|"Yes"| Q2{"Is an air gap required?"}
    Q2 -->|"Yes"| AIR["Air-gapped Model"]
    Q2 -->|"No"| Q3{"Must processing stay in a customer datacenter?"}
    Q3 -->|"Yes"| ONPREM["On-prem Model"]
    Q3 -->|"No"| HOSTED["Customer-hosted / Private VPC Model"]
    Q1 -->|"No"| Q4{"Is a region/provider boundary specified?"}
    Q4 -->|"Yes"| AZURE["Regional Azure OpenAI or approved provider"]
    Q4 -->|"No"| API["Approved External API model"]
    AIR --> OPS["Validate ops cost and model performance"]
    ONPREM --> OPS
    HOSTED --> OPS
    AZURE --> OPS
    API --> OPS
```

> This decision tree is a Solomon design recommendation, not a claim that Palantir ships a
> complete sensitivity-based auto-router out of the box.

| Path | External transmission | Ops control | Advantage | Main burden |
|---|---:|---:|---|---|
| External API | Yes | Low–medium | Latest models, fast adoption | Provider/region/contract dependence |
| Azure OpenAI | Yes | Medium | Region + enterprise contract integration | Azure boundary and service dependence |
| Customer-hosted | Depends on config | High | Customer network/key/log control | GPU and MLOps operations |
| On-prem | Generally none | Very high | Fully internal processing | Capacity, updates, model-choice limits |
| Air-gap | None | Highest | Disconnected, high-sensitivity | Model import, patching, eval cost |

```mermaid
flowchart LR
    REQ["Request Envelope"] --> CLASS["Data Classification"]
    CLASS --> REGION["Region Constraint"]
    REGION --> CAP["Capability Need"]
    CAP --> COST["Cost / Latency Budget"]
    COST --> ALLOW["Allowed Model Set"]
    ALLOW --> ROUTE["Selected Route"]
    ROUTE --> EXT["External"]
    ROUTE --> PRIV["Private VPC"]
    ROUTE --> LOCAL["On-prem"]
    ROUTE --> AIR["Air-gap"]
```

### Threat model and mitigating controls

```mermaid
flowchart TB
    GOAL["Sensitive-data exposure or unauthorized action"]
    GOAL --> PI["Prompt Injection"]
    GOAL --> EX["Excessive Retrieval"]
    GOAL --> TA["Tool Abuse"]
    GOAL --> OE["Output Exfiltration"]
    GOAL --> LP["Log Poisoning / Leakage"]
    GOAL --> MP["Model / Route Misconfiguration"]
    GOAL --> DP["Data or Index Poisoning"]
    PI --> C1["separate content from instructions, tool limits, approval"]
    EX --> C2["scoped-to-user search, minimum fields, rate limit"]
    TA --> C3["typed schema, re-authorization, allow-list, sandbox"]
    OE --> C4["output inspection, citation, egress limits"]
    LP --> C5["log redaction, separate ACL, TTL"]
    MP --> C6["policy-based routing, region lock, deployment validation"]
    DP --> C7["provenance verification, lineage, approved indexing"]
```

| Attack surface | Failure example | Primary defense |
|---|---|---|
| Prompt | Instructions embedded in a document override the system prompt | Separate content from instructions, hide out-of-scope tools |
| Retrieval | Broad search bulk-extracts in-scope data | Limit by purpose, scope, tokens, call volume |
| Tools | Model selects a dangerous argument or API | Typed schema, server re-authorization, approval |
| Output | Sensitive data repeatedly leaked in "answer" form | Output policy, DLP backstop, citation/evidence check |
| Logs | Secrets accumulate in prompts, responses, tool args | Minimal logging, redaction, log ACL and TTL |
| Routing | Confidential data misrouted to an external model | Classification-based deny-by-default model allow-list |
| Index | Malicious documents or stale permissions poison results | Lineage, freshness, deletion/revocation propagation |

```mermaid
flowchart TD
    INPUT["User / document / web / tool input"] --> LABEL["Source and trust label"]
    LABEL --> SPLIT["Instruction / data separation"]
    SPLIT --> POLICY["Tool and data policy"]
    POLICY --> MODEL["Model inference"]
    MODEL --> INTENT["Tool intent and argument check"]
    INTENT --> RISK{"High-risk or irreversible?"}
    RISK -->|"Yes"| APPROVAL["Human Approval"]
    RISK -->|"No"| EXEC["Scoped Execution"]
    APPROVAL --> EXEC
    EXEC --> VERIFY["Verify result and side effects"]
    VERIFY --> AUDIT["Trace + Audit"]
```

### Audit, observability, and the evidence graph

An audit event must link: request ID, session ID, parent trace ID; user/group/project/service
principal; purpose, policy version, allow/deny decision and obligations; retrieved-object and
chunk identifiers and versions; prompt template, context hash, model and model version; tool
name, argument hash, executor, result, latency; Action proposal, approver, execution result, diff
and rollback; citations, evidence-graph nodes, and the final answer; and retention period,
redaction state, access tier.

```mermaid
flowchart LR
    REQ["Request"] --> SESSION["Session"]
    SESSION --> POLICY["Policy Decision"]
    SESSION --> RETRIEVAL["Retrieved Evidence"]
    SESSION --> PROMPT["Prompt Version"]
    SESSION --> MODEL["Model Invocation"]
    MODEL --> TOOL["Tool Call"]
    TOOL --> ACTION["Action"]
    ACTION --> APPROVAL["Approval"]
    ACTION --> RESULT["Committed Result"]
    RETRIEVAL --> CITE["Citation"]
    CITE --> ANSWER["Final Answer"]
    RESULT --> ANSWER
    POLICY --> ANSWER
```

The log paradox: logging raises auditability, but a log that replicates prompts, responses,
document chunks, PII, and tool arguments becomes a new sensitive store in its own right — it
needs the same permission, encryption, redaction, and retention discipline as the source data.

### Solomon target architecture

```mermaid
flowchart TB
    subgraph UX["Experience"]
        DESKTOP["Desktop UI"]
        API["API / SDK"]
        WORKFLOW["Decision Workflow"]
    end
    subgraph EDGE["Ingress and gateway"]
        AUTH["Identity + Session"]
        LLMGW["LLM Gateway"]
        SECGW["Security Gateway"]
    end
    subgraph CONTROL["Control plane"]
        POLICY["Policy Engine"]
        PERM["Permission Engine"]
        RISK["Risk Engine"]
        ROUTER["Model Router"]
        AUDIT["Audit Service"]
    end
    subgraph KNOWLEDGE["Knowledge and context"]
        CONTEXT["Context Builder"]
        ONTOLOGY["Decision Ontology"]
        EVIDENCE["Evidence Graph"]
        MEMORY["Decision Memory"]
    end
    subgraph EXECUTION["Governed execution"]
        BROKER["Tool Broker"]
        ACTION["Action Engine"]
        APPROVAL["Human Approval"]
        ROLLBACK["Rollback / Compensation"]
    end
    subgraph MODELS["Model plane"]
        EXTERNAL["External Models"]
        PRIVATE["Private VPC Models"]
        LOCAL["On-prem / Air-gap Models"]
    end
    subgraph DATA["Enterprise data"]
        DB["Databases"]
        DOCS["Documents"]
        APIS["Enterprise APIs"]
        EVENTS["Events"]
    end
    UX --> AUTH --> SECGW --> LLMGW
    SECGW --> POLICY
    POLICY <--> PERM
    POLICY --> CONTEXT
    CONTEXT <--> ONTOLOGY
    CONTEXT <--> EVIDENCE
    CONTEXT <--> MEMORY
    ONTOLOGY --> DATA
    LLMGW --> ROUTER
    ROUTER --> EXTERNAL
    ROUTER --> PRIVATE
    ROUTER --> LOCAL
    LLMGW --> BROKER
    BROKER --> POLICY
    BROKER --> ACTION
    ACTION --> RISK
    RISK --> APPROVAL
    APPROVAL --> ACTION
    ACTION --> DATA
    ACTION --> ROLLBACK
    LLMGW --> AUDIT
    CONTEXT --> AUDIT
    ACTION --> AUDIT
```

| Component | Responsibility | Priority |
|---|---|---:|
| LLM Gateway | Model abstraction, prompt envelope, timeout, cost, fallback | P0 |
| Security Gateway | Ingress, auth, classification, egress, rate limit | P0 |
| Policy Engine | Purpose/data/tool/model/action policy | P0 |
| Permission Engine | User/group/resource permission evaluation | P0 |
| Context Builder | Permission-applied retrieval, minimization, redaction, citation | P0 |
| Audit Service | End-to-end trace and reproducibility | P0 |
| Decision Ontology | Operating model of objects, relations, decisions, actions | P1 |
| Evidence Graph | Claims and evidence, versions, lineage | P1 |
| Action Engine | Typed writes, approval, idempotency, rollback | P1 |
| Model Router | Route selection by sensitivity/region/performance/cost | P1 |
| Risk Engine | Risk scoring for actions and responses | P1 |
| Decision Memory | Long-term memory of approved decisions and outcomes | P2 |

```mermaid
flowchart LR
    R0["1. Authenticate"] --> R1["2. Classify"]
    R1 --> R2["3. Authorize Retrieval"]
    R2 --> R3["4. Build Context"]
    R3 --> R4["5. Route Model"]
    R4 --> R5["6. Generate"]
    R5 --> R6["7. Validate Output"]
    R6 --> R7["8. Authorize Tool"]
    R7 --> R8["9. Approve Action"]
    R8 --> R9["10. Execute"]
    R9 --> R10["11. Audit + Evidence"]
```

### Solomon recommended interfaces

> These JSON shapes aren't a copy of a Palantir public API — they're a recommended contract for
> applying the analysis above to Solomon.

**Request envelope**

```json
{
  "request_id": "req_01J...",
  "session_id": "ses_01J...",
  "principal": {
    "user_id": "user_123",
    "groups": ["research"],
    "project_id": "project_solomon"
  },
  "purpose": "decision_support",
  "data_labels": ["INTERNAL"],
  "region": "KR",
  "prompt": "question or task request",
  "max_risk": "medium"
}
```

**Policy decision**

```json
{
  "decision": "allow_with_obligations",
  "policy_version": "policy-2026-08-06.1",
  "allowed_object_types": ["Decision", "Evidence"],
  "allowed_properties": {
    "Decision": ["id", "title", "status"],
    "Evidence": ["id", "summary", "source_uri"]
  },
  "allowed_tools": ["search_evidence"],
  "allowed_models": ["private-general"],
  "obligations": ["redact_pii", "require_citations"],
  "max_context_tokens": 12000
}
```

**Evidence bundle**

```json
{
  "bundle_id": "evb_01J...",
  "request_id": "req_01J...",
  "items": [
    {
      "evidence_id": "ev_001",
      "source_uri": "ontology://Evidence/ev_001",
      "source_version": "42",
      "retrieved_at": "2026-08-06T03:00:00Z",
      "policy_decision_id": "pol_01J...",
      "content_hash": "sha256:..."
    }
  ]
}
```

**Action proposal**

```json
{
  "action_id": "act_01J...",
  "type": "Decision.ChangeStatus",
  "target": "decision_123",
  "arguments": {
    "status": "APPROVED"
  },
  "proposed_by": "model/private-general",
  "on_behalf_of": "user_123",
  "evidence_bundle_id": "evb_01J...",
  "risk": "high",
  "approval_required": true,
  "idempotency_key": "idem_01J..."
}
```

**Audit event**

```json
{
  "event_id": "evt_01J...",
  "trace_id": "trc_01J...",
  "event_type": "ACTION_COMMITTED",
  "principal_id": "user_123",
  "policy_version": "policy-2026-08-06.1",
  "model": "private-general",
  "prompt_template_version": "decision-agent-v7",
  "action_id": "act_01J...",
  "result_hash": "sha256:...",
  "timestamp": "2026-08-06T03:00:05Z"
}
```

### Multi-agent adoption criteria

Multi-agent should be a way to **separate permissions, tools, budget, and responsibility** — not
a way to add more roles.

```mermaid
flowchart TB
    USER["User"] --> ORCH["Orchestrator"]
    ORCH --> PLAN["Planner Agent"]
    ORCH --> RET["Research Agent"]
    ORCH --> RISK["Risk Agent"]
    ORCH --> ACT["Action Agent"]
    PLAN --> BROKER["Shared Tool Broker"]
    RET --> BROKER
    RISK --> BROKER
    ACT --> BROKER
    BROKER --> POLICY["Shared Policy + Permission"]
    POLICY --> DATA["Scoped Data / Tools"]
    BROKER --> AUDIT["Shared Trace + Budget"]
    ACT --> APPROVAL["Human Approval"]
```

Conditions for adopting it: each agent has its own service principal or explicit delegation
scope; a shared Tool Broker enforces tool schema and permissions; inter-agent messages follow
data classification and retention policy too; the whole task has token/cost/time/tool-call
budgets; and the orchestrator checks evidence conflicts and permission scope before merging
results — action-taking agents get narrower tools and higher approval bars than read-only agents.

When *not* to use multi-agent: a single deterministic workflow already suffices; permission
boundaries can't be split per agent; distributed inference's cost/latency/audit complexity
outweighs the benefit; or there's no evidence graph to resolve conflicting results.

### Implementation roadmap

```mermaid
flowchart LR
    P0["Phase 0\nThreat Model + Contracts"] --> P1["Phase 1\nRead-only Assistant"]
    P1 --> P2["Phase 2\nGrounded RAG + Evidence"]
    P2 --> P3["Phase 3\nGoverned Actions"]
    P3 --> P4["Phase 4\nModel Router + Risk"]
    P4 --> P5["Phase 5\nMulti-Agent + Hardening"]
    P0 -. "policy · audit schema" .-> G0["Exit: deny-by-default"]
    P1 -. "authorized retrieval" .-> G1["Exit: cross-user isolation"]
    P2 -. "citation · freshness" .-> G2["Exit: evidence replay"]
    P3 -. "approval · idempotency" .-> G3["Exit: safe rollback"]
    P4 -. "sensitivity routing" .-> G4["Exit: no forbidden egress"]
    P5 -. "distributed tracing" .-> G5["Exit: bounded autonomy"]
```

| Phase | Scope | Exit condition |
|---|---|---|
| Phase 0 | Threat model, data classification, request/policy/audit contracts | Deny-by-default and owner sign-off |
| Phase 1 | Read-only UI, identity, permission, audit | Passes a cross-user data-isolation test |
| Phase 2 | Context Builder, RAG, citation, evidence graph | Evidence reproducibility, deletion/revocation propagation |
| Phase 3 | Tool Broker, Action Engine, human approval | Idempotency, replay protection, rollback |
| Phase 4 | Model Router, Risk Engine, egress policy | Zero forbidden external routes for restricted data |
| Phase 5 | Multi-agent, evals, operational automation | Bounded autonomy under budget/permission/tracing |

### Security test checklist

**Identity and permissions** — objects from another user/org/project never mix into search
results · fields without property permission never appear in prompts, citations, or logs ·
group removal and permission revocation propagate to caches, vector search, and sessions · agents
and background jobs never run with excess system permissions.

**Prompt and RAG** — a malicious document can't override system instructions · result count,
context tokens, and repeated queries are bounded · citations link to the actual source version
used · deleted/edited documents stop appearing via stale indexes.

**Tools and actions** — the model can't manipulate tool name/arguments past the allowed scope ·
every tool execution re-authorizes under the same principal server-side · high-risk/irreversible
actions never execute without approval · idempotency keys and nonces block replay · partial
failure triggers compensation or rollback.

**Model, network, and data egress** — a forbidden external model is never selected based on data
classification · region restrictions and egress allow-lists are enforced at the network layer ·
PII/secrets/credentials are detected and handled in prompts and output · fallback never silently
downgrades to a weaker security path · air-gapped paths block external DNS, telemetry, and update
calls.

**Logs and audit** — logs never store full raw text, secrets, tokens, or unnecessary PII · log
access permissions stay consistent with source-data permissions · request → retrieval → model →
tool → approval → result reproduces as one trace · policy/prompt/model versions are included in
audit events · deletion policy for caches/logs/backups is verified after retention expiry.

### Adopt / don't-adopt decisions

**Patterns Solomon should adopt first**

| Pattern | Why |
|---|---|
| Identity propagation | Keeps every lookup and action user-accountable |
| Ontology-style contract | Exposes data/relations/logic/actions through a bounded surface |
| Policy before retrieval and action | Keeps the LLM from becoming the security decision-maker |
| Typed tools and Actions | Separates free text from real side effects |
| Evidence Graph | Reproducibility of claim ↔ evidence ↔ version ↔ decision |
| Human Approval | Control on high-risk, irreversible work |
| End-to-end audit | Operational, security, and regulatory readiness |
| Deployment-aware routing | Model path matched to sensitivity and region |

**Areas not to replicate early**

| Area | Why |
|---|---|
| The full Foundry data platform | Scope far exceeds Solomon's core value |
| Full Apollo fleet management | Over-investment before multi-environment scale exists |
| Gotham domain applications | No need to replicate mission-specific UX in a general product |
| A large service mesh | Operational complexity and headcount cost too high |
| A complex multi-tier marking scheme | Should be introduced gradually, matched to real classification needs |
| A general-purpose no-code builder ecosystem | Scope for after the security core and product are validated |

### Areas not confirmed by public sources

Treat these as design/procurement/PoC items to verify directly, not as claimed product features:
the exact internal vector DB product and its physical placement; a concrete implementation of a
universal prompt-injection firewall across every path; whether a complete sensitivity-based
dynamic model router ships by default; a universal output DLP covering every response and tool
result; full support for every BYOM-and-marking combination; the exact fields and default
retention of session/prompt/tool logs; provider-specific data-processing paths and
sub-processor lists; feature parity across customer-hosted/on-prem/air-gapped; and the scope of
approval, rollback, and compensating transactions across every Action type.

```mermaid
flowchart LR
    DOC["Documented"] --> DEMO["Product demo"]
    DEMO --> POC["Isolated PoC"]
    POC --> RED["Red team + egress test"]
    RED --> LEGAL["Contract · DPA · region review"]
    LEGAL --> PROD["Limited production rollout"]
    PROD --> MON["Continuous eval + policy regression"]
```

### Final design principles

```mermaid
flowchart TB
    P1["1. Keep the LLM outside the trust boundary"] --> P2["2. Query data under the user's own permissions"]
    P2 --> P3["3. Give the model only the minimum context"]
    P3 --> P4["4. Re-validate tools and actions server-side"]
    P4 --> P5["5. Require human approval for high-risk actions"]
    P5 --> P6["6. Keep evidence and execution in one trace"]
    P6 --> P7["7. Constrain model and deployment path by sensitivity"]
    P7 --> P8["8. Verify Unknowns — never assume they're features"]
```

**Summary:** what's worth learning from Palantir isn't a specific model or UI — it's a
**control-plane-centric architecture** that keeps a non-deterministic model inside existing data,
permission, action, and audit systems.

Full source material (research methodology in more depth, source citations, original document
composition): private `solomon` repo, `docs/palantir-llm-security-architecture.md`.

## Features

What this gives a team end to end:

| Area | What it does |
|---|---|
| Meetings | Run/record meetings, AI persona review, extract decisions & follow-ups |
| Decisions | Structured decisions with the framework/rationale behind each one |
| Directives | Tasks generated from meetings/decisions, owner + due date tracked |
| Documents | Source documents connected as review/report input |
| Reports | Generated from tracked decisions/directives, traceable to source |
| Agents | AI personas that assist review and can be published for reuse |
| Dashboard | Workspace-level signal & activity overview |

## Tech stack

| Layer | Stack |
|---|---|
| Client | Flutter, Dart, Riverpod (state/DI), Clean Architecture |
| Server | Rust, Axum (HTTP), sqlx (Postgres), Tokio |
| Auth | Supabase Auth (JWT), PKCE flow on the client |
| Data | Postgres |
| AI | LLM-backed meeting personas, RAG over connected documents |

## Repositories

| Repo | What it is |
|---|---|
| [`solomon`](https://github.com/Solomon-Platform/solomon) | Monorepo: Flutter client + Rust Cloud API (private) |
| [`solomon-workspace-ui`](https://github.com/Solomon-Platform/solomon-workspace-ui) | Offline UI shell extracted from the client — no backend, in-memory fixtures only |
| [`rfc8785-jcs`](https://github.com/Solomon-Platform/rfc8785-jcs) | RFC 8785 JSON Canonicalization (Rust) — the hashing primitive behind the content-addressed state transitions above |
| [`pg-lease-queue`](https://github.com/Solomon-Platform/pg-lease-queue) | Postgres job queue (Rust) — the claim/lease/heartbeat pattern behind the worker's safe-under-concurrency job processing above |
| [`openrouter_dart`](https://github.com/Solomon-Platform/openrouter_dart) | OpenRouter client (Dart) — tool-calling loop with parallel execution + reasoning-model recovery, plus a Tavily search client |
| [`jwks-verifier`](https://github.com/Solomon-Platform/jwks-verifier) | JWKS JWT verifier (Rust) — the auth boundary in front of the API above, framework-agnostic |
| [`idempotency-key`](https://github.com/Solomon-Platform/idempotency-key) | Idempotency-Key handling (Rust) — the header validation/request-hash pair behind the API's mutation endpoints |
| [`rubric-loop`](https://github.com/Solomon-Platform/rubric-loop) | Ten Rust CLIs for domain-specific content generation/review (SEO, GEO, ASO, marketing copy, PR review, app-store creative, icon design, business plans, secret-scan triage) — standalone tooling, not part of the Solomon product |
