---
title: Cursor Routing
date: 2025-11-20
hidden: true
---
### ABOUT CURSOR ROUTING
Yes. Cursor’s **Auto model selection is essentially a model router sitting in front of the coding models**, rather than a single model called “Auto.” Cursor has recently made much more of this architecture public.

### How the routing works

For the newer **Cursor Router** on Teams and Enterprise, Cursor says every agent request is classified *before* the coding model runs. The classifier considers the **user’s query, the context supplied from the codebase/conversation, task complexity, and task domain**, along with Cursor’s learned knowledge of which models perform well on different kinds of work. ([Cursor][1])

So your intuition is basically correct: it is trying to infer **what the developer is trying to accomplish**, but at the level of the current agent request rather than maintaining some magical high-level specification of your ultimate project goal.

A simplified backend looks roughly like:

```text
You: "Refactor this authentication flow"
          │
          ▼
Cursor agent/harness
  • conversation history
  • relevant files/code
  • errors/tool results
  • rules/instructions
          │
          ▼
Cursor Router classifier
  • What kind of task?
  • How difficult?
  • What domain?
  • How much context?
  • Which models perform well here?
  • Model availability/reliability
  • Cost-vs-quality mode
          │
     ┌────┼──────────────┐
     ▼    ▼              ▼
Composer/Grok    GPT       Claude/etc.
     │
     ▼
tools → code search → edit → terminal → etc.
     │
     ▼
next agent request can be routed again
```

Cursor gives concrete examples: relatively simple tasks can go to price-efficient models, UI work can go to models that have better “taste,” while difficult long-horizon reasoning can go to frontier reasoning models. The router was trained using **600,000+ live requests** and evaluated across millions of production requests. Cursor says it optimizes using signals such as whether users have to correct the agent and how much generated code remains in the codebase (“keep rate”). ([Cursor][1])

Importantly, **the model does not necessarily stay fixed for an entire conversation**. Cursor explicitly says real routing happens across a conversation—deciding both which model to use and when switching models makes sense. ([Cursor][1])

### The token-cost part is a little unusual

There are now two different Auto pricing behaviors.

For the traditional Auto / **Auto Cost** path, you **do not pay the actual underlying model's price**. Cursor charges a fixed Auto rate even if the router happens to give you an expensive model underneath:

| Token type                |             Auto rate |
| ------------------------- | --------------------: |
| Fresh input / cache write | **$1.25 / 1M tokens** |
| Output                    | **$6.00 / 1M tokens** |
| Cache read                | **$0.25 / 1M tokens** |

Those rates are independent of which underlying model Cursor picked. Cursor staff has explicitly confirmed that if Auto routes you to a more expensive frontier model, you still pay the Auto rate rather than that frontier model's normal price. ([Cursor Documentation][2])

So conceptually:

```text
Actual backend:
Auto → expensive frontier model

Cursor's cost to provider:
potentially expensive

What Auto Cost meters to you:
input × $1.25/M
+ output × $6/M
+ cache-read × $0.25/M
```

Cursor takes on the routing/economic risk there. Some Auto requests presumably have very healthy margins because they go to cheap models; others can cost Cursor more because they get routed to expensive models.

### Auto Balance and Auto Intelligence are different

On **Teams and Enterprise**, Cursor launched a newer Router on July 22, 2026 with three choices:

* **Auto Cost** — optimize aggressively for token cost; this is essentially the previous Auto behavior.
* **Auto Balance** — spend more when it materially improves quality.
* **Auto Intelligence** — prioritize frontier-level quality.

For **Balance and Intelligence, you pay the rate of the actual model Cursor routed the request to**, rather than the flat Auto Cost rate. ([Cursor][3])

So suppose three successive agent steps looked like this:

```text
1. "Rename these interfaces"
   → cheap model
   → charged at cheap model rate

2. "Understand this race condition across 17 files"
   → frontier reasoning model
   → charged at frontier model rate

3. "Update these repetitive tests"
   → cheaper model again
   → charged at cheaper model rate
```

That's one reason routing can save money versus simply selecting something like Opus for every request. Cursor reported that its early-access customers saw roughly **30–50% lower costs** than routing everything to Opus 4.8, and its larger A/B testing showed substantial savings while maintaining comparable user satisfaction. Those are Cursor's own measurements, so I'd treat them as vendor-reported results rather than universal guarantees. ([Cursor][1])

There's another Teams-specific detail: when Balance or Intelligence routes to a **third-party model**, Cursor says the Cursor Token Rate can also apply; **Auto Cost and Cursor first-party models such as Composer 2.5 and Grok 4.5 are exempt** from that extra rate. ([Cursor][4])

### Why you may see enormous token numbers

One prompt in the UI is not necessarily one LLM call.

For an agent request like:

> “Figure out why checkout is failing and fix it.”

Cursor might internally do:

```text
Router → model A

read files        ─┐
search code        │
reason             │
run command        ├─ many model/tool cycles
read error         │
inspect more files │
edit code          │
run tests          │
fix test           ┘
```

Every subsequent model turn may contain some combination of the system prompt, chat history, code context and tool results. Much of reused context becomes **cache-read tokens**, which explains why a task might report millions of processed tokens without costing what millions of completely fresh input tokens would cost. Cursor staff has specifically explained that this repeated agent context/tool-call behavior is why apparently simple operations can produce very large token totals. ([Cursor - Community Forum][5])

One subtle optimization Cursor recently disclosed is **dynamic tool calling**: it no longer necessarily stuffs every possible tool description into every prompt. Less-used tools can be loaded only when needed, reducing prompt-token overhead. ([Cursor][1])

Finally, there is a plan distinction worth knowing: **Cursor Router's new Cost/Balance/Intelligence system is currently a Teams/Enterprise feature**; Cursor staff said on July 24 that it isn't currently available on individual plans. Individual Auto therefore shouldn't automatically be assumed to behave exactly like Teams' Auto Balance/Intelligence. ([Cursor - Community Forum][6])

So the shortest mental model is:

**Cursor Auto = intent/task classifier → choose model → run agent → re-evaluate on later requests.**
**Auto Cost = fixed Cursor token price regardless of hidden model.**
**Auto Balance/Intelligence = routed dynamically, and you pay the selected model's token rate.**

That last distinction is the key to understanding the economics.

[1]: https://cursor.com/blog/router?utm_source=chatgpt.com "Introducing Cursor Router · Cursor"
[2]: https://docs.cursor.com/account/pricing?utm_source=chatgpt.com "Cursor – Models & Pricing"
[3]: https://cursor.com/en-US/changelog?utm_source=chatgpt.com "What's New in Cursor — Latest Updates & Release Notes"
[4]: https://cursor.com/business/teams?utm_source=chatgpt.com "Cursor for Teams — Your best engineer's setup, for everyone"
[5]: https://forum.cursor.com/t/anthropic-api-key-still-counts-towards-cursor-usage/162164?utm_source=chatgpt.com "Anthropic API Key still counts towards Cursor usage - Help - Cursor - Community Forum"
[6]: https://forum.cursor.com/t/introducing-cursor-router/166386/14?utm_source=chatgpt.com "Introducing Cursor Router - #14 by Colin - Release Discussions - Cursor - Community Forum"
