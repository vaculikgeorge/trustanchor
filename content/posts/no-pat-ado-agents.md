+++
title = 'No-PAT Azure DevOps agents on ACI with user-assigned managed identity'
date = '2026-05-23T10:00:00+02:00'
draft = true
tags = ['azure-devops', 'managed-identity', 'aci', 'devsecops', 'wif']
categories = ['Secure Delivery']
summary = 'Self-hosted Azure DevOps agents normally register with a Personal Access Token. PATs are user-bound, expire silently, and sit in plain text on the agent. This post walks through registering an Azure Container Instance agent with a user-assigned managed identity instead — no secret on disk, no rotation, no user binding.'
+++

## TL;DR

Self-hosted Azure DevOps agents register against an org with a PAT. PATs bind to the
user who issued them, expire silently, and live as plain text in the agent's config.
In any regulated tenant where Conditional Access blocks legacy auth — and especially
where org policy disallows both PATs and Entra app client secrets — the default
self-hosted agent flow is a hard stop.

The fix is unglamorous and works: attach a user-assigned managed identity to an Azure
Container Instance, have the container fetch an Entra access token for the ADO resource
ID, and pass that token to `config.sh --auth pat`. ADO accepts an Entra token where it
expects a PAT, the token is short-lived and never stored, and the UAMI is the only
durable principal. This post is the walkthrough — including the half-dozen gotchas
that ate an afternoon.

## The problem

Microsoft-hosted agent minutes run out. Past the free tier, you pay per minute and the
cost scales linearly with usage. Self-hosting flips the curve to capacity pricing,
which is the right shape for any team that runs more than an hour or so of pipeline a
day.

Microsoft's tutorial path for self-hosted agents is to install the agent on a VM or
container and register it with a PAT. The
[official docs](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/linux-agent)
make this look fine. In a regulated tenant it isn't:

- A PAT is bound to the user who issued it. When that user offboards, every agent
  registered with their PAT stops working — silently, at the next token check.
- PATs default to a finite lifetime. The longest you can set is one year. They expire
  at midnight UTC on the expiry date with no graceful warning to the agent.
- The agent config stores the PAT as plain text in `.credentials` under the agent
  install directory. Anyone with read access to the host has it.
- Rotation is all-or-nothing. Reissuing the PAT means re-registering every agent that
  uses it.
- In tenants where Conditional Access blocks legacy auth, the PAT issuance flow itself
  may be unavailable to most users.

In tenants designed against current Conditional Access guidance, the only
non-interactive principal the policy stack tends to accept is a managed identity. So
in those environments the question is rarely "should we use PATs" — it's "can we make
`config.sh` accept something that isn't a PAT".

## Why existing approaches fall short

Three patterns show up in community write-ups.

**System-assigned managed identity on a VM agent.** Works mechanically, but ties the
identity to the VM's lifecycle. Rebuild the VM, lose the identity. Share an agent pool
across teams, share the VM's identity scope — which is usually broader than any one
pipeline needs. Microsoft's own guidance now points to UAMI for this exact reason.

**Service principal with client secret.** The default Azure CLI / Terraform pattern.
The "secret" is still a secret you have to store, rotate, and audit. In a tenant where
client secrets are policy-blocked, this isn't on the table at all.

**Workload Identity Federation.** WIF is the right answer for what runs *inside* the
pipeline (Terraform plans, az CLI calls, Graph requests). It does not solve agent
registration, because at registration time there is no pipeline yet — there's a
container starting up and trying to attach to a pool. WIF needs an OIDC issuer
(GitHub, ADO) sitting on top of an existing workload; the agent bootstrap predates
that.

The undocumented (well, lightly-documented) fact that unlocks the rest of this post:
the ADO `config.sh` script's `--auth pat` mode accepts any bearer token for the ADO
resource. The "token" doesn't have to be a PAT. An Entra access token issued for
resource `499b84ac-1321-427f-aa17-267ca6975798` (the public ADO application ID) works.

That's the seam.

## The pattern

A user-assigned managed identity holds the durable trust relationship with ADO. The
agent host (an Azure Container Instance) attaches the UAMI at runtime. On startup,
the container's bootstrap script does three things:

1. Fetches an Entra access token for the ADO resource via the UAMI.
2. Passes that token to `config.sh --auth pat --token $TOKEN`.
3. Starts the agent under a non-root user.

No secret is written to disk. No rotation is needed. The token's one-hour lifetime is
irrelevant after registration — ADO issues the agent its own session credentials once
the agent is attached.

{{< mermaid >}}
flowchart LR
    UAMI[User-assigned<br/>managed identity] -->|attached| ACI[Azure Container<br/>Instance]
    ACI -->|IMDS request| Entra[Entra ID]
    Entra -->|access token| ACI
    ACI -->|config.sh auth pat| ADO[Azure DevOps<br/>agent pool]
    UAMI -.->|Pool Administrator| ADO
{{< /mermaid >}}

The UAMI needs one Azure DevOps permission, set once: **Pool Administrator** on the
target agent pool. No subscription-level RBAC is required for the agent itself.

## Walkthrough

I'll use synthetic identifiers throughout. Adjust to your tenant:

- ADO organisation: `dev.azure.com/contoso`
- Agent pool name: `self-hosted-aci`
- UAMI name: `id-ado-agent-demo-weu-001`
- Resource group: `rg-ado-agents-weu-01`

### 1. Create the user-assigned managed identity

```powershell
$rg     = 'rg-ado-agents-weu-01'
$loc    = 'westeurope'
$uami   = 'id-ado-agent-demo-weu-001'

az group create --name $rg --location $loc
az identity create --resource-group $rg --name $uami --location $loc

$uamiId       = az identity show -g $rg -n $uami --query id     -o tsv
$uamiClientId = az identity show -g $rg -n $uami --query clientId -o tsv
```

### 2. Add the UAMI to ADO and grant Pool Administrator

ADO doesn't know what a managed identity is until something with the UAMI's object ID
signs in. The cleanest path: add the UAMI as a Service Principal user in the org
(ADO refers to managed identities as "Service Principals" in the UI), then grant Pool
Administrator on the target pool.

```powershell
$uamiPrincipalId = az identity show -g $rg -n $uami --query principalId -o tsv
# Use this object ID in ADO:
#   Organization settings -> Users -> Add user -> Add a service principal
#   Paste $uamiPrincipalId, set access level to Stakeholder
#
#   Organization settings -> Agent pools -> self-hosted-aci -> Security
#   Add the service principal with role: Administrator
```

The CLI alternative uses the ADO REST API; for a single agent pool the UI path is
faster and just as auditable (it lands in the ADO audit log).

### 3. Bootstrap script

The container runs `mcr.microsoft.com/azure-cli:latest`, which is CBL-Mariner under
the hood — not Alpine, not Ubuntu. The package manager is `tdnf`. This trips up
every blog post that assumes `apk` or `apt-get`.

Save this as `aci-agent-bootstrap.sh`:

```bash
#!/usr/bin/env bash
# Runs as root inside the ACI container.
# Required env vars: ADO_ORG_URL, ADO_POOL, UAMI_CLIENT_ID, AGENT_VERSION
set -euo pipefail

tdnf install -y shadow-utils sudo tar gzip ca-certificates which

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

- The `--allow-no-subscriptions` flag matters. A bare UAMI with no Azure RBAC roles
  fails `az login --identity` without it, because the CLI tries to enumerate
  subscriptions by default.
- `--client-id` replaces the deprecated `--username` flag. Old samples still use
  `--username`; they break on current Azure CLI.

### 4. Deploy the ACI

Inline command syntax with PowerShell escaping is painful for a multi-line bootstrap.
Use an ARM template (or Bicep) and pass the script as a parameter. The shape:

```powershell
$adoOrgUrl     = 'https://dev.azure.com/contoso'
$adoPool       = 'self-hosted-aci'
$agentVersion  = '3.245.0'

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

The ARM template attaches `uamiResourceId` to the container, sets `restartPolicy` to
`Never` while you're debugging (so failed runs leave logs intact), and runs:

```bash
/bin/bash -c "echo $BOOTSTRAP_B64 | base64 -d > /tmp/start.sh && bash /tmp/start.sh"
```

Once it works end-to-end, flip `restartPolicy` to `OnFailure` so transient network
blips self-heal.

### 5. Verify

```powershell
az container logs -g $rg -n aci-ado-agent-001 --follow
```

You're looking for the `Listening for Jobs` line. In the ADO UI, the pool's Agents tab
should show one online agent named after the container's hostname. Run a one-line
pipeline against the pool to confirm:

```yaml
pool: self-hosted-aci
steps:
  - pwsh: Write-Host "Hello from $(hostname) — no PAT involved."
```

## Pitfalls and gotchas

This is the part that's worth the price of admission. The architecture is obvious once
written down. These eat hours one at a time.

**`vstsagentpackage.azureedge.net` is unreachable from ACI in several regions.** The
DNS resolves, the connection hangs. Microsoft documents this as a known constraint for
some egress configurations. Use `https://download.agent.dev.azure.com/agent/...`
instead — same package, supported URL.

**Agent refuses to run as root.** Newer agent versions (3.x) explicitly check and exit
if `whoami` returns root. Hence the `useradd -m azp` and `sudo -u azp` calls. Skipping
this is the most common reason "but it worked on my VM" doesn't translate to ACI.

**`az login --identity --username` is deprecated.** It still half-works against older
CLI versions and silently does the wrong thing against new ones. Always use
`--client-id`.

**UAMI without subscription roles needs `--allow-no-subscriptions`.** The bootstrap
script's UAMI typically has zero Azure RBAC — its scope is Pool Administrator on an
ADO pool, which is outside ARM. `az login --identity` defaults to enumerating
subscriptions, fails, and the CLI exits.

**The azure-cli image is CBL-Mariner, not Alpine.** `apk add` and `apt-get install`
both fail. Use `tdnf install -y shadow-utils sudo`. If you build your own image, you
can pick whatever base you want — but the trade is image hosting, scanning, and
patching, which a public Microsoft image gives you for free.

**Pool Administrator must be granted before the first registration attempt.** If the
UAMI doesn't yet have Pool Administrator on the target pool, `config.sh` fails with a
generic "TF400813" error that doesn't mention permissions. Easy to misdiagnose as a
token issue.

**`restartPolicy: OnFailure` swallows logs.** During debugging, the container restarts
on bootstrap failure and ACI returns `None` for the previous run's logs. Set
`restartPolicy: Never` until the registration flow is stable, then flip it.

**ADO accepts Entra tokens at `config.sh` — but only at the configuration step.** Once
the agent is registered, it manages its own session lifetime against ADO. You don't
need to refresh the Entra token; the agent only needs it for the few seconds of
`config.sh --auth pat`.

## When not to use this

This pattern is a poor fit for:

- **High-throughput pools.** ACI is sized for predictable, low-to-medium concurrency.
  A single container running one agent is fine; running ten agents per container is
  not. If you need dozens of concurrent agents, look at AKS-hosted agents or VM scale
  sets. The auth pattern transfers, the compute substrate doesn't.
- **Long-running build jobs (multi-hour) that need on-host secrets.** This pattern
  doesn't change how the *pipeline* gets secrets — only how the *agent* registers.
  Pipelines should use WIF for in-job auth; if your team isn't there yet, address
  that separately.
- **Strict workload isolation across tenants on shared infrastructure.** ACI
  containers share more network namespace than some compliance frameworks prefer.
  Deploy one ACI per isolation boundary rather than multiplexing.

## How this plays out in the reference platform

The principle — *remove every long-lived credential from the agent and lean on
Entra-issued, short-lived tokens for everything* — applies to any regulated tenant.
In the reference platform, it's load-bearing.

This worked example is distilled from real multi-customer delivery in regulated Microsoft cloud environments, generalized here rather than any single customer's deployment.

**The constraint that forced it.** The reference platform's Conditional Access stack disallows
both PATs and Entra app client secrets organisation-wide. There's no exception
process for "but we need to run a build agent" — and there shouldn't be, because the
exception you carve once is the exception you forget to expire. The pattern in this
post is the only path that doesn't require carving one.

**One ACI per customer subscription.** Per-customer workload isolation rules out
multiplexing agents across customers. The agent pool model is one ACI deployment per
customer subscription, with the customer's own UAMI scoped to a customer-specific
agent pool. The bootstrap script is the same; the parameters change per deployment.
The blast radius of a compromised agent is bounded to one customer tenant.

**Pool Administrator as the only durable grant.** The UAMI's only ADO right is Pool
Administrator on the customer-specific pool. The pipelines that *run* on that pool
authenticate via WIF for everything they touch (Terraform plans against the
customer's subscription, Graph writes against the customer's Entra tenant). The
agent's registration identity and the pipeline's runtime identity are deliberately
distinct.

**Audit trail.** Every agent registration emits an ADO audit-log event tied to the
UAMI's object ID. Combined with the Sentinel rule that watches for Pool Administrator
grants outside the IaC pipeline, the reference platform has a real-time signal for "someone
manually attached an agent to a customer pool" — which is exactly the human
behaviour that should not be silently possible.

## Wrap-up

The no-PAT pattern collapses the agent's footprint to one durable principal (the
UAMI), one durable grant (Pool Administrator on the target pool), and zero secrets at
rest. Rotation goes from "rotate a PAT and re-register all agents" to "nothing to
rotate". Offboarding the issuing user has no effect because there is no issuing user.

In any tenant where Conditional Access already blocks PATs and client secrets, this
isn't a nice-to-have — it's the only path that doesn't require carving a policy
exception. And the policy exception you don't carve is the one that never lapses.

A companion lab repo with the sanitized ARM template and bootstrap script will follow
in a separate post.

**References:**

- [Microsoft Learn — Self-hosted Linux agents](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/linux-agent)
- [Microsoft Learn — User-assigned managed identities](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities)
- [Azure DevOps resource ID `499b84ac-...`](https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/service-principal-managed-identity) — Microsoft's authoritative reference for using SPs and MIs against ADO.
