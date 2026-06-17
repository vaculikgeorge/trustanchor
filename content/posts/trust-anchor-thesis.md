+++
title = 'Two perimeters, one trust anchor'
date = '2026-05-26T10:00:00+02:00'
draft = false
tags = ['thesis', 'identity-governance', 'devsecops', 'iac', 'positioning', 'everything-as-code']
categories = ['Meta']
summary = 'The problem that started this blog: a Microsoft cloud tenant that grew faster than any person could track. The solution: everything-as-code — CA, PIM, Defender, Sentinel, M365 settings, Azure infrastructure all declared in version-controlled repos and delivered by pipeline. And the two-perimeter identity architecture that makes it trustworthy.'
+++

> The principles are universal. The implementations are Microsoft.

## TL;DR

At some point, every environment — Microsoft cloud included — hits the same wall: you stop knowing what's set, where the gaps are, and whether the configuration that exists today matches what was intended. Manual discovery doesn't scale. Neither does institutional memory.

The only durable answer is to declare tenant configuration in code — and then design the identity architecture so that the pipelines delivering that configuration are as tightly governed as the human access they sit alongside. The pipeline is the authorized change channel. Admin accounts remain — held as eligible PIM roles that require peer approval before activation, not as standing access — but for break-glass and portal-only gaps, not for routine configuration changes. Everything outside the pipeline is drift, and drift demands a decision: integrate the change into the repository or revert it.

This blog is about building Microsoft cloud environments you can prove. The two layers that make this possible — the identity architecture that governs every principal (human or workload) that can change the tenant, and the IaC pipeline that delivers those changes — are the same design problem at different scopes.

## Platform focus, transferable principles

The problems this blog addresses — configuration drift, implicit trust in pipeline identities, governance that doesn't survive the first manual exception — are structural. They appear in AWS environments, GCP projects, on-premises Active Directory estates, and hybrid architectures just as reliably as they appear in Microsoft cloud tenants. The patterns that address them — declare desired state in version-controlled text, deliver via a governed pipeline, treat deviation as a defect — apply regardless of which cloud vendor's primitives you're working with.

This blog's working context is Microsoft. That is my practice area, where I have production experience and can write with precision. But Microsoft cloud is not a walled garden: Intune manages Linux, macOS, Android, and iOS alongside Windows; Defender for Cloud extends posture management to AWS and GCP workloads; Entra ID federates with Okta, Google Workspace, ADFS, and any SAML or OIDC-compliant provider; Microsoft Sentinel connects to over three hundred data sources including Rapid7, Palo Alto, CrowdStrike, Cisco, and Splunk; Defender XDR ingests telemetry from non-Microsoft endpoint, network, and cloud tools. The Microsoft stack is the integration layer for a significant proportion of the enterprise security market — not an alternative to it.

The practical implication: when a post describes a PIM activation policy, a Workload Identity Federation pattern, or a Conditional Access authentication context, the specific API calls are Microsoft-specific. The underlying principle — least-privilege just-in-time access, credential-less pipeline authentication, step-up authentication for high-risk operations — is not. Read the Microsoft implementation as a worked example. The discipline transfers; the API surface does not.

## The problem: tenant state you can't account for

Spend enough time inside Microsoft cloud environments and a pattern emerges. Conditional Access exceptions that made sense when they were created. PIM role assignments that outlived the project that created them. Defender settings configured correctly on day one and never reviewed since. M365 self-service licence settings that opened a surface nobody tracked. Tenant security defaults with known gaps that were documented and never closed because the fix felt too risky to apply manually.

The frustration isn't that those changes happened. The frustration is that there is no systematic way to discover what's set, what's missing, and what has drifted — unless you already know where to look. In a tenant that spans Conditional Access, Defender XDR, tenant security defaults, M365 service configurations, Azure control plane resources, and automation coverage, there is no single place to look. You find things one at a time, usually after something breaks or a question you can't answer comes up in a review.

The honest question that raises: if you don't have a complete declaration of what your tenant is supposed to look like, what does "fix the gap" actually mean?

## Everything-as-code changes the question

Infrastructure-as-code, policy-as-code, configuration-as-code: the pattern is the same regardless of toolchain — declare desired state in version-controlled text, enforce it through a pipeline, treat deviation as a defect. The shift from portal administration to everything-as-code changes the question you're able to ask. Instead of *what is set?* you can ask *does the tenant match the declaration?* That is a substantially more useful question.

This is the everything-as-code (EaC) commitment: Conditional Access as code, PIM as code, Defender baselines as code, Sentinel rules as code, M365 tenant settings as code, Azure infrastructure as code. Every layer of tenant state that can be drifted can instead be declared.

When those policies, assignments, baselines, rules, and settings are declared in version-controlled repos and delivered by pipeline, gap analysis becomes mechanical. The source of truth is the repository. The audit trail is the pipeline run log. Replicating a known-good configuration to a new tenant — a new customer, a new environment, a migration target — means running the pipeline against a new scope.

This is the architecture that makes tenant governance tractable. The worked example this blog returns to is a reference MSSP platform — drawn from real multi-customer delivery, generalized here rather than any single customer's deployment. It manages Conditional Access, PIM, Entra ID settings, and Landing Zone configurations for multiple customers from Azure DevOps, with various Workload Identity Federation service principals each scoped to its purpose, no PATs, no client secrets anywhere in the chain. Deployment is JSON-driven: a single config file is the source of truth for all deployment flags; the pipeline composes it into Terraform calls, Graph API writes, and RBAC assignments. PIM is a system across three activation tiers with Conditional Access authentication contexts enforced at activation — the policy lives in Git, the enforcement is in the Entra primitive, the pipeline run is the change record. M365 tenant migrations where the destination configuration is fully declared before the first user moves.

In each case, the practical approach is the same: pipelines using Workload Identity Federation, service principals that authenticate via OIDC token exchange, Terraform or Bicep declaring the desired state. The pipeline runs are the change record. The repository is the configuration snapshot. Assessment is a diff.

The governance model this creates is more than documentation — it is a deliberate inversion of how control plane changes normally work. The pipeline is the authorized change channel, not one method among several. Admin accounts are not standing privileged access: they are eligible PIM roles held by named individuals, requiring peer approval before activation, reserved for break-glass scenarios and the platform API gaps the next section covers. The operating principle is to minimize their direct write footprint on the tenant control plane, not to eliminate human access entirely. Every change that happens outside the pipeline surfaces as drift against the declared baseline — and drift is not noise to suppress, it is a decision point: integrate the change into the repository and let the pipeline redeploy it, or revert it. That loop — declare, deliver, detect, decide — is what makes the governance resilient rather than aspirational. A tenant where every authorized change originates in a pull request, and every detected deviation is either committed to code or rolled back, is one you can prove.

## The tools that exist — and the gap they leave

The ecosystem has not ignored this problem. The assessment side has matured significantly, and four tools between them cover much of the right-hand side of a governance loop — detect, snapshot, diff, and report.

**Unified Tenant Configuration Management (UTCM)** is Microsoft's strategic direction. In staged public preview, it snapshots over three hundred resource types across Entra, Exchange, Intune, Teams, and Defender on a six-hour cycle, surfaces drift against a declared baseline, and integrates into automated reporting pipelines. If Microsoft's direction holds, this is the platform-layer monitoring story for M365 configuration governance.

**Microsoft365DSC** is the community answer that predates UTCM — the closest existing tool to a full governance loop: declarative DSC configuration, apply operations, fifteen-minute drift detection, optional auto-remediation across all M365 workloads. Community-maintained, not Microsoft-supported; Nik Charlebois — one of its primary authors — now leads UTCM at Microsoft, which makes the succession feel less like displacement than translation: the same institutional knowledge, migrating into the supported platform. Its DSC execution model does not fit cleanly into API-first pipelines, and it carries significant operational overhead, but it has more production history than anything else in this space.

**Maester** (v2.0, February 2026) is a read-only security test framework: 285-plus tests drawn from EIDSCA and CIS baselines, covering Entra, Conditional Access, Intune, and Azure, producing HTML and Markdown reports. It does not remediate anything — it is a post-apply validation gate, the step that confirms a pipeline run against a known-good security baseline.

**The Zero Trust Assessment** is Microsoft's GA posture tool: point-in-time across all Zero Trust pillars, useful for a periodic review, not for CI/CD integration. Large tenants take over twenty-four hours to assess; there is no machine-readable diff output.

I run all four — but as the double-check layer after delivery, not as the delivery mechanism. Maester runs as the final stage of every pipeline apply. UTCM monitors run continuously and alert on drift. ZTA and an M365DSC full-tenant export run periodically as compliance snapshots. What these tools catch is exactly what the pipeline can't reach: portal-only toggles, undocumented endpoints, settings that exist in the tenant but have no writable API surface. The gap the tools reveal is the same gap the pipeline is built to close where an API exists, and to document as a permanent exception where it doesn't.

What none of them own is the left side of the loop: declaring desired state in a versioned artifact, delivering it through a governed pipeline, and producing an auditable diff as the authoritative change record. That gap is what the everything-as-code approach fills — and it is what this blog covers.

## The gaps the tools can't close — and where the work goes next

Closing the loop on every layer of Microsoft tenant state is harder than the architecture describes. The Microsoft API surface has real limits, and understanding those limits is as important as understanding the pattern.

**The UTCM write gap.** UTCM is detection-only. The strategic Microsoft tool for drift detection cannot remediate — it can tell you something changed, but it cannot write the change back. Until Microsoft ships write operations against the UTCM API, the left side of the loop — declare, deliver, apply — requires direct Graph API calls and Exchange Online PowerShell. The custom pipeline approach exists because of this gap, not despite it.

**Defender for Endpoint Advanced Features.** Approximately thirty toggle switches under the Defender portal's Advanced Features page have no public Graph API. The community-discovered `XDRInternals` module can read and write these settings using the same session tokens the portal uses internally — acknowledged risk; undocumented endpoints, no Microsoft support guarantee, subject to breaking without notice. MDCA App Connectors each require their own OAuth consent flow with no bulk API. App Governance has a portal-only enable toggle. For initial tenant deployment, either XDRInternals or a manual runbook; UTCM monitors ongoing drift.

**M365 Admin Center tenant-level settings.** Several high-value security settings have no Graph API for write operations: Customer Lockbox, Release Preferences, Idle Session Timeout for unmanaged devices, Sway and Forms external sharing controls. Self-service purchase control requires the separate `MSCommerce` PowerShell module iterated per-product — no single toggle, no Graph API path. These are not obscure settings; Customer Lockbox and self-service purchase control are meaningful security hygiene items, and they cannot be delivered by any pipeline that relies on documented public APIs.

**The exception registry as a first-class artifact.** Portal-only settings are not failures of the EaC approach — they are a category the approach handles explicitly. Each gets a runbook (screenshot-based, operator-executed at initial setup), an entry in an exception registry recording what cannot be automated and why, and a UTCM monitor where the API supports it. The exception registry matters as much as the pipeline configuration: it makes the boundary between what is pipeline-managed and what is manually configured explicit, auditable, and intentional. A documented gap is not a governance failure; an undocumented one is.

**What this means for the series.** The Secure Delivery category is a workload-by-workload account of how this loop closes in practice — Conditional Access, PIM, and Entra ID settings first, then Defender for Office 365, Exchange Online, Intune, Purview, Teams, and SharePoint. Each workload follows the same shape — export desired state to JSON, review in a pull request, deliver via Graph API or PowerShell — but each has its own API surface, its own portal-only corner cases, and its own set of things that do not go in code. The pattern scales; the edge cases do not collapse.

## Where identity and pipeline are the same surface

In any environment — not just Azure — the pipeline that deploys infrastructure carries credentials that can modify the environment it targets, which makes the pipeline's own identity as consequential as the permissions it provisions. Here is the constraint that sharpens the design problem: the pipeline delivering tenant configuration is itself an identity actor in the tenant. The service principal that writes Conditional Access policies, assigns PIM roles, and configures Defender settings holds significant scope over the tenant's security posture. Designing the tenant's identity controls without designing the pipeline's identity controls leaves a back door that is easier to walk through than the front.

The "two perimeters" in the title describes this exactly. The identity controls over human principals — PIM, Conditional Access, JIT, authentication contexts — and the identity controls over workload principals — Workload Identity Federation, service principal scoping, role-assignable group boundaries, no-secret delivery — are two separate constructs that answer the same question: who and what is allowed to make what change to this tenant, under what conditions, and can you prove it?

Both have to be designed against the same threat model. A tenant strict about human access but casual about pipeline identity has a gap that is often structurally simpler to exploit. A pipeline scoped carefully but running under an identity that bypasses the same Conditional Access policies governing human admins does not have a demonstrably better security posture.

One principle that recurs throughout this work: the security boundary often sits at a deeper layer than the surface permission suggests. A pipeline service principal holding tenant-wide `RoleManagementPolicy.ReadWrite.AzureADGroup` looks alarming in isolation. Read in context — the `isAssignableToRole` boolean on the target groups — and the actual boundary is in the Entra primitive, not in the Graph grant. Whether the pipeline that creates those groups is safe depends entirely on that boolean being set correctly at group-creation time. The entire Graph permission is defensible because of a single attribute.

That pattern — defensible boundaries sitting at a layer below where the surface tooling draws the line — is why both perimeters have to be understood together, not handed off to separate teams.

## What the four categories cover

The blog is organised around four categories that carry this thesis across different parts of the Microsoft security stack. The first two categories carry the thesis directly. The third and fourth ground it in the operational reality of real platforms — because governance architecture only proves itself when it runs.

**Identity & Privileged Access** — PIM design at scale across roles, groups, and Azure resources; Conditional Access authentication contexts in practice; RBAC design for tenant isolation in multi-customer deployments; JIT patterns for high-value workloads such as Azure DevOps; the security boundary that role-assignable groups draw around tenant-wide Graph permissions; CA for the new AI-agent identity type.

**Secure Delivery** — Workload Identity Federation patterns for ADO pipelines; self-hosted agents without PATs or stored secrets; three-domain Terraform separation of duties for tenants that write to Entra, Azure ARM, and Microsoft Graph in the same run; supply-chain isolation for security-sensitive IaC libraries; M365 configuration as code ahead of UTCM write operations.

**Security Operations** — Sentinel hunting queries grounded in real privileged-access audit data; Defender for Cloud at management-group scope; KQL parsers for ADO audit logs; assessment pipelines for Zero Trust posture; Sentinel cost optimisation against the March 2027 Azure-portal retirement deadline; Workspace Manager distribution across customer-isolated Sentinel workspaces.

**Landing Zone Architecture** — management-group hierarchy design for multi-customer tenants; subscription-vending automation; sovereign controls and Managed HSM; hub-and-spoke patterns at multi-customer scale; formula-driven customer numbering; three-tier logging architecture across application, security analytics, and archive tiers.

## What comes next

The first technical post takes up the identity half of the thesis: PIM designed as one system across roles, groups, and Azure resources — not as three separate workloads administered in three separate places.

The pipeline-side posts follow. Posts in Security Operations and Landing Zone Architecture interleave with the lead categories rather than running as a separate stream, because the operational evidence for the thesis lives there.

Whatever brought you here — the editorial position the rest of the content is written against is this: tenant state that cannot be reconstructed from code is a liability, and the identity architecture that governs the pipelines which reconstruct it is the other half of the same design problem. One side note worth coming back to: PIM activation for Azure DevOps access is a case where the pipeline approval loop governs the change record in a way that makes it more auditable — and in practice more operationally streamlined — than a break-glass admin activation. That argument gets its own post.
