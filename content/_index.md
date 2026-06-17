+++
title = "Trust Anchor"
+++

> The principles are universal. The implementations are Microsoft.

At some point, every environment — Microsoft cloud included — hits the same wall: you stop knowing what's set, where the gaps are, and whether the configuration that exists today matches what was intended. Manual discovery doesn't scale. Neither does institutional memory. The underlying problems — configuration drift, implicit trust in pipeline identities, governance that doesn't survive the first manual exception — are structural. They appear in AWS environments, GCP projects, on-premises Active Directory estates, and hybrid architectures just as reliably as they appear in Microsoft cloud.

```yaml
run: everything-as-a-code --source git --on-drift enforce
```

This blog covers the architecture of auditable Microsoft cloud environments: identity governance designed to be inspectable, delivery pipelines that carry their own proof, and the design space where both connect. The writing comes from production projects — identity modernisation (AD FS to Microsoft Entra ID, AD and Entra hardening), banking application security, M365 tenant migrations, multi-tenant platform design, Azure Arc deployments, global cybersecurity policy delivery, and more.
