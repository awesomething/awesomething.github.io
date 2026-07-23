---
title: Arch or Software Design
date: 2025-12-11
---

This segment surfaced the most important gap in the first demo: **the clarification rule must consider the user’s authorized-location count, not merely whether the question contains a location.**

## Best answer to the one-location question

A stronger response would be:

> “Correct. If the user has access to exactly one location, we should not ask them to choose a location. The readiness check can resolve that location automatically and continue. We only ask a clarification when the user has access to multiple locations and the request does not identify one clearly. If the user has no authorized locations, we return the appropriate access message.”

That gives you a clean decision table:

| Authorized locations | Location in question      | Result                         |
| -------------------- | ------------------------- | ------------------------------ |
| 0                    | Any                       | Stop with an access message    |
| 1                    | Missing                   | Auto-resolve the only location |
| 1                    | Explicit but unauthorized | Reject or clarify safely       |
| Multiple             | Exact authorized match    | Continue                       |
| Multiple             | Missing                   | Ask which location             |
| Multiple             | Ambiguous match           | Ask the user to choose         |
| Multiple             | No reliable match         | Fall through or clarify        |

This should become part of the demo because it shows the readiness check is based on **business context**, not just keywords.

## Feedback on how the discussion went

### What went well

The team converged on a more realistic hybrid design:

* Deterministic checks for high-confidence cases.
* Existing state reused for authorized locations.
* Exact matching attempted first.
* LLM or orchestrator fallback for uncertain matching.
* Shadow mode and measurement retained.
* Human-in-the-loop resume behavior treated separately from the readiness decision.

That is a much stronger architecture than presenting rules and agents as mutually exclusive.

### Where your answer became unclear

When asked about “human in the loop,” you responded by asking how it differed from your readiness check. That was useful for clarification, but you should then explicitly separate two concepts:

1. **Decision:** Does this request require clarification?
2. **Execution:** How do we pause, store the pending request, collect the answer, and resume?

The readiness gate handles the first. Human-in-the-loop or resumable state handles the second.

A polished answer would be:

> “The readiness check determines whether input is missing. Human-in-the-loop is the mechanism used after that decision: persist the pending request, ask the user, receive the response, and resume execution. They are complementary rather than competing approaches.”

## The architectural distinction to emphasize

There are really three layers in the conversation:

### 1. Hard request guards

Examples:

* Maximum request length.
* Supported language.
* Authentication present.
* User has no authorized locations.

These are deterministic and may terminate immediately.

### 2. Readiness resolution

Examples:

* User has one authorized location, so auto-resolve it.
* User has multiple locations but did not identify one.
* User named an exact authorized location.
* Several locations match the supplied phrase.

This produces `READY`, `NEEDS_CLARIFICATION`, `NOT_ALLOWED`, or `INCONCLUSIVE`.

### 3. Orchestration and reasoning

Examples:

* Interpreting semantically complex requests.
* Resolving uncertain entity references.
* Selecting tools and downstream agents.
* Answering the business question.

This framing will make the discussion much easier to follow.

## Answer to “where can these skills live without a pre-orchestrator?”

Avoid continuing to call them “skills” unless they are actually agent-invoked tools.

A strong answer is:

> “The reusable deterministic logic should live in a shared readiness or request-validation module, not inside a pre-orchestrator agent. The API entry path calls that module before invoking the orchestrator. Agentic or LLM-based entity resolution can remain available as a fallback after the deterministic pass, either through the orchestrator or through a separately defined resolution service.”

You can describe the flow as:

```text
Request
  ↓
Existing safety and access guards
  ↓
Deterministic readiness checks
  ├─ Ready → Orchestrator
  ├─ Clarification → Persist pending state and ask user
  ├─ Not allowed → Safe response
  └─ Inconclusive → Orchestrator or semantic resolver
```

## Important concern raised about event storage

One teammate pointed out that if the request terminates before entering the agent or graph, certain events may not be stored through the existing mechanism.

Do not dismiss this as implementation detail. It is a legitimate observability requirement.

A good answer would be:

> “That is a valid integration concern. Even when the request exits before orchestration, the readiness layer still needs to write a standard trace or conversation event through the shared persistence path. The decision should not depend on starting the agent graph, but its outcome must remain visible in conversation history and telemetry.”

This means the gate should not bypass:

* Audit logging.
* Conversation event storage.
* Request IDs and trace IDs.
* Streaming terminal events.
* Standard error or rejection envelopes.

## Be careful with the LLM fallback proposal

The other implementation described:

1. Exact deterministic matching.
2. An “optimistic” LLM call to match the request against allowed locations.

This is reasonable, but it changes the cost and architecture story. You should distinguish it from your zero-model-call POC.

Say:

> “For the current POC, the enforced path uses deterministic matching only. An LLM-assisted matcher is a possible fallback for ambiguous aliases, but it should be measured separately because it reintroduces latency, model cost, and nondeterminism.”

Also, do not pass sensitive JWT contents directly into a prompt. Only provide the minimum required normalized information, such as authorized location IDs and display names, and confirm that this aligns with your security controls.

## Questions you are likely to get next

### “What if the user has one location but explicitly mentions a different one?”

> “We should not silently replace the location they named. We validate the supplied location against their authorized set. If it is unauthorized or unrecognized, we return a safe clarification or access response.”

### “What if two locations have similar names?”

> “The deterministic matcher should only auto-resolve exact or uniquely normalized matches. If multiple authorized locations match, the result is `NEEDS_CLARIFICATION`, and the user is shown only those matching options.”

### “What if the user says ‘my usual location’?”

> “That requires an explicitly defined preference or recent-context rule. Without one, the readiness result should be inconclusive rather than guessing.”

### “Should a single location be injected automatically into every request?”

> “Only for workflows where location is a required scoped parameter and the user has exactly one authorized option. It should be added as structured context, not by rewriting the user’s text.”

### “Where is the pending clarification stored?”

> “In durable or session-scoped pending state bound to the user, conversation, original request ID, expected field, and expiration time. The user response must match that state before resuming.”

### “Do we resume the same graph?”

> “That depends on the runtime design. The core requirement is to resume the same logical request with preserved state. In ADK, that may use session state and a new invocation; in a graph runtime, it may use a native pause-and-resume mechanism.”

## Recommended wording for your second demo

Use this when you reach the missing-location example:

> “Before asking a clarification, the checker reads the user’s authorized locations from existing session state. If there are none, it stops with an access message. If there is exactly one, it fills that location automatically. If there are several, it checks whether the user named one clearly. Only when several locations remain possible does it ask the user to choose.”

That single explanation answers the concern before the audience raises it and makes the POC appear substantially more mature.

This section sounds like an architecture alignment discussion rather than part of the main demo. The core conclusion is useful, but it needs to be stated much more directly.

## What the team appears to have agreed on

The final choice between a **pre-orchestrator agent** and a **deterministic readiness check** is still pending.

However, several pieces of work remain useful under either architecture:

* Location, time-frame, language, and menu capabilities should be independently callable.
* The same capabilities could be invoked by either the pre-orchestrator or the existing orchestrator.
* Streaming-event translation is required regardless of which agent calls the capability.
* Human-in-the-loop and pending-state work can continue independently.
* Testing of the individual capabilities can begin before the architecture decision.

A concise way to summarize that in the meeting would be:

> “The caller is still undecided, but the reusable capabilities and streaming contract are needed in either design. So the immediate work should focus on making those capabilities caller-independent and defining how their events are translated for the front end.”

## The streaming problem

The important technical point is that raw ADK events should not be sent directly to the front end.

The current concern appears to be:

1. ADK emits many internal events.
2. Those events may expose internal names, tool details, or implementation-specific structures.
3. The backend currently merges or morphs those events into a frontend payload.
4. That transformation is becoming complicated and tightly coupled to ADK and individual tools.

The cleaner architecture is:

```text
ADK or readiness capability
        ↓
Raw internal event
        ↓
Central event normalization layer
        ├─ Persist full internal event for logs/tracing
        └─ Produce approved public stream event
                    ↓
                Front end
```

The key rule should be:

> Log the complete internal event, but expose only a stable, sanitized public event contract.

For example, a raw event from a time-frame detector might contain internal tool names, arguments, and execution details. The public stream should instead contain something like:

```json
{
  "type": "context_resolved",
  "field": "time_range",
  "status": "resolved",
  "display_message": "Using yesterday as the reporting period."
}
```

The frontend should not need to know whether the result came from ADK, a deterministic check, an LLM tool, or a future service.

## Strong answer to the architecture question

You asked:

> “Are we deciding to use the pre-orchestrator check or the pre-orchestrator agent?”

A stronger answer during the meeting would be:

> “The final caller is not decided yet. To avoid rework, I suggest we separate the decision logic from the caller. Location, time-frame, language, and menu resolution should expose stable interfaces and neutral events. Then either the readiness layer or the orchestrator can invoke them once the architecture decision is made.”

This avoids making your work depend on Monday’s decision.

## Clarify the one-question limitation

The criticism that the current POC only asks one location question is fair, but the response became somewhat defensive.

You should answer it this way:

> “The current POC intentionally demonstrates a single clarification cycle. It is not intended to support an unrestricted interview with the user. The production design needs an explicit policy: either ask one combined clarification for all high-confidence missing fields, or ask one prioritized question and let complex cases fall through to the orchestrator.”

That introduces an important product decision.

### Option A: One combined clarification

> “Which location and reporting period should I use?”

Advantages:

* Fewer user turns.
* Better when several known required fields are missing.

Risks:

* More complex UI and validation.
* May overwhelm the user.
* Harder to map free-text answers reliably.

### Option B: One prioritized clarification

Ask only the highest-priority missing field.

Advantages:

* Simple interaction.
* Easier validation.
* Matches the current POC.

Risks:

* The request may still be incomplete afterward.
* Can create repeated back-and-forth.

### Recommended policy

Use a single clarification interaction, but allow it to request multiple structured values when the use case requires them. If more uncertainty remains after that interaction, pass the request to the orchestrator rather than repeatedly interrogating the user.

## Important terminology correction

The conversation uses “tool,” “skill,” “check,” and “agent” interchangeably. That will create confusion during the demo.

Use these definitions consistently:

* **Check:** deterministic validation or resolution with no LLM reasoning.
* **Tool:** a callable capability available to an agent or service.
* **Agent:** an LLM-driven component that decides what actions to take.
* **Streaming adapter:** converts internal events into the public frontend contract.
* **Human-in-the-loop handler:** persists pending state, gathers user input, and resumes the request.

A deterministic time parser could technically be exposed as a tool, but it remains a deterministic capability. Calling it a tool does not make it an agent.

## Feedback on your responses

### What you did well

You correctly emphasized that:

* The work should not be thrown away if the architecture changes.
* Complex edge cases can still fall through to the orchestrator.
* The POC is intentionally scoped.
* Rule sets can expand gradually.
* Latency remains a legitimate concern.
* Shadow mode allows evidence to guide the decision.

### What to improve

Avoid phrases like:

> “That’s why I said you should go through the branch.”

Even when justified, it may sound defensive. A more collaborative response is:

> “The branch currently enforces one clarification cycle by design. Let me show where that behavior is implemented so we can decide whether the next iteration should support a combined multi-field clarification.”

Also avoid accepting the description that the implementation is “lame.” Redirect to scope:

> “It is deliberately narrow because it is a feasibility POC. The next question is which production behaviors we want to add before evaluating it.”

## Questions you may be asked next

### “Should each capability emit its own custom frontend event?”

> “Each capability may produce domain-specific internal data, but it should map into a small shared set of public event types. We should avoid creating a completely different streaming protocol for every tool.”

### “Should we stream every ADK event?”

> “No. We should log every relevant event internally, but only stream events that improve the user experience, such as progress, clarification, completion, safe termination, or a resolved context message.”

### “Where should the mapping live?”

> “In a centralized streaming adapter or event-normalization service. Individual tools should return structured results rather than constructing frontend payloads themselves.”

### “How do we avoid coupling the adapter to tool names?”

> “Map based on stable event categories and structured result schemas, not internal class names or ADK-specific event names. A registry can associate a capability type with a public event formatter.”

### “What if the architecture changes from pre-orchestrator to orchestrator?”

> “The capability contracts and public event contract remain unchanged. Only the caller and invocation point change.”

### “Can we test before the decision?”

> “Yes. We can independently test each capability’s input, result schema, authorization handling, failure behavior, latency, and public event mapping. The architecture decision does not block those tests.”

### “Do we need Streaming Helper V2?”

> “We need the behavior, but the final component should not remain a temporary V2 fork. We should identify the reusable changes, integrate them into the primary streaming path, and retire the duplicate path after compatibility testing.”

### “What should be stored?”

> “Store the raw event for diagnostics where permitted, plus a normalized trace record containing request ID, capability, outcome, latency, reason code, and public event type. Sensitive arguments should be redacted.”

## Best summary to use in the next discussion

> “Until the architecture decision is final, we should avoid implementing logic that depends on whether a pre-orchestrator exists. The reusable capabilities should expose stable structured results, and a neutral streaming adapter should translate those results into frontend-safe events. Then the final design can invoke them from either the readiness layer or the orchestrator without rewriting the capabilities or streaming path.”

