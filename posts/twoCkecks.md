---
title: Two Checks
date: 2023-02-15
---

The proposed architecture is directionally strong because it avoids running every domain-specific check for every request.

The central idea is:

```text
Global checks
    ↓
Routing
    ↓
Route-specific readiness checks
    ↓
Clarification when required
    ↓
Selected agent
```

That is better than putting location, time range, menu, and other business-context checks in a global pre-orchestration path.

## What is good about the proposal

The distinction between **global checks** and **route-specific checks** is the strongest part.

Global checks apply to nearly every request:

* Authentication and access prerequisites
* Request-size limits
* DLP or sensitive-data controls
* Supported-language detection
* Basic safety checks

These should run before routing because their result does not depend on whether the request is analytical, RAG-based, or something else.

Domain-specific checks should run only after routing:

* Location requirements
* Time-range requirements
* Menu or item resolution
* Business-context loading
* Analytical metric requirements

A RAG request should not pay the cost of loading locations or validating analytical time ranges when those values are irrelevant.

## Recommended architecture

```text
User request
    ↓
Global deterministic guardrail layer
- authentication
- size limits
- language
- DLP
- basic safety
    ↓
Existing routing agent
    ├── RAG route
    │      ↓
    │   RAG-specific readiness, only if needed
    │      ↓
    │   RAG agent
    │
    ├── Analytics/POC route
    │      ↓
    │   Load authorized business context
    │      ↓
    │   Analytics readiness checks
    │   - location
    │   - date range
    │   - metric
    │      ↓
    │   clarification or analytics agent
    │
    └── Manual/other route
           ↓
        Route-specific readiness
           ↓
        Selected agent
```

This preserves the current routing logic and limits additional latency to the route that needs the extra work.

## Important improvement: route first, but do not load everything immediately

Even after the router selects the analytics path, avoid eagerly calling every API.

Use progressive checks:

1. Determine which fields the selected use case requires.
2. Inspect what is already available in the question and session.
3. Load only the missing context needed to make the readiness decision.
4. Ask a clarification only when the result cannot be safely inferred.

For example:

```text
Analytics route selected
    ↓
Does this intent require location?
    ├── No → skip location loading
    └── Yes
          ↓
      Is location already resolved?
          ├── Yes → continue
          └── No → load authorized locations
```

Otherwise, even within the analytics route, the implementation could recreate the same latency problem at a smaller scale.

## Feedback on the latency argument

The statement that two to four seconds for clarification is automatically acceptable is too broad.

A clarification response may feel more acceptable than a silent delay because the user receives an interactive result, but it should still be measured. Four seconds before asking a simple question such as “Which location?” could feel slow.

Track at least:

* Time to first visible clarification
* Routing latency
* Context-loading latency
* Readiness-decision latency
* Resume latency after the user answers
* Total time to final answer
* Clarification success rate
* Unnecessary clarification rate

Also separate these measurements:

```text
No-clarification request:
global checks + routing + agent response

Clarification request, first turn:
global checks + routing + readiness + clarification

Clarification request, resumed turn:
pending-state lookup + validation + selected agent
```

The resumed request should ideally avoid repeating routing and expensive context loading.

## The biggest unresolved issue: what happens on the second user message?

This was the most important technical question at the end of the discussion.

When the system is waiting for a location and the user replies:

> “What were my sales yesterday?”

that may be:

* An answer to the pending clarification
* A replacement of the original request
* A new unrelated request
* An answer that also introduces another missing field

The system should not blindly treat every next message as the clarification answer.

Use an explicit pending-state resolver:

```text
New user message
    ↓
Is there a pending clarification?
    ├── No → normal global checks and routing
    └── Yes
          ↓
      Does the message satisfy the expected field?
          ├── Yes → validate, merge, and resume
          ├── Clearly a new request → cancel pending state and route normally
          └── Ambiguous → ask a concise confirmation
```

A useful pending record would contain:

```text
conversation_id
user_id
original_request_id
selected_route
original_query
expected_fields
allowed_values or validation reference
already_loaded_context
expiration_time
clarification_attempt_count
```

## Do not assume native graph suspension is required

There are two valid implementation patterns.

### Persist and reinvoke

Store the pending state, end the current request, and process the next message as a new invocation that resumes the logical workflow.

This is usually simpler and more robust for chat systems.

### Native pause and resume

Suspend a graph execution and resume the same execution when input arrives.

This can preserve runtime state, but introduces lifecycle, timeout, deployment, and recovery complexity.

For your POC, persisted logical continuation is probably sufficient. The product requirement is continuity of the user’s task, not necessarily continuation of the same in-memory graph execution.

## One clarification versus multiple missing fields

The team needs an explicit policy.

A good default is:

> Ask one clarification interaction, but allow it to collect multiple tightly related structured fields.

For example:

> “Which location and reporting period should I use?”

This is preferable to:

1. Which location?
2. Which date?
3. Which metric?
4. Which menu?

However, do not combine unrelated or complex questions merely to avoid another turn. If several fields are missing because the request itself is poorly formed, it may be better to let the orchestrator handle it.

## Suggested response during the meeting

You could say:

> “I agree with the separation. Global deterministic safeguards should run before routing because they apply to every request. Once the existing router selects a route, we should execute only the readiness checks required for that domain. For an analytics request, that may include location and time range. For a RAG request, those checks would not run unless that RAG use case explicitly requires them.
>
> The remaining design question is the resume behavior. We need pending state that preserves the selected route and loaded context, but the next user message must first be classified as either an answer to the clarification or a new request. We should not automatically attach every follow-up message to the pending question.”

## Main risk to call out

The design could gradually create a second orchestration system if each route accumulates custom tools, branching, context loaders, and clarification logic.

To prevent that:

* Keep the global layer small.
* Keep route readiness declarative.
* Use shared outcome types.
* Limit clarification depth.
* Fall through to the agent when confidence is low.
* Avoid duplicating routing decisions inside readiness logic.

A declarative rule could look like:

```text
Intent: sales_summary
Required fields:
- time_range
- location, only when user has multiple authorized locations

Clarification policy:
- one interaction maximum

Fallback:
- orchestrator
```

The best summary of the proposal is:

> **Run universal safeguards globally, route using the existing orchestrator, and perform only the readiness checks required by the selected route. Preserve the route and context during clarification, but validate whether the next message is truly an answer before resuming.**

There is still a readiness check. It just moves to the right place.

Use two levels:

* **Global readiness checks before routing** for things that apply to every request, such as supported language, request length, authentication, DLP, and basic safety.
* **Route-specific readiness checks after routing** for things such as location, time range, metric, or menu item.

So the design becomes:

```text
User request
  → global checks
  → existing router
  → route-specific readiness check
  → clarification if needed
  → selected agent
```

What should probably go away is a single global readiness layer that runs every possible check on every request. That would add unnecessary latency—for example, loading location context for a RAG question that does not require a location.

A good way to say it in the meeting is:

> “We are not removing readiness checks. We are separating universal checks from domain-specific readiness. Universal checks run before routing, while business-context checks run only after the relevant route is selected.”

Generally, **no**. Location readiness should usually be **route-specific**, not global.

A global check should apply to nearly every request regardless of destination, such as:

* Authentication
* Request size
* Supported language
* DLP or safety controls

Location is only relevant for certain workflows, such as sales analytics or store-level reporting. A RAG or general knowledge request may not need location at all, so checking or loading it globally adds unnecessary latency and coupling.

A good rule is:

> Run location readiness globally only if location is a universal tenancy or authorization boundary for every request.

Otherwise, use:

```text
User request
  → global checks
  → router
  → route requires location?
      → no: continue
      → yes: run location readiness
```

Even within a location-dependent route:

* One authorized location: auto-resolve.
* Multiple locations and one clear match: continue.
* Multiple locations and none specified: clarify.
* No authorized locations: return an access response.
* Ambiguous or complex reference: fall through to semantic resolution or the orchestrator.

So for your current example, I would classify location as an **analytics-route readiness check**, not a global check.

Yes—with one important nuance.

**Not all deterministic checks should come after routing.** Split them into two categories:

### 1. Global deterministic checks (before routing)

These are universal and cheap.

Examples:

* Authentication/session valid
* Request size limit
* Supported language
* DLP/basic safety
* Malformed request checks

These should always run first because they don't depend on the user's intent.

### 2. Route-specific deterministic readiness checks (after routing)

These depend on **what the user is asking**.

Examples:

* Location required for sales analytics
* Time range required for trend reports
* Menu item required for menu analytics
* Store or organization context
* Business-specific required parameters

These should run **after the router** has identified the request type.

So the flow I'd recommend is:

```text
User request
    ↓
Global deterministic checks
    ↓
Router (existing orchestrator routing)
    ↓
Route-specific deterministic readiness
    ↓
Clarify if needed
    ↓
Selected agent
```

This actually addresses the concern your teammate raised about latency:

> "Why should a RAG question pay the cost of location detection?"

It shouldn't. The router first decides it's a RAG request, and the location readiness check is simply never invoked.

### One point I'd raise in the architecture discussion

Rather than saying **"the readiness check comes after routing,"** I'd say:

> **"Universal readiness comes before routing; domain readiness comes after routing."**

That wording makes it clear you're not removing the readiness layer—you are **moving business-specific readiness to the point where the system knows whether it is actually needed.**

