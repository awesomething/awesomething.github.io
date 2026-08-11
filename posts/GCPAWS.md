---
title: Args
date: 2018-04-04
---

At this point, the AWS side is making a legitimate architectural argument, so I would avoid defending GCP purely on “we already built it.” The stronger contribution is to force the team to define the **decision criteria** and distinguish strategic architecture from migration cost.

You could say:

> “I think the AWS argument is valid if we assume the orchestrator’s center of gravity is going to be live operational actions and AWS-hosted services. In that case, colocating orchestration with AgentCore and the operational APIs gives us simpler IAM, potentially lower latency, and fewer cross-cloud trust boundaries.
>
> But I don’t think the decision should be ‘AWS has IAM, therefore orchestration belongs in AWS.’ We should compare the complete control plane. For each option, we should score identity and authorization, safety controls, observability, latency, cross-cloud calls, operational ownership, security certification, and how much functionality we have to rebuild.
>
> If AWS wins that comparison long-term, then we should choose AWS. But we should explicitly account for the fact that the GCP path already has production-tested safety, streaming, DLP, model protection, telemetry, and deployment controls. Those are architectural capabilities, not just sunk implementation effort.”

Then I would add one particularly important question:

> “What percentage of an average request actually requires AWS operational data or an AWS action, versus reporting and conversational reasoning over GCP-resident data? That should influence where the orchestration boundary belongs.”

That question matters because the team is currently arguing from assumptions about workload locality.

### A useful challenge to Kevin’s latency argument

Kevin is essentially saying: “put the brain near AWS because that’s where the workload is.”

You can respond:

> “I agree with workload locality, but we should validate what ‘the workload’ actually is. If most turns are reporting against BigQuery and only a minority invoke a menu or transactional action in AWS, moving the orchestrator to AWS could actually invert the cross-cloud traffic rather than eliminate it.”

That is a very strong neutral architecture point.

### Also clarify the IAM argument

Their AWS authorization point is good, but it isn’t free.

You can say:

> “AWS-native IAM definitely simplifies AWS-to-AWS authorization. But the user identity still originates outside that service boundary, and we still need tenant scope, restaurant scope, tool-level authorization, auditing, and propagation of that identity into downstream calls. So AWS IAM removes one integration problem; it doesn’t remove the application authorization model.”

That prevents the discussion from oversimplifying “IAM solves security.”

### The direction I would recommend now

The meeting seems to be converging toward a **shared orchestration implementation with separate deployments** for R/S/N. That changes the question.

I’d propose this:

> “If we agree the orchestrator is the same logical codebase but independently deployed for R, S, and N, then maybe our architectural requirement should be cloud-portability at the orchestration layer. Define tools behind interfaces, keep the domain logic cloud-neutral, and make AWS versus GCP a deployment decision rather than embedding cloud-specific services deeply into orchestration.”

That is likely the most constructive contribution you can make right now.

It gives both camps something:

* AWS can host the N orchestrator close to operational data.
* GCP can continue hosting R where the production controls already exist.
* The team avoids maintaining two fundamentally different “brains.”
* MCP/tool contracts become the stable abstraction.
* You can later consolidate on AWS or GCP without rewriting the agent behavior.

If they ask you directly **“what does GCP provide that AWS does not?”**, don’t claim unique capabilities unless you can prove them. Answer:

> “Probably very little that AWS fundamentally cannot provide. The GCP advantage here is maturity of *our implementation*: existing production controls, security validation, operational telemetry, integrations, and known behavior. The AWS advantage is locality to operational systems, native authorization, and likely easier infrastructure ownership. So the real decision is whether those AWS operational advantages outweigh the cost and risk of recreating the production control plane.”

That is the cleanest answer to the argument currently on the table.
