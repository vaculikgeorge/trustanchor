+++
title = 'PIM at scale: roles, groups, and Azure resources as one system'
date = '2026-06-06T10:00:00+02:00'
draft = false
tags = ['pim', 'entra-id', 'azure-rbac', 'identity-governance', 'authentication-contexts', 'privileged-access']
categories = ['Identity & Privileged Access']
summary = 'PIM is one service in Microsoft Entra admin center, with three management areas — Entra roles, Azure resources, Groups — and no default-policy mechanism in any of them. The integrated view, with consistent tiered policies bound by authentication context, has to be reconstructed in code. This post walks the design and the API patterns that make it work, including the non-inheritance gotcha, the role-assignable-group Member-vs-Owner setup clash, and the service-principal exemption that has to be governed instead of forgotten. The reference-platform section at the end shows how the whole pattern lands in a pipeline with drift detection.'
+++

> Three management areas. Three setup flows. No default-policy mechanism in any of them. The integrated view has to be reconstructed in code.

## TL;DR

Privileged Identity Management is one service with three management areas — Entra roles, Azure resources, Groups — all under the same roof. The unification ends there. Entra activation settings get configured **per role** (eighty-something blades, no default-across-all toggle). Azure resource activation settings get configured **per role per scope** (no inheritance from parent management groups). Group activation settings get configured **per group — twice** (one pane for Members, an independent pane for Owners that most teams never even configure).

It's still one system at the design layer. The thing that holds it together is the authentication context — a label both PIM and Conditional Access can reference. Bind every PIM activation rule to the same context that a Conditional Access policy targets, and the three management areas share one gate at activation time. This post walks the integrated design (three management areas, two API surfaces — Microsoft Graph and Azure ARM — and one binding primitive), the four worked walkthrough patterns, and the gotchas that don't make it into the Microsoft tutorial path. The whole pattern runs unattended in a pipeline with drift detection on the apply side — the reference-platform section at the end shows how that works end to end.

## The problem

There are a few ways to get into PIM — the Microsoft Entra admin center via ID Governance → Privileged Identity Management, the Azure Portal's PIM blade, or a direct jump from a specific role's or group's settings page. They all funnel to the same UI; the canonical path most docs use is Microsoft Entra admin center → ID Governance → Privileged Identity Management. From there you pick **Microsoft Entra roles**, **Azure resources**, or **Groups**. PIM is the unified service.

The unification ends at the menu.

Inside **Microsoft Entra roles**, you configure activation settings *per role*. Eighty-something role blades, each with its own form, no default-across-all toggle. Tier the top thirteen, leave the rest at Microsoft defaults, and you've quietly told the activation flow for *Application Administrator* to look nothing like the activation flow for *Privileged Role Administrator* — which matters as soon as your tenant has more than a few app registrations.

Inside **Azure resources**, you configure activation settings *per role per scope*. The same forms repeat at every management group, every subscription, every resource group. **No inheritance from parent scopes.** Setting an activation policy at the root management group does not propagate downward; every child scope holds its own copy of the policy at Microsoft defaults until the API writes to it explicitly. The number of policies that actually need configuring depends on where your RBAC lives: a pattern that keeps every role assignment at management-group level only needs MG-level PIM policies (child subscription policies stay at Microsoft defaults, which is fine because nobody activates *at* those scopes). A flatter setup with per-subscription role assignments needs per-subscription PIM policies too. Either way, there's no "cascade to descendants" toggle; every scope you choose to manage needs an explicit API write.

Inside **Groups**, you configure activation settings *per group — twice*. Each group has **two independent settings panes**: one for Member activation, one for Owner activation. Neither inherits from the other. Configure the Member side carefully so users have to step up to access the role behind the group; leave the Owner side at Microsoft defaults; and you've quietly authorised a free bypass — anyone with Owner on the group can add themselves (or anyone else) to it directly, without activating, without the auth-context gate, without the approval flow. This is the most common production PIM-for-Groups misconfiguration I see. Covered in depth in the PIM-for-Groups walkthrough below.

Three management areas. Four distinct setup units (per role / per role-per-scope / per group / per *side of group* — meaning the Member settings pane and the Owner settings pane, each its own form). Zero "apply this as the default" mechanism in any of them. A platform team operating manually does the same five clicks eighty-something times for Entra, another N × M times for Azure (N roles × M scopes), and another 2 × G times for Groups (G groups × Member-and-Owner sides). The portal punishes scale and rewards forgetting one of the layers.

The right answer — consistent tiered policies at every role and every scope and both sides of every group, unified by auth-context binding — is mechanically unreachable from the UX, even though all three management areas live in the same portal. It has to be scripted. The rest of this post is what that script, and the design behind it, actually looks like.

## Why existing approaches fall short

The Microsoft docs unify PIM at the conceptual level — the [PIM overview](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/pim-configure) presents it as one service governing Entra roles, Azure resource roles, and Groups. The unification ends at the overview. From there the docs branch into three separate walkthrough sets, one per management area, each describing its own settings flows in isolation. The auth-context binding that lets the three management areas share a policy spine gets one paragraph in the role-management-policy reference and zero paragraphs in any of the user-facing setup guides. That's the integration gap this post fills.

Community write-ups tend to inherit the same split — separate posts on tiered Entra PIM at scale, Azure PIM for subscriptions, and PIM-for-Groups as a standalone pattern. What's missing from most of them is the auth-context binding that lets those three pieces share a policy spine — and the operational reality that you need a scripted path to apply consistent settings at every role, every scope, and both sides of every group, because the UX will not let you do it any other way.

The closest existing community work in this space is [EntraOps](https://github.com/Cloud-Architekt/EntraOps) by Thomas Naunheim — a PowerShell framework that classifies *principals* against Microsoft's Enterprise Access Model (Control Plane / Management Plane / User Access tiers) across Entra directory roles, Azure RBAC, PIM-for-Groups, Microsoft Graph application permissions, Intune RBAC, and Identity Governance delegations. EntraOps reads the privilege landscape, tags classified principals with custom security attributes, surfaces tier violations and attack paths, and emits to Microsoft Sentinel and Azure Workbooks. It writes back where the writeback strengthens governance — creating Restricted Management Administrative Units and Conditional Access security groups derived from the classification — but its centre of gravity is observation and tagging of the existing state.

This post extends the EntraOps principle into the settings layer. EntraOps establishes that the privilege landscape should be treated as code — classified against the Enterprise Access Model into Control / Management / User Access tiers, with the classification driving custom security attributes, Restricted Management AUs, and Sentinel emission. The patterns here take that principle and run it down into the settings that govern *how* activations against the classified landscape get gated: PIM activation duration, approval workflow, expiration rules, and auth-context binding — authored per role, per role-per-scope, and per group-per-side. EntraOps reads the landscape and classifies who holds what; the patterns here write the policy spine across that landscape, at the granularity where the Microsoft portal leaves the real scaling gaps unsolved.

## The pattern

Three management areas, two API surfaces, one binding primitive:

| Management area | What it governs | API surface | Activation gate |
|---|---|---|---|
| **A** — Entra directory roles | Entra ID role activations | Microsoft Graph | Shared step-up auth context |
| **B** — Azure resource roles | Azure RBAC activations (Owner, Contributor, etc. at any scope) | Azure ARM | Shared step-up auth context |
| **C** — PIM-for-Groups | Group-membership activations on role-assignable security groups | Microsoft Graph | Shared step-up auth context |

**All three PIM management areas bind activations to the same step-up auth context.** The differentiation between management areas is *not* in the context — it's in **which roles you configure, at which scopes, with which per-role policy settings** (duration, approval). One context, three management areas, one gate. (The reference-platform context IDs, the device + network + credential controls layered on that context, and how multiple contexts coexist for adjacent purposes like Protected Actions and guest activation — all detailed in the reference-platform section and [I-4 (forthcoming)](/posts/auth-contexts-in-practice/).)

Each management area has its own API surface (Graph for Entra and Groups, ARM for Azure resources). Each has its own *type* of PIM policy primitive (Entra `unifiedRoleManagementPolicy` on the Graph side, `Microsoft.Authorization/roleManagementPolicies` on the ARM side). What unifies them is the authentication context — a label the PIM rule references and the Conditional Access policy targets. At activation time, Conditional Access fires its normal evaluation against that context, regardless of which API requested the activation.

A note on the IDs: authentication context IDs (the strings PIM and CA both reference) are **operator-chosen labels, not Microsoft primitives**. Any non-colliding string works — what matters is that the PIM rule and the Conditional Access policy reference the *same* identifier. That's the binding. One tenant might label its PIM-activation context one way and another label it completely differently — both are equally correct; the choice is local. The reference-platform section names the contexts this worked example uses.

The picture, with the three management areas converging on one gate:

{{< mermaid >}}
flowchart LR
    A["<b>A</b> — Entra directory role activation"]    -->|shared context| Gate["CA evaluation<br/>on the shared step-up context"]
    B["<b>B</b> — Azure resource role activation"]     -->|shared context| Gate
    C["<b>C</b> — PIM-for-Groups membership activation"] -->|shared context| Gate
    Gate -->|device + network + auth strength| Granted["Granted"]
{{< /mermaid >}}

**How to read it.** Three activation paths enter from the left:

- `A` — a user activating an Entra directory role through Microsoft Graph.
- `B` — the same user activating an Azure resource role through ARM.
- `C` — the same user activating membership of a role-assignable security group through Microsoft Graph (PIM-for-Groups). The user is an *eligible member* of the group; activation flips them to an active member for the configured window, and the group's standing role/permission grant takes effect for that window.

All three carry the same shared step-up auth context as a claim on the activation request. Conditional Access doesn't care which API the request came from — it sees the claim, matches it to the policy set targeting that context, and runs the same evaluation for everyone: device posture, network location, authentication strength. If all three controls pass, the activation is granted; if any one fails, the activation fails closed regardless of which management area initiated it. **One binding primitive (the auth-context ID), one evaluator (CA), one outcome surface (granted / denied) for three management areas** — that's the integrated design this post is about. (The reference-platform section covers when a *different* context is appropriate — a separate one for Protected Actions, another for guest activations — but those are separate use cases, not different gates for the same internal-user PIM flows.)

The PIM-side reference to the context is a dedicated rule object that sits alongside the enablement, expiration, and approval rules — its own `unifiedRoleManagementPolicyAuthenticationContextRule` derived type with rule ID `AuthenticationContext_EndUser_Assignment`:

```json
{
  "@odata.type": "#microsoft.graph.unifiedRoleManagementPolicyAuthenticationContextRule",
  "id": "AuthenticationContext_EndUser_Assignment",
  "isEnabled": true,
  "claimValue": "<your-context-id>"
}
```

**That's the joining primitive — and it's worth understanding precisely.** The authentication context is *just a link*. It's a label, nothing more — a string both PIM and Conditional Access can reference. The link goes one way: PIM points at the context (`claimValue` on the PIM rule), and Conditional Access targets the same context (in the policy's conditions). Neither side knows anything about the other; they only know the shared identifier. At activation time, CA's normal evaluation fires against whatever conditions the policies on that context specify — same gate as any other CA-protected operation. The same primitive supports adjacent use cases on separate contexts, with no coupling between them; that's why a single PIM-activation context can govern Entra-role activation, Azure-resource activation, and Platform-MG-role activation without anything special on either side. One primitive, multiple consumers, no cross-talk.

## Walkthrough

Every subsection that follows ships at least one runnable, sanitised API call. The Microsoft docs cover each call in isolation; the integration is the part that lives in scripts, not in tutorials.

### Entra directory roles via Microsoft Graph

The Entra side is the most documented and the most often half-implemented. A common pattern: somebody tiers Global Administrator, Privileged Role Administrator, and Security Administrator carefully — and leaves the other ~80 directory roles at Microsoft defaults. The activation policy for *Application Administrator* matters as much as the policy for Global Administrator if your tenant has more than a few app registrations; the default is far more permissive than it should be.

A tiered model with three tiers and a common shape works well at scale:

| Tier | Scope | Max activation | Approval | Activation gate (end-user) | Admin direct assignment |
|---|---|---|---|---|---|
| **Tier-0** | Critical control-plane roles | **Short** (sized to a single high-risk operation) | Required (second human) | Bound to the shared step-up auth context; CA evaluation on that context enforces a strict device + network + credential gate | `MultiFactorAuthentication` + `Justification` enablement rule. **No auth-context binding** — activation contexts don't apply to direct admin assignments. |
| **Tier-1** | All remaining admin roles | **Medium** (admin-work window) | Self-service | Same auth-context gate as Tier-0 | Same as Tier-0 |
| **Tier-2** | Reader / read-only roles | **Long** (reader-shift window) | Self-service | Same auth-context gate as Tier-0 | Same as Tier-0 |

**All three tiers use the same activation gate.** The credential bar, the device bar, and the network bar are identical across tiers. The tiers differ in two things only: **duration** and **approval** (Tier-0 yes, others no).

A few design notes the table doesn't capture:

- **Tier-0's short window is calibrated to a specific operational pattern.** A short Tier-0 activation makes sense when every privileged change is executed by an automated workflow that completes inside that window, with the human exiting as soon as the change is initiated. Tenants without that pattern — human-only operations, longer change-windows, or interactive troubleshooting — will need to widen the Tier-0 window. The reference-platform calibration appears in the reference-platform section.
- **The credential gate is not "MFA" in the generic sense.** The CA-side `AuthenticationStrengths` policy carries the specifics — which factors are accepted, whether attestation is required, what hardware/software combinations qualify. Always name the strength explicitly (a dedicated authentication-strength policy) rather than saying "MFA"; the difference between a phishing-resistant strength and a hardware-attested strength is significant. The reference platform's specific strength choices appear in the reference-platform section and the underlying mechanics are covered in [I-4 (forthcoming)](/posts/auth-contexts-in-practice/) and [I-13 (forthcoming)](/posts/fido2-aaguid-attestation-whitelisting/).
- **Tier-0 approval is the harm-reduction fail-safe.** Every Tier-0 activation passes through a second human, so a compromised credential or careless click can't reach the role on its own. Self-service Tier-1 / Tier-2 accept the risk on the assumption that the activation gate (the device + network + credential controls on the shared context) already raised the bar enough.
- **Auth context applies to end-user activation, not admin direct assignment.** The PIM rule that carries the context binding is `AuthenticationContext_EndUser_Assignment`. There is no equivalent for admin direct assignments — those are gated by `Enablement_Admin_Assignment` (MFA + justification) with no context binding. So when an admin makes a direct active assignment instead of going through PIM activation, the auth context never fires.
- **Tier-0 role selection is an opinionated subset — and Microsoft's `isPrivileged = true` list is a starting point, not a sufficient one.** The natural starting point is Microsoft's [highly privileged roles](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/privileged-roles-permissions) guidance — the roles flagged `isPrivileged = true` in `roleDefinitions`. That set is necessary but not sufficient. Several roles Microsoft flags `isPrivileged = false` escalate to Tier-0 effective privilege in specific scenarios: *Application Administrator* and *Cloud Application Administrator* can grant client credentials to apps and (depending on existing permissions) take over Global-Admin-equivalent identities; *Hybrid Identity Administrator* controls sync infrastructure that backdoors the cloud directory in hybrid tenants; *Helpdesk Administrator* and *Authentication Administrator* can reset passwords or auth methods on principals they're scoped over, which means without an Administrative Unit scope to constrain them they reach high-privilege accounts. Don't take Microsoft's flag as the boundary; map roles to *effective privilege under your tenant's configuration* — `isPrivileged = true` is the floor, not the ceiling. The specific Tier-0 role list the reference platform uses appears in the reference-platform section.
- **One break-glass exception to the duration rule.** Global Administrator gets a carefully-justified exception to the standard expiration policy for break-glass purposes — covered in the pitfalls section.

Enumerating roles to apply policies to:

```powershell
# Microsoft Graph: list all directory role definitions in the tenant.
# Permission required: RoleManagement.Read.Directory (least privilege).
$roles = Invoke-MgGraphRequest -Method GET `
  -Uri 'https://graph.microsoft.com/v1.0/roleManagement/directory/roleDefinitions' |
  Select-Object -ExpandProperty value
```

Each role has a `roleManagementPolicyAssignment` that binds it to a specific `roleManagementPolicy`. To update the policy that governs a role's activation behaviour, you fetch the assignment, follow it to the policy, and PATCH the policy's rules:

```powershell
# Permission required: RoleManagementPolicy.ReadWrite.Directory.
# scopeType 'DirectoryRole' is Entra; scopeId '/' is the tenant.
$assignments = Invoke-MgGraphRequest -Method GET `
  -Uri "https://graph.microsoft.com/v1.0/policies/roleManagementPolicyAssignments?`$filter=scopeId eq '/' and scopeType eq 'DirectoryRole'" |
  Select-Object -ExpandProperty value

# Pick the assignment that points at the role you want to govern.
$policyId = ($assignments | Where-Object { $_.roleDefinitionId -eq $globalAdminRoleId }).policyId
```

The PATCH body is the load-bearing piece. A Tier-0 policy is four sibling rule objects — enablement, auth-context binding, expiration, approval:

```json
{
  "rules": [
    {
      "@odata.type": "#microsoft.graph.unifiedRoleManagementPolicyEnablementRule",
      "id": "Enablement_EndUser_Assignment",
      "enabledRules": ["MultiFactorAuthentication", "Justification"]
    },
    {
      "@odata.type": "#microsoft.graph.unifiedRoleManagementPolicyAuthenticationContextRule",
      "id": "AuthenticationContext_EndUser_Assignment",
      "isEnabled": true,
      "claimValue": "<pim-activation-context-id>"
    },
    {
      "@odata.type": "#microsoft.graph.unifiedRoleManagementPolicyExpirationRule",
      "id": "Expiration_EndUser_Assignment",
      "isExpirationRequired": true,
      "maximumDuration": "<tier-0-duration>"
    },
    {
      "@odata.type": "#microsoft.graph.unifiedRoleManagementPolicyApprovalRule",
      "id": "Approval_EndUser_Assignment",
      "setting": {
        "isApprovalRequired": true,
        "approvalStages": [{ "primaryApprovers": [/* approver objects */] }]
      }
    }
  ]
}
```

Tier-1 and Tier-2 policies both drop the approval requirement (self-service) and keep justification on activation. Tier-1's `maximumDuration` is a medium admin-work window; Tier-2's is a longer reader-shift window. All three tiers keep the auth-context binding on the same PIM-activation context.

The body above is the bulk-PATCH shape that works when you send a single `PATCH /policies/roleManagementPolicies/{policyId}` with all rules at once. **Production code typically PATCHes each rule individually** at `/policies/roleManagementPolicies/{policyId}/rules/{ruleId}` — same semantic effect, clearer per-rule error reporting, lets one bad rule fail without rolling back the others. Both patterns are supported by the API; the per-rule pattern is what the reference implementation in the reference-platform section uses.

Classifying the ~95 directory roles into tiers by hand is one of those tasks that's *almost* automatable. The shape that works:

```powershell
function Get-RoleTier {
    param([string]$DisplayName)
    # $tier0 = a tenant-specific list of high-privilege role display names.
    # See the reference-platform section for the specific Tier-0 role list.
    if ($tier0 -contains $DisplayName)    { return 'Tier-0' }
    if ($DisplayName -match '\bReaders?\b') { return 'Tier-2' }
    return 'Tier-1'
}
```

The fallthrough to `Tier-1` is the bit that makes this forward-compatible: when Microsoft ships a new directory role (and they do — several per year), it gets the operational defaults until you decide otherwise. The `\bReaders?\b` whole-word regex for Tier-2 is more precise than a bare substring (it won't match a role named "MyReaderHelper"), but it's still fragile against custom roles like "Reporting Reader" or "Quarterly Review Reader" that should arguably be Tier-1, and against future Microsoft roles that may use "Reader" in a name but warrant Tier-0. Audit the classification annually and override per-role where the heuristic gets it wrong.

### Azure resource roles via ARM

Azure resource PIM uses a different API surface: ARM, against the `Microsoft.Authorization/roleManagementPolicies` resource type. The current stable api-version is `2020-10-01`; the latest preview is `2024-09-01-preview`. The endpoints are described in Microsoft's [ARM template reference](https://learn.microsoft.com/azure/templates/microsoft.authorization/rolemanagementpolicies).

The single design choice that matters most: **Azure PIM does not inherit policies from parent scopes.** Setting Owner activation to a tighter policy at the root management group does not propagate to the child subscriptions. Each subscription, each child management group, each resource group has its own copy of the policy at Microsoft defaults. The portal does not surface this; the API does, the first time you query it.

The fix is the Management Groups Descendants API:

```powershell
# ARM: enumerate every child MG and subscription under a root MG in one call.
$rootMgId = '<your-root-mg-id>'
$descendants = Invoke-AzRestMethod -Method GET `
  -Path "/providers/Microsoft.Management/managementGroups/$rootMgId/descendants?api-version=2021-04-01"
$scopes = ($descendants.Content | ConvertFrom-Json).value
```

Do *not* iterate `Get-AzManagementGroup` recursively for this. The Descendants API returns a flat list of every management group and every subscription beneath the root in one round trip. Iterating PowerShell cmdlets adds N more calls and N more chances to hit throttling on a tenant with hundreds of subscriptions.

Applying the policy at each scope is then a loop:

```powershell
foreach ($scope in $scopes) {
    $scopePath = $scope.id   # e.g. /providers/Microsoft.Management/managementGroups/<mg-id>
                              # or  /subscriptions/<guid>

    # Find the role-management-policy assignment for Owner at this scope.
    $assignmentsUri =
      "$scopePath/providers/Microsoft.Authorization/roleManagementPolicyAssignments?api-version=2020-10-01"
    $resp = Invoke-AzRestMethod -Method GET -Path $assignmentsUri
    $ownerAssignment = ($resp.Content | ConvertFrom-Json).value |
        Where-Object { $_.properties.roleDefinitionId -like '*8e3af657-a8ff-443c-a75c-2fe8c4bcb635*' }

    # PATCH the underlying policy.
    $policyUri =
      "$($ownerAssignment.properties.policyId)?api-version=2020-10-01"
    Invoke-AzRestMethod -Method PATCH -Path $policyUri -Payload $tierOwnerPolicyBody
}
```

The `roleDefinitionId` GUID above is Owner. The full Azure built-in role GUID list is in the [Microsoft RBAC docs](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles).

The body shape mirrors the Graph side conceptually, but the ARM PIM rule names are different — `RoleManagementPolicyEnablementRule`, `RoleManagementPolicyExpirationRule`, `RoleManagementPolicyApprovalRule`, `RoleManagementPolicyAuthenticationContextRule`. The same idea applies: the auth-context rule binds the shared PIM-activation context, an expiration rule sets the `maximumDuration`, an approval rule controls whether approval is required.

A two-tier model with a common shape works at the ARM layer:

| Tier | Example roles at MG / subscription scope | Max eligible duration | Max active-assignment duration | Max activation duration | Activation gate (end-user) |
|---|---|---|---|---|---|
| **Resource-admin tier** | Owner, User Access Administrator, Role Based Access Control Administrator | **Long** (multi-month eligibility window) | **Medium** (short active-assignment ceiling) | **Short** (single high-risk operation) | Shared step-up auth context; **same CA evaluation as the Entra Tier-0 gate** |
| **Resource-operator tier** | Contributor, Key Vault Administrator, Storage Blob Data Owner, Network Contributor | **Longer** | **Longer** | **Longer** (work-session window) | Same shared step-up auth context |

The activation durations matter — too short and operators thrash; too long and you've quietly recreated standing access in a different shape. The relative shape (shorter for resource-admin, longer for resource-operator) holds across tenants; concrete number choices (with the trade-offs that drove them) appear in the reference-platform section.

A further refinement worth flagging here, even though the worked example lives in the reference-platform section: the two-tier shape above applies *uniformly* per role. Some (role, scope) tuples warrant promotion above the table — a Contributor at a child subscription isn't the same risk as the same role at a tier-0 management group; a Sentinel Contributor on a security-tier MG is admin work, not operator work. A small classifier function over (role, scope) tuples extends this two-tier model into per-(role, scope) tiering without changing the gate or the API. The reference-platform section shows the implementation.

### PIM-for-Groups — the two-layer pattern, and the Member-vs-Owner trap

PIM-for-Groups is the third management area, and its use cases reach beyond Azure resource roles. The pattern shows up wherever a privilege binding lives outside PIM's direct reach — anywhere a system uses group claims or group membership as the authorisation surface:

- **Azure resource roles at scale**, where direct per-user RBAC assignments would hit the subscription role-assignment limit (a few thousand assignments per subscription, reached sooner if every user gets their own per-resource grant). Assign the role permanently to a security group; make users eligible members.
- **Microsoft Defender XDR Unified RBAC roles**, which are assigned to groups, not to users directly. Putting PIM-for-Groups on those groups is how Defender Unified RBAC role memberships become JIT-activatable.
- **Azure DevOps project access**, where ADO permissions are evaluated against Entra group membership. PIM-for-Groups on the project-access group makes ADO access JIT — covered in [I-2 (forthcoming)](/posts/jit-ado-via-pim-and-ca-auth-contexts/).
- **Any third-party SaaS** that consumes Entra group claims for authorisation — group-claim membership becomes the access decision, and PIM-for-Groups governs when membership is active.

The shape is the same in every case: assign the privilege permanently to a security group, make users eligible members of the group, activate group membership through PIM. The group-to-privilege binding is stable; the user-to-group binding is the dynamic part.

The group has to be `isAssignableToRole = true` to be safe — that flag means the group itself is protected against modification by non-privileged admins. (The `isAssignableToRole` flag has a much longer story behind it; that's the subject of [I-6 (forthcoming)](/posts/role-assignable-pim-groups-security-boundary/). For now, take it as load-bearing.)

```powershell
# Create a role-assignable security group.
# Graph permission required: Group.ReadWrite.All on Microsoft Graph.
# Directory role required on the calling principal: Privileged Role Administrator
# (or Global Administrator) — needed because isAssignableToRole = true protects
# the group against modification by non-privileged admins.
$groupBody = @{
    displayName        = '<your-group-name>'
    description        = 'Eligible-only owners of a specific Azure scope'
    mailEnabled        = $false
    mailNickname       = '<your-mail-nickname>'
    securityEnabled    = $true
    isAssignableToRole = $true
} | ConvertTo-Json

$group = Invoke-MgGraphRequest -Method POST `
  -Uri 'https://graph.microsoft.com/v1.0/groups' -Body $groupBody
```

Then permanently assign the Azure role to the group:

```powershell
New-AzRoleAssignment `
  -ObjectId $group.id `
  -RoleDefinitionName 'Owner' `
  -Scope '/subscriptions/00000000-0000-0000-0000-000000000000'
```

Then make a user eligible as a *member* of the group via PIM:

```powershell
$memberRequestBody = @{
    accessId      = 'member'
    action        = 'adminAssign'
    groupId       = $group.id
    principalId   = $userObjectId
    justification = 'Eligible owner via group membership for the relevant Azure scope'
    scheduleInfo  = @{
        startDateTime = (Get-Date).ToString('o')
        expiration    = @{ type = 'noExpiration' }
    }
} | ConvertTo-Json -Depth 4

Invoke-MgGraphRequest -Method POST `
  -Uri 'https://graph.microsoft.com/v1.0/identityGovernance/privilegedAccess/group/eligibilityScheduleRequests' `
  -Body $memberRequestBody
```

Most write-ups stop here. That's the bug.

A role-assignable security group has two layers of access: members and owners. Members get the role behind the group when they activate. Owners can add or remove members directly, without activating anything. If you put PIM activation on the Member side and leave the Owner side wide open, anyone with Owner on that group can grant themselves the role behind it — or grant it to anyone else — without ever going through the activation flow, the auth-context gate, the approval workflow, or the audit signal that activation produces.

This is the most common PIM-for-Groups misconfiguration I see. Most organisations close the front door (Member activation through PIM) and leave the back door (Owner direct edits) wide open.

Two fixes, in order.

**First, govern both sides of every role-assignable group — even when the Owner side is currently empty.** Apply the same PIM-for-Groups configuration to the Owner pane as to the Member pane: eligible-only assignment, the same auth-context binding, the same activation duration, the same approval workflow. Empty today does not mean empty tomorrow — a default Owner can appear the moment somebody creates an `adminAssign` to the Owner side via the portal or a misconfigured script. Configure the policy now so the gate exists whether or not it's currently exercised. The `accessId` field on `eligibilityScheduleRequests` is what controls which side you're configuring:

```powershell
$ownerRequestBody = @{
    accessId      = 'owner'   # the side most orgs forget
    action        = 'adminAssign'
    groupId       = $group.id
    principalId   = $ownerObjectId
    justification = 'Eligible owner-administrator for the group itself'
    scheduleInfo  = @{
        startDateTime = (Get-Date).ToString('o')
        expiration    = @{ type = 'noExpiration' }
    }
} | ConvertTo-Json -Depth 4

Invoke-MgGraphRequest -Method POST `
  -Uri 'https://graph.microsoft.com/v1.0/identityGovernance/privilegedAccess/group/eligibilityScheduleRequests' `
  -Body $ownerRequestBody
```

**Second, minimise the Owner population on top of the gated configuration.** Drive the Owner side toward zero individual Owners: prefer "owned by a role-assignable governance group" over "owned by a list of people," and prune Owners inherited from group-creation defaults (the creating principal becomes an Owner unless you intervene). The combination is the point: the Owner side has a gate, and what's behind the gate is as small as the design allows. Either the Owner side is governed and near-empty, or the cheaper layer is a free bypass.

A useful complement: a Sentinel rule that fires on any direct group-membership change (Add or Remove) on a role-assignable group, when the change did *not* originate from a PIM activation event. The change is technically possible — group Owners exist, after all — but every occurrence of it should be a deliberate, justified, audited event, not a routine click.

## Pitfalls and gotchas

Seven items, ordered by how often they catch people.

**1. Azure PIM doesn't inherit.** The first pitfall, repeated for emphasis. Every management group and every subscription holds its own copy of the activation policy. The Descendants API gives you a flat list of scopes in one call; iterate `Get-AzManagementGroup` recursively only if you enjoy throttling. New subscriptions need a re-run of the policy application; an Event Grid subscription on `Microsoft.Resources/subscriptions/write` events feeding an Automation Runbook is the closing-the-loop pattern.

**2. Approval groups must be `isAssignableToRole = true`.** Tier-0 activation requires approval; the approver group has to be configured. If the approver group is a regular security group, a non-privileged administrator who can modify group membership can grant themselves the role by adding themselves to the approver list. The fix: every approver group is itself role-assignable, with its own PIM-for-Groups governance on both Member and Owner sides. [I-6 (forthcoming)](/posts/role-assignable-pim-groups-security-boundary/) covers the `isAssignableToRole` boundary in depth.

**3. `isExpirationRequired = false` is the exemption for permanent PIM-governed active assignments — and that exemption needs to be governed, not forgotten.** A note on terminology first, because PIM uses *assignment* and *activation* for two distinct things:

- An **assignment** is the act of granting a principal a role. PIM supports two assignment states: *eligible* (the principal is a candidate to step up to the role) and *active* (the principal currently holds the role). Both are created by an admin (`adminAssign`) or by a self-service request, depending on policy. An *eligible assignment* and an *active assignment* are two different ways of holding the same role.
- An **activation** is the distinct, time-bounded JIT transition that moves an existing eligible assignment into the active state for a defined window. This is PIM's just-in-time step: the principal already has an eligible assignment; activation turns it active for the duration of the activation window.

Active status can therefore be reached two ways: (1) admin-assigned active directly, or (2) admin-assigned eligible followed by user-initiated activation. The recommended flow for human users is (2) — eligible + activation — because it forces the JIT step and the auth-context gate. Direct active assignment bypasses the activation flow and its gate (covered in the Entra walkthrough's design-notes bullet).

The `Expiration_Admin_Assignment` rule constrains how long an active assignment can persist before it must expire or be renewed; it enforces a maximum duration on the active-status side. Setting `isExpirationRequired = false` is the explicit exemption from that rule — it allows an active assignment to persist permanently.

Two production scenarios need the exemption:

1. **Break-glass Global Administrator.** Two cloud-only accounts with permanent active assignment (`isExpirationRequired = false`), explicitly excluded from every Conditional Access policy, quarterly tested. The whole point of break-glass is that it works when everything else is broken; subjecting it to a step-up policy that may also be broken is a circular dependency. The exception you don't test is the one that surprises you the day you need it — document the quarterly validation cadence as a calendar event, not a hope.

2. **Service principals that need an Entra directory role itself, not just Graph permissions.** Some workloads — automation scripts, deployment pipelines, integration agents, scheduled jobs — need a service principal to hold an actual directory role (e.g., *Authentication Policy Administrator*, *User Administrator*, *Cloud Application Administrator*) because the operation isn't covered by Graph application permissions alone. SPs can't go through PIM activation (no human, no MFA prompt to clear), so when the role is PIM-governed those SP assignments need permanent **active** assignment via this exemption. The need is general to any non-interactive identity, not specific to one type of workload. (SPs that need only Graph permissions sit outside PIM entirely — they get application permissions on the resource, and the `isExpirationRequired` setting is irrelevant.)

The exemption is necessary in both cases. The governance around the exemption list is the deliverable: maintain a tenant inventory of every principal (human break-glass or SP) that holds an `isExpirationRequired = false` PIM-governed active assignment, record the role and the scope, review quarterly the same way you'd review break-glass, and require a pull request review for any new entry. The exemption itself is fine. The unmonitored list of exemptions is the gap that quietly grows past the point where anyone remembers what's on it.

**4. PIM-for-Groups: govern both sides first, then minimise the Owner population.** Covered above; repeated here because it's the most common production misconfiguration. The asymmetric configuration (Member behind PIM, Owner wide open) leaves a free bypass that's invisible until somebody uses it. The fix in order: (1) apply the same eligible-only PIM governance to the Owner pane as to the Member pane on every role-assignable group, including groups whose Owner side is currently empty — empty today does not mean empty tomorrow; (2) on top of that gated configuration, drive the Owner population toward zero individual Owners — prefer "owned by a role-assignable governance group" over "owned by people," and prune Owners inherited from group-creation defaults. The audit signal worth shipping: alert on direct group-membership changes on role-assignable groups that didn't originate from a PIM activation.

**5. Tier-1 fallthrough is forward-compatible — but watch new Microsoft roles.** The `Get-RoleTier` function above defaults any unrecognised role to Tier-1, which means new Microsoft directory roles get sensible operational defaults the moment they appear. That's deliberate. The risk is that Microsoft eventually ships a role that should have been Tier-0 — something like "Application Approver" or a future agent-identity admin — and your tenant runs it at Tier-1 settings for months because nobody noticed. The mitigation: review new Microsoft directory roles every quarter and decide whether the fallthrough placed them correctly.

**6. `\bReaders?\b` name match for Tier-2 is fragile.** The classifier above uses a whole-word regex (more precise than a bare substring — it won't catch "MyReaderHelper") to route Reader roles to the long-duration tier. This works for Microsoft's current naming convention. It does not work for custom roles named "Reporting Reader" or "Quarterly Review Reader" that should still be Tier-1, and it does not work for future Microsoft roles named "Settings Reader" that may need to be Tier-0. The same audit cadence applies: review the classification annually, override per-role where the heuristic gets it wrong. Long-term direction: replace the name heuristic with a deterministic classifier — either an explicit per-role override config (`pim-role-archetypes.json`-style), or classify by role permissions / `isPrivileged` flag rather than name pattern. (The reference platform shrinks the audit latency from annual-by-hand to *daily-by-pipeline*: an assessment job enumerates the current `roleDefinitions` set every day, compares the count and the per-role classification to a baseline file, and surfaces any new or renamed Microsoft role as drift before the operator notices. The reference-platform section covers the implementation.)

**7. Auth context IDs are operator-chosen labels, and bind to *activation* only.** Auth context IDs are not Microsoft primitives. Any non-colliding string the operator chooses works — what matters is the match between the PIM rule's `claimValue` on the `AuthenticationContext_EndUser_Assignment` rule and the same identifier targeted by the CA policy's conditions. Two important constraints: (a) auth contexts bind to **end-user activations only** — there is no equivalent rule for admin direct assignments (`Enablement_Admin_Assignment` carries only MFA/Justification, no context), so direct assignments bypass the context gate; (b) the same primitive can be used for adjacent purposes on separate contexts (e.g., Protected Actions on one context, internal-user PIM activation on another, guest PIM activation on a third) — every tenant defines its own naming scheme. The CA-side mechanics that the context resolves to (the device gate, network gate, and credential-strength grant control) are covered in [I-4 (forthcoming)](/posts/auth-contexts-in-practice/).

## When not to use this

A small tenant with one subscription and a handful of admins doesn't need *the Azure-resource side* of this framework — the Descendants walk and per-scope ARM policy maintenance assume many MGs and many subscriptions, and that's the operational overhead a single-subscription tenant doesn't need. The Entra side (tiered directory-role activation policies + auth-context binding) and the PIM-for-Groups patterns still apply at any size, because Entra roles and group-claim consumers exist in every tenant. The threshold for the Azure-resource walk specifically is *more than a handful of scopes* — once policies need to be applied to ten-plus management groups or subscriptions, the framework here pays for itself. Below that threshold, standard activation through the portal — even if it leaves some defaults wider than they should be — may be the right call for the Azure side, while still adopting the Entra and PIM-for-Groups pieces.

Tenants without Microsoft Entra ID P2 (or Microsoft Entra Suite) licensing don't have PIM. The features assumed throughout this post — eligible assignments, activation policies, role-management policy rules, PIM-for-Groups — require P2 floor licensing. The patterns transfer the moment you have the licensing; until then, conditional access for standing assignments is the substitute, and a much weaker one.

Tenants that haven't adopted group-based PIM — where users are still directly assigned to roles — should plan that migration before adopting the patterns here. The two-layer PIM-for-Groups model assumes you've already decided to push role-via-group rather than role-direct. Migrating from direct to group-based is its own design exercise.

## How this plays out in the reference platform

The principle body above is the design. This section is the concrete implementation in the reference platform behind that design — a worked example drawn from real multi-customer MSSP delivery, generalized here rather than any single customer's deployment. It covers the auth-context IDs the platform chose, the durations calibrated to a pipeline-driven operations model, the role-list opinions for Tier-0, the platform-MG paths, the authentication strengths layered onto the shared step-up context, and the pipeline that ships the lot with drift detection.

### Three auth contexts, one PIM-activation gate

The platform uses three authentication contexts, each an operator-chosen label local to the tenant. What matters is their *purpose*, not the string:

| Context | What it gates | What's bound to it |
|---|---|---|
| **Protected Actions** | sensitive admin operations like modifying CA policies or cross-tenant settings. Not PIM activations. | Entra Protected Actions configuration. |
| **Internal-user PIM activation** | every Entra / Azure / Group PIM activation by an internal admin hits this one context. | All three PIM management areas bind activation to it. |
| **Guest PIM activation** | step-up for B2B / guest users activating eligible PIM roles. Distinct from the internal-user context because guests typically can't satisfy the strict hardware-attested-only credential requirement and need a phishing-resistant-but-not-hardware-attested gate. | Guest-principal PIM activation paths. |

The principle body's claim — *"all three PIM management areas bind activations to the same step-up auth context"* — refers to the internal-user PIM-activation context here.

### Triple-gate on the PIM-activation context

The internal-user PIM-activation context triggers three Conditional Access policies forming a triple-gate (device + network + credential):

1. **Device** — block any PIM activation on a device that is not Entra-joined and not compliant (Intune-evaluated).
2. **Network** — block any PIM activation from outside the Global Secure Access network. (Trusted-IP-via-named-location is the older variant; GSA-network-based filtering replaces it on this platform.)
3. **Credential** — require a hardware-attested authentication strength: FIDO2 hardware-attested keys only.

This hardware-attested strength is *not* generic MFA. It's a Conditional Access authentication strength that accepts only FIDO2 hardware keys whose AAGUIDs match the tenant's allow-list — restricting to **hardware-attested keys only** (YubiKey, Feitian, HID, and similar). Explicitly excluded from this strength: Microsoft Authenticator passkeys, Windows Hello for Business (WHfB), Windows Hello passkeys — none of those support hardware attestation. SMS, voice, software TOTP are also out, but for a different reason: they're disabled at the tenant level via the Authentication Methods policy.

A separate, phishing-resistant strength exists for cases that don't need hardware attestation; it accepts WHfB + FIDO2 + x509. The platform binds that strength to the guest-activation context, where requiring guest principals to carry hardware-attested keys would be operationally unviable for most engagements. [I-13 (forthcoming)](/posts/fido2-aaguid-attestation-whitelisting/) deep-dives the AAGUID-filtering primitive.

The CA-side mechanics — what each context *actually* enforces in the tenant, what session controls it layers, what grant controls it requires — is the subject of [I-4 (forthcoming)](/posts/auth-contexts-in-practice/). The contract between the PIM-side scripts and the CA-side policies is the auth-context ID, nothing else.

### Concrete tier durations

The platform's Tier-0 / Tier-1 / Tier-2 tier model with concrete durations — these govern **Entra directory-role activation in the portal / Graph**, not Azure DevOps activation. ADO uses a separate set of PIM-for-Groups bindings with its own activation policies, covered in [I-2 (forthcoming)](/posts/jit-ado-via-pim-and-ca-auth-contexts/):

| Tier | Max activation | Calibration rationale |
|---|---|---|
| **Tier-0** | `PT1H` | Calibrated to the blast radius of Tier-0 directory roles — these are roles that can directly escalate privileges, bypass security controls, or compromise the entire identity plane (Global Administrator, Privileged Role Administrator, etc.). One hour is a deliberately short window for a single high-risk portal-management operation: enough time to navigate to the right blade, make the change, validate, and exit. Not a routine work window — Tier-0 is supposed to feel uncomfortable. |
| **Tier-1** | `PT2H` | Admin-work window for interactive Tier-1 portal operations — long enough to complete a coherent admin task, short enough to force re-activation between sessions. |
| **Tier-2** | `PT8H` | Reader-shift window — covers an investigative or support session without thrashing. |

The Tier-0 `PT1H` is a blast-radius calibration. Tenants whose Tier-0 activity is rare and tightly bounded (most operations land on a service principal with permanent active assignment via the `isExpirationRequired = false` exemption above) may shrink Tier-0 even further; tenants with longer interactive change windows may need `PT2H` or `PT4H` Tier-0.

### Tier-0 role list (13 roles)

The platform's Tier-0 list — an opinionated subset of Microsoft's [highly privileged roles](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/privileged-roles-permissions) guidance:

```text
Global Administrator
Privileged Role Administrator
Privileged Authentication Administrator
Security Administrator
Conditional Access Administrator
Application Administrator
Cloud Application Administrator
Hybrid Identity Administrator
Exchange Administrator
SharePoint Administrator
Intune Administrator
Domain Name Administrator
External Identity Provider Administrator
```

Thirteen roles. Widen for your tenant if your threat model warrants it. Application Administrator and Cloud Application Administrator carry hidden privilege escalation paths via service principal credentials — they belong at Tier-0 in any tenant with a non-trivial app-registration count.

### Platform-MG roles and scopes

The platform's `Get-TierForRole` function — concrete platform-MG scope IDs and the platform-admin role list:

```powershell
function Get-TierForRole {
    param([string]$RoleName, [string]$ScopePath)

    $platformMgs = @(
        '/providers/Microsoft.Management/managementGroups/<management-mg>',
        '/providers/Microsoft.Management/managementGroups/<security-mg>',
        '/providers/Microsoft.Management/managementGroups/<connectivity-mg>'
    )
    $platformAdminRoles = @(
        'Microsoft Sentinel Contributor',
        'Microsoft Sentinel Responder',
        'Log Analytics Contributor',
        'Monitoring Contributor',
        'Automation Contributor'
    )

    if ($platformMgs -contains $ScopePath -and $platformAdminRoles -contains $RoleName) {
        return 'Tier-0'   # platform-admin role at a platform MG — PT1H + approval
    }
    return 'Tier-1'       # general resource activation — PT2H + self-service
}
```

(MG names are illustrative; the reference platform uses customer-coded names like `<customer>-management`, `<customer>-security`, `<customer>-connectivity` under a per-customer root MG.)

### Pipeline orchestration

The platform splits the PIM workload across two pipeline tracks — *policy track* (the per-role / per-scope / per-side rules) and *membership track* (who is eligible at each tier).

**Policy track** — three deployment scripts, one per PIM management area:

| Script | Management area | API surface |
|---|---|---|
| `Import-PIMRoleManagementPolicies.ps1` | Entra directory roles | Microsoft Graph |
| `Import-PIMAzureResourcePolicies.ps1` | Azure resource roles | Azure ARM |
| `Import-PIMAzurePlatformPolicies.ps1` | Platform-MG Azure roles | Azure ARM (with the `Get-TierForRole` classifier) |

All three are idempotent. All three support `-WhatIf`. All three are invoked from a single pipeline stage that fans out to the three management areas in parallel.

**Membership track** — a user-lifecycle pipeline governs *who* is eligible for which PIM-for-Groups membership at any time. The lifecycle stages:

| Stage | Script | What it does |
|---|---|---|
| **Onboard** | `Invoke-UserOnboard.ps1` (drives `Get-UserTierBundle.ps1` → tier classification → adds the user as an *eligible member* to the matching role-assignable groups via the `eligibilityScheduleRequests` API) | Admits a new admin to the platform with explicit tier and scope bundle. Idempotent against existing eligibilities. |
| **Govern** | `Invoke-UserGovern.ps1` (drives `Get-UserLifecycleSnapshot.ps1` → enumerates per-tier members + per-admin state → `Compare-UserLifecycleSnapshot.ps1` → diffs vs the previous daily snapshot) | Read-only daily reconciliation. Snapshots every group's eligibility set, diffs it against yesterday's snapshot, opens work items for any drift the pipeline didn't author. |
| **Offboard** | (paired with HR-driven trigger) | Removes the principal from every eligibility schedule and from the SP-exemption registry if present. |

The role-assignable security groups that hold the per-scope Owner / Contributor / Sentinel-Contributor / etc. assignments are generated by a separate script (`New-PlatformPIMGroups.ps1`) that owns the naming convention (`pim-{scope}-{role}` shape — illustrative; the reference platform uses a customer-coded prefix), ensures every group is `isAssignableToRole = true` at creation time, and configures both the Member-side and the Owner-side activation policies before any membership flows in. Eligibility on both Member and Owner sides is configured through the same PIM-for-Groups API calls shown in the PIM-for-Groups walkthrough; the symmetry is enforced in code, not relied on as a convention. The Owner-pane policy is set on every group regardless of whether the group currently has Owners, so the gate exists when membership ever appears.

A single CA policy *set* (the three policies forming the triple-gate) handles the internal-user PIM-activation context. The PIM-side scripts know which context ID they bind to; they do not know — and do not need to know — the contents of those CA policies. The contract is the auth-context ID, nothing else. The Protected Actions and guest-activation contexts exist for separate purposes and bind to their own CA policies, which is what makes the auth-context primitive scale.

### SP-exemption registry (as code)

The service-principal exemption list is **code** — a YAML file checked into the same repository as the policy scripts, version-controlled, code-reviewed, and read by the pipeline at runtime. One entry per SP that holds a permanent active assignment. Every entry has a role, a scope, an owner (the human accountable for the SP), and a justification. The pipeline reads the file and the PIM scripts honour it as the only legitimate source of `isExpirationRequired = false` SP assignments; anything that exists in the tenant but not in the file is flagged as drift at the next run. Quarterly review of the file is a calendar event, not a hope. Adding an entry requires a pull request review from a second engineer — the exemption is governance-as-code, with the same review surface as any other policy change.

### Sentinel rules for direct PIM changes (membership, role assignment, and policy)

Three sister Sentinel rules cover the three ways the PIM pattern can be circumvented out-of-band — direct edits to a role-assignable group's membership, direct directory-role assignments that skip the PIM workflow, and direct edits to PIM policy itself. Two of the three are deployed in the platform's KQL repo today (Rules A and B below); the third (Rule C) is the policy-side companion still on the to-do list. All three anchor detection on PIM signals — `LoggedByService == "PIM"`, ARM `roleManagementPolicies` operations, and watchlist-resolved role-assignable group IDs.

**Rule A — Direct membership changes on role-assignable groups (deployed).** The detection logic: a group-membership change event on a role-assignable group, where the operation was *not* logged by the PIM service (PIM-driven activations and admin assignments show `LoggedByService == "PIM"`; direct portal/Graph edits to a role-assignable group's membership do not), and the initiator is not the pipeline service principal.

```kql
let allowed_initiators = dynamic([
    "<pipeline-sp-app-id>"
]);
// Sentinel watchlist of role-assignable group object IDs, refreshed by a
// scheduled job that queries Microsoft Graph for groups where
// isAssignableToRole = true. Anchors the detection to the PIM-governance scope
// rather than to a group-name pattern.
let role_assignable_groups =
    _GetWatchlist("RoleAssignableGroups")
    | project SearchKey;
AuditLogs
| where OperationName in (
    "Add member to group",
    "Remove member from group",
    "Add owner to group",
    "Remove owner from group"
  )
| where LoggedByService != "PIM"                     // PIM-driven changes are excluded
| extend group_id = tostring(TargetResources[0].id)
| where group_id in (role_assignable_groups)         // only role-assignable groups
| extend init_id = coalesce(
    tostring(InitiatedBy.app.appId),
    tostring(InitiatedBy.user.id)
  )
| where init_id !in (allowed_initiators)             // not the pipeline SP
| project TimeGenerated, OperationName, group_id, InitiatedBy
```

**Rule B — Direct directory role assignment outside pipeline (deployed).** Rule A covers the *group* side of out-of-band privilege change — someone adding themselves (or another user) to a role-assignable group via the portal or Graph. Rule B covers the *direct role* side — someone assigning a directory role to a user without going through the platform's role-management pipeline. Same family of attack, different vector: Rule A bypasses the group-then-PIM-activation path; Rule B bypasses the whole pipeline-as-the-only-writer model.

The detection logic anchors on `AuditLogs` role-assignment operations (`Add member to role`, `Add eligible member to role`, and the PIM-driven variants) and filters out any change whose initiator is the pipeline service principal. The query then walks `TargetResources[0].modifiedProperties` to surface the role display name on the alert.

```kql
let allowed_initiators = dynamic([
    "<pipeline-sp-app-id>"
]);
AuditLogs
| where TimeGenerated > ago(1h)
| where OperationName in (
    "Add member to role",
    "Add eligible member to role",
    "Add member to role in PIM requested (permanent)",
    "Add member to role completed (PIM activation)"
  )
| extend init_app  = tostring(InitiatedBy.app.appId)
| extend init_user = tostring(InitiatedBy.user.userPrincipalName)
| where isnotempty(init_app) or isnotempty(init_user)
| where init_app !in (allowed_initiators)
| extend target_user = tostring(TargetResources[0].userPrincipalName)
// modifiedProperties is an array — expand and filter to find Role.DisplayName
| mv-expand mp = TargetResources[0].modifiedProperties
| where tostring(mp.displayName) == "Role.DisplayName"
// newValue is JSON-string-encoded (quoted), parse_json strips the quotes
| extend role_name = tostring(parse_json(tostring(mp.newValue)))
| project TimeGenerated, OperationName, target_user, role_name, init_user, init_app, ResultDescription
```

This rule and Rule A are complementary, not redundant. Rule A fires on a role-assignable *group*'s membership change (the most common attack against a PIM-for-Groups setup). Rule B fires on a *directory role*'s assignment (the attack against the role side directly — e.g., a compromised user with `Privileged Role Administrator` assigning Global Administrator to another account, bypassing the platform pipeline entirely).

**Rule C — Direct PIM policy changes (planned, policy-side companion).** The third rule, on the policy side. The operation lands in different log tables depending on which side wrote it:

- ARM side — `Microsoft.Authorization/roleManagementPolicies/write` and `roleManagementPolicyAssignments/write` in `AzureActivity`. These ARM operations *are* PIM (they're the only way to author a PIM policy on Azure resources), so the operation name alone is the PIM signal.
- Graph side — `AuditLogs` filtered by `LoggedByService == "PIM"`. The PIM service tags its own role-management-policy updates with this service name; filtering by service rather than operation-name string survives display-name renames over time.

A representative shape that joins both sources (the platform's KQL repo currently deploys Rules A and B — the policy-side Rule C below is on the to-do list):

```kql
let allowed_initiators = dynamic([
    "<pipeline-sp-app-id>"
]);
// ARM side — all PIM policy and policy-assignment writes on Azure resources.
let arm_pim_writes =
    AzureActivity
    | where OperationNameValue endswith "ROLEMANAGEMENTPOLICIES/WRITE"
        or OperationNameValue endswith "ROLEMANAGEMENTPOLICYASSIGNMENTS/WRITE"
    | extend init_id = tostring(parse_json(Authorization).evidence.principalId)
    | where init_id !in (allowed_initiators)
    | project TimeGenerated, Source = "ARM", OperationNameValue, init_id, ResourceId;
// Graph side — anything the PIM service logs (catches policy updates regardless
// of the activity display name Microsoft picks for it).
let graph_pim_writes =
    AuditLogs
    | where LoggedByService == "PIM"
    | where Result == "success"
    | where ActivityDisplayName has_any (
        "Update role setting",
        "Update role management policy",
        "Update rule",
        "Update policy"
      )
    | extend init_id = coalesce(
        tostring(InitiatedBy.app.appId),
        tostring(InitiatedBy.user.id)
      )
    | where init_id !in (allowed_initiators)
    | project TimeGenerated, Source = "Graph (PIM)",
              OperationNameValue = ActivityDisplayName,
              init_id, ResourceId = tostring(TargetResources[0].id);
union arm_pim_writes, graph_pim_writes
| sort by TimeGenerated desc
```

Any match alerts the platform on-call channel and creates an incident. The expected answer to any such alert is "someone clicked something in the portal that should have been a pull request" — the alert closes that gap.

### Drift detection — pipeline side, paired with Sentinel rules above

Pipeline drift detection and the Sentinel rules above watch the same gap from opposite directions. Pipeline drift detection asks **"does what's deployed match what's declared?"** at a point in time. The Sentinel rules ask **"was anything changed out-of-band?"** in the event stream. The two cover each other:

- Anything Rule A, Rule B, or Rule C catches as an out-of-band edit will also surface in the next Plan-stage `-WhatIf` run or the next daily Govern snapshot as a state delta — but the Sentinel rule fires *at the moment of the change*, which is the right cadence for incident response. The pipeline catches up later.
- Anything the pipeline catches as drift that did *not* produce a corresponding Sentinel signal is a higher-priority finding: it means a state change reached the tenant without an audit event, which warrants a deeper investigation than the routine "someone clicked something in the portal" answer.

To make the pairing operationally useful, the pipeline writes its drift findings to a Log Analytics custom table (`PIMPolicyDrift_CL`-style) on every run. A Sentinel scheduled query then reconciles each drift finding against the same workspace's `AzureActivity` / `AuditLogs` records inside the same window — drift with a matching out-of-band event closes as "expected, alert fired"; drift without a matching event opens as "no-event drift" and goes to the on-call channel.

Drift detection itself runs in two complementary places.

**Plan-stage `-WhatIf` runs (assess-only).** The platform's `graph-plan-apply.yml` stage template wraps every PIM-policy script in a two-stage Plan → Apply pattern. The Plan stage invokes the script with `-WhatIf` and posts the proposed diff to the pipeline run as a build comment; the Apply stage requires manual approval before running the script for real. A pipeline run that stops at Plan is a full assessment of *deployed state vs declared baseline* without changing anything in the tenant — the same script runs in both stages, only the `-WhatIf` flag differs. Running just the Plan stage on a schedule (or on every pull request that touches the baseline files) gives the platform a continuous declared-vs-deployed compare report without any apply risk.

**Daily Govern-stage snapshots.** The membership-track lifecycle pipeline runs a Govern stage every day that is *implicitly* assess-only: `Get-UserLifecycleSnapshot.ps1` enumerates every PIM-for-Groups membership across the platform, `Compare-UserLifecycleSnapshot.ps1` diffs the snapshot against the previous day's snapshot, and any difference the pipeline didn't author opens a work item. No apply step is involved at any point in this stage.

**Post-apply validation.** After an Apply run, the validation script (`Get-PIMPolicyRoleMapping.ps1`-style) fetches the current policy state from the tenant and diffs it against the declared baseline a second time. This catches the case where the apply itself produced an unexpected result — the script writes a policy, the next read returns something different, and the Plan-stage `-WhatIf` couldn't have predicted that mismatch.

One concrete trade-off the platform hit: Microsoft's Graph API lets you configure a `roleManagementPolicy`, but it's the `roleManagementPolicyAssignment` that binds a policy to a specific role. It is mechanically possible to update a policy and not notice that the assignment is still pointing at an older policy GUID — silent drift between intent and effect. The validation script catches this by checking that every role's `policyId` matches the policy whose contents were updated by the last pipeline run.

### Daily role-classification audit

Tying the fragile-classifier pitfall back to the reference-platform implementation: the role-classification heuristic (`\bReaders?\b` for Tier-2, opinionated 13-role list for Tier-0, fallthrough to Tier-1) is fragile in principle, so the platform runs a daily assessment job that enumerates the current `roleDefinitions` set from Microsoft Graph, computes the per-tier count, and compares both the count and the per-role classification against a baseline file in the same repository as the policy scripts. Any new Microsoft role (the fallthrough to Tier-1 catches it silently) or any rename of an existing role (the regex might re-classify it) shows as drift and creates a work item. The audit reduces "review the classification annually" to "review the drift work item the day it appears."

## Wrap-up

PIM is one system at the design layer and three management areas at the implementation layer. The thing that unifies them is the authentication context — a free-form string that any PIM rule can reference and any Conditional Access policy can target. Get the binding right and the three management areas share a coherent policy spine, an audit lens, and a governance story.

**A note on what you've just seen.** This post described the PIM policy spine — one slice of the reference platform that ships it. The reference-platform section above named the components that wire into PIM at the seam (CA policy set on the auth context, pipeline scripts in two tracks, Sentinel rules, drift reconciliation, an SP-exemption registry as code) — but it didn't show how the *rest* of the platform fits together: how PIM connects to Conditional Access Persona Flows, Global Secure Access, Defender XDR Unified RBAC, Sentinel content distribution, Terraform domains, the ADO bootstrap. You've seen the PIM slice; the whole picture is a post of its own. That's coming next: **[reference platform — the high-level scheme (forthcoming)](/posts/mssp-platform-architecture-overview/)** — every component and every dependency, in one Mermaid diagram. Read this one for the policy-spine deep-dive; follow for the scheme post to put it on the platform map.

The mechanics from this post also sit at the foundation of several other upcoming posts. **[I-2 (forthcoming)](/posts/jit-ado-via-pim-and-ca-auth-contexts/)** — JIT Azure DevOps access via PIM groups and CA auth contexts — uses the same shared step-up context defined here. **[I-4 (forthcoming)](/posts/auth-contexts-in-practice/)** is the auth-context deep-dive — the CA-side mechanics that this post deliberately stays out of. **[I-6 (forthcoming)](/posts/role-assignable-pim-groups-security-boundary/)** covers the `isAssignableToRole` security boundary that the role-assignable groups above are leaning on. **[I-3 (forthcoming)](/posts/rbac-for-tenant-isolation/)** sits on top of the Azure-resource side and looks at RBAC scoping in multi-customer tenants. Cross-links resolve as those posts publish.

If you're treating PIM as three management areas today and the audit question takes you three places to answer — the joining primitive is the auth context. Pick the IDs, write the scripts, retire the portal clicks.

## Get the scripts

The PowerShell that implements the pattern above and the Sentinel KQL that defends it are both in a public reference repo: **[github.com/vaculikgeorge/trustanchor-scripts](https://github.com/vaculikgeorge/trustanchor-scripts)**. PowerShell on top, detections on the bottom, same topic anchor on both sides.

**PowerShell — [`pim/`](https://github.com/vaculikgeorge/trustanchor-scripts/tree/main/pim):**

- **`New-PlatformPIMGroups.ps1`** — creates the platform PIM groups (`isAssignableToRole = true`) that Terraform consumes via `data.azuread_group`. Idempotent, `-WhatIf`-safe.
- **`Import-PIMRoleManagementPolicies.ps1`** — applies the tiered PIM policy (Tier-0 / Tier-1 / Tier-2) to *every* Entra directory role via Microsoft Graph. The auth-context binding (the internal-user PIM-activation context) is set here; the per-tier differences are activation duration and approval routing only.
- **`Import-PIMAzureResourcePolicies.ps1`** — applies per-role PIM policy to Azure resource roles via ARM. Walks the full MG-hierarchy descendants automatically — the non-cascading gotcha from the Pitfalls section above is what this script exists to patch.
- **`Get-PIMPolicyRoleMapping.ps1`** — dry-run validator. Loads the JSON template, classifies every directory role in the connected tenant against the Tier-0 / Tier-1 / Tier-2 model, and flags roles that have no PIM policy assignment.

**Sentinel KQL — [`kql/pim/`](https://github.com/vaculikgeorge/trustanchor-scripts/tree/main/kql/pim):**

- **`pim-group-membership-outside-pipeline.kql`** — Rule A above. Watchlist-anchored on `isAssignableToRole = true`; filters `LoggedByService != "PIM"`; coalesces app + user initiators.
- **`role-assigned-outside-pipeline.kql`** — Rule B above. Direct directory-role assignment outside the pipeline.
- **`tier0-role-activated.kql`** — informational fire on every Tier-0 activation. Pairs with the same 13-role classification used in the PowerShell.

All seven files are **anonymised reference copies**, not turnkey deployables. Placeholder IDs (`00000000-...`), placeholder email (`admin@contoso.com`), placeholder service-principal token (`<pipeline-sp-app-id>`), the example tier-classification lists, and the management-group names need to be reviewed and replaced for your tenant. Every PowerShell script has a `CONFIGURE BEFORE USE` block at the top; every KQL file has a `CONFIGURE BEFORE USE` block in the header comment. The KQL also requires a `RoleAssignableGroups` watchlist refreshed daily from a Microsoft Graph query — `kql/pim/README.md` gives the source query. Always run the PowerShell with `-WhatIf` first.

The repo is [MIT-licensed](https://github.com/vaculikgeorge/trustanchor-scripts/blob/main/LICENSE). Copy, adapt, ship in your own work. Issues and discussion welcome on GitHub.

## References

- Microsoft Learn — [Securing privileged access](https://learn.microsoft.com/en-us/security/privileged-access-workgroup/)
- Microsoft Learn — [Privileged Identity Management overview](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)
- Microsoft Learn — [Microsoft Entra highly privileged roles](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/privileged-roles-permissions)
- Microsoft Learn — [`roleManagementPolicies` (Microsoft Graph)](https://learn.microsoft.com/graph/api/policyroot-list-rolemanagementpolicies)
- Microsoft Learn — [`unifiedRoleManagementPolicyAuthenticationContextRule`](https://learn.microsoft.com/graph/api/resources/unifiedrolemanagementpolicyauthenticationcontextrule)
- Microsoft Learn — [`Microsoft.Authorization/roleManagementPolicies` (ARM)](https://learn.microsoft.com/azure/templates/microsoft.authorization/rolemanagementpolicies)
- Microsoft Learn — [PIM for Groups API — `eligibilityScheduleRequests`](https://learn.microsoft.com/graph/api/privilegedaccessgroup-post-eligibilityschedulerequests)
- Microsoft Learn — [Management Groups descendants API](https://learn.microsoft.com/en-us/rest/api/managementgroups/management-groups/list-descendants)
- Microsoft Learn — [Azure built-in roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)
- Thomas Naunheim — [EntraOps](https://github.com/Cloud-Architekt/EntraOps) (operational counterpart to the design patterns in this post)
