# DRIS Skill Graph

> Interactive version: [**skill-graph.html**](./skill-graph.html) · Data index: [**SKILL_GRAPH.json**](./SKILL_GRAPH.json)

## Overview — All Clusters

```mermaid
graph LR
    subgraph DOM["🟡 Domain Knowledge"]
        CSR[CSR Compliance India]
        NGO[NGO Operations]
        DSG[Dev Sector Grants]
        GMO[Grant Mgmt Ops]
        FCRA[FCRA Compliance]
        DMS[Data & MIS Design]
    end

    subgraph DES["🟢 Design Skills"]
        FD[Frontend Designer]
        UXR[UX Reviewer]
        AA[Accessibility Auditor]
        RCP[React Patterns]
        FUD[Frappe UI Design]
        SA[System Architecture]
        F2P[Figma → PowerBI]
    end

    subgraph PRD["🔴 Product Workflows"]
        BA[Business Analyst]
        FS[Feature Spec]
        TPM[Technical PM]
        BIS[BRD → Impl Spec]
        FDT[Frappe DocType]
        PR[PR Raiser]
        RN[Release Notes]
        ED[Email Drafter]
        TK[Ticketing]
        SS["🔶 Skill Selector"]
        MGD[mGrant Donor]
        MGC[mGrant CSR]
        MGN[mGrant NuO]
        MBU[mGrant Bulk Upload]
        MST[mGrant Setup]
        PD[Product Docs]
        WU[Weekly Update]
    end

    subgraph BLD["🔵 Builder / Technical"]
        FRD[Frappe Development]
        QA[QA Testing]
        PMW[PM Workflow]
        ANG[Angular + Node]
        MDB[MongoDB Patterns]
        MSV[Microservices]
        DLT[Load Testing]
        DSC[Scheduling]
        GHA[GitHub Analytics]
        PFP[Postgres + Frappe]
        GCC[Tech CSR Compliance]
        KB[Knowledge Base]
        DW[Doc Workflow]
    end

    subgraph TPL["⚪ Templates & Guides"]
        CT[CLAUDE.md Template]
        ET[Evals Template]
        SPT[Spec Template]
        TR[Tooling Reference]
        VCG[Vibe Coding Guide]
        SAS[Skill Authoring]
    end

    %% Key cross-cluster edges
    BA -->|golden path| FS
    FS -->|golden path| TPM
    TPM -->|golden path| BIS
    BIS -->|golden path| FDT
    FDT -->|golden path| FRD
    FRD -->|golden path| QA
    QA -->|golden path| PR
    PR -->|golden path| RN
    RN -->|golden path| ED

    MGD -.->|requires| GMO
    MGD -.->|requires| CSR
    MGC -.->|requires| CSR
    MGN -.->|requires| FCRA
    MGN -.->|requires| NGO
    BIS -.->|requires| SA

    classDef domain fill:#FEF3C7,stroke:#C8991F,color:#78350F
    classDef design fill:#CCFBF1,stroke:#2D6A6A,color:#134E4A
    classDef product fill:#FEE2E2,stroke:#8B1A1A,color:#7F1D1D
    classDef builder fill:#DBEAFE,stroke:#1E40AF,color:#1E3A5F
    classDef template fill:#F3F4F6,stroke:#9CA3AF,color:#374151
    classDef router fill:#FEF3C7,stroke:#C8991F,stroke-width:3px,color:#78350F

    class CSR,NGO,DSG,GMO,FCRA,DMS domain
    class FD,UXR,AA,RCP,FUD,SA,F2P design
    class BA,FS,TPM,BIS,FDT,PR,RN,ED,TK,MGD,MGC,MGN,MBU,MST,PD,WU product
    class SS router
    class FRD,QA,PMW,ANG,MDB,MSV,DLT,DSC,GHA,PFP,GCC,KB,DW builder
    class CT,ET,SPT,TR,VCG,SAS template
```

---

## ⭐ The Golden Path — Client Call to Shipped Code

```mermaid
graph TD
    START((Client Call)) --> BA["1️⃣ Business Analyst<br/><i>BRD + Requirements Matrix</i>"]
    BA --> FS["2️⃣ Feature Spec<br/><i>.docx spec</i>"]
    FS --> TPM["3️⃣ Technical PM<br/><i>Solution architecture (COVE)</i>"]
    TPM --> BIS["4️⃣ BRD → Impl Spec<br/><i>Claude Code-ready tech spec</i>"]
    BIS --> CTX{{"Load Domain Context<br/><i>mGrant Donor / CSR / NuO<br/>+ compliance skills</i>"}}
    CTX --> FDT["5️⃣ Frappe DocType<br/><i>JSON + workflows</i>"]
    FDT --> FRD["6️⃣ Frappe Development<br/><i>Production code</i>"]
    FRD --> DRC{{"Design Review Chain<br/><i>Frontend Designer → UX Reviewer<br/>→ Accessibility Auditor</i>"}}
    DRC --> QA["7️⃣ QA Testing<br/><i>E2E + POM</i>"]
    QA --> PR["8️⃣ PR Raiser<br/><i>GitHub PR</i>"]
    PR --> RN["9️⃣ Release Notes<br/><i>Internal + customer</i>"]
    RN --> ED["🔟 Email Drafter<br/><i>Client update</i>"]
    ED --> DONE((Shipped ✓))

    style START fill:#C8991F,stroke:#8B1A1A,color:white
    style DONE fill:#16a34a,stroke:#166534,color:white
    style CTX fill:#FEF3C7,stroke:#C8991F,color:#78350F
    style DRC fill:#CCFBF1,stroke:#2D6A6A,color:#134E4A
    style BA fill:#FEE2E2,stroke:#8B1A1A,color:#7F1D1D
    style FS fill:#FEE2E2,stroke:#8B1A1A,color:#7F1D1D
    style TPM fill:#FEE2E2,stroke:#8B1A1A,color:#7F1D1D
    style BIS fill:#FEE2E2,stroke:#8B1A1A,color:#7F1D1D
    style FDT fill:#FEE2E2,stroke:#8B1A1A,color:#7F1D1D
    style FRD fill:#DBEAFE,stroke:#1E40AF,color:#1E3A5F
    style QA fill:#DBEAFE,stroke:#1E40AF,color:#1E3A5F
    style PR fill:#FEE2E2,stroke:#8B1A1A,color:#7F1D1D
    style RN fill:#FEE2E2,stroke:#8B1A1A,color:#7F1D1D
    style ED fill:#FEE2E2,stroke:#8B1A1A,color:#7F1D1D
```

---

## Domain Knowledge Cluster (6 skills)

```mermaid
graph LR
    CSR["CSR Compliance<br/>India"] <-->|cross-ref| FCRA["FCRA<br/>Compliance"]
    CSR <-->|cross-ref| GMO["Grant Mgmt<br/>Ops"]
    CSR <-->|cross-ref| NGO["NGO<br/>Operations"]
    CSR <-->|cross-ref| DSG["Dev Sector<br/>Grants"]
    FCRA <-->|cross-ref| NGO
    FCRA <-->|cross-ref| GMO
    DSG <-->|cross-ref| GMO
    DSG <-->|cross-ref| DMS["Data & MIS<br/>Design"]
    NGO <-->|cross-ref| DMS
    GMO <-->|cross-ref| DMS

    classDef dom fill:#FEF3C7,stroke:#C8991F,color:#78350F
    class CSR,FCRA,GMO,NGO,DSG,DMS dom
```

---

## Design Skills Cluster (7 skills)

```mermaid
graph LR
    RCP["React<br/>Patterns"] -->|enhances| FD["Frontend<br/>Designer"]
    FD -->|hands off to| UXR["UX<br/>Reviewer"]
    UXR -->|hands off to| AA["Accessibility<br/>Auditor"]
    AA -->|enhances| FUD["Frappe UI<br/>Design"]
    FD -->|hands off to| F2P["Figma →<br/>PowerBI"]
    UXR -->|enhances| F2P
    SA["System<br/>Architecture"] -->|hands off to| BIS["BRD → Impl Spec<br/>(Product cluster)"]

    classDef des fill:#CCFBF1,stroke:#2D6A6A,color:#134E4A
    classDef prd fill:#FEE2E2,stroke:#8B1A1A,color:#7F1D1D
    class FD,UXR,AA,RCP,FUD,SA,F2P des
    class BIS prd
```

---

## Entry Points by Role

| I am a… | Start with | What it does |
|---------|-----------|-------------|
| **Product Manager** | `dris-feature-spec` or `dris-skill-selector` | Spec a feature or get routed to the right skill |
| **Business Analyst** | `dris-business-analyst` | Structure client calls/emails into BRD |
| **Engineer** | `frappe-development` or `brd-to-implementation-spec` | Build from specs with battle-tested patterns |
| **QA Engineer** | `qa-testing` or `ux-reviewer` | E2E testing or UI quality review |
| **Designer** | `frontend-designer` or `figma-to-powerbi` | Build UI or convert Figma → PowerBI |
| **Leadership** | `dris-skill-selector` | Understand the ecosystem |

---

## Contributors

| Person | Affiliation | Contribution |
|--------|------------|-------------|
| **Affaan Mustafa** | Community | Everything Claude Code — 120+ base skills, hooks, commands, agents |
| **Ankit Jangir** | Dhwani RIS | 13 technical skills, 2 Frappe agents, 5 rules, safety guardrails |
| **Swapnil Agarwal** | Dhwani RIS | BRD-to-implementation-spec, spec template, tooling reference |
| **Nihaan Mohammed** | Dhwani RIS | 28 skills — domain (6), design (6), product workflows (16) |

---

*For the interactive visualization, see [`docs/skill-graph.html`](./skill-graph.html). For the machine-readable index, see [`docs/SKILL_GRAPH.json`](./SKILL_GRAPH.json).*
