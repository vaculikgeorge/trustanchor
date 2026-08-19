+++
title = 'Secret-less auth for Azure DevOps runners with WIF'
date = '2026-07-25T10:00:00+02:00'
tags = ['azure-devops', 'managed-identity', 'aci', 'devsecops', 'wif', 'service-connection', 'workload-identity-federation', 'microsoft-hosted-agents']
categories = ['Secure Delivery']
showShareBottom = false
liVideos = [
  { id = "7492864792047632384", cap = "The walkthrough" },
]
summary = 'A build pipeline is one of the most privileged identities in your tenant — the service principals and managed identities behind it form a perimeter into the platform. Workload Identity Federation makes the pipeline auth to Azure, Microsoft Graph, other API endpoints, and Azure DevOps secret-less on any runner; a user-assigned managed identity does the same for registering a self-hosted runner. This post covers both, and why PATs, client secrets, and certificates are the credentials to design out.'
+++

> A pipeline reaches further than most of your admins — it writes Azure, Microsoft Entra, and
> Azure DevOps, and calls a dozen other API endpoints besides. Hand it a long-lived credential and
> you have copied that reach into a string that can be read from a config file, echoed into a log,
> or lifted off a disk — and it outlives whoever created it. A PAT expires and takes the pipeline
> down at midnight; a client secret ages out a year later; a certificate rots in a store. The most
> defensible credential on a runner is the one that was never issued.

## TL;DR

A build pipeline is one of the most privileged identities in your tenant: it writes infrastructure,
identity, and policy across Azure, Microsoft Entra, Microsoft Graph and other API endpoints, and
Azure DevOps itself. The service principals and managed identities behind it are a **perimeter into
the platform** — as real as your human admins, and far more likely to be left holding a long-lived
secret in a config store where it can be read or leaked. **Workload Identity Federation (WIF) is how
you secure that perimeter.** WIF is an OpenID Connect trust: instead of holding a password, secret, or
certificate, the identity trusts a *federation* — at run time the pipeline presents a short-lived OIDC
token, Entra checks it against the configured trust and returns an access token. The credential is
minted per run, never stored, and expires on its own, so there is nothing at rest to steal or rotate.

A pipeline authenticates in two places, and WIF (with a managed identity) covers both with no stored
secret:

- **When a job runs** — reaching Azure, Microsoft Graph, other API endpoints, and Azure DevOps's own
  resources (repositories, artifact feeds, the REST API). A **WIF service connection** handles this
  the same way on a Microsoft-hosted or self-hosted runner, because the pipeline *task* presents the
  token — not the machine.
- **When a self-hosted runner joins its pool** — an **add-on** that applies only if you run your own
  runners. Registering the runner is a separate, one-time step; a **user-assigned managed identity**
  does it with a short-lived Entra token, so no PAT is written to the host.

The enemy in both places is the long-lived credential — the PAT, the client secret, the certificate.
The rest of this post is how WIF removes it.

<style>
.ta-diagram { max-width: 100%; margin: 1.75rem auto; overflow-x: auto; }
.ta-diagram img { display: block; box-sizing: border-box; height: auto; max-width: 100%; max-height: 82vh; margin: 0 auto; cursor: zoom-in; }
.ta-diagram .ta-diagram-dark { display: none; }
.dark .ta-diagram .ta-diagram-light { display: none; }
.dark .ta-diagram .ta-diagram-dark { display: block; }
</style>

<figure class="ta-diagram">
  <img class="ta-diagram-light nozoom" src="/img/diagrams/s1-overview-light.svg?v=9" alt="The whole picture: two runner pools — Microsoft-hosted, and self-hosted (ACI + UAMI, no PAT) — both feed one pipeline task; the task hands off to a WIF service connection that presents an OIDC token (issuer, subject, audience) to Entra ID, which validates the federated trust and issues a short-lived access token that reaches Microsoft Graph (Entra, Intune, M365), Azure Resource Manager, other Azure APIs (Key Vault, Storage), and Azure DevOps — with least-privilege grants and no stored secret." loading="lazy" decoding="async" />
  <img class="ta-diagram-dark nozoom" src="/img/diagrams/s1-overview-dark.svg?v=9" alt="The whole picture: two runner pools — Microsoft-hosted, and self-hosted (ACI + UAMI, no PAT) — both feed one pipeline task; the task hands off to a WIF service connection that presents an OIDC token (issuer, subject, audience) to Entra ID, which validates the federated trust and issues a short-lived access token that reaches Microsoft Graph (Entra, Intune, M365), Azure Resource Manager, other Azure APIs (Key Vault, Storage), and Azure DevOps — with least-privilege grants and no stored secret." loading="lazy" decoding="async" />
</figure>

## The shared part: WIF service connections

This is the surface that matters on every pipeline, whatever it runs on — and the one the post's
title is about.

**The pipeline to Microsoft's APIs.** The identity behind a **Workload Identity Federation** service
connection is a normal Entra principal, so it reaches *any* Microsoft API that accepts an Entra
token — **Microsoft Graph** (Entra, Intune, and much of the Microsoft 365 surface), **Azure Resource
Manager**, and the Azure **data-plane** APIs beside it (**Key Vault**, **Azure Storage** — each its
own token audience, not ARM's), or a custom API. Azure Resource Manager is one endpoint among many. The connection (generally available, and the recommended
credential type) trusts a **federated credential** on either an **app registration** or a
**user-assigned managed identity** instead of a stored secret or certificate: at run time the pipeline
task gets an OpenID Connect (OIDC) token, presents it to Entra ID at the audience
`api://AzureADTokenExchange`, and Entra — having checked that the token's issuer, subject, and
audience match the configured federated identity credential exactly — returns a short-lived access
token for whichever API the task asked for. Newly created connections use an **Entra-issued** ID token
(issuer `login.microsoftonline.com/<tenant-id>/v2.0`, subject
`<entra-prefix>/sc/<org-id>/<service-connection-id>`). One federated identity fans out to every
endpoint; none needs a stored secret, and nothing is kept in Azure DevOps.

**The pipeline to Azure DevOps's own resources.** Reaching Azure is only half of what a pipeline
authenticates to. Checking out a second repository, restoring from an artifact feed, or calling the
Azure DevOps REST API has historically meant a PAT. The **Microsoft Entra workload-identity service
connection** removes that: a service principal or managed identity reaches Azure DevOps resources
through the same federated exchange, PAT-free, with per-pipeline permissions instead of the shared
build-service-account identity — and every attempt lands in the Azure DevOps audit log.

**Why the runner does not matter.** In both cases the federated exchange is a property of the
**service connection and the task**, not of the machine. The task asks for the OIDC token, the task
presents it, the task receives the access token. Move the job from a Microsoft-hosted pool to a
self-hosted one and nothing about what it can reach changes — the deployment identity rides the
service connection, not the compute.

## The two surfaces

There is really **one identity plus one add-on**. The deployment identity — the WIF service
connection above — is the same on every runner. Self-hosting adds a single extra, one-time step:
registering the runner into its pool. Keep those two apart and the rest of the post falls into place:

| Surface | What authenticates | Which runners | Secret-less mechanism |
|---|---|---|---|
| **Deployment identity** — pipeline to Azure / Graph / other APIs / Azure DevOps | the pipeline **task** (not the runner) | Microsoft-hosted **and** self-hosted | WIF service connections (OIDC) — an ARM connection for Azure/Graph/other APIs, an Entra workload-identity connection for Azure DevOps; nothing stored |
| **Registration** *(a self-hosted add-on)* — a runner to its pool | the runner, once, at `config.sh` time | self-hosted **only** | UAMI Entra token in place of a PAT |

<figure class="ta-diagram">
  <img class="ta-diagram-light nozoom" src="/img/diagrams/s1-two-surfaces-light.svg?v=2" alt="Both runner pools converge on one pipeline task: a Microsoft-hosted runner (the default) feeds it directly, and a self-hosted ACI runner reaches it only after a registration add-on — the ACI runner's user-assigned managed identity registers it into the pool with no PAT. From the pipeline task onward the deployment identity is identical; the full deployment path is in the overview above." loading="lazy" decoding="async" />
  <img class="ta-diagram-dark nozoom" src="/img/diagrams/s1-two-surfaces-dark.svg?v=2" alt="Both runner pools converge on one pipeline task: a Microsoft-hosted runner (the default) feeds it directly, and a self-hosted ACI runner reaches it only after a registration add-on — the ACI runner's user-assigned managed identity registers it into the pool with no PAT. From the pipeline task onward the deployment identity is identical; the full deployment path is in the overview above." loading="lazy" decoding="async" />
</figure>

## Microsoft-hosted runners

Microsoft-hosted runners are the default, and the **majority of pipelines should stay here**: a
public, patched image, a fast cold start, and no runner infrastructure to own. Because you do not
control the host, the **only** secret-less lever you have is the service connection from the shared
part above — and that is enough, because there is **no registration surface**. Microsoft registers
and manages the runners; you never run `config.sh`, so no PAT, secret, or managed identity is
involved in attaching a runner to a pool. Nothing sits on disk for you to rotate.

What you trade for that simplicity is capacity, not identity. For a private project the free grant is
one parallel job, capped at **60 minutes a run and 1,800 minutes (≈30 hours) a month**; past that,
concurrency is bought per parallel job — one concurrent slot each, roughly **$40/month** at the time
of writing (rates change; see the [Azure DevOps pricing page](https://azure.microsoft.com/pricing/details/devops/azure-devops-services/)). A paid parallel job also
removes the monthly-minute ceiling and raises the per-run cap from 60 to **360 minutes**. When that
ceiling or a confidentiality requirement bites, you reach for a self-hosted pool — and inherit the
registration add-on.

## Self-hosted runners: ACI and a managed identity

Self-hosting is the **exception, justified on a few triggers**. First, **network reach** — the job
has to reach resources behind **private endpoints** or inside a VNet (Key Vault, storage, a private
API) that a Microsoft-hosted runner on the public internet cannot. Second, **confidentiality or
sovereignty** — work that must run on infrastructure you control rather than Microsoft's shared
image. Third, **cost at volume** — a self-hosted parallel job has no minute cap (one is free, extras
**~$15/month**), and on **Azure Container Instances you pay only the underlying per-second compute**,
so once a busy platform runs past the 1,800-minute managed ceiling a self-hosted pool is both
unbounded and cheaper than stacking $40 hosted slots.

Everything in the shared part still applies unchanged: the jobs authenticate to Azure and Azure
DevOps through the same WIF service connections. Self-hosting adds exactly one thing — the
**registration** add-on: the runner has to attach itself to the pool before it can run anything.

**None of the built-in registration methods is both secret-less and unattended.** As of runner
version 3.227.1, `config.sh` accepts several authentication types, used *only* at registration and
never persisted:

| Registration method | `--auth` value | Secret-less? | Unattended? |
|---|---|:---:|:---:|
| Personal access token | `pat` (`--token` = a PAT string) | no | yes |
| Service principal + client secret | `SP` (`--clientID` / `--tenantId` / `--clientSecret`) | no (a client secret) | yes |
| Entra device-code sign-in | `AAD` | no — interactive **user** sign-in | no — a human every time |
| **UAMI Entra token** | `pat` (`--token` = a short-lived Entra / OAuth 2.0 token) | **yes** | **yes** |

A few things the table hides. The **service-principal registration path takes a client secret** — the
`config.sh` prompts are Client ID, Tenant ID, and client *secret*, with no certificate option. The app
registration can of course hold a certificate for its *other* work — authenticating to Azure or
Graph — but it cannot present that certificate to *register the runner*, so even the "first-class"
service-principal path hands a stored secret to the host. `AAD` is Entra **device-code flow**: a
person finishes the sign-in in a browser
at `microsoft.com/devicelogin` *every* time a runner registers — it is a user credential, not a
machine one, and cannot drive an unattended boot. That leaves the last row. `config.sh --auth pat`
accepts any bearer token for the Azure DevOps resource — the docs note it takes an OAuth 2.0 token as
well as a PAT — so an **Entra access token** issued for the public Azure DevOps application ID
(`499b84ac-1321-427f-aa17-267ca6975798`) works, and a **user-assigned managed identity** on the host
can mint that token with no stored secret. This is a *different surface with a different lever* from
the WIF service connection above: the service-connection exchange is a property of the task and the
connection, which exist only when a pipeline runs — there is no task at host-registration time, so
the registration add-on needs its own secret-less answer, and the UAMI is it.

### The pattern

A user-assigned managed identity holds the durable trust relationship with Azure DevOps. The runner —
an Azure Container Instance — attaches the UAMI at runtime. On startup the bootstrap script:

1. fetches an Entra access token for the Azure DevOps resource via the UAMI (through IMDS);
2. passes it to `config.sh --auth pat --token $TOKEN`;
3. starts the runner under a non-root user.

Nothing is written to disk, nothing needs rotating, and the token's one-hour lifetime is irrelevant
after registration — Azure DevOps issues the runner its own listener OAuth token once it attaches.

<figure class="ta-diagram">
  <img class="ta-diagram-light nozoom" src="/img/diagrams/s1-registration-flow-light.svg?v=1" alt="Self-hosted registration flow: a user-assigned managed identity attached to an Azure Container Instance requests an access token for the Azure DevOps resource from Entra ID via IMDS; the container's bootstrap passes that token to config.sh --auth pat, which attaches the runner to the pool; the UAMI's only standing grant is Pool Administrator on that pool." loading="lazy" decoding="async" />
  <img class="ta-diagram-dark nozoom" src="/img/diagrams/s1-registration-flow-dark.svg?v=1" alt="Self-hosted registration flow: a user-assigned managed identity attached to an Azure Container Instance requests an access token for the Azure DevOps resource from Entra ID via IMDS; the container's bootstrap passes that token to config.sh --auth pat, which attaches the runner to the pool; the UAMI's only standing grant is Pool Administrator on that pool." loading="lazy" decoding="async" />
</figure>

The UAMI needs one Azure DevOps permission, set once: **Pool Administrator** on the target pool. No
subscription-level RBAC is required for the runner itself.

### Walkthrough

Synthetic identifiers throughout; adjust to your tenant:

- Azure DevOps organisation: `dev.azure.com/contoso`
- Agent pool name: `self-hosted-aci`
- UAMI name: `id-ado-agent-demo-weu-001`
- Resource group: `rg-ado-agents-weu-01`

#### 1. Create the user-assigned managed identity

```powershell
# PowerShell 7+ with Azure CLI 2.x
$rg   = 'rg-ado-agents-weu-01'
$loc  = 'westeurope'
$uami = 'id-ado-agent-demo-weu-001'

az group create --name $rg --location $loc
az identity create --resource-group $rg --name $uami --location $loc

$uamiId       = az identity show -g $rg -n $uami --query id       -o tsv
$uamiClientId = az identity show -g $rg -n $uami --query clientId -o tsv
```

#### 2. Add the UAMI to Azure DevOps and grant Pool Administrator

Azure DevOps does not know a managed identity exists until something with the UAMI's object ID is
added. The cleanest path: add the UAMI as a Service Principal user in the organisation (Azure DevOps
refers to managed identities as "Service Principals" in the UI), then grant Pool Administrator on the
target pool.

```powershell
# PowerShell 7+ with Azure CLI 2.x
$uamiPrincipalId = az identity show -g $rg -n $uami --query principalId -o tsv
# Use this object ID in Azure DevOps:
#   Organization settings -> Users -> Add user -> Add a service principal
#   Paste $uamiPrincipalId, set access level to Basic
#
#   Organization settings -> Agent pools -> self-hosted-aci -> Security
#   Add the service principal with role: Administrator
```

The CLI alternative uses the Azure DevOps REST API; for a single pool the UI path is faster and just
as auditable (it lands in the Azure DevOps audit log).

#### 3. Bootstrap script

The container runs `mcr.microsoft.com/azure-cli:latest`, which is CBL-Mariner under the hood — not
Alpine, not Ubuntu. The package manager is `tdnf`. This trips up every blog post that assumes `apk`
or `apt-get`.

```bash
#!/usr/bin/env bash
# Runs as root inside the ACI container.
# Required env vars: ADO_ORG_URL, ADO_POOL, UAMI_CLIENT_ID, AGENT_VERSION
set -euo pipefail

tdnf install -y shadow-utils sudo tar gzip ca-certificates which icu libstdc++

useradd -m -s /bin/bash azp

az login --identity --client-id "${UAMI_CLIENT_ID}" --allow-no-subscriptions

# 499b84ac-1321-427f-aa17-267ca6975798 is the public Azure DevOps resource ID.
ENTRA_TOKEN=$(az account get-access-token \
  --resource 499b84ac-1321-427f-aa17-267ca6975798 \
  --query accessToken -o tsv)

mkdir -p /home/azp/agent
cd /home/azp/agent

# The legacy agentpackage CDN is unreachable from inside ACI in some regions.
# This URL is the supported alternative.
curl -fsSL -o agent.tar.gz \
  "https://download.agent.dev.azure.com/agent/${AGENT_VERSION}/vsts-agent-linux-x64-${AGENT_VERSION}.tar.gz"

tar -xzf agent.tar.gz
chown -R azp:azp /home/azp/agent

sudo -u azp ./config.sh \
  --unattended \
  --url "${ADO_ORG_URL}" \
  --auth pat \
  --token "${ENTRA_TOKEN}" \
  --pool "${ADO_POOL}" \
  --agent "$(hostname)" \
  --acceptTeeEula \
  --replace

exec sudo -u azp ./run.sh
```

Two things worth calling out:

- The `--allow-no-subscriptions` flag matters. A bare UAMI with no Azure RBAC roles fails
  `az login --identity` without it, because the CLI tries to enumerate subscriptions by default.
- `--client-id` replaces the deprecated `--username` flag. Old samples still use `--username`; they
  break on current Azure CLI.

#### 4. Deploy the ACI

Inline command syntax with PowerShell escaping is painful for a multi-line bootstrap. Use an ARM
template (or Bicep) and pass the script as a parameter. The shape:

```powershell
# PowerShell 7+ with Azure CLI 2.x
$adoOrgUrl    = 'https://dev.azure.com/contoso'
$adoPool      = 'self-hosted-aci'
$agentVersion = '4.271.0'

# Build the command as a single base64-encoded payload for cleanliness.
$scriptPath = '.\aci-agent-bootstrap.sh'
$scriptB64  = [Convert]::ToBase64String([IO.File]::ReadAllBytes($scriptPath))

az deployment group create `
  --resource-group $rg `
  --template-file .\aci-agent.json `
  --parameters `
    aciName='aci-ado-agent-001' `
    uamiResourceId=$uamiId `
    uamiClientId=$uamiClientId `
    adoOrgUrl=$adoOrgUrl `
    adoPool=$adoPool `
    agentVersion=$agentVersion `
    bootstrapScriptB64=$scriptB64
```

The ARM template attaches `uamiResourceId` to the container, sets `restartPolicy` to `Never` (so a
failed run stays put rather than silently restarting), and runs:

```bash
/bin/bash -c "echo $BOOTSTRAP_B64 | base64 -d > /tmp/start.sh && bash /tmp/start.sh"
```

Leave it on `Never` while you stabilise — with the log-capture trick under gotchas, a crashed
container keeps its output. Move to `OnFailure` / `Always` only once you want the runner to self-heal
after a crash; in practice many self-hosted setups just stay on `Never`.

#### 5. Verify

```powershell
# PowerShell 7+ with Azure CLI 2.x
az container logs -g $rg -n aci-ado-agent-001 --follow
```

Look for the `Listening for Jobs` line. In the Azure DevOps UI the pool's Agents tab should show one
online runner named after the container's hostname. Run a one-line pipeline against the pool to
confirm:

```yaml
# azure-pipelines.yml
pool: self-hosted-aci
steps:
  - pwsh: Write-Host "Hello from $(hostname) — no PAT involved."
```

### Pitfalls and gotchas

The architecture is obvious once written down. These eat hours one at a time.

- **`vstsagentpackage.azureedge.net` is unreachable from ACI in several regions.** The DNS resolves,
  the connection hangs. Use `https://download.agent.dev.azure.com/agent/...` instead — same package,
  supported URL.
- **The runner refuses to run as root.** Newer (3.x) runners check and exit if `whoami` returns root.
  Hence `useradd -m azp` and the `sudo -u azp` calls. Skipping this is the most common reason "it
  worked on my VM" does not translate to ACI.
- **`az login --identity --username` is deprecated.** It half-works on older CLI and silently does the
  wrong thing on new ones. Always use `--client-id`.
- **A UAMI without subscription roles needs `--allow-no-subscriptions`.** The bootstrap UAMI typically
  holds zero Azure RBAC — its scope is Pool Administrator on an Azure DevOps pool, outside ARM — so
  the default subscription enumeration fails and the CLI exits.
- **The azure-cli image is CBL-Mariner, not Alpine.** `apk add` and `apt-get install` both fail — the
  package manager is `tdnf`. And because the agent is a .NET application, install **`icu`** (and
  `libstdc++`) or it exits at startup with *"Couldn't find a valid ICU package installed"* —
  `tdnf install -y shadow-utils sudo icu libstdc++`.
- **Pool Administrator must be granted before the first registration attempt.** Without it,
  `config.sh` fails with a generic `TF400813` error that never mentions permissions — easy to
  misdiagnose as a token problem.
- **A failed container drops its logs — hold it open to read them.** With `restartPolicy: Never` a
  crashed bootstrap exits and ACI soon returns `None` for the logs; with `OnFailure` it just restarts
  and the failed run's output is gone anyway. Keep the container alive on failure instead — append
  `|| { echo FAILED; sleep 3600; exit 1; }` to the risky steps — so `az container logs` still works.
  This is why many self-hosted runners sit on `Never` with a hold rather than a self-healing policy.
- **Azure DevOps accepts the Entra token only at the configuration step.** Once registered, the runner
  manages its own session lifetime; you never refresh the Entra token.

### When not to use this

A poor fit for:

- **High-throughput pools.** ACI suits predictable, low-to-medium concurrency. For dozens of
  concurrent runners, look at AKS-hosted agents or VM scale sets — the auth pattern transfers, the
  compute substrate does not.
- **Jobs that need on-host secrets.** This changes how the *runner registers*, not how the *pipeline*
  gets secrets. In-job auth should ride the WIF service connection above; if it does not yet, address
  that first.
- **Strict workload isolation on shared infrastructure.** ACI containers share more network namespace
  than some frameworks prefer. Deploy one ACI per isolation boundary rather than multiplexing.

## The dangers: PATs, secrets, and certificates

These are not just a runner-registration problem. A PAT, a client secret, or a certificate is a
**key to the whole project and its pipelines** — a PAT with **Code (read/write)** scope can push a
commit that triggers CI, a service-connection secret is usable by any pipeline granted the connection,
and either can be used from anywhere the holder takes it. Registration is only the most visible place
they show up. Set the four credential types side by side:

| Credential | Lifetime | At rest | Blast radius | Potential flaw |
|---|---|---|---|---|
| **PAT** | ≤ 1 year (expiry not always a hard stop) | a reusable bearer string, stored wherever it is used | everything the issuing **user** can reach across the org | a leaked token works from any network Conditional Access does not IP-fence; dies silently when the user offboards |
| **Client secret** | up to 2 years | stored in Azure DevOps / config; usable by tasks on the connection | everything the **service principal** is granted | any task on the connection can use or exfiltrate it; rotation is manual and easy to forget |
| **Certificate** | renewal cycle; expiry not always enforced | private key in a store or file; still a bearer credential | everything the **service principal** is granted | private-key theft; and an expired certificate can still authenticate where the relying party does not check `NotAfter` |
| **WIF** | ~one job, minutes | **nothing persisted** | everything the **service principal** is granted | none at rest to steal — but the RBAC scope is unchanged, so least-privilege on the identity still matters |

Two honest points from that last row. **WIF does not shrink the blast radius** — the federated
identity holds exactly the RBAC an equivalent service principal would, so scoping the grant tightly is
still your job. What WIF removes is the *stored credential*: there is nothing on disk or in Azure
DevOps to leak, expire, or rotate. And **expiry is not the backstop people treat it as**: a token
minted from a credential outlives the moment of issue, and some relying parties keep accepting an
expired certificate — so "it will expire eventually" is not a control.

**A PAT also sidesteps most of your Conditional Access posture.** Conditional Access does *not* stop
PATs. For interactive, web sign-in Azure DevOps enforces the full range of Conditional Access
policies, but for **non-interactive flows such as a PAT the only Conditional Access control that
applies is IP-fencing** — and only if an administrator has enabled the *IP Conditional Access policy
validation on non-interactive flows* organization policy. Device-compliance and
authentication-strength policies never evaluate on a PAT call, so a PAT minted on a compliant device
keeps working from a non-compliant one. If you want to actually stop PATs, the lever is an **Azure
DevOps policy**, not Conditional Access: the organisation-level *Restrict personal access token
creation* policy, and the tenant-level policies on the Microsoft Entra tab (restrict global PATs,
restrict full-scoped PATs, enforce a maximum lifespan, auto-revoke leaked tokens).

**Client secrets can be partly contained — but the controls protect different planes, and one plane
stays open.**

- **Management plane — closed only if the operator closes it.** This protection is opt-in, never
  automatic. Stored the right way — a service-connection secret, or a variable explicitly marked
  *secret* — Azure DevOps will not display the value again and it cannot be read back or exported,
  which stops an admin or a stolen console session from *retrieving* it. Stored any other way — a
  plain variable, a value committed in a YAML file, an unmarked field — none of that applies. So the
  plane is shut only when someone deliberately shut it, and wide open by default.
- **Runtime plane — a job that uses the connection can still get the credential.** This is the plane
  the write-only property does *not* cover, and the one people miss. A task that legitimately runs on
  the connection can pull the service principal's key into its own process — the Azure CLI task's
  `addSpnToEnvironment` option does exactly that, by design. So a malicious or compromised task in the
  pipeline can exfiltrate the secret even though no human can read it in the UI. This is the risk
  Workload Identity Federation removes: with WIF there is no key to hand out — the task gets a
  short-lived federated token instead.
- **Conditional Access for workload identities** can block the service principal from outside a
  **named location**, but it needs a **Workload Identities Premium** licence, applies to
  **single-tenant service principals only** (managed identities are exempt entirely), and is
  **block-only** on location or Entra ID Protection risk. A named-location rule fits a fixed-egress
  self-hosted pool, but it **cannot allow-list Microsoft-hosted runners** — their egress is
  Microsoft's own broad, shifting IP space — so the moment a job runs Microsoft-hosted, the location
  control is off the table.

{{< warning >}}
**One control that does *not* count here:** pipeline authorization, approvals, and branch checks
apply to a WIF connection exactly as much as a secret-based one, so they harden the pipeline — not
the secret.
{{< /warning >}}

**WIF removes the category.** There is no persistent secret to leak, expire, or rotate; the token is
short-lived, minted per exchange, and issued by Entra with its full protection around the ID token.
The gating on these principals then comes from RBAC scope, the federated-credential trust, and Azure
DevOps approvals — not from a network rule that cannot see a Microsoft-hosted runner.

**WIF is not magic — four things it does not fix.** It is the right default, but treat these as
standing warnings, not footnotes:

- **The live token is still exfiltratable in-job.** WIF removes the *stored* secret, not the
  short-lived token it mints. A task in the job can read that token — the Azure CLI task's
  `addSpnToEnvironment` option puts the credential in the environment on purpose — and use it for its
  roughly one-hour life. WIF shrinks the window; it does not stop a malicious task inside it.
- **A run is a token.** The federated trust means anyone who can make the pipeline *run* — a pull
  request that triggers CI, a tampered YAML file — can obtain a token as the identity. Guard the
  triggers (fork-PR settings, required reviewers, protected branches) and **authorize the service
  connection per pipeline**, never "grant access to all pipelines".
- **The trust is only as tight as the subject.** Entra matches the federated credential's issuer,
  subject, and audience **exactly and case-sensitively** — a sloppy or over-broad subject widens who
  can mint a token as the identity.
- **A managed-identity-backed connection is invisible to Conditional Access and ID Protection.**
  Managed identities are out of scope for both, so you cannot gate or risk-flag them there; the only
  controls left are RBAC scope and the federation configuration.

## How this plays out in the reference platform

Here's how the pattern tends to land in practice — generalized. In the
platform's map, the runner is
one node — [the reference platform's Azure DevOps runners](/posts/reference-platform-architecture-overview/#node-runners) —
and the full token exchange is the pipeline lane in [the access flow](/posts/reference-platform-architecture-overview/#access-flow).
What that map assumes, this post is the deep-dive for.

**The constraint that forces it.** The platform disallows both long-lived options org-wide — but
through the *correct* levers: an **Azure DevOps organisation/tenant policy** blocks PAT creation, and
an **Entra application-management policy** blocks client-secret creation and caps certificate lifetime
on app registrations (secrets it can forbid outright; certificates it can lifetime-limit or pin to a
trusted issuer). There is no
"but we need a build runner" exception, and there should not be — the exception you carve once is the
one you forget to expire. With PATs and secrets off the table, the only non-interactive principal the
policy stack still admits is a managed identity, which is exactly what the registration add-on above
leans on.

**One ACI per slice owner.** Per-engagement workload isolation rules out multiplexing runners across
slice owners. The model is one ACI deployment per slice-owner subscription, with that slice owner's own
UAMI scoped to a slice-owner-specific pool — same bootstrap, different parameters — so the blast radius
of a compromised runner is bounded to one tenant.

**Pool Administrator as the only durable grant.** The UAMI's only standing Azure DevOps right is Pool
Administrator on its pool. The pipelines that *run* on that pool authenticate via the WIF service
connection for everything they touch. The runner's registration identity and the pipeline's runtime
identity are deliberately distinct — the same split this whole post is built on.

**Audit trail.** The audited surface is the whole delivery path. Azure DevOps records the
security-relevant actions of every run — the **service connection being executed**, the **checks and
approvals** on protected resources, deployments, resource authorization, agent-pool changes and
registration — and **audit streaming** forwards them to a Log Analytics workspace (the
`AzureDevOpsAuditing` table). The runtime side is just as huntable: **Entra diagnostic settings** route
service-principal and managed-identity sign-ins to the same workspace — `AADServicePrincipalSignInLogs`
and `AADManagedIdentitySignInLogs`, each carrying a `FederatedCredentialId` column, so a WIF sign-in
filters out in one line — every Microsoft Graph call lands in `MicrosoftGraphActivityLogs`, and ARM
control-plane writes in `AzureActivity`. The value is the **correlation** across them: those sources
share join keys — the identity's object ID across the audit and sign-in logs, and `SignInActivityId` ↔
`UniqueTokenIdentifier` between `MicrosoftGraphActivityLogs` and the sign-in tables — so one KQL join
reconstructs a per-run, per-identity timeline: which run acted, through which pipeline and approvals, as
which federated identity, and every Graph and ARM call that followed. Detections then fire on the
**joined pattern**: any run or action that departs from the declared, authorized path emerges as a
shape once the sources are joined.

## Wrap-up

One identity plus one add-on, one principle: no long-lived credential lives on the runner or in the
project. The pipeline's auth to Azure, Microsoft Graph, other API endpoints, and Azure DevOps rides a
WIF service connection that presents a short-lived, subject-scoped token from the *task* — identical on
a Microsoft-hosted or self-hosted runner. A self-hosted runner's *registration* is a separate add-on
WIF does not touch, closed with a user-assigned managed identity that registers the runner with a
short-lived Entra token — no PAT on the host. WIF does not shrink what the identity can reach, so
least-privilege still matters; what it removes is the credential at rest — no PAT to expire, no secret
to rotate, no certificate to renew, and no issuing user to offboard. The credential you never issued is
the one that never leaks.

This post secured the pipeline's *machine* perimeter. The human one — who may approve a run,
JIT-elevated and never able to clear their own change — is its companion:
**[JIT ADO access via PIM groups and CA auth contexts](/posts/jit-ado-via-pim-and-ca-auth-contexts/)**.

{{< share >}}

**References:**

- [Microsoft Learn — Connect to Azure with an ARM service connection (workload identity federation)](https://learn.microsoft.com/azure/devops/pipelines/library/connect-to-azure)
- [Microsoft Learn — Access Azure DevOps with Microsoft Entra workload identity](https://learn.microsoft.com/azure/devops/pipelines/library/add-devops-entra-service-connection)
- [Microsoft Learn — Self-hosted agent authentication options](https://learn.microsoft.com/azure/devops/pipelines/agents/agent-authentication-options)
- [Microsoft Learn — Register an agent using a service principal](https://learn.microsoft.com/azure/devops/pipelines/agents/service-principal-agent-registration)
- [Microsoft Learn — Register an agent using device code flow](https://learn.microsoft.com/azure/devops/pipelines/agents/device-code-flow-agent-registration)
- [Microsoft Learn — Manage PATs with policies (for administrators)](https://learn.microsoft.com/azure/devops/organizations/accounts/manage-pats-with-policies-for-administrators)
- [Microsoft Learn — Stream Azure DevOps audit events (Azure Monitor Logs)](https://learn.microsoft.com/azure/devops/organizations/audit/auditing-streaming)
- [Microsoft Learn — Microsoft Entra diagnostic settings & log categories](https://learn.microsoft.com/entra/identity/monitoring-health/concept-diagnostic-settings-logs-options)
- [Microsoft Learn — Set up Conditional Access policies on Azure DevOps](https://learn.microsoft.com/azure/devops/organizations/accounts/conditional-access-policies)
- [Microsoft Learn — Conditional Access for workload identities](https://learn.microsoft.com/entra/identity/conditional-access/workload-identity)
- [Microsoft Learn — Configure and pay for parallel jobs](https://learn.microsoft.com/azure/devops/pipelines/licensing/concurrent-jobs) · [Azure DevOps pricing](https://azure.microsoft.com/pricing/details/devops/azure-devops-services/)
- [Microsoft Learn — Self-hosted Linux agents](https://learn.microsoft.com/azure/devops/pipelines/agents/linux-agent)
- [Microsoft Learn — User-assigned managed identities](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities)
- [Azure DevOps resource ID `499b84ac-...`](https://learn.microsoft.com/azure/devops/integrate/get-started/authentication/service-principal-managed-identity)
