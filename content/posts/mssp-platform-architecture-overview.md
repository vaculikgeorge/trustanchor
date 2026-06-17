+++
title = 'MSSP platform — the high-level scheme'
date = '2026-06-05T11:00:00+02:00'
draft = true
tags = ['mssp', 'architecture', 'platform', 'identity-governance', 'azure-landing-zones', 'm365-tenant-config']
categories = ['Architecture']
summary = 'A reference MSSP platform — a multi-tenant, multi-customer, regulated-cloud delivery vehicle for Microsoft Entra + Azure + M365 governance. This post is the high-level map — one Mermaid diagram with ~32 components stacked in five layers (Delivery / Substrate / Identity / Security operations / Assurance) and the load-bearing dependencies between them. Every other post on this blog deep-dives one node; this one is the territory each of them sits on.'
+++

> The reference platform is one diagram. Every post on this blog deep-dives one node.

## TL;DR

The reference platform behind these posts is an MSSP-class delivery vehicle for Microsoft cloud governance — Entra, Azure, M365, Defender, Sentinel, ADO pipelines, Terraform delivery, policy-as-code, observability — designed to onboard multiple regulated customers into a shared substrate without losing per-customer isolation. It builds on Microsoft's Azure Landing Zone (ALZ) and Cloud Adoption Framework (CAF) reference architectures and extends them with the multi-tenant separation, supply-chain isolation, and detection-as-code patterns that an MSSP context demands.

This post is the **map**. The diagram below has ~32 components stacked into five layers — Delivery (everything-as-code that builds the platform) / Azure substrate (what exists in customer tenants) / Identity & Access (governance over who can act on the substrate) / Security operations (observability + response over the first three) / Assurance (validation that the first four meet declared standards). Every other post on this blog covers one slice — I-1 (PIM at scale) is a deep-dive into the PIM node; I-2 (JIT ADO access) drills into the auth-context binding between PIM and CA; S-1 (no-PAT agents) walks the Delivery-layer node for the self-hosted ADO agents; and so on. Read this post first, then jump to whichever slice-post matches what you're working on; this one stays useful as a "where am I" reference whenever a slice-post references a component from another layer.

The post is intentionally **sanitised and principle-level**. No customer-specific data, no internal management-group names, no role-list specifics. Those land in the slice-posts. This post is the architectural pattern, not the deployed instance.

## The high-level scheme

{{< mermaid >}}
flowchart BT
    subgraph delivery["Layer 1 — Delivery (everything-as-code)"]
        direction LR
        ET["Engagement Template<br/>DOCX + JSON Schema v3.4"]
        COB["Platform Delivery Pipeline<br/>multi-stage unified"]
        ADO["Azure DevOps<br/>two organizations + Backup Chain"]
        AGSH["Self-hosted agents<br/>ACI + UAMI + WIF"]
        AGMS["Microsoft-hosted agents"]
        TF["Terraform delivery<br/>Identity / Platform / Workload"]
        SUBV["Subscription Vending<br/>lz-vending"]
        EPAC["EPAC<br/>policy-as-code, dual EPAC"]
        AMBA["AMBA-ALZ<br/>alert baseline + Action Groups"]
        CFM["Compliance Framework Mapping<br/>~12 frameworks"]
        ET --> COB
        COB --> ADO
        ADO --> AGSH
        ADO --> AGMS
        AGSH --> TF
        AGMS --> TF
        COB --> SUBV
    end

    subgraph substrate["Layer 2 — Azure substrate"]
        direction LR
        MG["MG hierarchy<br/>Platform / LZs / Sandbox / Decommissioned"]
        CONF["Confidential Isolated MG<br/>L3 sovereign"]
        HUB["Hub and Spoke<br/>Firewall + Bastion + DDoS + DNS + AVNM"]
        GSA["Global Secure Access<br/>Internet + Private Access"]
        LH["Azure Lighthouse<br/>cross-tenant management"]
        COST["Cost Management<br/>billing + budgets + anomaly"]
        HSM["Managed HSM<br/>FIPS 140-3 L3, customer-managed keys"]
        MG --> CONF
        HSM --> CONF
    end

    subgraph identity["Layer 3 — Identity and Access"]
        direction LR
        PIM["PIM<br/>3 management areas"]
        CA["Conditional Access<br/>per-persona policy set, auth contexts"]
        AS["Authentication Strengths<br/>and AAGUID allow-list"]
        AMP["Authentication Methods policy<br/>tenant allow-list"]
        AU["Administrative Units"]
        PAW["Privileged Access Workstations"]
        AMP --> AS
        AS --> CA
        PIM -->|step-up claimValue| CA
        AU --> CA
        PAW --> CA
    end

    subgraph secops["Layer 4 — Security operations"]
        direction LR
        SENT["Microsoft Sentinel<br/>central Tier-2 workspace"]
        WM["Workspace Manager<br/>parent to member"]
        DL["Sentinel Data Lake<br/>Tier 3, up to 12yr"]
        DFC["Defender for Cloud<br/>MCSB v2 baseline"]
        DXDR["Defender XDR<br/>Unified RBAC"]
        LOG["Three-tier logging<br/>Tier 1 / 2 / 3"]
        LOG --> SENT
        SENT --> WM
        SENT --> DL
    end

    subgraph assurance["Layer 5 — Assurance"]
        direction LR
        UTCM["UTCM<br/>Microsoft Graph beta"]
        MAES["Maester<br/>test framework"]
        ZTA["Zero Trust Assessment<br/>200+ tests"]
    end

    TF --> MG
    SUBV --> MG
    TF --> HUB
    TF --> HSM
    TF --> LH
    TF --> COST
    EPAC --> MG
    AMBA -->|DINE| MG

    ADO -->|Graph + ARM| PIM
    ADO -->|Graph + ARM| CA
    ADO -->|Graph| AMP
    ADO -->|Graph| AU

    GSA -->|network gate| CA

    MG -->|diag settings + DINE| LOG
    DFC -->|compliance read| CFM
    DXDR -->|Unified RBAC scope| MG

    MAES --> SENT
    ZTA --> SENT
    ZTA --> DFC
    UTCM -.->|reads config| MG
    UTCM -.->|reads config| PIM

    MAES -.->|findings inform next delivery wave| ET
    ZTA -.->|findings inform next delivery wave| ET

    classDef futureCls stroke-dasharray: 5 5,stroke-width:3px
    class GSA,UTCM,WM,DL futureCls

    classDef deliveryCls fill:#14532d,stroke:#22c55e,stroke-width:2px,color:#f1f5f9
    classDef substrateCls fill:#1e293b,stroke:#94a3b8,stroke-width:2px,color:#f1f5f9
    classDef identityCls fill:#1e3a8a,stroke:#60a5fa,stroke-width:2px,color:#f1f5f9
    classDef secopsCls fill:#7f1d1d,stroke:#ef4444,stroke-width:2px,color:#f1f5f9
    classDef assuranceCls fill:#581c87,stroke:#a855f7,stroke-width:2px,color:#f1f5f9

    class ET,COB,ADO,AGSH,AGMS,TF,SUBV,EPAC,AMBA,CFM deliveryCls
    class MG,CONF,HUB,GSA,LH,COST,HSM substrateCls
    class PIM,CA,AS,AMP,AU,PAW identityCls
    class SENT,WM,DL,DFC,DXDR,LOG secopsCls
    class UTCM,MAES,ZTA assuranceCls

    style delivery stroke:#22c55e,stroke-width:2px,color:#22c55e,fill:#0f172a
    style substrate stroke:#94a3b8,stroke-width:2px,color:#94a3b8,fill:#0f172a
    style identity stroke:#60a5fa,stroke-width:2px,color:#60a5fa,fill:#0f172a
    style secops stroke:#ef4444,stroke-width:2px,color:#ef4444,fill:#0f172a
    style assurance stroke:#a855f7,stroke-width:2px,color:#a855f7,fill:#0f172a
{{< /mermaid >}}

*Nodes with dashed borders (GSA, UTCM, Workspace Manager, Sentinel Data Lake) are documented as design-intent or Microsoft Preview, not deployed in production tenants today. Everything else is deployed. The diagram reads bottom-up: **Delivery → Substrate → Identity → Security operations → Assurance**, with a dashed feedback edge from Assurance back to the Engagement Template — assurance findings drive the next delivery wave.*

## Reading the scheme

The diagram is a **five-layer stack**. Each layer is a distinct *kind* of thing: artefacts that build the platform, the platform itself, the identity controls that gate access to it, the security operations that observe it, the assurance frameworks that validate it. Arrows flow bottom-up — Delivery produces Substrate, Substrate is governed by Identity and emits to Security operations, and Assurance closes the cycle by feeding findings back into the next Delivery wave.

- **Layer 1 — Delivery (bottom, ten nodes).** Everything-as-code that produces platform state. The Engagement Template (DOCX + JSON Schema) drives the platform delivery pipeline, a multi-stage unified pipeline that orchestrates Azure DevOps across two organizations (a platform management org for customer Terraform delivery and a supply-chain-isolated platform security org for detections + the shared-library repo). The pipeline runs on a mix of self-hosted agents (ACI + UAMI + WIF for sovereign workloads) and Microsoft-hosted agents (parallel pool for non-sovereign workloads). Terraform delivery covers three domains — Identity / Platform / Workload. Subscription Vending (`lz-vending`) creates per-customer subscriptions inline with the pipeline. EPAC and AMBA-ALZ are the policy and alert baselines as code; Compliance Framework Mapping ties Defender for Cloud regulatory dashboards to ~12 frameworks (CIS / ISO 27001 / SOC 2 / NIS2 / PCI DSS / GDPR / NIST 800-53 / HITRUST / CMMC / DORA / EU AI Act / NIST CSF). None of these artefacts live inside customer tenants — they live in the platform ADO organizations and *produce* tenant state.
- **Layer 2 — Azure substrate (seven nodes).** What exists in customer tenants after delivery runs. The MG hierarchy (Platform MGs / per-customer Landing Zone MGs / Sandbox / Decommissioned) is the structural backbone; the Confidential Isolated MG sits alongside it for L3 sovereign workloads. Hub & Spoke networking (per-customer hubs carrying Firewall + Bastion + DDoS + Private DNS, with AVNM for large fleets) is the connectivity layer. Global Secure Access (GSA) is the future-state network identity — Internet Access + Private Access, dashed-bordered in the diagram because it's design-intent, not yet universal substrate. Azure Lighthouse provides cross-tenant management for customer-owned tenants. Cost Management runs per-customer billing + budgets + anomaly detection. Managed HSM (FIPS 140-3 Level 3) holds the customer-managed keys for sovereign L2/L3 encryption.
- **Layer 3 — Identity & Access (six nodes).** Cross-cutting governance over who can act on Layer 2. PIM operates in three management areas (Entra directory roles, Azure resource roles, PIM-for-Groups) and carries `claimValue` references that bind activations to Conditional Access auth contexts (the internal-user PIM activation context is the reference platform's primary step-up gate). CA's per-persona framework evaluates every sign-in through a triple-gate of device + network + authentication strength, across auth contexts for Protected Actions, internal-user PIM activation, and guest step-up. The Authentication Methods policy sits underneath — a tenant-level allow-list of credential types (FIDO2, WHfB, MS Authenticator, TAP), with weak methods like SMS and voice disabled at the floor. Authentication Strengths layer the more-restrictive grant control on top of that floor, and surface the FIDO2 AAGUID allow-list (hardware-attested keys only for the most sensitive contexts). Administrative Units scope CA + PIM for delegated administration. Privileged Access Workstations (PAW) are the hardware prerequisite for privileged personas — registration on a non-PAW device is blocked for the admin tier. *How* these primitives actually compose at runtime — registration journey, CA evaluation, PIM step-up, WIF — lives in the Access Flow companion diagram below.
- **Layer 4 — Security operations (six nodes).** Observation + response surface over Layers 1–3. Microsoft Sentinel is the central Tier-2 security workspace; Workspace Manager (Microsoft Preview, future-state) pushes content one-way to per-customer member workspaces; Sentinel Data Lake (future-state Tier 3, up to 12-year retention) handles long-term audit storage. Defender for Cloud holds the MCSB v2 baseline and the compliance dashboard that Compliance Framework Mapping populates. Defender XDR is the cross-product detection plane with Unified RBAC (scoped to MGs via the Layer 2 substrate). Three-tier logging takes signals via DINE-deployed diagnostic settings — Tier 1 per-LZ Log Analytics, Tier 2 central, Tier 3 Data Lake.
- **Layer 5 — Assurance (top, three nodes).** Validation that Layers 1–4 meet declared standards. UTCM (Unified Tenant Configuration Management, Microsoft Graph beta, future-state) reads tenant config across MGs + Entra for drift detection. Maester is a test framework for tenant security posture (200+ assertions across CA / PIM / authentication / device controls); Zero Trust Assessment is a fork with 200+ Zero Trust-specific tests. Both feed findings into Sentinel and DfC, and the dashed feedback edges to the Engagement Template close the cycle — assurance findings inform the next Delivery wave.

**Read it as a cycle:** Delivery writes Substrate; Substrate is gated by Identity and observed by Security operations; Assurance scores all four; findings flow back into the next Engagement Template revision and the next Delivery run. The platform is operated by walking this cycle, not by walking the layers individually.

## Access flow: registration, human, pipeline

The master diagram shows the Layer 3 *components* — PIM, CA, Authentication Strengths, Authentication Methods policy, AUs, PAW — sitting alongside one another. This companion diagram shows the **runtime sequence** in which those components actually compose: how a method gets registered onto a user, how a human sign-in passes through the CA + PIM stack to reach a resource, and how a pipeline run reaches the same resource via Workload Identity Federation without touching the human-access stack at all. Three lanes, converging at the same Layer 2 target on the right.

{{< mermaid >}}
flowchart LR
    subgraph registration["Registration lane"]
        direction TB
        ONB["User onboarded<br/>persona assigned"]
        AMP_R["Authentication Methods policy<br/>tenant allow-list<br/>(FIDO2, WHfB, MS Auth, TAP)"]
        PAW_R["PAW required<br/>for privileged personas"]
        REG["User registers method<br/>aka.ms/mysecurityinfo"]
        BIND["Method bound to user"]
        ONB --> AMP_R
        AMP_R --> PAW_R
        PAW_R --> REG
        REG --> BIND
    end

    subgraph human["Human-access lane"]
        direction TB
        USER["Operator or Admin"]
        PORTAL["Portals<br/>Azure, Entra, Defender,<br/>Intune, Purview, ADO Web"]
        SIGNIN["Entra ID sign-in"]
        CA_USER["CA for users<br/>per-persona policy set<br/>triple-gate device + network + auth strength"]
        AS_GATE["Auth strength enforces<br/>AAGUID allow-list<br/>for FIDO2"]
        STEPUP["Step-up auth context<br/>if privileged"]
        PIM_ACT["PIM activation<br/>JIT, time-bound, MFA-bound"]
        USERTOKEN["User access token<br/>JIT-elevated"]
        USER --> PORTAL
        PORTAL --> SIGNIN
        SIGNIN --> CA_USER
        CA_USER --> AS_GATE
        AS_GATE --> STEPUP
        STEPUP --> PIM_ACT
        PIM_ACT --> USERTOKEN
    end

    subgraph machine["Pipeline-access lane"]
        direction TB
        PIPELINE["ADO pipeline run<br/>YAML, repo-scoped"]
        AGENT2["Self-hosted or<br/>Microsoft-hosted agent"]
        OIDC["ADO OIDC token<br/>issuer vstoken.dev.azure.com<br/>subject sc://org/project/sc-name"]
        ENTRAEXCH["Entra ID token exchange<br/>audience api://AzureADTokenExchange"]
        SPTOKEN["UAMI or SP token<br/>scoped to target MG/sub"]
        PIPELINE --> AGENT2
        AGENT2 --> OIDC
        OIDC --> ENTRAEXCH
        ENTRAEXCH --> SPTOKEN
    end

    BIND -.->|enables sign-in| USER

    USERTOKEN --> RES["Layer 2 resource<br/>MG / subscription / resource"]
    SPTOKEN --> RES

    classDef regCls fill:#78350f,stroke:#f59e0b,stroke-width:2px,color:#f1f5f9
    classDef humanCls fill:#1e3a8a,stroke:#60a5fa,stroke-width:2px,color:#f1f5f9
    classDef machineCls fill:#14532d,stroke:#22c55e,stroke-width:2px,color:#f1f5f9
    classDef gateCls fill:#7f1d1d,stroke:#ef4444,stroke-width:2px,color:#f1f5f9
    classDef targetCls fill:#1e293b,stroke:#94a3b8,stroke-width:3px,color:#f1f5f9

    class ONB,AMP_R,PAW_R,REG,BIND regCls
    class USER,PORTAL,SIGNIN,USERTOKEN humanCls
    class PIPELINE,AGENT2,OIDC,ENTRAEXCH,SPTOKEN machineCls
    class CA_USER,AS_GATE,STEPUP,PIM_ACT gateCls
    class RES targetCls

    style registration stroke:#f59e0b,stroke-width:2px,color:#f59e0b,fill:#0f172a
    style human stroke:#60a5fa,stroke-width:2px,color:#60a5fa,fill:#0f172a
    style machine stroke:#22c55e,stroke-width:2px,color:#22c55e,fill:#0f172a
{{< /mermaid >}}

**The registration lane.** Before a human can authenticate, a credential has to land on their account. The Authentication Methods policy at the tenant level sets the *floor* — SMS, voice, software TOTP are disabled tenant-wide; only the strong methods (FIDO2 hardware keys, Windows Hello for Business, Microsoft Authenticator, Temporary Access Pass for bootstrapping) are eligible. For privileged personas the registration *must* happen on a Privileged Access Workstation; CA registration policies block non-PAW devices for the admin tier. Once the method is bound to the user, that user can begin the Human-access lane.

**The human-access lane.** An operator opens a browser, lands on a Microsoft portal (Azure, Entra admin centre, Defender, Intune, Purview, or the ADO web UI), and Entra signs them in. The full Conditional Access framework applies — the triple-gate of device + network + authentication strength, plus auth-context evaluation (Protected Actions, internal-user PIM activation, or guest step-up) if the operator is activating a PIM-eligible role. The Authentication Strengths gate enforces the FIDO2 AAGUID allow-list for the most sensitive contexts — only hardware-attested keys from the approved manufacturer list satisfy the grant. Only after CA grants and PIM JIT-elevates does the operator's access token reach the Layer 2 resource. Every portal click is a user-CA event.

**The pipeline path.** An ADO pipeline run gets a pipeline-scoped OIDC token from `vstoken.dev.azure.com/<org-id>` with subject `sc://<org>/<project>/<service-connection>`. The self-hosted agent (UAMI on ACI) presents that token to Entra ID at the audience `api://AzureADTokenExchange`. Entra validates the federated credential (issuer + subject + audience must all match the configured `federatedIdentityCredential`) and issues an access token for the underlying identity — a service principal or a user-assigned managed identity. That access token then drives the ARM / Graph call. No password, no secret, no PAT anywhere in the chain.

**The CA-coverage gap is real.** Conditional Access for users is the full triple-gate with auth-context binding. Conditional Access for *workload identities* is much narrower: only single-tenant service principals can be targeted, the conditions are limited to location and Entra ID Protection risk, the grant control is **block-only** (workload identities can't satisfy MFA / device controls), and **managed identities are exempt from Conditional Access entirely**. That last point matters: the platform's self-hosted-agent identities are UAMIs, so CA does *not* gate them. The gating comes from elsewhere — RBAC scope, the federated-credential trust (issuer + subject + audience must match), the ADO pipeline approvals/checks layered on the service connection, and the OIDC token's pipeline-scoped subject claim. Workload identity governance lives in the deployment plane (RBAC + WIF setup + ADO approvals), not in the identity plane.

**Why this matters for the reference platform.** Read the master diagram with both paths in mind. Portal-driven changes (one-off admin actions, platform delivery stage approvals, PIM activations, break-glass) flow **USER → portal → user-CA → PIM → ARM/Graph**. Pipeline-driven changes (Terraform applies, EPAC desired-state runs, Workspace Manager content pushes, KQL deployments) flow **PIPELINE → agent → OIDC → Entra exchange → ARM/Graph**. The same platform components on the right of the diagram receive both. Sensitive admin operations are routed through portals on purpose, so the triple-gate plus PIM activation can apply. Bulk-deploy, drift-reconciliation, and continuous-enforcement run through WIF on purpose, so no human credential or stored secret is involved — and so the change is reproducible from version control.

**Sources:** [Workload identity federation for Azure DevOps](https://learn.microsoft.com/azure/devops/pipelines/release/configure-workload-identity?view=azure-devops), [Access Azure DevOps with Microsoft Entra workload identity](https://learn.microsoft.com/azure/devops/pipelines/library/add-devops-entra-service-connection?view=azure-devops), [Conditional Access for workload identities](https://learn.microsoft.com/entra/identity/conditional-access/workload-identity).

## Per-node detail

The master diagram has expanded to ~32 components, but only 15 carry a full sub-diagram below. The remaining nodes — the platform delivery pipeline, Cost Management, Maester, Zero Trust Assessment, Subscription Vending, Compliance Framework Mapping, Managed HSM, Administrative Units, Microsoft-hosted agents, Authentication Methods policy, AAGUID, and PAW — are introduced by short paragraphs at the end of this section rather than full sub-diagrams. Each of those will get its own slice-post over time; this map names them so the diagram and the future deep-dives line up.

Each of the 15 sub-diagrammed components below carries a TB Mermaid showing its internal pieces, a 2–4-paragraph explanation, and a pointer to the slice-post that deep-dives the node.

### 1. PIM

{{< mermaid >}}
flowchart TB
    PIM["PIM"]
    PIM --> ENT["Entra directory roles<br/>(Microsoft Graph)"]
    PIM --> AZR["Azure resource roles<br/>(Azure ARM)"]
    PIM --> P4G["PIM-for-Groups<br/>(Microsoft Graph)"]
    ENT --> RULES["Rule types per policy:<br/>Enablement / Expiration / Approval<br/>+ AuthenticationContext"]
    AZR --> RULES
    P4G --> ESR["EligibilityScheduleRequest<br/>(eligible member / eligible owner)"]
    P4G --> RULES
    RULES --> TIER["Tier-0 / Tier-1 / Tier-2<br/>(activation duration + approval)"]
{{< /mermaid >}}

PIM is the JIT identity primitive in three management areas — Entra directory roles, Azure resource roles, and PIM-for-Groups. Each management area has its own `roleManagementPolicy` shape (Graph for Entra and Groups; ARM for Azure resources), and each policy is composed of sibling rule objects: enablement (which controls fire at activation — MFA, justification), expiration (how long the active assignment can persist), approval (whether activation routes through an approver), and the authentication-context binding (which Conditional Access context the activation request claims).

The tiered model (Tier-0 / Tier-1 / Tier-2) is the reference platform's way of normalising activation policy across the Entra directory roles, every (role × scope) tuple on the Azure side, and both sides (Member + Owner) of every role-assignable group. All three management areas share one auth-context binding, so all activations clear the same Conditional Access evaluation — the per-tier difference is only duration and approval, not the gate.

PIM-for-Groups extends the JIT pattern to anything that consumes group claims for authorisation — Defender XDR Unified RBAC roles assigned to groups, Azure DevOps project access evaluated against group membership, third-party SaaS that consumes Entra group claims. The eligible-member / eligible-owner pattern (governed symmetrically — see I-1's Member-vs-Owner trap) extends the policy spine into those downstream consumers without duplicating the activation primitive.

*Covered in depth in [I-1](/posts/pim-at-scale-three-domains/), [I-5 (forthcoming)](/posts/enforce-ca-on-pim-activation/), [I-6 (forthcoming)](/posts/role-assignable-pim-groups-security-boundary/).*

### 2. Conditional Access

{{< mermaid >}}
flowchart TB
    CA["Conditional Access framework"]
    CA --> PER["Per-persona policy ranges<br/>CA-GLOBAL / CA-ADMIN / CA-INTERNAL / CA-WORKLOAD / CA-GSA / CA-AGENT / CA-GUEST / CA-BREAKGLASS"]
    CA --> CTX["Auth contexts<br/>Protected Actions / step-up PIM / guest step-up"]
    CA --> STR["Authentication strengths<br/>hardware-attested / phishing-resistant / registration / guest variants"]
    CTX --> GATE["Triple-gate per context<br/>device + network + strength"]
    PER --> POL["Per-persona policy set<br/>baseline + admins + internals + workloads + guests"]
    POL --> RES["Conflict resolution<br/>Block beats grant; auth strength beats MFA; shortest SIF"]
{{< /mermaid >}}

Conditional Access is the gate every privileged operation passes through, but the framework is what makes it scale. Per-persona ranges — CA-GLOBAL, CA-ADMIN, CA-INTERNAL, CA-WORKLOAD, CA-GSA, CA-AGENT (the new AI-agent identity primitive), CA-GUEST, CA-BREAKGLASS — cover every sign-in surface the reference platform has to govern. The guest range is templated: a set of policies cloned per customer with the named location and authentication strength substituted per customer.

This worked example is distilled from real multi-customer MSSP delivery, generalized here rather than any single customer's deployment.

The three authentication contexts — Protected Actions for sensitive admin operations, internal-user PIM activation, and guest-side step-up (used for both guest lifecycle actions and guest PIM activation) — each trigger a **triple-gate**: a device control (block non-compliant or non-managed devices), a network control (block off-trusted-network — GSA where available, named locations where not), and a credential control (require a specific authentication strength). Three policies per context, evaluated as a block-wins-on-conflict, auth-strength-overrides-MFA, shortest-sign-in-frequency-wins set of rules.

The authentication-strength catalog underneath spans several strengths — a hardware-attested strength (FIDO2 hardware-attested only, via an AAGUID allow-list), a phishing-resistant strength (WHfB + FIDO2 + x509), and registration and guest strengths for cases where a hardware key isn't operationally viable. The Authentication Methods policy at the tenant level sets the floor (SMS, voice, software TOTP are disabled at the tenant); CA Authentication Strengths layer the more-restrictive grant control on top.

*Covered in depth in [I-4 (forthcoming)](/posts/auth-contexts-in-practice/), [I-8 (forthcoming)](/posts/auth-strengths-vs-auth-contexts/), [I-9 (forthcoming)](/posts/protected-actions-in-entra/), [I-10 (forthcoming)](/posts/persona-based-ca-at-scale/), [I-11 (forthcoming)](/posts/ca-for-ai-agent-identity/), [I-13 (forthcoming)](/posts/fido2-aaguid-attestation-whitelisting/).*


### 3. Hub & Spoke network

{{< mermaid >}}
flowchart TB
    HUB["Hub VNet<br/>(per customer)"]
    HUB --> FW["Azure Firewall<br/>+ Firewall Policy"]
    HUB --> BAS["Azure Bastion"]
    HUB --> DDOS["DDoS Protection Plan"]
    HUB --> DNS["Private DNS zones<br/>+ DNS Resolver"]
    HUB --> GW["VPN / ExpressRoute gateway"]
    HUB --> AVNM["Azure VNet Manager<br/>(optional, large fleet)"]
    SPK["Spoke VNets<br/>(per environment)"] -->|peering| HUB
    SPK --> NSG["NSGs + UDRs"]
    SPK --> WL["Workload subnets"]
    NSG -->|force-tunnel| FW
{{< /mermaid >}}

The network plane is per-customer hub-and-spoke, with the hub living in a dedicated connectivity subscription per customer. The hub carries Azure Firewall (with a per-customer firewall policy), Bastion for tenant-side jump access, DDoS Protection Plan, Private DNS zones with a DNS Resolver for hybrid resolution, and a VPN / ExpressRoute gateway when the engagement template enables it.

Spokes — typically one per environment (prod / dev / test / sandbox) — peer to the hub. UDRs on every spoke subnet force-tunnel traffic to the firewall; NSGs scope ingress/egress per subnet; workload subnets host the actual application surfaces.

For tenants with a large customer footprint (dozens of hubs to keep peered), Azure Virtual Network Manager automates the peering topology. AVNM is optional — small fleets can manage peering directly. The placement matrix (which resources live in the hub vs. in the spoke vs. in the workload subscription) follows a deterministic rule: shared infrastructure (firewall, bastion, DNS, DDoS, gateways) sits in the hub; per-environment isolation (subnets, NSGs, UDRs, workloads) sits in the spoke.

*Covered in depth in [L-4 (forthcoming)](/posts/hub-and-spoke-multi-customer/).*

### 4. Global Secure Access (GSA)

{{< mermaid >}}
flowchart TB
    GSA["Global Secure Access"]
    GSA --> ACQ["Traffic acquisition<br/>(client tunnel)"]
    GSA --> SWG["Secure Web Gateway<br/>(internet egress)"]
    GSA --> ZTNA["Private Access<br/>(replaces VPN)"]
    GSA --> NWGATE["CA network condition<br/>(referenced by auth-context triple-gates)"]
{{< /mermaid >}}

GSA is Microsoft's identity-centric SSE (Secure Service Edge): traffic acquisition via a client tunnel, a Secure Web Gateway for outbound internet, Private Access as a VPN replacement for on-prem and Azure-private workloads, and — most importantly for this platform — a network identity that Conditional Access can target as a network condition.

GSA's role in the reference platform is the **network gate**. The triple-gate pattern across the auth contexts references a network control; GSA is the natural fit because it gives each session an identity-tied network attribute that CA can evaluate. Tenants without GSA fall back to named locations (IP-range based), which works but ties the network control to physical IP allocation rather than identity.

A note on documentation status: GSA isn't yet a first-class component in the canonical architecture references for the reference platform. It's a forward-looking element of the design — instantiated in tenants that have rolled it out, but not yet codified as universal substrate. Slice-posts that touch it will treat it as "design intent" until that codification lands.

*Covered in depth in [I-4 (forthcoming)](/posts/auth-contexts-in-practice/) (notes), with a dedicated GSA-deployment post likely on the longer-term slate.*

### 5. Azure DevOps (two organizations)

{{< mermaid >}}
flowchart TB
    ADO["Azure DevOps"]
    ADO --> MGT["Platform management org ADO<br/>customer Terraform delivery + platform pipeline"]
    ADO --> SEC["Platform security org ADO<br/>supply-chain-isolated shared-library fork"]
    MGT --> REPOS["Repos + Pipelines<br/>+ Variable groups + Service connections"]
    MGT --> ART["Artifacts<br/>(AVM module feed)"]
    MGT --> BACKUP["Backup chain<br/>Arc-MI → CMK → WORM → Managed HSM → GitLab"]
    SEC --> FORK["Shared-library fork<br/>(no auto-sync)"]
{{< /mermaid >}}

Azure DevOps is the orchestration substrate, deployed across two ADO organizations by design. The platform management org runs the platform delivery pipeline and Terraform delivery for the platform's tenants. The platform security org holds an independent fork of the shared-library repo (the stage/step templates and helper scripts that the management org's pipelines extend from) — fork rather than sync, so a compromise of the management org can't reach the security org's policies and detections.

Repos hold Terraform configs, PowerShell scripts, YAML pipelines, KQL detection rules, and policy definitions. Pipelines orchestrate the deployment; service connections (no PATs — always workload-identity-federated to a UAMI) carry the deployment-time identity; variable groups hold non-secret parameters; an Artifacts feed publishes Azure Verified Modules for downstream consumption.

The ADO backup chain is the post's most distinctive piece. An Arc-enabled managed identity replicates repos, work items, and pipelines to on-prem GitLab through a chain of customer-managed-key encryption, WORM immutability tiers (30 days / 90 days / 1 year / 7 years per artifact type), and a Managed HSM holding the encryption keys. The pattern is sovereign-data-as-a-feature: the customer's tenant-as-code lives in the customer's authoritative storage, not Microsoft's.

*Covered in depth in [S-1 (forthcoming)](/posts/no-pat-ado-agents/) (agent side), [S-7 (forthcoming)](/posts/ado-backup-sovereign-data/) (backup side).*

### 6. Self-hosted ADO agents (ACI + UAMI + WIF)

{{< mermaid >}}
flowchart TB
    ACI["Azure Container Instances<br/>(agent host)"]
    ACI --> UAMI["User-Assigned Managed Identity"]
    UAMI --> FIC["Federated credentials<br/>(OIDC trust to ADO)"]
    FIC --> TOK["OIDC token exchange<br/>(no stored secret)"]
    TOK --> SC["ADO service connection<br/>(workload-identity-federated)"]
    UAMI --> RBAC["RBAC assignments<br/>(Storage Blob Data Contributor,<br/>Key Vault Crypto User, …)"]
    ACI --> HIMDS["Arc HIMDS<br/>(for on-prem replication agent)"]
{{< /mermaid >}}

The self-hosted ADO agent runs on Azure Container Instances and gets its deployment identity from a User-Assigned Managed Identity. The UAMI carries federated credentials trusting Azure DevOps' OIDC issuer — no secrets are stored anywhere in the chain. The agent presents an OIDC token; ADO exchanges it for a service-connection token; the service connection then drives the target-tenant ARM and Graph calls under whatever RBAC and Graph permissions the UAMI carries.

The RBAC assignments are scope-minimal: the UAMI gets exactly the roles its pipelines need — Storage Blob Data Contributor for backend state, Key Vault Crypto User for sealing/unsealing, Reader at MG scope for inventory queries — and no more. New pipelines that need additional permissions go through a PR that adds the assignment explicitly.

For the on-prem replication side of the ADO backup chain (covered above), the same UAMI pattern extends via Azure Arc and the Hybrid Instance Metadata Service (HIMDS): the on-prem replication agent gets an Arc-MI identity that the ADO service connection trusts, exchanging tokens the same way as the cloud-side agent. One identity model, two compute substrates.

*Covered in depth in [S-1 (forthcoming)](/posts/no-pat-ado-agents/).*

### 7. Terraform delivery (three domains)

{{< mermaid >}}
flowchart TB
    TF["Terraform delivery"]
    TF --> IDD["Identity domain<br/>(Entra, RBAC, PIM groups)"]
    TF --> PLD["Platform domain<br/>(MG hierarchy, hubs, EPAC)"]
    TF --> WLD["Workload domain<br/>(spokes, app subscriptions)"]
    IDD --> WIF1["WIF provider block"]
    PLD --> WIF2["WIF provider block"]
    WLD --> WIF3["WIF provider block"]
    WIF1 --> ST["Remote state<br/>(Azure Storage + lock)"]
    WIF2 --> ST
    WIF3 --> ST
    IDD --> AVM["Azure Verified Modules<br/>(terraform-azurerm-*)"]
    PLD --> AVM
    WLD --> AVM
    WLD --> VAR["Variable substitution<br/>CIDR + CA range formulas"]
{{< /mermaid >}}

Terraform delivery is split into three domains. Identity owns Entra objects, RBAC group lifecycle, PIM group provisioning. Platform owns the MG hierarchy, hub networks, and the EPAC initiative assignments. Workload owns the per-customer spokes, application subscriptions, workload-specific resources. Each domain is a separate Terraform root module with its own remote state in Azure Storage (state-locked for concurrent-run safety) and its own WIF provider block bound to a UAMI scoped to that domain's responsibilities.

Workload-identity federation replaces the SP-secret pattern entirely on the deployment side. The Terraform provider block names the UAMI client_id and tenant_id and a federated token URL; the runtime exchanges the agent's OIDC token for an ARM access token. Same model as Section 6, applied at the Terraform-provider layer.

Azure Verified Modules — Microsoft's `terraform-azurerm-*` registry of resource modules — are the building blocks. Custom local modules wrap AVMs where the reference platform needs MSSP-specific composition (e.g., the per-customer hub assembly). Variable substitution pulls per-customer values from the deployment configuration template: the customer index drives per-customer subnetting and policy ranges via a customer-index-derived formula, with a fixed arithmetic capacity ceiling enforced by the formula's arithmetic.

*Covered in depth in [S-2 (forthcoming)](/posts/three-domain-terraform-wif/), [S-12 (forthcoming)](/posts/cross-tenant-wif-assessment/).*

### 8. EPAC + AMBA

{{< mermaid >}}
flowchart TB
    EPAC["EPAC<br/>(Enterprise Policy as Code)"]
    EPAC --> DEFS["Policy definitions<br/>(custom + built-in)"]
    EPAC --> INIT["Initiatives<br/>(grouped definitions)"]
    EPAC --> ASGN["Assignments<br/>(scope + parameters + exemptions)"]
    ASGN --> DINE["DINE / Modify<br/>auto-remediation"]
    ASGN --> DUAL["Dual EPAC<br/>scheduled every 6h + notScopes"]
    AMBA["AMBA-ALZ alerts"] --> ACT["Action Groups"]
    ACT --> ROUTE["Alert routing<br/>(on-call channels)"]
    DINE --> AMBA
{{< /mermaid >}}

Enterprise Policy as Code (EPAC) is the policy-as-code wrapper around Azure Policy: definitions, initiatives, assignments, and exemptions all version-controlled, deployed by a pipeline that runs `Build-DeploymentPlans` to produce a policy-plan and a roles-plan, then applies them. The dual-EPAC pattern is the tamper-resistance mechanism: a scheduled run every six hours re-asserts the desired state; the policy-management identity is itself restricted via `notScopes` RBAC so even an admin who can compromise the runtime can't disable the schedule.

AMBA-ALZ (Azure Monitor Baseline Alerts for Azure Landing Zones) layers a curated alert baseline on top: pre-built metric and log-search alerts for the platform-management resources (compute, storage, key vaults, network, IAM events). Action Groups route firing alerts to on-call channels — Teams, email, ITSM webhooks — based on severity and tag-derived ownership.

The DINE (Deploy If Not Exists) and Modify policies are the platform's continuous-enforcement substrate. They cascade from MG scope down to subscriptions — diagnostic settings get auto-deployed to every new resource, Defender plans auto-enabled at subscription create, the AMBA alert pack auto-instantiated. Drift detection on the policy side closes the loop: anything that drifts from the EPAC desired state is re-applied within six hours.

*Covered in depth in [S-13 (forthcoming)](/posts/dual-epac-tamper-proof/).*

### 9. Engagement Template + platform delivery pipeline

{{< mermaid >}}
flowchart TB
    ET["Engagement Template"]
    ET --> DOCX["DOCX<br/>(customer-facing)"]
    ET --> JSON["JSON Schema<br/>(pipeline-consumed)"]
    JSON --> PIPE["Platform delivery pipeline<br/>multi-stage"]
    PIPE --> S1["Preflight / Bootstrap / Validate"]
    PIPE --> S2["ConfigDiscover<br/>(derive deployment flags)"]
    PIPE --> S3["EntraProvision / BlobPersist / CAPrepare"]
    PIPE --> S4["SubscriptionVend / HubCore / SpokeNetworks"]
    PIPE --> S5["AppGateway or FrontDoor<br/>(mutex)"]
    PIPE --> S6["DevOpsSetup / GuestOnboard / Report"]
{{< /mermaid >}}

The Engagement Template is the reference platform's deployment configuration template. The DOCX side is the document a customer engagement workshop fills out — sections on regions, BCDR, IR ownership, naming, tagging, guest identity, CA policy needs. The JSON Schema side is the structured representation the platform delivery pipeline reads. Every field in the DOCX maps to a JSON path; an appendix table holds the traceability between document section and pipeline stage that consumes it.

The platform delivery pipeline runs through multiple stages depending on the deployment configuration template's flags. Preflight checks permissions; Bootstrap establishes Terraform state backends; Validate runs the schema check; ConfigDiscover derives deployment flags (whether to deploy Application Gateway vs. Front Door, whether VPN/ExpressRoute is enabled, etc.); EntraProvision creates the Entra objects (RBAC groups, named location, authentication strength, terms-of-use, guest admin unit); BlobPersist saves the enriched JSON (with Entra object IDs) for downstream stages; CAPrepare generates Conditional Access policy JSONs from the template; SubscriptionVend runs the Terraform `lz-vending` module; HubCoreNetwork and SpokeNetworks deploy the network plane; AppGateway or FrontDoor (mutually exclusive based on `firewall_pattern`) deploys the WAF edge; DevOpsSetup creates the customer's ADO project; GuestOnboard runs B2B invitations; Report emits a deployment summary and RBAC audit.

The customer index is the lynchpin. From the deployment configuration template, the pipeline derives the per-customer subnet allocation and CA policy range via a customer-index-derived formula, the RBAC group naming (per the convention shown in the slice-post for naming), and the Guest-Lifecycle Administrative Unit name. One index, multiple derived values; the formula carries a fixed arithmetic capacity ceiling per platform instance.

*Covered in depth in a dedicated delivery-pipeline post (forthcoming, slot in the per-category list — likely 2026 H2).*

### 10. UTCM

{{< mermaid >}}
flowchart TB
    UTCM["UTCM<br/>Unified Tenant Configuration Management"]
    UTCM --> WL["5 workloads"]
    WL --> ENT["Entra ID"]
    WL --> DXDR2["Defender XDR"]
    WL --> EXO["Exchange Online"]
    WL --> INT["Intune"]
    WL --> PUR["Purview"]
    UTCM --> RES["70+ resource types"]
    UTCM --> MON["configurationMonitor<br/>30 monitors/tenant<br/>800 resources/day<br/>6h drift cycle"]
    UTCM --> PRIO["5-priority tooling matrix:<br/>UTCM strategic → TF AVM → Graph → community → M365DSC"]
{{< /mermaid >}}

UTCM (Unified Tenant Configuration Management) is Microsoft's strategic API surface for M365 tenant configuration-as-code. It covers five workloads — Entra ID, Defender XDR, Exchange Online, Intune, and Purview — across roughly seventy resource types. The `configurationMonitor` API watches deployed state and emits drift signals on a six-hour cycle, scoped to a quota of thirty monitors per tenant and eight hundred resources per day, with seven-day drift retention.

UTCM is preview at the time of writing and not yet covering every M365 surface that the platform needs to govern. The platform's tooling matrix runs at five priority levels: UTCM is the strategic primary where coverage exists; Terraform with AVM modules handles Azure-side IaC; Microsoft Graph + PowerShell SDK fills the gaps where UTCM is still read-only; community frameworks (EntraOps for principal classification, EPAC for policy-as-code) cover gaps the Microsoft-supplied tools don't; M365DSC is kept as a backup / compliance-snapshot tool only — no longer primary deploy.

What UTCM doesn't yet cover: portal-only settings (idle session timeout, group creation controls, Copilot data-access scope), Teams meeting/messaging policies (partial support), audit log search permissions (no automation — manual role assignment), and self-service purchases (limited automation via `MSCommerce` PowerShell). Each of these is on the gap list for slice-posts to deep-dive.

*Covered in depth in [S-9 (forthcoming)](/posts/utcm-concept-and-primitives/), [S-10 (forthcoming)](/posts/utcm-drift-detection-mssp-scale/), [S-11 (forthcoming)](/posts/utcm-vs-m365dsc-vs-epac/).*

### 11. Defender XDR Unified RBAC

{{< mermaid >}}
flowchart TB
    DXDR["Defender XDR"]
    DXDR --> URBAC["Unified RBAC<br/>(group-based role assignments)"]
    URBAC --> SECG["Security groups<br/>(role-assignable)"]
    SECG --> P4G2["PIM-for-Groups<br/>(JIT activation governance)"]
    DXDR --> MPM["Defender portal<br/>(multi-tenant management)"]
    DXDR --> MIG["Sentinel-in-Defender migration<br/>(March 2027 deadline)"]
{{< /mermaid >}}

Defender XDR Unified RBAC is Microsoft's consolidated role model across Defender for Endpoint, Defender for Identity, Defender for Cloud Apps, Defender for Office 365, and Microsoft Sentinel. Roles are assigned to security groups rather than directly to users; PIM-for-Groups (covered in node 1) governs the JIT activation of those group memberships, so a SOC analyst with eligible membership in `Defender-IR-Tier2` only carries the role when they activate.

The Defender portal multi-tenant management surface is the SOC operator's console — a single pane that pulls signals across all customer tenants. Combined with Workspace Manager-distributed content (analytics rules, hunting queries, workbooks), one SOC team can operate detection-and-response across the full customer fleet without each operator needing a per-customer login.

The March 2027 Defender portal migration deadline — Sentinel-in-Azure-portal retires; everything routes through the unified Defender experience — is the time-pressure factor in this node. Slice-post O-9 covers what the migration changes for Sentinel-as-code and the `EyesOn` onboarding pattern.

*Covered in depth in [O-9 (forthcoming)](/posts/defender-portal-migration-march-2027/).*

### 12. Microsoft Sentinel (+ Workspace Manager + Data Lake)

{{< mermaid >}}
flowchart TB
    SENT2["Microsoft Sentinel<br/>(Tier 2 central security workspace)"]
    SENT2 --> CONN["Data connectors<br/>(Defender / Azure AD / Activity / custom CEF)"]
    SENT2 --> AR["Analytic rules<br/>(KQL from shared-library repo)"]
    SENT2 --> AUTO["Automation rules<br/>(grouping + auto-remediation)"]
    SENT2 --> WL2["Watchlists<br/>(e.g., RoleAssignableGroups)"]
    SENT2 --> WB["Workbooks + incidents"]
    SENT2 --> WM["Workspace Manager<br/>(parent → member, one-way push)"]
    WM --> MEMB["Per-customer Tier 1 workspaces"]
    SENT2 --> DL["Sentinel Data Lake<br/>(Tier 3, up to 12yr retention)"]
{{< /mermaid >}}

Sentinel is the reference platform's central security analytics workspace, deployed at Tier 2 of the three-tier logging architecture (Tier 1 = per-LZ workspaces; Tier 3 = Data Lake). Data connectors pull from Defender, Entra ID, Azure Activity, custom CEF/Syslog, and platform-specific feeds (PIM operations, ADO audit events). Analytics rules — KQL definitions held in the shared-library repo, version-controlled, code-reviewed — fire on the ingested signals.

Watchlists hold reference data the rules need at query time — the canonical example is `RoleAssignableGroups`, a watchlist refreshed by a scheduled Graph query that lists every group with `isAssignableToRole = true`. Rules join against the watchlist to anchor membership-change detection on the right group set without name-pattern guessing.

Workspace Manager is what makes the multi-customer SOC pattern work without log-visibility bleed. Central Sentinel acts as the manager parent and pushes content — analytics rules, hunting queries, workbooks, playbooks — one-way down to per-customer member workspaces. Customer logs stay in the per-customer workspace; SOC operators query across them via Defender XDR multi-tenant management. The Data Lake (Tier 3) holds long-tail logs in cheaper-storage tiers, queryable on-demand with retention up to twelve years for compliance evidence.

*Covered in depth in [O-1 (forthcoming)](/posts/sentinel-hunting-pim-activations/), [O-8 (forthcoming)](/posts/sentinel-cost-optimization-mssp/), [O-10 (forthcoming)](/posts/workspace-manager-sentinel-per-customer/).*

### 13. Defender for Cloud

{{< mermaid >}}
flowchart TB
    DFC2["Defender for Cloud"]
    DFC2 --> MCSB["MCSB v2 baseline"]
    DFC2 --> REG["~12 compliance frameworks"]
    REG --> R1["CIS / ISO 27001 / SOC 2"]
    REG --> R2["NIS2 / PCI DSS / GDPR"]
    REG --> R3["NIST 800-53 / HITRUST / CMMC"]
    REG --> R4["DORA / EU AI Act / NIST CSF"]
    DFC2 --> DASH["Regulatory compliance dashboard"]
    DFC2 --> PLAN["Defender plans<br/>(Servers / Storage / SQL / Containers / …)"]
    PLAN -.->|DINE-deployed| MG2["MG hierarchy"]
{{< /mermaid >}}

Defender for Cloud sits alongside Sentinel and covers the compliance posture side. The baseline is the Microsoft Cloud Security Benchmark v2 — a control catalog mapped to common standards. On top of MCSB, the platform layers around twelve regulatory standards: CIS, ISO 27001, SOC 2, NIS2, PCI DSS, GDPR, NIST 800-53, HITRUST, CMMC, DORA, EU AI Act, NIST CSF. Each is selectable per customer based on the engagement template; the regulatory compliance dashboard surfaces the gap analysis for any of them.

The Defender plans — Defender for Servers, Storage, SQL, Containers, Kubernetes, etc. — get DINE-deployed at MG scope. New subscriptions inherit the plan set automatically; new resources within those subscriptions get assessment coverage on creation. The customer-specific compliance frameworks layer on top of this auto-applied posture.

*Covered in depth in [O-2 (forthcoming)](/posts/defender-for-cloud-mg-scope/), [O-7 (forthcoming)](/posts/compliance-framework-mapping-dfc/).*

### 14. MG hierarchy

{{< mermaid >}}
flowchart TB
    TRG["Tenant Root Group"]
    TRG --> INT["Intermediate root<br/>(platform-level policy base)"]
    INT --> PLAT["Platform MGs"]
    PLAT --> MNG["Management MG"]
    PLAT --> SEC2["Security MG"]
    PLAT --> CON["Connectivity MG"]
    INT --> LZ["Landing Zones MG"]
    LZ --> CUST["Per-customer MGs"]
    INT --> SAND["Sandbox MG"]
    INT --> DEC["Decommissioned MG"]
    INT --> CONF["Confidential Isolated MG<br/>(L3 sovereign policies)"]
    INT -->|policy inheritance| TRG
    LZ -->|cross-tenant| LH["Azure Lighthouse<br/>(customer-owned tenants)"]
{{< /mermaid >}}

The Management Group hierarchy is the organisational substrate that every other plane lands on. Below the Tenant Root Group, an intermediate root carries the platform-level policy base. Platform MGs (Management, Security, Connectivity) host shared services. Landing Zones MG is the parent of per-customer MGs — one branch per customer, with the customer's connectivity hub, application subscriptions, and identity scoping all nested below. Sandbox MG holds relaxed policies for experimentation; Decommissioned MG holds soft-deleted subscriptions during the off-boarding cool-down.

The fourth top-level branch is Confidential Isolated MG — a separate MG (not a subtree of Platform or Landing Zones) for L3 sovereign workloads that demand confidential compute, encryption in use, and a separate policy lineage. It's its own tier in the hierarchy, not a customer-LZ sub-branch.

For customer-owned tenants (where the customer keeps their tenant root but delegates platform management to the MSSP), Azure Lighthouse provides cross-tenant resource management. Lighthouse projects an MSSP-side identity (group + delegation manifest) into the customer tenant; the MSSP operator can operate the customer's resources without per-customer logins, while RBAC scoping limits what the projected identity can do.

*Covered in depth in [L-1 (forthcoming)](/posts/mg-hierarchy-multi-customer/), [L-7 (forthcoming)](/posts/multi-region-without-region-mgs/).*

### 15. Three-tier logging

{{< mermaid >}}
flowchart TB
    T1["Tier 1<br/>per-LZ Log Analytics<br/>(30–90 days)"]
    T2["Tier 2<br/>central security workspace<br/>(90d hot + 2yr interactive + 7yr queryable)"]
    T3["Tier 3<br/>Sentinel Data Lake<br/>(up to 12 years)"]
    SRC["Diagnostic settings<br/>(DINE-deployed from MG hierarchy)"] --> T1
    T1 -->|security signals| T2
    T2 -->|long-tail| T3
    T2 -->|cross-workspace queries<br/>max 20, recommended ≤5| T1
    T3 -.->|on-demand restore| T2
{{< /mermaid >}}

The three-tier logging architecture splits log-data ownership by retention need and query frequency. Tier 1 is the per-Landing-Zone Log Analytics workspace — 30–90-day retention, hot tier, scoped to the customer's own LZ. Tier 2 is the central security workspace where Sentinel runs — 90 days hot, two years interactive, seven years queryable in the basic-logs tier. Tier 3 is the Sentinel Data Lake — colder storage, retention up to twelve years, on-demand restore to Tier 2 when an audit or investigation needs the long tail.

DINE-deployed diagnostic settings push signals from every resource into Tier 1; Sentinel connectors aggregate security-relevant signals up into Tier 2; Tier 2 archives long-tail signals into Tier 3. Cross-workspace queries from Tier 2 back into Tier 1 are supported (Sentinel's own queries can join across workspaces) but rate-limited — Microsoft's recommendation caps the number of joined workspaces at twenty in any single query, with five or fewer being the operational sweet spot for performance.

The architectural payoff is delegation without bleed: each customer's logs live in the customer's own Tier 1 workspace; the SOC reads Tier 2 (their content + the per-customer Tier 1 references); audit/forensic restores from Tier 3 happen on the central workspace without exposing other customers' logs.

*Covered in depth in [L-8 (forthcoming)](/posts/three-tier-logging-architecture/), [O-8 (forthcoming)](/posts/sentinel-cost-optimization-mssp/).*

### 16. New nodes — one-paragraph introductions

These twelve nodes appear in the master diagram but don't yet have a full sub-diagram in this post. Each will get its own slice-post over time; the paragraphs below name the node, place it in its layer, and cite the canonical doc anchor.

- **Platform delivery pipeline (Layer 1 — Delivery).** Multi-stage unified pipeline that consumes the JSON Schema produced from the Engagement Template DOCX, then orchestrates every downstream stage: identity bootstrap, MG vending, Subscription Vending (`lz-vending`), Hub & Spoke deployment, EPAC + AMBA assignment, Sentinel connector wire-up, CA persona deployment, customer cut-over. One pipeline, one engagement record, idempotent. Lives in the platform management org with the platform security org carrying the shared-library fork.
- **Cost Management (Layer 2 — Azure substrate).** Per-customer billing rollup, budget enforcement at MG and subscription scope, anomaly detection that fires AMBA Action Groups when a customer's spend slope crosses threshold. Tied to the reference platform's chargeback model — each customer is a separate cost-management scope so the MSSP can split invoices per engagement without re-tagging at billing time.
- **Subscription Vending (Layer 1 — Delivery).** `lz-vending` Terraform module that creates per-customer subscriptions inline with the platform delivery pipeline. The same pipeline run creates the subscription, places it under the correct per-customer LZ MG, assigns the policy-set, and creates the subscription-scoped RBAC groups (already PIM-eligible at creation — no follow-up stage, no race condition).
- **Compliance Framework Mapping (Layer 1 — Delivery).** A documented mapping between platform policy assignments and ~12 regulatory frameworks (CIS / ISO 27001 / SOC 2 / NIS2 / PCI DSS / GDPR / NIST 800-53 / HITRUST / CMMC / DORA / EU AI Act / NIST CSF). The mapping is the artefact; Defender for Cloud reads it to populate the compliance dashboard.
- **Managed HSM (Layer 2 — Azure substrate).** FIPS 140-3 Level 3 customer-managed key store. Backs the sovereign L2 and L3 encryption requirements: customer keys never leave the HSM boundary, the MSSP can't read them, and the Confidential Isolated MG references HSM keys for storage + compute encryption.
- **Administrative Units (Layer 3 — Identity & Access).** AU-scoped administration for delegated roles — particularly relevant for tenants where the MSSP holds only a subset of admin authority (e.g., security AU only, not identity AU). CA policies can target AU-scoped admins separately from tenant-global admins.
- **Maester (Layer 5 — Assurance).** PowerShell + Pester-based test framework that asserts tenant security posture against ~200 checks (CA configuration, PIM tier policies, authentication methods, device controls, app consent). Run on a schedule from the platform security org; results pushed into Sentinel as a custom data source so detections can fire on regressions.
- **Zero Trust Assessment (Layer 5 — Assurance).** A fork of the open-source Zero Trust Assessment with 200+ Zero Trust-specific tests added by the platform team. Findings feed into Defender for Cloud as additional posture signals and into Sentinel as a separate evidence stream.
- **Microsoft-hosted agents (Layer 1 — Delivery).** Parallel agent pool used alongside ACI-based self-hosted agents for non-sovereign workloads where the public Microsoft image is acceptable and the cold-start time matters more than the workload-identity-only constraint. Self-hosted (UAMI + WIF, no PAT) carries the sovereign workloads; Microsoft-hosted carries the rest.
- **Authentication Methods policy (Layer 3 — Identity & Access).** Tenant-level allow-list of credential types — FIDO2, WHfB, MS Authenticator, TAP — with weak methods (SMS, voice, software TOTP) disabled at the floor. CA Authentication Strengths then layer the more-restrictive grant control on top. The policy is the *floor*; the auth strength is the *ceiling*.
- **AAGUID allow-list (Layer 3 — Identity & Access, surfaced inside the auth strength).** The FIDO2 Authenticator Attestation GUID allow-list — only hardware-attested keys from approved manufacturers satisfy the hardware-attested authentication strength. The master diagram folds it into the Authentication Strengths node label rather than calling it out as a separate node.
- **PAW — Privileged Access Workstations (Layer 3 — Identity & Access).** Dedicated hardware for the privileged tier. Registration of credentials for privileged personas is blocked from non-PAW devices via CA registration policy. The PAW node is the hardware-rooted prerequisite that lets the rest of the Identity stack make defensible assertions about *who* is on the keyboard.

## How the layers connect

The master diagram above shows ~32 nodes across five layers, with the load-bearing inter-layer edges already drawn. The full dependency map is wider; the additional load-bearing edges hidden in the prose:

- **MG hierarchy → DINE cascade** — assignment at MG scope auto-deploys child resources (diagnostic settings, Defender plans, AMBA alerts) into every subscription. Layer 1 (EPAC / AMBA) writes once at MG; the cascade hits every new subscription on creation, without re-running the pipeline.
- **Subscription Vending → RBAC groups (PIM-enabled)** — the `lz-vending` Terraform module creates subscription-scoped groups inline, and the same pipeline run flips them to PIM-eligible immediately. A Layer 1 → Layer 3 cross-edge that hides behind the visible Layer 1 → Layer 2 edge.
- **AMBA-ALZ → DINE → Action Groups → Alert routing** — the alert baseline cascades through Layer 1 (DINE deploys the alerts), Layer 4 (Action Groups route firings), and the operational on-call channel that receives the routed alert. A four-stage chain across three layers.
- **Engagement Template → CAPrepare stage → CA policy JSONs (manual portal deployment)** — CA policy generation is partly automated (the platform delivery pipeline generates the JSON), partly manual (a portal admin imports each generated JSON because CA policy creation via Graph is still partially gated). One of the documented Layer 1 → Layer 3 gaps the reference platform tracks.
- **Defender for Cloud → MG hierarchy** — DfC's regulatory compliance dashboard reads against the MG hierarchy; the compliance scores feed back into Sentinel as an additional signal source. A Layer 4 → Layer 2 read-edge with a Layer 4 → Layer 4 fan-out into Sentinel.
- **PIM-for-Groups ↔ Defender XDR Unified RBAC ↔ Sentinel** — three nodes in a triangle across Layers 3, 4, and 4: PIM-for-Groups governs activation of groups assigned Defender XDR roles; Sentinel detection rules watch for direct membership changes that bypass the PIM activation flow (covered in I-1's Sentinel rule design).
- **Maester + ZTA → Sentinel → Layer 1 next delivery wave** — the Layer 5 → Layer 4 feed lands assurance findings as Sentinel evidence; the dashed Layer 5 → Layer 1 feedback edge closes the cycle by feeding findings back into the Engagement Template revision pipeline. Without this loop, assurance is just observation; with it, the platform reconfigures itself in response to its own findings.
- **Platform security org ADO ← (no auto-sync) ← Platform management org ADO** — the shared-library repo lives in two repos, one per ADO organization, intentionally without automatic synchronisation. Code review on the management org's side never lands in the security org's side without a separate explicit PR; supply-chain compromise of the management org can't reach detections or policies.

The combined picture is ~28 master-diagram edges + eight cross-layer edges hidden in the prose + a large set of implicit dependencies inside the per-node detail diagrams. That's the size of an MSSP-class reference platform — too dense for one viewable graph, manageable as a layered map.

## Where each piece is covered in depth

The closing table — every top-level node, the slice-post that deep-dives it, status.

| # | Component | Slice-post(s) | Status |
|---|---|---|---|
| 1 | PIM | [I-1](/posts/pim-at-scale-three-domains/) | **Published** |
| 1 | PIM (CA enforcement) | [I-5](/posts/enforce-ca-on-pim-activation/) | Planned |
| 1 | PIM-for-Groups boundary | [I-6](/posts/role-assignable-pim-groups-security-boundary/) | Planned |
| 2 | Conditional Access — auth contexts | [I-4](/posts/auth-contexts-in-practice/) | Planned |
| 2 | Conditional Access — strengths vs contexts | [I-8](/posts/auth-strengths-vs-auth-contexts/) | Planned |
| 2 | Conditional Access — Protected Actions | [I-9](/posts/protected-actions-in-entra/) | Planned |
| 2 | Conditional Access — persona scale | [I-10](/posts/persona-based-ca-at-scale/) | Planned |
| 2 | Conditional Access — AI-agent persona | [I-11](/posts/ca-for-ai-agent-identity/) | Planned |
| 2 | Conditional Access — AAGUID attestation | [I-13](/posts/fido2-aaguid-attestation-whitelisting/) | Planned |
| 3 | Hub & Spoke | [L-4](/posts/hub-and-spoke-multi-customer/) | Planned |
| 4 | GSA (notes) | [I-4](/posts/auth-contexts-in-practice/) | Planned |
| 5 | ADO backup | [S-7](/posts/ado-backup-sovereign-data/) | Planned |
| 6 | Self-hosted ADO agents | [S-1](/posts/no-pat-ado-agents/) | Draft in progress |
| 7 | Terraform delivery (three domains) | [S-2](/posts/three-domain-terraform-wif/) | Planned |
| 7 | Terraform delivery (cross-tenant WIF) | [S-12](/posts/cross-tenant-wif-assessment/) | Planned |
| 8 | EPAC + AMBA (dual EPAC) | [S-13](/posts/dual-epac-tamper-proof/) | Planned |
| 9 | Platform delivery pipeline | dedicated delivery-pipeline post (forthcoming) | Planned |
| 10 | UTCM — concept | [S-9](/posts/utcm-concept-and-primitives/) | Planned |
| 10 | UTCM — drift detection | [S-10](/posts/utcm-drift-detection-mssp-scale/) | Planned |
| 10 | UTCM vs M365DSC vs EPAC | [S-11](/posts/utcm-vs-m365dsc-vs-epac/) | Planned |
| 11 | Defender portal migration | [O-9](/posts/defender-portal-migration-march-2027/) | Planned |
| 12 | Sentinel hunting | [O-1](/posts/sentinel-hunting-pim-activations/) | Planned |
| 12 | Sentinel cost optimisation | [O-8](/posts/sentinel-cost-optimization-mssp/) | Planned |
| 12 | Workspace Manager / per-customer Sentinel | [O-10](/posts/workspace-manager-sentinel-per-customer/) | Planned |
| 13 | Defender for Cloud | [O-2](/posts/defender-for-cloud-mg-scope/) | Planned |
| 13 | Compliance framework mapping | [O-7](/posts/compliance-framework-mapping-dfc/) | Planned |
| 14 | MG hierarchy | [L-1](/posts/mg-hierarchy-multi-customer/) | Planned |
| 14 | Multi-region without region MGs | [L-7](/posts/multi-region-without-region-mgs/) | Planned |
| 15 | Three-tier logging | [L-8](/posts/three-tier-logging-architecture/) | Planned |
| 16 | Platform delivery pipeline | dedicated delivery-pipeline post (forthcoming) | Planned |
| 16 | Cost Management | dedicated post (forthcoming) | Planned |
| 16 | Subscription Vending (`lz-vending`) | dedicated post (forthcoming) | Planned |
| 16 | Compliance Framework Mapping | [O-7](/posts/compliance-framework-mapping-dfc/) (see also row 13) | Planned |
| 16 | Managed HSM (sovereign keys) | dedicated post (forthcoming) | Planned |
| 16 | Administrative Units | dedicated post (forthcoming) | Planned |
| 16 | Maester | dedicated post (forthcoming) | Planned |
| 16 | Zero Trust Assessment | dedicated post (forthcoming) | Planned |
| 16 | Microsoft-hosted agents | dedicated post (forthcoming) | Planned |
| 16 | Authentication Methods policy | dedicated post (forthcoming) | Planned |
| 16 | AAGUID allow-list | [I-13](/posts/fido2-aaguid-attestation-whitelisting/) (see also row 2) | Planned |
| 16 | Privileged Access Workstations (PAW) | dedicated post (forthcoming) | Planned |

## Wrap-up

This post is the map. Each slice-post is the territory of one node — the depth, the runnable code, the design trade-offs, the pitfalls. The map doesn't replace the territory; it lets you navigate between territories without losing context.

The diagram and the dependency map will evolve as the reference platform evolves and as new slice-posts land. Any time a forthcoming slice-post above ships, it's worth coming back to this one to walk the relevant node and check that the picture still holds. The discipline of keeping one canonical map — not letting it splinter across slice-posts — is the same discipline that keeps a platform manageable at scale.

## References

- Microsoft Learn — [Azure Landing Zone reference architecture](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/)
- Microsoft Learn — [Microsoft Cloud Adoption Framework (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/)
- Microsoft Learn — [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
- Microsoft Learn — [Conditional Access authentication context](https://learn.microsoft.com/entra/identity/conditional-access/concept-conditional-access-cloud-apps#authentication-context)
- Microsoft Learn — [Unified Tenant Configuration Management (UTCM)](https://learn.microsoft.com/graph/unified-tenant-configuration-management-concept-overview)
- Microsoft Learn — [Workspace Manager for Microsoft Sentinel](https://learn.microsoft.com/azure/sentinel/workspace-manager)
- Microsoft Learn — [Microsoft Cloud Security Benchmark v2](https://learn.microsoft.com/security/benchmark/azure/)
- Microsoft Learn — [Azure Lighthouse cross-tenant management](https://learn.microsoft.com/azure/lighthouse/overview)
- Microsoft Learn — [Sentinel Data Lake](https://learn.microsoft.com/azure/sentinel/datalake/overview)
