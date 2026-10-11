+++
title = 'Just-in-time admin access to Azure DevOps: the group holds the permission, never the human'
date = '2026-10-11T14:30:00+02:00'
draft = false
tags = ['azure-devops', 'pim', 'conditional-access', 'entra-id', 'just-in-time', 'privileged-access', 'authentication-contexts', 'identity-governance']
categories = ['Identity & Privileged Access']
summary = 'Azure DevOps organizations tend to carry permanent administrators, because the usual fix, adding people to Project Collection Administrators behind a Conditional Access policy, time-boxes nothing. This post builds the alternative: the permission sits permanently on an Entra group, the human holds only eligible membership, and an authentication context lets a Conditional Access policy gate the activation itself. The design runs on Microsoft Entra tokens with personal access tokens turned off, ends access through a block policy when the membership ends, and names what it does not prevent: the organization owner and direct assignments.'
+++

> An Azure DevOps organization with no permanent administrators still has to be administered. The way
> out is that nothing on the Azure DevOps side ever changes: the only thing that moves is one group
> membership.

## TL;DR

Just-in-time access to Azure DevOps rests on one inversion: the permission is assigned permanently to
a group, and the human only ever holds *eligible membership* in it. The target-side assignment never
changes, so there is no drift to reconcile and no orphaned direct grant to find later. What makes the
pattern a control rather than only a time limit is the **authentication context**, the link that lets
a Conditional Access policy gate the activation itself, not just the sign-in that follows it.

Azure DevOps reads Entra group membership rather than owning the grant, and three consequences follow:
an access level is licensing rather than permission and must not be made just-in-time; the
organization owner can never be a group, which is the one slot the pattern cannot cover; and
membership takes up to an hour to reach Azure DevOps unless the user refreshes. The design runs on
Microsoft Entra tokens alone, with personal access tokens turned off. The reference-platform section
at the end shows how code delivers the arrangement and corrects its drift every day.

## Three terms this post leans on

The pattern below depends on three Privileged Identity Management primitives, and Azure DevOps treats
them differently.

| Term | What it means here |
|---|---|
| **Eligible assignment** | The principal is a *candidate* to hold the access. It holds nothing until it activates. This is the standing state in the design below, and it grants no permission at all. |
| **Active assignment** | The principal holds the access right now. It can get there by activating, or by an administrator assigning it directly, and that second path skips the activation policy. |
| **Activation** | The move from eligible to active, for a bounded window. It is a distinct event with its own policy, its own audit record, and its own opportunity to demand something of the person doing it. |

What makes activation a control is the **authentication context**: a label that both Privileged
Identity Management and Conditional Access can reference. Privileged Identity Management names it on
the activation rule; a Conditional Access policy targets the same label and decides what the person
must present. The context is the only link between the two services.

What that context resolves to is a Conditional Access **authentication strength**, a named set of
acceptable credentials. I refer to strengths by what they accept rather than by name, because the
names are per-tenant: *hardware-attested* accepts only physical security keys whose attestation
identifier is on an allow-list, while *phishing-resistant* is the broader category that also admits
platform credentials such as Windows Hello for Business and Platform Credential for macOS, and passkeys
in Microsoft Authenticator or synced passkeys
([phishing-resistant methods](https://learn.microsoft.com/entra/identity/authentication/overview-authentication#phishing-resistant-authentication-methods)).
Those get their own treatment in
[FIDO2 AAGUID attestation allow-listing](/posts/fido2-aaguid-attestation-whitelisting/), and the
contexts get theirs in [Authentication contexts in practice](/posts/auth-contexts-in-practice/).

<style>
.ta-diagram { max-width: 100%; margin: 1.75rem auto; overflow-x: auto; }
.ta-diagram img { display: block; box-sizing: border-box; height: auto; max-width: 100%; max-height: 82vh; margin: 0 auto; cursor: zoom-in; }
.ta-diagram .ta-diagram-dark { display: none; }
.dark .ta-diagram .ta-diagram-light { display: none; }
.dark .ta-diagram .ta-diagram-dark { display: block; }
</style>

<figure class="ta-diagram">
  <img class="ta-diagram-light nozoom" src="/img/diagrams/i2-activation-path-light.svg?v=1" alt="Azure DevOps just-in-time administration in two lanes. On the transient side an eligible admin activates a PIM group membership, firing an authentication context that a Conditional Access policy uses to demand step-up on a hardware-attested credential, granting membership for a bounded window. On the permanent side that Entra group is a standing member of the Azure DevOps security group, so permission is inherited, never granted to the human; a second policy blocks the application for everyone outside those groups." loading="lazy" decoding="async" />
  <img class="ta-diagram-dark nozoom"  src="/img/diagrams/i2-activation-path-dark.svg?v=1"  alt="Azure DevOps just-in-time administration in two lanes. On the transient side an eligible admin activates a PIM group membership, firing an authentication context that a Conditional Access policy uses to demand step-up on a hardware-attested credential, granting membership for a bounded window. On the permanent side that Entra group is a standing member of the Azure DevOps security group, so permission is inherited, never granted to the human; a second policy blocks the application for everyone outside those groups." loading="lazy" decoding="async" />
</figure>

## Permanent admins, and why the obvious fix is not one

An Azure DevOps organization is a management endpoint. It holds the pipelines that write your tenant,
the service connections those pipelines authenticate with, and the branch policies that decide what
ships. The people who administer it are administrators of the change channel itself.

Most organizations govern that with two moves: put the admins in **Project Collection
Administrators**, and write a Conditional Access policy so they have to sign in from a managed device
with a strong credential. Both are worth doing. Neither one time-boxes anything. The membership is
permanent, so the only thing Conditional Access gates is *how* a standing administrator authenticates,
not *whether* they are one. Revoking access means remembering to remove a person from a group.

A common answer is "we already have Conditional Access on Azure DevOps". Conditional Access decides
what a principal must present in order to authenticate. It has no opinion about what that principal is
*allowed to do* once it is in, because authorization in Azure DevOps lives in security-group
membership, not in the sign-in. A standing Project Collection Administrator behind the strictest policy
in your tenant is still a standing Project Collection Administrator.

The second answer is to remove people and hand access out on request. Access granted on request has
no expiry, so removing it depends on someone remembering to.

What both answers miss is the property this design delivers: **the absence of an activation means the
absence of the permission**. The rest of this post is how, and
[what this does not fix](#what-this-does-not-fix) lists the four limits that remain.

## What Microsoft's guidance covers, and where it stops

Microsoft documents this pattern, and the documentation is correct.

[Manage access with Microsoft Entra groups](https://learn.microsoft.com/azure/devops/organizations/accounts/manage-azure-active-directory-groups?view=azure-devops)
carries a section called *Configure just-in-time access for admin groups*. Its setup has two steps:
create a role-assignable group in Entra ID, then add that group to the Azure DevOps group. Its usage
has three: activate the access, refresh your permissions in Azure DevOps, take the action. It adds two
notes: a user with elevated access must **also retain standard access** to the
organization so they can load the pages and refresh at all, and users keep elevated access "for up to
1 hour after their PIM group access gets deactivated".

That is an accurate description of the mechanism. It leaves out three things the design depends on:

- **It says nothing about Conditional Access, or about authentication contexts.** Activation is
  presented as something the user simply does. Whether activating should require anything of them,
  such as a specific credential or device, is outside its scope.
- **It does not separate an access level from a permission.** Azure DevOps has three distinct systems
  here, and conflating the licensing one with the authorization one produces a design that either
  fails to work or fails to protect. More on that below.
- **Its just-in-time section never mentions the organization owner.** The page's prerequisites note
  that owners are automatically Project Collection Administrators, and the owner is a single user
  account, which no group can hold. It is the one place the pattern structurally cannot reach, so a
  design that does not name it has a permanent administrator it has not accounted for.

Its first setup step stays: the groups must be created *role-assignable*, even though no Entra role
is assigned here. By default, only Privileged Role Administrators can change the membership of a
role-assignable group, and group owners only if you add them
([how role-assignable groups are protected](https://learn.microsoft.com/entra/identity/role-based-access-control/groups-concept#how-are-role-assignable-groups-protected)).
That narrows who can write the membership this design depends on.

## Two layers, and only one of them moves

The design has two layers, and they move at different speeds.

**The permanent layer is the grant itself.** An Entra group is added to the Azure DevOps security
group that carries the permission: Project Collection Administrators at organization scope, Project
Administrators at project scope. That assignment is made once and never touched again. Nothing about
it is time-boxed, nothing about it changes, and there is no reconciliation to do, because from Azure
DevOps's point of view the answer to "who administers this organization?" is a single group and it
never changes.

**The transient layer is one membership.** Humans are *eligible* members of that Entra group and
nothing more. When someone needs to administer, they activate; for a bounded window they are an active
member; then they are not. The permission was never granted to them and never revoked from them. The
only thing that changed is whether they were in a group.

**The reader group grants read access, project by project.** Beside the administrator group sits a
reader group. Both are eligible rather than active, both activate through the same authentication
context, and both are excluded from the resource policy described below. The administrator group is a
member of Project Collection Administrators. The reader group is a member of each project's
[Readers group](https://learn.microsoft.com/azure/devops/organizations/security/permissions?view=azure-devops#project-level-groups),
because Azure DevOps has no Readers group at organization level. Reading a project therefore never
needs the administrator group.

There is no per-person permission to audit in Azure DevOps, no direct assignment that might have been
left behind, and no drift between what the organization's permissions say and what was intended. The
target side is static by construction, and the governance sits in the eligibility list and the
activation policy.

### The authentication context in the PIM policy

In Microsoft Graph, PIM settings are rules on the group's policy, and the authentication context sits
in `AuthenticationContext_EndUser_Assignment`, whose value is the context identifier
([rules mapping](https://learn.microsoft.com/graph/identity-governance-pim-rules-overview)).

**It applies to end-user activations only.** The rule for an administrator's direct active assignment
is `Enablement_Admin_Assignment`, which can require multifactor authentication and a justification on
that assignment, and has no authentication context. An administrator who assigns active membership
directly therefore skips the step-up, which is why the direct-assignment path matters as much as the
activation path.

**MFA on activation is not an authentication strength.** The activation-side setting *On activation,
require multifactor authentication* lives in a different rule, `Enablement_EndUser_Assignment`, and
users might not be prompted by it if they
[provided multifactor authentication earlier in the session](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/groups-role-settings#on-activation-require-multifactor-authentication).
The same page recommends the authentication context together with authentication strengths when a
credential has to be presented at activation. In the portal both are options of one setting,
*On activation, require*
([rules mapping](https://learn.microsoft.com/graph/identity-governance-pim-rules-overview)), and only
the context reaches a strength.

**Nothing checks that a policy targets the context.** The identifier comes from a fixed set (`c1`,
`c2` and so on), and PIM lets you bind one that no policy targets. For that case it has a backup:
activation then requires multifactor authentication, as though *On activation, require multifactor
authentication* were set. The backup
[does not fire](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/groups-role-settings#on-activation-require-microsoft-entra-conditional-access-authentication-context)
when a policy exists but is turned off, is in report-only mode, or excludes the eligible users, so a
policy left in report-only leaves activation with no step-up from the context.

### Two Conditional Access policies, doing two different jobs {#two-policies}

There are two policies, and they gate different events:

| | What it targets | What it does | Carries the context? |
|---|---|---|---|
| **The activation policy** | the authentication context | Demands step-up at the moment of activation: a hardware-attested credential, on a device that meets your bar | **Yes.** This is the link to PIM |
| **The resource policy** | the Azure DevOps application (`499b84ac-1321-427f-aa17-267ca6975798`) | Blocks the application for all users, excluding the just-in-time groups and break-glass | **No.** It knows nothing about activation |

The resource policy is written as a **block with exclusions** rather than as a conditional allow. All users are blocked from the Azure DevOps
application; the just-in-time groups are excluded from that block. Activation therefore does not
"satisfy a policy"; it moves the principal into the exclusion set, and the block stops applying to
them.

Block-with-exclusions is the shape to use. A conditional allow grants access to everyone who meets
the conditions, so a new principal that happens to meet them is admitted by default. A block with an
explicit exclusion list admits nobody unless they are named. It costs you an exclusion list to
maintain, and that list must be matched by **object id, not by display name**, because a name is
squattable: an attacker who can create a group can create one that matches your exclusion by name.
Resolve the names to object ids once, when you write the policy, and the name never appears in it.

External identities are out of scope for this design and are covered by their own set of Conditional
Access policies.

Put together: the activation policy decides *whether you may become an administrator*, the resource
policy decides *whether you may reach Azure DevOps at all*, and one group membership is the only thing
connecting them.

**The two policies apply at different times.** The activation policy applies once, when the person
activates. The resource policy is evaluated every time a client requests a token for Azure DevOps, so
once the membership ends, the next request meets the block.

### Where Azure DevOps is not Azure

[PIM at scale](/posts/pim-at-scale-three-domains/) covers this same two-layer idea across Entra
directory roles, Azure resources and groups. Azure DevOps is a different kind of consumer: it does not
own the grant, it *reads* Entra group membership and maps it onto its own permission model. Four
consequences follow.

**Azure DevOps has three separate systems, and only one of them is authorization.** An *access level*
(Stakeholder, Basic, Basic + Test Plans) decides which portal features a user may use, which makes it
licensing. *Security-group membership* is what carries permissions. *Permissions* themselves are the
Allow/Deny entries inherited from those groups. An access level is not a permission:
[to access project resources](https://learn.microsoft.com/azure/devops/organizations/security/about-permissions?view=azure-devops#valid-user-groups),
a user also needs the permissions those resources require.

**So do not drive the access level from the just-in-time group.** It fails twice, and both failures
are documented. Group-rule re-evaluation happens
[on demand, when a rule changes, or automatically every 24 hours](https://learn.microsoft.com/azure/devops/organizations/accounts/assign-access-levels-by-group-membership?view=azure-devops)
(Azure DevOps refreshes Entra group membership hourly, and Entra itself may take up to 24 hours for
dynamic groups), which is too slow to track an activation measured in hours. And when a user leaves
the group, "if the group was the user's sole source of access, Azure DevOps automatically removes them
from the organization", so deactivating would remove the person from the organization rather than
de-elevate them. That is why Microsoft's guidance says to retain standard access.
{#access-level}

The working arrangement is therefore: **every administrator stays a standing organization member with a
standing access level**, which grants no permission by itself, and *every permission* is reached
through activation. Zero standing permission is achievable. Zero standing membership is not, because
leaving the only group that grants access removes a user from the organization.

That direct access-level assignment lives in Azure DevOps and does not get a user past Conditional
Access: the Azure DevOps application is reachable only with an activated group membership, because
the resource policy blocks everyone outside the just-in-time groups.

**The organization owner can never be a group.** The owner is a single user account, automatically a
member of Project Collection Administrators, and the slot accepts no group, so there is no activation
to gate. Give the slot to a dedicated account that does no daily work and is excluded from no policy.
Any Project Collection Administrator can
[change the owner](https://learn.microsoft.com/azure/devops/organizations/accounts/change-organization-ownership?view=azure-devops#prerequisites),
so an activated administrator can move ownership whenever it is needed, and the owner account does not
have to be usable day to day. The new owner needs an active access level; Stakeholder is not enough.
{#organization-owner}

**Membership changes can take up to an hour to reach Azure DevOps.** Entra changes
[may take up to an hour to become visible in Azure DevOps](https://learn.microsoft.com/azure/devops/organizations/accounts/manage-azure-active-directory-groups?view=azure-devops#add-a-microsoft-entra-group-to-an-azure-devops-group),
and the user can re-evaluate immediately instead: after activating, sign out and back in, or use
[refresh permissions](https://learn.microsoft.com/azure/devops/organizations/security/request-changes-permissions?view=azure-devops#refresh-or-reevaluate-your-permissions).

### Walkthrough

Eight steps in the Entra admin center and the Azure DevOps organization settings. The order matters in
two places, and those steps say why.

#### 1. Create the authentication context

Under Conditional Access, create an
[authentication context](https://learn.microsoft.com/entra/identity/conditional-access/concept-conditional-access-cloud-apps#configure-authentication-contexts)
whose name says what it gates, and select *Publish to apps* so that policies can target it. Its ID
comes from a fixed set (`c1`, `c2` and so on), and the ID, not the name, is what PIM binds in step 6.

#### 2. Create the activation policy, and enable it before step 6

Create a Conditional Access policy that targets the authentication context and is scoped to all users
or to the eligible users, never to the just-in-time groups: during activation the person is not a
member yet, so a policy scoped to the group
[does not apply](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/groups-role-settings#on-activation-require-microsoft-entra-conditional-access-authentication-context).
Grant access with the authentication strength that activation should demand, and under session
controls set sign-in frequency to *Every time*, so that users
[reauthenticate each time they activate, even with an active session](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/groups-role-settings#on-activation-require-microsoft-entra-conditional-access-authentication-context).
A [known issue](https://learn.microsoft.com/entra/identity/authentication/concept-authentication-strengths#known-issues)
applies: a strength and a sign-in frequency can be satisfied at two different times, for example the
strength by an earlier passkey sign-in and the frequency by a Windows Hello unlock. After a
reauthentication, a 10-minute window applies
in which further activations do not prompt again. Enable the policy before step 6, because PIM's backup
covers only a context that no policy targets, not one whose policy is off or in report-only.

#### 3. Create the resource policy, in report-only

Create a second policy for the Azure DevOps application (`499b84ac-1321-427f-aa17-267ca6975798`) that
blocks all users and excludes the just-in-time groups, by object id, and two break-glass accounts.
Leave it in report-only until the sign-in logs show the principals you expect and nobody else; step 8
enforces it.

#### 4. Create the two groups as role-assignable

Create the administrator group and the reader group as security groups with *Microsoft Entra roles can
be assigned to the group* set to Yes, one pair per scope. No Entra role is assigned to them. The setting
is there because it limits who can change their membership to Privileged Role Administrators, and to
owners only if you add them. It can be set only when the group is created and is
[immutable](https://learn.microsoft.com/entra/identity/role-based-access-control/groups-concept#restrictions-for-role-assignable-groups),
and a tenant can hold at most 500 such groups.

#### 5. Map both groups into Azure DevOps

Add the administrator group to Project Collection Administrators at organization scope, or to Project
Administrators per project, and the reader group to each project's Readers group, since Azure DevOps
has no Readers group at organization level. Organization-level groups sit under Organization settings
→ Permissions and project-level groups under Project settings → Permissions; in both, open the group
and add the Entra group under Members
([add Entra groups to built-in groups](https://learn.microsoft.com/azure/devops/organizations/security/add-ad-aad-built-in-security-groups?view=azure-devops)).
This is the permanent layer, and it needs Project Collection Administrator, at least Basic access, and
an organization connected to Entra ID. The same memberships can be written through the Azure DevOps
Graph REST API
([Memberships - Add](https://learn.microsoft.com/rest/api/azure/devops/graph/memberships/add?view=azure-devops-rest-7.1)),
which works with Azure DevOps descriptors rather than Entra object ids. The first time you add a group
in the portal you have to type its name; after that, browsing finds it.

#### 6. Bind the context in PIM, for members and owners

In Privileged Identity Management, open each group's settings for the Member role and set
[*On activation, require Microsoft Entra Conditional Access authentication context*](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/groups-role-settings#on-activation-require-microsoft-entra-conditional-access-authentication-context)
to the context from step 1. Then apply the same settings to the Owner role. Owners can add members
directly without activating anything, and the settings for owners are
[independent of those for members](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/groups-role-settings#overview),
so govern both sides even while the Owner side is empty, as
[the member versus owner trap](/posts/pim-at-scale-three-domains/#member-vs-owner-trap) describes.

#### 7. Make the administrators eligible, permanently

Assign each administrator as an eligible *member* of the groups, not an owner, with no end date. The
eligibility is permanent and the activation is what expires. An owner could add active members
directly, which is why the groups have none.

#### 8. Move the owner, then enforce

Move organization ownership to the [dedicated account](#organization-owner) from
[Organization settings → Overview](https://learn.microsoft.com/azure/devops/organizations/accounts/change-organization-ownership?view=azure-devops).
Changing the owner
[doesn't remove the previous owner](https://learn.microsoft.com/azure/devops/organizations/accounts/change-organization-ownership?view=azure-devops#remove-the-previous-owner-from-project-collection-administrators)
from Project Collection Administrators, so remove that account from the group. Before you switch the
resource policy on, check its
[report-only results](https://learn.microsoft.com/entra/identity/conditional-access/concept-conditional-access-report-only#reviewing-results)
in the sign-in logs or the policy impact view: sign-ins by activated members of the just-in-time
groups and by the break-glass accounts show *Report-only: Not applied*, and every other sign-in to
Azure DevOps shows *Report-only: Failure*. Then switch the resource policy from report-only to on, and
verify in this order: activate, confirm
that the prompt demands the credential you specified, refresh permissions in Azure DevOps, take an
administrative action, deactivate, and confirm that the action is refused again. If no prompt
appeared, check that the step 2 policy is on rather than in report-only.

### Pitfalls and gotchas {#pitfalls}

**Personal access tokens sit outside most of Conditional Access, so this design turns them off.**
Azure DevOps
[enforces Conditional Access on interactive (web) flows](https://learn.microsoft.com/azure/devops/organizations/accounts/conditional-access-policies?view=azure-devops#conditional-access-behavior-on-web)
and enforces MFA policies on web flows only. On a personal access token the documented check is IP
fencing, and only after a Project Collection Administrator enables the *IP Conditional Access policy
validation on non-interactive flows* organization policy. The authentication strength that activation
demands is never asked of a token. A Microsoft Entra token is the opposite case: it comes from a
sign-in, Conditional Access
[is enforced on that sign-in](https://learn.microsoft.com/entra/identity/conditional-access/overview)
once the first factor completes, and the sign-in is recorded in the sign-in logs. So this design runs on Entra
tokens alone, and having no personal access tokens is one of its conditions.

Two levels of policy do that
([PAT policies](https://learn.microsoft.com/azure/devops/organizations/accounts/manage-pats-with-policies-for-administrators?view=azure-devops)).
At organization level, a Project Collection Administrator turns on *Restrict personal access token
(PAT) creation*, which exists only in Entra-backed organizations and also blocks global tokens in that
organization. At tenant level, an Azure DevOps Administrator can restrict global and full-scoped tokens
and cap their lifespan. Both apply to new tokens only, so revoke the existing ones; the audit log
records them as
[`Token.PatCreateEvent` and `Token.PatRevokeEvent`](https://learn.microsoft.com/azure/devops/organizations/audit/auditing-events?view=azure-devops#token-events).
Pipelines and automation authenticate as workload identities instead. People authenticate their git
operations with their own Microsoft Entra identity, through
[Git Credential Manager](https://learn.microsoft.com/azure/devops/repos/git/set-up-credential-managers?view=azure-devops)
set to request Entra tokens rather than personal access tokens
(`credential.azreposCredentialType oauth`). In my experience git is the main reason people create
personal access tokens, so this removes most of the demand for them.

**Deactivation takes effect through the block.** PIM removes the membership
[within seconds](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/groups-activate-roles#overview),
and from then on every token request for Azure DevOps meets the resource policy's block. Azure DevOps
documents that elevated access can last
[up to an hour after deactivation](https://learn.microsoft.com/azure/devops/organizations/accounts/manage-azure-active-directory-groups?view=azure-devops#use-access);
in my own test, with the block in place, an administrative action was refused about a minute after I
deactivated. If access has to end at once, revoke the user's sessions as well
([revoke user access](https://learn.microsoft.com/entra/identity/users/users-revoke-access)).

**A direct assignment skips the step-up, so it is detected rather than prevented.** An active
assignment made in PIM follows `Enablement_Admin_Assignment`, which has no authentication context, and
a member added through the Groups experience
[does not pass through PIM at all](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/groups-role-settings#overview).
Two facts keep that path narrow. Membership of a role-assignable group is managed by Privileged Role
Administrators, and by owners only if you add them, so the just-in-time groups get no owners. And
both kinds of change are visible: the assignment schedule instance records `assignmentType` as
`assigned` rather than `activated` (next entry), and the Entra audit log records every
[Add member to group and Add owner to group](https://learn.microsoft.com/entra/identity/monitoring-health/reference-audit-activities#core-directory)
operation. Alert on both.

**The step-up governs the activation, not everything after it.** Once membership is activated, users
[aren't prevented from using another browsing session, device, or location](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/groups-role-settings#on-activation-require-microsoft-entra-conditional-access-authentication-context)
to use it, so a compliant device demanded at activation is not demanded afterwards. Microsoft's remedy
is a Conditional Access policy scoped at the eligible users directly, for example one that requires a
compliant device whenever they sign in.

**"Active" is three different states, and the membership list shows you one.** Eligible assignments are
invisible exactly where you would look for them: the PIM for Groups reference states that when a
principal is made eligible, "the members and owners relationships of the group aren't updated". Active
assignments do populate those relationships, and that is deliberate, because populating them is how the
permission reaches Azure DevOps at all. The catch is that three quite different things land in the same
list: a permanent active assignment with no start or end date, a time-bound active assignment that
expires on its own, and a real just-in-time activation, where the person moved themselves from
eligible to active for a bounded window. Read the group's members and all three look identical.

What separates them is `assignmentType` on the
[assignment schedule instance](https://learn.microsoft.com/graph/api/resources/privilegedaccessgroupassignmentscheduleinstance?view=graph-rest-1.0),
either `assigned` or `activated`, alongside an `activatedUsing` relationship that points back at the
eligibility an activation came from and is null otherwise. A permanent active member reads as
`assigned` with no end date: standing access inside a PIM-managed group, which is what this design
removes.

A tool that reads group membership cannot tell a working activation from a permanent grant, so audit
against the schedule instances rather than the member list.

**A service principal in the group holds the permission and does not appear in the member list.**
[Microsoft states it outright](https://learn.microsoft.com/azure/devops/organizations/accounts/manage-azure-active-directory-groups?view=azure-devops):
"Due to a functional limitation on Microsoft Graph, service principals don't appear in any list of
Microsoft Entra group members on Azure DevOps. Permissions set on any Microsoft Entra groups still
apply to any service principals in the group that were added to the organizations, even if they aren't
displaying on the web UI." So a workload identity that was added to the organization and sits in a
group mapped to Project Collection Administrators has organization-admin rights that the Azure DevOps
member list does not show. Govern and audit the group's members in Entra; the Azure
DevOps member list is not an inventory. Separately, group rules for licensing
[do not apply to service principals or managed identities](https://learn.microsoft.com/azure/devops/organizations/accounts/assign-access-levels-by-group-membership?view=azure-devops),
so their access levels must be assigned directly.

**Project-scoped visibility is a user-experience guardrail, not a boundary.** Limiting user visibility
and collaboration to specific projects applies to the web portal only. REST APIs and the `az devops`
CLI can still read data the portal hides. Permissions still enforce on those paths (Deny is Deny
everywhere), but if your isolation story depends on people not *seeing* other projects, it does not
survive a script.

**If you inherited this tenant, look for an Azure Resource Manager exclusion that no longer has a
reason to exist.** Before September 2025 Azure DevOps sign-in depended on the ARM audience, so
administrators routinely excluded Azure DevOps users from ARM Conditional Access policies to keep
sign-in working. That dependency is gone and a policy targeting ARM no longer covers Azure DevOps
sign-ins at all, so the exclusion protects nothing while exempting real people from real policies.
Remove it, keeping only the groups that still need ARM, which are billing administrators and
service-connection creators.

### The design assumes you removed the standing access, and usually you have not

Applying the pattern to an organization that already exists is a different job, because just-in-time
membership governs only the membership you made just-in-time. Any other route to the same authority is
untouched by it. The resource policy does not close these routes; it holds them back until the person
activates. The policy targets the Azure DevOps application, and Entra validates the tenant's policies
at sign-in to every organization the tenant backs
([Conditional Access behavior on web](https://learn.microsoft.com/azure/devops/organizations/accounts/conditional-access-policies?view=azure-devops#conditional-access-behavior-on-web)).
Activating any just-in-time group, even the reader group, therefore lifts the block in all of them,
and every standing right the person still holds works again for that window, in other projects and
other organizations too.

There are six such routes, and five of them are memberships:

| # | Route | Can a group hold it? |
|---|---|---|
| 1 | A direct assignment on the organization's Users page, carrying an access level | Only indirectly, via a group rule, which [this design does not use](#access-level) |
| 2 | The organization owner | **No.** See [the owner](#organization-owner) |
| 3 | Membership of Project Collection Administrators | Yes |
| 4 | Membership of Project Administrators, per project | Yes |
| 5 | Membership of a team (teams nest inside the project's Contributors group) | Yes, mapped as a project group |
| 6 | **Approval authority** on a pipeline environment, a service connection, a classic release gate, or as a required reviewer in a branch policy | Yes, see [below](#approval-authority) |

Routes 3, 4 and 5 share one remedy: replace direct human members with one eligible group per scope,
and never add a human directly again. Route 1 stays, because an access level carries no permission.
Route 2, the owner, cannot be removed, only given to a dedicated account. Route 6 is different in kind.

#### The sixth route: approval authority {#approval-authority}

**Approval authority is the right to let a run through a gate.** An approver on a pipeline environment
or a service connection, on a classic release gate, or a required reviewer in a branch policy decides
what ships. That authority attaches to the resource rather than to a security group, so PIM does not
sit in its path and the membership cleanup above does not touch it. These guardrails cover it:

- Put an eligible group, never a named person, in every approver slot;
  [one member's approval is enough](https://learn.microsoft.com/azure/devops/pipelines/process/approvals?view=azure-devops#approvals).
  The documentation does not say when the group's membership is read, so test that a member who
  activates after a run has paused can still approve.
- Change the approver list before a run starts. The list of users who can review an approval
  [is fixed](https://learn.microsoft.com/azure/devops/pipelines/process/approvals?view=azure-devops#approvals)
  when approvals and checks start running, so edits do not reach runs already waiting.
- Govern the right to edit the check like an administrator group. Members of the resource's
  [User, Creator or Administrator role](https://learn.microsoft.com/azure/devops/pipelines/security/resources?view=azure-devops#permissions),
  Project Administrators and Project Collection Administrators can change or delete the check, so
  those roles belong to eligible groups as well.
- Keep the approval timeout well above the activation lag. Entra membership can take up to an hour to
  reach Azure DevOps, and an approval that times out
  [marks the stage skipped](https://learn.microsoft.com/azure/devops/pipelines/process/approvals?view=azure-devops#approvals),
  together with the stages that
  [depend on it](https://learn.microsoft.com/azure/devops/server/release-notes/azuredevops2022u2#retry-a-stage-when-approvals-and-checks-time-out).
- Give approvers a way to see a waiting approval other than email. For classic releases, Azure DevOps
  [doesn't expand Azure Active Directory groups when delivering notifications](https://learn.microsoft.com/azure/devops/pipelines/release/approvals/approvals?view=azure-devops#predeployment-approvals),
  and the YAML documentation says nothing either way.

### What this does not fix {#what-this-does-not-fix}

The pattern removes standing administrator permission. Four limits remain, each covered above.

**The owner is a permanent administrator you choose rather than eliminate.** It is a single user
account, and no group can hold the slot; see [the organization owner](#organization-owner).

**Native Azure DevOps groups cannot become eligible.** Some of them sit inside the administrator groups
by design, the service-accounts group for one, and they have no Entra object for PIM to manage. The
accurate claim is no standing *human* administrators, plus a named list of what cannot move.

**A direct assignment is detected, not prevented.** It skips the step-up; see the
[pitfalls](#pitfalls).

**The step-up proves the credential once, at activation.** Only a policy scoped at the eligible users
keeps demanding it afterwards; see the [pitfalls](#pitfalls).

### When not to use this

Two of these are hard blockers and the rest are judgement.

**The organization must be backed by Microsoft Entra ID.** The whole pattern rides on Azure DevOps
reading Entra group membership. An organization backed by Microsoft accounts has no such path, and
nothing here applies to it.

**Every eligible member needs a licence.** Privileged Identity Management for Groups requires
Microsoft Entra ID P2 or Microsoft Entra ID Governance for every user who is eligible for membership
or ownership of a managed group. This is per-person and it is not optional, so the cost scales with
the number of administrators rather than with the number of organizations, which usually makes it
cheap, but check before designing around it.

**If you cannot issue phishing-resistant credentials, activation is a weaker control.** The step-up is
only as strong as the credential it demands. A code from an authenticator app can be relayed in real
time by a proxy between the user and the sign-in page, so an attacker can complete the activation too.
The pattern still gives time-boxing and a clean permission model at any credential strength; assurance
about *who* activated comes only with phishing-resistant credentials.

**If a team cannot work without personal access tokens, this design does not cover that work.** The
permission model still improves, but on a token Conditional Access reaches only IP fencing, as the
[pitfall above](#pitfalls) describes.

**Below a certain size the owner problem dominates.** In a very small organization the standing owner
may be most of the administrative authority there is, and a placeholder-owner account is bureaucracy
without much return. The pattern earns its keep once "who can administer this, and since when?" stops
being answerable from memory.

## How this plays out in the reference platform {#reference-platform}

*This worked example is distilled from real multi-org delivery in regulated Microsoft cloud
environments, generalized here rather than any single organization's deployment.*

The reference platform runs this pattern across several Azure DevOps organizations. The groups were
created once. Their mapping into Azure DevOps and their PIM settings are delivered by code,
which checks them every day and corrects any drift. An assessment also runs over each organization
every day and asserts that:

- the resource policy is enabled rather than report-only, targets the Azure DevOps application, and
  excludes exactly the just-in-time groups and the declared break-glass accounts, matched by object id;
- the administrator group is mapped into Project Collection Administrators, and no human holds a
  direct administrator membership at organization or project scope;
- the organization owner is a dedicated placeholder account, kept disabled.

The reader group is a member of each project's Readers group. Personal access token creation is
restricted at organization level with an empty allowlist, and the tenant policies block global and
full-scoped tokens. Neither setting can be read back through an API, so both are watched through their
effects: the token events in the audit log and the tokens themselves. Pipelines authenticate with workload identity federation, so no secret sits in the
delivery path ([Secret-less auth for Azure DevOps runners with WIF](/posts/secretless-ado-runners-wif/)).
External identities are not part of this design and are governed by their own Conditional Access
policies.

Drift is a finding with a decision attached: fold the change into the declaration, revert it, or
record it as a governed exception with a date. The assessment is one part of a wider governance
pipeline for the Azure DevOps control plane, which works from a declared baseline and brings the
organizations back in line with it rather than only reporting on them. That pipeline is covered in a
separate article, [Governing the Azure DevOps control plane as code](/posts/ado-control-plane-as-code/).

Where this sits in the wider platform is the
[Privileged Identity Management node](/posts/reference-platform-architecture-overview/#node-pim) of the
[reference platform map](/posts/reference-platform-architecture-overview/), alongside the
[Azure DevOps node](/posts/reference-platform-architecture-overview/#node-ado) that this post's
permanent layer plugs into.

## Wrap-up

The pattern is small once stated: assign the permission permanently to a group, keep humans eligible
rather than active, and bind the activation to an authentication context so a Conditional Access
policy can demand something real at the moment it happens. The target side never changes, which is
what removes the drift and the orphaned assignments. Two Conditional Access policies do two different
jobs: one gates the activation, the other gates Azure DevOps itself.

Four facts are not in the setup guide. An access level is
licensing rather than permission, and driving it from the just-in-time group will eject your
administrators from the organization instead of de-elevating them. The owner cannot be a group and
never will be. The design runs on Microsoft Entra tokens alone, with personal access tokens turned off,
because Conditional Access reaches a personal access token only through IP fencing. And deactivation needs no clean-up step: PIM ends the membership within seconds, and the block refuses
the next token request.

If you check one thing tomorrow, check the direct-assignment path. Every control in this post gates
the activation, and a direct assignment skips it, so look for active assignments of type `assigned`
and for any owner on the just-in-time groups.

**Coming next in this area:** [Enforce CA on PIM role activation](/posts/enforce-ca-on-pim-activation/)
covers the reauthentication on every PIM activation that became generally available in April 2026,
which is what step 2's *Every time* setting configures, and is the natural follow-on for anyone
implementing the activation policy above.

**References:**
[Manage access with Microsoft Entra groups](https://learn.microsoft.com/azure/devops/organizations/accounts/manage-azure-active-directory-groups?view=azure-devops) ·
[Set up Conditional Access policies on Azure DevOps](https://learn.microsoft.com/azure/devops/organizations/accounts/conditional-access-policies?view=azure-devops) ·
[Manage personal access tokens with policies](https://learn.microsoft.com/azure/devops/organizations/accounts/manage-pats-with-policies-for-administrators?view=azure-devops) ·
[Configure PIM for Groups settings](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/groups-role-settings) ·
[Role-assignable groups](https://learn.microsoft.com/entra/identity/role-based-access-control/groups-concept) ·
[Assign access levels with group rules](https://learn.microsoft.com/azure/devops/organizations/accounts/assign-access-levels-by-group-membership?view=azure-devops) ·
[About permissions and security groups](https://learn.microsoft.com/azure/devops/organizations/security/about-permissions?view=azure-devops) ·
[About access levels](https://learn.microsoft.com/azure/devops/organizations/security/access-levels?view=azure-devops) ·
[Change the organization owner](https://learn.microsoft.com/azure/devops/organizations/accounts/change-organization-ownership?view=azure-devops) ·
[PIM for Groups](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/concept-pim-for-groups) ·
[Refresh or reevaluate your permissions](https://learn.microsoft.com/azure/devops/organizations/security/request-changes-permissions?view=azure-devops#refresh-or-reevaluate-your-permissions) ·
[About security, authentication, and authorization](https://learn.microsoft.com/azure/devops/organizations/security/about-security-identity?view=azure-devops)
