---
title: Modes
date: 2023-12-15
---

NB: **modes could help resolve the architecture tension**, provided each mode represents a genuinely different execution path, not just a different screen.

The two experiences would represent different contracts:

* **Standard mode:** answer normal questions through the existing RAG or conversational pipeline.
* **Find Menu mode:** search menu data using menu-specific tools, filters, and structured results.

## How modes help the architecture

An explicit mode can determine the broad route before the orchestrator is called.

```text
User request
    ↓
Global deterministic checks
    ↓
Was a mode explicitly selected?
    ├─ Standard
    │      → Existing RAG or general-answer pipeline
    │
    ├─ Analyze business data
    │      → Analytics readiness checks
    │      → Location/time-range clarification
    │      → Analytics agent
    │
    ├─ Find Menu
    │      → Menu-specific readiness checks
    │      → Menu search tools
    │      → Structured menu response
    │
    └─ Auto
           → Existing orchestrator selects the route
```

This does not give the orchestrator another job. When the user explicitly selects a mode, the application can bypass general routing and invoke the relevant pipeline directly.

## Recommended modes

### Standard

For normal questions and straightforward RAG retrieval.

* Fast response
* Existing knowledge or document-retrieval flow
* No location or menu checks unless required by the selected use case
* Minimal additional processing

### Analyze business data

For questions involving sales, performance, locations, time ranges, metrics, or operational data.

* Runs analytics-specific readiness checks
* Automatically uses the location when the user has exactly one
* Clarifies only when multiple valid locations remain
* Resolves time ranges and other required business parameters
* Uses analytics tools and agents

### Find Menu

For requests such as:

* “Show me the dinner menu.”
* “Find vegetarian items.”
* “Which menu items contain chicken?”
* “What desserts are available at this location?”
* “Show me items under $15.”

This mode can:

* Search menu items directly
* Apply category, dietary, ingredient, price, and availability filters
* Use location only when menu availability differs by location
* Return structured menu cards or grouped results
* Avoid running analytics or general RAG logic unnecessarily

## Menu-specific readiness

The menu pipeline should run only the checks required for the request.

```text
Find Menu selected
    ↓
Is location required for menu availability?
    ├─ No → Search the shared menu
    └─ Yes
          ↓
      One authorized location?
          ├─ Yes → Auto-resolve
          └─ Multiple locations
                 ↓
          Was one clearly specified?
              ├─ Yes → Continue
              └─ No → Ask which location
```

Other readiness fields may include:

* Menu category
* Meal period
* Dietary preference
* Ingredient restriction
* Price range
* Availability date or time

These should not all be mandatory. The system should only clarify when a missing value is genuinely required to produce a valid result.

## Do not force a mode selection every time

The default can remain **Auto** or **Standard**.

The system may suggest another mode when appropriate:

> “This looks like a menu search. Open it in Find Menu?”

or:

> “This question requires business reporting data. Continue in Analyze mode?”

The user should not be interrupted for every request.

## Modes do not replace readiness checks

Modes define **which readiness rules apply**.

```text
Standard
- Global checks only
- Usually no location readiness

Analyze business data
- Location
- Time range
- Metric
- Business access

Find Menu
- Location when menu-specific
- Category or meal period when essential
- Dietary or ingredient filters when supplied
```

This solves the latency concern because a normal RAG question does not run location, sales, or menu-specific checks.

## Comparison

| Standard                       | Analyze business data          | Find Menu                                 |
| ------------------------------ | ------------------------------ | ----------------------------------------- |
| General questions              | Sales and operational analysis | Menu discovery                            |
| RAG or conversational answers  | Analytics tools and agents     | Menu-search tools                         |
| Low latency                    | May require clarification      | Structured filtering                      |
| Usually no location dependency | Often location-dependent       | Location-dependent only when menus differ |
| Narrative response             | Tables, metrics, and insights  | Menu items, categories, and availability  |

## Recommended routing policy

```text
Explicit mode selected
    → Invoke that pipeline directly

No mode selected
    → Existing orchestrator selects the route
```

A concise way to present it to the team is:

> “Modes let the user declare the type of task they want. Standard handles general questions, Analyze handles business data, and Find Menu handles menu discovery. When a mode is explicitly selected, we can bypass general orchestration and run only the readiness checks and tools relevant to that pipeline. The existing orchestrator remains the fallback when no mode is selected.”
