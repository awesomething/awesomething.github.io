---
title: Agent trade offs
date: 2025-06-12
---

# Meeting Summary: Agent Status and Friendly Streaming Messages

## Executive Summary

The discussion focused on improving the streaming experience for an agent-based application by showing user-friendly progress messages while suppressing internal orchestration details.

A prototype was demonstrated in which the existing response structure remains unchanged for backward compatibility, while an additional `friendly_message` object communicates progress such as:

* Understanding the user’s question
* Detecting the language
* Retrieving information from the knowledge base
* Completing a tool operation
* Preparing the final response

The team generally agreed with the direction but raised concerns about how these messages should be represented, whether they should use existing event types, and whether temporary progress messages might accidentally appear in the final response.

The key design challenge is balancing backward compatibility, frontend simplicity, extensibility, and clear separation between transient progress events and permanent response content.

---

# Cleaned-Up Meeting Minutes

## 1. Initial Setup

The meeting began with troubleshooting related to Google Meet and Calendar behavior. The organizer experienced issues where deleted or modified meetings appeared to retain previous limits or settings.

The team briefly discussed waiting for another participant, sharing the meeting link, and recording the session.

## 2. Prototype Overview

A prototype was presented to demonstrate user-friendly streaming messages.

The implementation was designed with the following principle:

> Existing response fields and behavior should remain unchanged to avoid breaking the current frontend.

Instead of modifying older response formats, a separate or additional structure was introduced for friendly progress messages.

The prototype showed a sequence similar to:

1. Understanding the user’s question
2. Identifying the language of the question
3. Running retrieval from the knowledge base
4. Completing retrieval
5. Returning or summarizing the final answer

## 3. Suppression of Internal Transitions

The presenter explained that internal transitions between the orchestrator, agents, and tools had been suppressed.

Examples of internal messages that should not be shown to users include:

* Transferring control from one agent to another
* Starting an internal tool call
* Switching from the orchestrator to the RAG component
* Internal execution or routing events

The intention is to expose only meaningful progress updates while hiding implementation details.

## 4. Tool Naming

Tool names are currently converted into readable labels by replacing underscores with spaces.

For example:

`retrieval_from_knowledge_base`

becomes:

`Retrieval from knowledge base`

The team noted that more intentional, logical, or product-friendly labels may be needed rather than relying only on automatic formatting.

## 5. Event Type Discussion

A suggestion was made to use or customize the existing event type instead of adding only a separate friendly message field.

The response was that event types originate from the Agent Development Kit and may not be freely customizable in all cases.

The prototype therefore introduced a separate structure to preserve backward compatibility and avoid changing existing event behavior.

A possible alternative discussed was:

* Retain the original event type
* Add a customizable event-type label or display text
* Generate a separate progress event when necessary

## 6. Final Response Contamination

A concern was raised that friendly progress information should not appear in the final response served to the user.

The team agreed that progress events should be temporary and must remain separate from the final answer content.

## 7. Extensible Message Structure

The presenter explained that the friendly-message design was intentionally created as a structured object rather than a single string.

Potential future message categories include:

* Informational progress
* Completion or terminal messages
* Human feedback required
* User confirmation required
* Citations or sources
* Warnings
* Errors

The goal is to support additional UI behaviors later, such as:

* Toast messages
* Inline progress indicators
* Approval prompts
* Citation panels
* Human-in-the-loop interactions

---

# Key Decisions and Agreements

## Confirmed or Strongly Agreed

1. Existing response behavior should remain backward compatible.

2. Internal agent-to-agent and orchestrator-to-tool transitions should generally be hidden from end users.

3. User-facing progress messages should use friendly, readable language.

4. Progress messages should not be included in the final answer content.

5. The message format should be extensible enough to support more than informational status updates.

6. Early feedback is needed before implementing the same pattern across all agents and tools.

## Not Yet Finalized

1. Whether friendly messages should be represented through:

   * Existing event types
   * A new event type
   * Event display text
   * A separate `friendly_message` object

2. Whether progress events should be generated by:

   * The backend
   * The orchestrator
   * Individual tools
   * The frontend based on raw events

3. The exact schema for informational, terminal, citation, and human-feedback messages.

4. How the frontend will distinguish transient messages from persistent response content.

5. Whether generic tool names are sufficient or require a centrally managed display-name mapping.

---

# Action Items

| Action                                                                              | Suggested Owner               | Priority |
| ----------------------------------------------------------------------------------- | ----------------------------- | -------- |
| Define the final schema for friendly and progress messages                          | Backend and frontend leads    | High     |
| Confirm how the frontend currently consumes `event_type`                            | Frontend team                 | High     |
| Ensure progress events are excluded from the final response payload                 | Backend team                  | High     |
| Define which internal events must be suppressed                                     | Agent platform team           | High     |
| Create a standard mapping between tool names and user-friendly labels               | Product and engineering       | Medium   |
| Decide whether friendly messages are separate events or metadata on existing events | Architecture team             | High     |
| Add terminal, warning, error, citation, and human-input message categories          | Backend team                  | Medium   |
| Test compatibility with clients that do not recognize the new field                 | QA and backend                | High     |
| Validate behavior with the actual RAG and NLP integrations                          | RAG/NLP team                  | High     |
| Review the prototype before scaling the change to all tools                         | Technical leads               | High     |
| Document event lifecycle and frontend rendering rules                               | Architecture or platform team | Medium   |
| Add automated tests to prevent progress messages from leaking into final answers    | QA/backend                    | High     |

---

# Technical Feedback

## What Is Working Well

### 1. Backward Compatibility

Adding an optional field or structured object is safer than changing existing response fields.

Older clients can ignore the new field, while newer clients can use it to improve the experience.

### 2. Suppression of Internal Orchestration

Hiding messages such as “transferring control” or raw tool-call events is the correct user-experience direction.

Most users care about what the system is doing at a meaningful level, not how agents are internally routed.

### 3. Structured Message Object

Using an object instead of a simple string creates room for future capabilities.

A structured message can support:

* Type
* Display text
* Severity
* Lifecycle
* Tool identity
* Citations
* Required user action
* Completion status

### 4. Early Prototype Review

Getting architectural feedback before implementing the same behavior across many tools is appropriate and reduces rework.

## Risks and Concerns

### 1. Duplication Between Event Type and Friendly Message

If both fields attempt to describe the same event, they may become inconsistent.

For example:

* `event_type`: `tool_call_started`
* `friendly_message`: “Analyzing your question”
* Actual tool: knowledge retrieval

The schema needs to clearly define the responsibility of each field.

### 2. Progress Messages in Final Output

If the final response is created by aggregating streamed text, friendly messages may accidentally be included.

Progress events should therefore not use the same content channel as final answer tokens.

### 3. Too Many Progress Messages

Displaying every tool event can make the interface noisy and slower-feeling.

A long workflow might produce dozens of events even when the actual execution is fast.

### 4. Automatically Generated Tool Labels

Replacing underscores with spaces is useful as a fallback, but may produce awkward or overly technical language.

Examples:

* `run_rag_query_v2`
* `fetch_customer_360`
* `invoke_sub_agent`

These should not be exposed directly to users.

### 5. Event Ordering and Timing

Parallel tools, retries, and nested agents may cause status events to arrive out of sequence.

The schema should include identifiers that allow the frontend to correctly update, replace, or close a progress item.

### 6. Localization

Hard-coded English friendly messages may conflict with the detected user language.

Language detection occurs early in the workflow, so subsequent progress messages should ideally follow the user’s language.

### 7. Ambiguous Completion State

A message such as “Retrieval completed” does not indicate whether retrieval:

* Succeeded
* Returned no results
* Partially succeeded
* Timed out
* Fell back to another source

The status model should distinguish these cases.

---

# Recommendations

## 1. Separate Protocol Events From Display Text

Keep machine-readable events separate from user-facing language.

Recommended structure:

```json
{
  "event_type": "tool_execution",
  "event_phase": "started",
  "event_id": "evt_123",
  "parent_event_id": "evt_100",
  "tool": {
    "name": "retrieve_knowledge",
    "display_name": "Searching the knowledge base"
  },
  "ui": {
    "message": "Searching for relevant information…",
    "message_type": "progress",
    "visibility": "transient"
  }
}
```

The event type should be stable and machine-readable. The friendly message can change without affecting integrations.

## 2. Introduce an Explicit Visibility or Persistence Field

Do not rely on the frontend to infer whether a message belongs in the final response.

Recommended values:

* `transient`
* `persistent`
* `final`
* `internal`

This would allow the frontend to safely ignore internal events and prevent transient messages from appearing in conversation history.

## 3. Define a Small, Controlled Message Taxonomy

Avoid creating too many overlapping message types.

A practical initial taxonomy would be:

* `progress`
* `success`
* `warning`
* `error`
* `action_required`
* `citation`
* `final`

The execution phase can remain separate:

* `started`
* `in_progress`
* `completed`
* `failed`
* `cancelled`

## 4. Use Event IDs and Update Existing UI Elements

Instead of adding a new message for every transition, reuse the same UI element.

Example:

* “Searching the knowledge base…”
* Updated to “Found relevant information”
* Then removed or marked complete

This reduces clutter and gives the frontend a cleaner progress experience.

## 5. Do Not Expose Raw Tool Names by Default

Create a central display-name registry.

Example:

```json
{
  "retrieve_from_kb": {
    "started": "Searching the knowledge base…",
    "completed": "Relevant information found"
  }
}
```

Automatic underscore replacement should only be a fallback for development environments.

## 6. Apply Progress Message Throttling

Very fast operations should not always show a progress message.

For example:

* Do not display an event if the operation completes in under 300–500 milliseconds.
* Show a status only if processing lasts long enough for the message to be useful.
* Avoid repeated updates more frequently than a defined interval.

This prevents flickering and unnecessary noise.

## 7. Distinguish User-Relevant Work From Internal Work

Recommended user-visible stages:

1. Understanding the request
2. Searching relevant information
3. Analyzing the results
4. Preparing the answer
5. Waiting for user input, when applicable

Internal routing, retries, agent handoffs, token processing, and implementation-specific tool events should remain hidden unless needed for debugging.

## 8. Preserve Language Consistency

Once the user’s language is identified, progress messages should use that language.

Until language detection is complete, the system may:

* Show no initial message
* Use a neutral loading indicator
* Use the application’s default language

## 9. Treat Citations Separately

Citations should not be modeled only as friendly informational messages.

They should have structured fields such as:

```json
{
  "message_type": "citation",
  "source_id": "source_123",
  "title": "Internal Knowledge Article",
  "url": "...",
  "supports": ["response_segment_2"]
}
```

This allows the frontend to render sources accurately and associate them with specific claims.

## 10. Add a Formal Human-in-the-Loop Event

User confirmation should be a dedicated event type rather than an informational toast.

Recommended fields include:

* Prompt
* Available actions
* Whether input is mandatory
* Expiration or timeout
* Whether execution is paused
* Correlation ID for resuming execution

## 11. Define the Streaming Contract Before Full Integration

The team should document:

* Which component emits each event
* Which events are public
* Which events are internal
* How events are ordered
* How retries are represented
* How final content is distinguished
* How old clients behave
* How errors and cancellations are handled

This should be agreed before integrating all RAG and NLP components.

## 12. Recommended Implementation Direction

The strongest option is:

* Keep existing ADK event types unchanged.
* Add a separate optional `ui` or `presentation` object.
* Mark each event as internal, transient, persistent, or final.
* Use stable IDs to update existing progress items.
* Keep final answer tokens in a separate response event.
* Maintain a controlled mapping of backend tools to user-friendly labels.
* Suppress internal orchestration events by default.
* Allow debug clients to opt in to raw events.

This approach provides backward compatibility while keeping the protocol clean and extensible.

---

# Suggested Acceptance Criteria

The feature should be considered ready when:

1. Existing clients operate without modification.
2. New clients can display friendly progress messages.
3. Internal orchestration details are not visible to standard users.
4. Friendly progress text never appears in the final answer.
5. Messages are displayed in the user’s language where supported.
6. Parallel and repeated tools do not create confusing duplicate statuses.
7. Errors, cancellations, and user-input requests have distinct behavior.
8. Tool labels are understandable and not generated solely from internal names.
9. Citations remain structured and traceable.
10. Automated tests verify ordering, compatibility, suppression, and final-response separation.

---

# Proposed Final Decision Statement

The team should retain the existing event structure for backward compatibility and introduce a separate optional presentation layer for user-facing progress information. This layer should clearly identify message type, execution phase, visibility, persistence, and event correlation. Internal agent transitions should remain suppressed, and transient messages must never be merged into the final response.

## Latency vs. Result Quality: Recommendation

The concern is valid: placing a full LLM-driven pre-orchestrator in front of the existing orchestrator can add noticeable latency. However, enrichment, validation, clarification, and human-in-the-loop processing will add some cost regardless of where they are implemented.

The key question is not whether there will be additional latency, but **which requests genuinely need that additional processing**.

### Recommended decision

Keep the new pre-orchestration layer, but **do not run it as a fully autonomous LLM agent for every request**.

Use it as a lightweight, code-controlled preprocessing pipeline:

```text
User request
   ↓
Fast deterministic checks
   ├─ Language
   ├─ Existing metadata
   ├─ Time/date extraction
   ├─ Location availability
   └─ Basic validation
   ↓
Conditional LLM enrichment only when ambiguous
   ↓
Existing orchestrator
   ↓
Existing RAG/tool workflow
```

The existing orchestrator should remain unchanged and continue handling classification, RAG routing, and execution.

## Why this is the best compromise

### Result-quality benefits

The preprocessing layer can:

* Reject unsupported languages before expensive downstream processing.
* Detect missing mandatory information.
* Normalize time, location, and other entities.
* Request clarification before sending an incomplete request downstream.
* Add transparency and progress messages.
* Preserve the proven behavior of the current orchestrator.
* Provide a clean rollback boundary if the new functionality causes problems.

### Latency risks

A separate LLM agent may introduce:

* Another model invocation.
* Serial execution before the main orchestrator starts.
* Additional prompt construction and context processing.
* Tool-selection overhead.
* Increased tail latency when the model retries or calls multiple tools.
* Duplicate reasoning between the pre-orchestrator and main orchestrator.

The suggested two-to-three-second increase is possible, but it should be measured rather than treated as a fixed number. The actual impact will depend on the model, prompt size, tool calls, infrastructure, and whether operations run sequentially or concurrently.

## Separate hard gates from soft enrichment

Not every enrichment item should block the orchestrator.

### Hard gates

Complete these before forwarding:

* Is the language supported?
* Is the request safe and structurally valid?
* Is mandatory information missing?
* Does the workflow require explicit confirmation?
* Is there a reason execution must terminate?

### Soft enrichment

These should not automatically delay every request:

* Optional location enhancement
* Optional timezone inference
* Rewriting an already clear request
* Generating friendly status text
* Extracting metadata that the existing orchestrator does not require
* Adding nonessential context

Soft enrichment can run conditionally, in parallel, or be skipped when confidence is already high.

## Recommended implementation model

### 1. Use tools as a fixed pipeline, not agent-selected actions

The current description includes predictable steps such as:

1. Detect language.
2. Detect time.
3. Detect location.
4. Validate metadata.
5. Decide whether to continue or terminate.

These are known steps. They do not require an agent to decide what to do every time.

Implement them as an explicit asynchronous pipeline:

```python
async def preprocess_request(request):
    language_result, time_result, location_result = await asyncio.gather(
        detect_language(request),
        detect_time(request),
        detect_location(request),
    )

    return process_results(
        request=request,
        language=language_result,
        time=time_result,
        location=location_result,
    )
```

This avoids an extra LLM planning cycle and allows independent skills to run concurrently.

### 2. Use an LLM only when deterministic processing is insufficient

For example:

```text
Clear language → no LLM
Clear ISO date → no LLM
Known user timezone → no LLM
Ambiguous “next Friday evening” → lightweight LLM or clarification
Ambiguous location reference → clarification or contextual model call
```

This changes the latency model from:

```text
Every request pays the LLM cost
```

to:

```text
Only ambiguous requests pay the LLM cost
```

### 3. Keep skill execution and processing together

The proposed pattern of keeping each skill and its processing logic close together is reasonable.

A useful structure would be:

```text
skills/
  language/
    tool.py
    processor.py
    models.py
    test_skill.py

  time/
    tool.py
    processor.py
    models.py
    test_skill.py
```

For very small skills, `tool.py` and `processor.py` can remain in one file. Split them only when the logic becomes difficult to maintain.

Each skill should return a structured result rather than only a message:

```python
class SkillResult:
    status: str
    value: object | None
    confidence: float | None
    message: str | None
    action: str  # continue, default, clarify, terminate
    metadata: dict
```

The user-facing message should be presentation metadata, not the primary result.

## Processing outcomes

Standardize the processor outcome across all skills:

```text
CONTINUE
DEFAULT
CLARIFY
TERMINATE
DEGRADED_CONTINUE
```

### Examples

**Supported language**

```text
Result: CONTINUE
Metadata: language=en
User message: optional transient progress message
```

**Unsupported language**

```text
Result: TERMINATE
Final message: supported-language guidance
Skip summarization: true
```

**Time not provided but optional**

```text
Result: DEFAULT
Metadata: timezone=user_default
Continue to orchestrator
```

**Time is required but ambiguous**

```text
Result: CLARIFY
Execution paused until the user responds
```

**Location service unavailable**

```text
Result: DEGRADED_CONTINUE
Forward request without location
Do not terminate unless location is mandatory
```

## Avoid serial skill execution

A flow like this will accumulate latency:

```text
Detect language
   ↓
Detect time
   ↓
Detect location
   ↓
Enrich request
   ↓
Call orchestrator
```

Instead, run independent checks concurrently:

```text
             ┌─ Detect language ─┐
User input ──┼─ Detect time ─────┼─ Aggregate ─ Orchestrator
             └─ Detect location ─┘
```

Only dependencies should be sequential. For example, timezone conversion may need location, but language detection does not need to wait for location detection.

## Use fast paths

Define a fast path for normal requests.

Example:

```text
Request is clear
Language is supported
No mandatory metadata is missing
No clarification is required
```

In this case, the pre-layer should immediately forward the request to the orchestrator with minimal processing.

A slow path should activate only when:

* Confidence is below a threshold.
* Required metadata is absent.
* The request contains ambiguous temporal or location language.
* User confirmation is required.
* A policy requires termination or escalation.

## Streaming recommendations

Streaming can improve perceived responsiveness, but it does not reduce actual execution time.

Send an immediate lightweight event such as:

```text
Understanding your request…
```

Then update the same progress item rather than appending several messages:

```text
Understanding your request…
        ↓
Checking the required details…
        ↓
Searching for relevant information…
```

Do not generate these messages through an LLM. They should come from predefined templates associated with processing stages.

Also, do not delay orchestration just to emit a progress message.

## Fail-open versus fail-closed

The processing layer should not become a new single point of failure.

### Fail closed

Use termination only when proceeding would be invalid:

* Unsupported language with no translation capability
* Mandatory consent missing
* Required data is definitely invalid
* A policy prohibits execution

### Fail open

Continue with the original request when optional enrichment fails:

* Location could not be detected
* Timezone is unavailable
* Friendly message generation fails
* Optional metadata extraction times out
* A noncritical enrichment tool is unavailable

A suitable rule is:

> Enrichment failure should not become request failure unless the enrichment is mandatory for a correct or safe result.

## Timeout and budget controls

Give the preprocessing layer its own strict latency budget.

For example:

```text
Deterministic checks: strict sub-second target
Optional enrichment: short timeout
Clarification model call: only when required
Overall preprocessing: capped before fallback
```

When an optional skill exceeds its budget:

```text
Cancel or ignore it
Record the timeout
Forward the original request
```

Do not let the pre-orchestrator wait indefinitely for optional metadata.

## Caching opportunities

Cache results that are stable across requests:

* User’s preferred language
* Supported-language status
* User timezone
* Default location, with appropriate privacy controls
* Tenant configuration
* Feature flags
* Tool-display labels

Do not repeatedly invoke a model to detect information already available in session or user metadata.

## Query enrichment

Avoid replacing the original user query with an LLM-rewritten query. Pass both:

```json
{
  "original_query": "Can I schedule it next Friday evening?",
  "normalized_query": "Schedule the event on 2026-07-24 during the evening.",
  "metadata": {
    "language": "en",
    "timezone": "America/New_York",
    "date_resolution_confidence": 0.93
  }
}
```

This protects against enrichment errors and allows the downstream orchestrator to inspect the user’s exact wording.

## Rollback strategy

The new layer should be independently switchable.

Recommended controls:

* Feature flag for all preprocessing
* Per-skill feature flags
* Shadow mode
* Fail-open mode
* Ability to route directly to the original orchestrator
* Versioned schemas
* Logging of original and enriched requests

In shadow mode, the pre-layer processes the request and records what it would have done, but the original request still goes directly to the orchestrator. This allows latency and correctness comparison before making it blocking.

## Metrics required for the decision

Do not assess the design only by average latency. Track:

| Metric                                 | Purpose                              |
| -------------------------------------- | ------------------------------------ |
| Preprocessing p50 latency              | Typical additional cost              |
| Preprocessing p95/p99 latency          | Slow-user experience                 |
| End-to-end time to first visible event | Perceived responsiveness             |
| End-to-end time to final answer        | Actual user wait                     |
| Clarification rate                     | How often the layer interrupts users |
| Prevented downstream failures          | Value created by validation          |
| Unsupported-language termination rate  | Whether the gate is useful           |
| Metadata extraction accuracy           | Enrichment quality                   |
| Orchestrator success rate              | Downstream impact                    |
| User correction rate                   | Incorrect inference indicator        |
| Fail-open rate                         | Reliability of enrichment tools      |
| Token and model cost per request       | Operational impact                   |

## Evaluate result quality, not enrichment volume

Success should not mean “more metadata was extracted.”

It should mean:

* Fewer failed downstream executions
* Fewer incorrect assumptions
* Better answers
* Fewer unnecessary clarification turns
* No regression in existing orchestrator performance
* Acceptable latency increase
* Better user trust and transparency

## Phase-one recommendation

For the initial version:

1. Keep the existing orchestrator unchanged.
2. Retain the new wrapper or pre-orchestration boundary.
3. Implement language validation as a code-controlled skill.
4. Add time and location skills only where they are required.
5. Run independent skills concurrently.
6. Use deterministic parsing before invoking an LLM.
7. Terminate only for genuine hard failures.
8. Fail open for optional enrichment.
9. Use predefined streaming messages.
10. Put the whole layer behind a feature flag.
11. Compare direct routing, shadow mode, and enabled mode.
12. Establish a maximum latency budget before broader rollout.

## Longer-term recommendation

The pre-orchestrator can remain an agent conceptually, but its runtime behavior should resemble a **bounded workflow**, not an open-ended autonomous agent.

```text
Bounded workflow:
Known skills
Known order or dependency graph
Strict timeouts
Structured outcomes
Limited model usage
Explicit termination rules
Predictable handoff
```

This provides the architectural separation and rollback strategy the team wants without imposing a full additional agent-reasoning cycle on every request.

## Proposed decision statement

> We will preserve the existing orchestrator and introduce a lightweight preprocessing layer for validation, metadata enrichment, clarification, and user-facing progress. The layer will use deterministic and parallel skill execution by default, invoke an LLM only for ambiguous cases, and fail open when optional enrichment is unavailable. Hard validation must complete before orchestration, while soft enrichment must not unnecessarily block execution. The feature will be measured using end-to-end latency, downstream success, clarification frequency, and enrichment accuracy before full rollout.

The core trade-off is therefore not **pre-orchestrator versus no pre-orchestrator**. It is **a full LLM agent on every request versus a selective, bounded preprocessing workflow**. The second option provides most of the expected quality and risk-reduction benefits with substantially better latency predictability.

## Follow-up questions

1. **Execution model:** Which tasks are independent enough to run in parallel, and which have strict data or ordering dependencies requiring sequential execution?
2. **Decision logic:** What exact criteria will the pre-orchestration classifier use to route requests to the “pause,” “right,” or “US” agent?
3. **Terminology:** Are “right agent” and “US agent” the intended names, or should these be clarified?
4. **Classifier output:** Will routing use fixed labels, confidence scores, rules, or a combination?
5. **Low-confidence handling:** What should happen when the classifier is uncertain—fallback agent, retry, human review, or clarification?
6. **Prompt reuse:** Will the existing production orchestration prompt be reused exactly, or adapted to work in the pre-orchestration layer?
7. **Context transfer:** What request context, history, and metadata must be passed from the classifier to the selected agent?
8. **Smart operation layer:** What conditions identify a smart operation, and why must this decision happen above the current orchestration layer?
9. **Failure handling:** How will partial failures, timeouts, retries, and duplicate executions be managed during parallel processing?
10. **MCP hosting:** Does the MCP server need to be public, private, region-specific, autoscaled, or accessible only within the internal network?
11. **State management:** Will the orchestration workflow be stateless, or must it preserve execution state across agents and tools?
12. **Success metrics:** How will the new design be evaluated—latency, routing accuracy, cost, completion rate, or production errors?

## Recommendations

### 1. Define the workflow as a dependency graph

Represent each operation as a node with explicit dependencies:

* Run independent nodes in parallel.
* Run dependent nodes sequentially.
* Merge results only after all required upstream nodes complete.

This is safer than manually hard-coding parallel and sequential paths.

### 2. Keep the pre-orchestration classifier lightweight

The classifier should only determine:

* Request type
* Target agent
* Execution mode
* Whether smart-operation handling is required
* Confidence level

Avoid putting full task execution logic into this layer.

### 3. Preserve the existing prompt, but separate responsibilities

Since the production prompt is working well, reuse its core instructions. However, split the architecture conceptually:

* **Pre-orchestration:** classification and routing
* **Agent orchestration:** planning and execution
* **Post-orchestration:** validation, result merging, and error handling

This reduces regression risk while making the workflow easier to maintain.

### 4. Use structured routing output

The classifier should return a validated object similar to:

```json
{
  "target_agent": "pause_agent",
  "execution_mode": "parallel",
  "is_smart_operation": false,
  "confidence": 0.94,
  "reason": "Request requires independent tool calls"
}
```

Use a strict schema and reject or repair invalid outputs.

### 5. Add a fallback route

When confidence is below an agreed threshold, route to the existing orchestration flow rather than making a potentially incorrect specialized decision. This allows gradual rollout without disrupting production.

### 6. Introduce parallel execution selectively

Start with low-risk operations that:

* Do not modify shared state
* Do not depend on one another’s output
* Are idempotent or safe to retry
* Have clearly defined merge logic

Keep write operations and state-dependent operations sequential initially.

### 7. Treat smart operations as a policy layer

Instead of creating another full agent, add a policy or capability-detection layer above orchestration. It should determine whether the request needs the smart-operation flow, then pass control to the existing orchestration system.

### 8. Add observability before rollout

Log:

* Classifier decision
* Confidence
* Selected agent
* Parallel and sequential execution paths
* Tool latency
* Retry count
* Final outcome
* Fallback usage

This will help identify misrouting and performance bottlenecks.

### 9. Evaluate MCP hosting through a small proof of concept

Compare at least:

* In-process or local deployment
* Private internal service
* Containerized service on the existing platform
* Managed serverless or Kubernetes deployment

Evaluate authentication, network access, scaling, secrets, latency, monitoring, and failure recovery before selecting the hosting model.

### 10. Roll out incrementally

A practical sequence would be:

1. Document current orchestration behavior.
2. Add the classifier in shadow mode without affecting routing.
3. Compare classifier decisions with actual production outcomes.
4. Enable routing for a small percentage of traffic.
5. Add parallel execution for low-risk tasks.
6. Introduce smart-operation detection.
7. Finalize the MCP hosting model after load and security testing.

The discussion reinforces the recommendation, with one important clarification: **do not describe or implement the preprocessing layer as a full agent unless it performs genuinely agentic work.**

## Concise Summary

The team agrees that preprocessing capabilities are necessary, including:

* Language and entity detection
* User-preference loading
* Location and data-scope validation
* Metadata enrichment
* Clarification and human-feedback handling
* Prevention of invalid or unsafe downstream requests

The disagreement is primarily about implementation and terminology.

A full pre-orchestrator agent was challenged because most of these tasks are deterministic preprocessing rather than specialized reasoning. Making it an agent would add unnecessary routing, model calls, latency, complexity, and behavior that differs from the system’s other agents.

The preferred approach is therefore:

1. Use direct application logic for simple deterministic operations.
2. Use Python tools or services for structured extraction and validation.
3. Use an LLM only when interpretation is genuinely ambiguous.
4. Keep the existing orchestrator as the sole routing agent.
5. Run independent preprocessing checks in parallel where possible.
6. Treat preprocessing as a bounded request-readiness layer, not an autonomous agent.

## Does the Previous Recommendation Still Stand?

**Yes. It still stands and is more strongly supported by this discussion.**

The recommendation was never dependent on adding a second full agent. It proposed a lightweight preprocessing layer that uses deterministic, parallel execution and invokes an LLM only when needed.

That directly addresses the concerns raised:

* It preserves the existing orchestrator.
* It avoids an additional agent-planning cycle.
* It reduces latency and operational complexity.
* It keeps essential validation before orchestration.
* It supports a clean rollback boundary.
* It allows specialized agent behavior to be introduced later only for tasks that truly require it.
* It prevents simple operations, such as loading preferences or validating scope, from unnecessarily consuming LLM resources.

The main adjustment should be semantic and architectural:

> Replace “pre-orchestrator agent” with “preprocessing layer,” “request-readiness layer,” or “pre-orchestration pipeline.”

## Revised Decision Statement

We will preserve the existing orchestrator as the system’s primary routing agent and introduce a lightweight request-readiness layer for validation, metadata enrichment, clarification, privacy and scope checks, and user-facing progress.

This layer will use direct application logic and deterministic tools by default, execute independent checks in parallel, and invoke an LLM only when the request contains genuine ambiguity that cannot be resolved reliably through code.

Hard validation must complete before orchestration. Optional enrichment must not unnecessarily block execution and should fail open when unavailable. The preprocessing layer will not perform routing or duplicate capabilities already handled by the orchestrator.

The design will be evaluated using end-to-end latency, time to first response, downstream success rate, clarification frequency, enrichment accuracy, failure-prevention rate, and operational cost before full rollout.

The core trade-off is not pre-orchestrator versus no pre-orchestrator. It is:

**a full generative agent on every request versus a selective, bounded preprocessing workflow.**

The selective workflow remains the recommended option because it provides the required validation and enrichment benefits without introducing unnecessary agent latency and complexity.
