---
title: Menu Lot
date: 2025-03-10
---

## Thoughts

The overall direction is sound: keep the existing production agents intact, introduce a new pre-orchestrator, and roll out entity enrichment incrementally. Making each detector an independently testable tool is especially strong because language, time, location, menu items, modifiers, and quantities will require different implementation strategies.

The embedding approach also appears practical. Using a dedicated `embedding_text` field with filterable metadata such as account ID, location ID, entity type, and status should reduce irrelevant semantic matches and prevent unnecessary downstream RAG and SQL execution.

The main concern is that the architecture currently seems more mature than the entity contract and decision policy. The team is building detectors before fully defining:

* Which entities are supported
* How each entity is grounded
* How conflicts are resolved
* When the system clarifies, defaults, rejects, or continues
* What gets stored as preference versus conversation history

Those decisions will affect every detector and the orchestrator state schema.

## Verdict

**Proceed with the architecture, but do not scale implementation across all entities until the common entity model and orchestration rules are finalized.**

The proposed architecture is a good foundation for phased rollout. However, entity detection should not become a collection of independently behaving tools with inconsistent outputs and confidence logic. The orchestrator needs a deterministic contract governing all detectors.

The strongest parts are:

* Phased rollout without replacing existing production behavior
* Independent and parallel development of entity detectors
* Metadata-filtered semantic search
* Early short-circuiting for unsupported languages or inaccessible resources
* Collecting all required clarifications in one user interaction
* Preserving enrichment results in shared state
* Separating the new streaming implementation from the legacy helper

The greatest risks are:

* Undefined overlapping-entity behavior
* Stale embeddings after menu changes
* Too many false short-circuits
* Persisting temporary choices as long-term preferences
* Parallel tools producing contradictory state
* Streaming internal detector output directly to users
* Multiple BigQuery tables with overlapping responsibilities

## Recommendations

### 1. Finalize a canonical entity contract first

Every detector should return the same structure, for example:

```json
{
  "entity_type": "menu_item",
  "raw_text": "large coke",
  "normalized_value": "Coca-Cola",
  "canonical_id": "menu_item_123",
  "confidence": 0.91,
  "grounding_status": "grounded",
  "candidates": [],
  "source": "semantic_search",
  "scope": {
    "account_id": "account_1",
    "location_id": "location_4"
  },
  "requires_clarification": false
}
```

Without this, integration effort will increase with every new detector.

### 2. Separate detection, grounding, and resolution

These are three different responsibilities:

1. **Detection:** “pizza” appears to be a menu-related entity.
2. **Grounding:** It matches four menu items available at this location.
3. **Resolution:** Select one, apply a safe default, or ask the user.

Do not combine all three inside every detector. Detectors should identify candidates; a centralized resolution policy should determine the next action.

### 3. Create an explicit ambiguity policy

Define thresholds centrally:

* High confidence and one grounded result: continue
* Multiple strong results: clarify
* Low confidence: leave unresolved or clarify
* Unsupported or unauthorized entity: short-circuit
* Optional entity missing: continue with default
* Required entity missing: clarify

This will address the overlapping-entity issue without forcing every tool to invent its own behavior.

### 4. Run detectors in parallel, but resolve centrally

Parallel execution is appropriate for latency, but all results should pass through a single reconciliation step before classification.

For example, “large Washington pizza” could produce:

* `large` → modifier or quantity
* `Washington` → location or menu-item name
* `pizza` → menu category or menu item

The reconciliation layer should inspect all candidates together rather than accepting detector outputs independently.

### 5. Avoid streaming raw enrichment-tool output

Entity detection, grounding, and authorization checks are internal processing steps. Streaming every tool response could expose noisy or confusing information.

Stream only user-relevant status or final clarification messages, such as:

* “I found several sandwich options.”
* “I cannot access the requested location.”
* “Please confirm the menu item and location.”

Internal embeddings, confidence values, metadata filters, and intermediate candidates should remain internal unless transparency requirements explicitly demand them.

### 6. Treat short-circuiting carefully

Short-circuiting is valuable, but only for definitive conditions:

* Unsupported language
* Invalid authentication
* No accessible locations
* Explicitly unavailable entity
* Missing required context that cannot be defaulted

Do not short-circuit solely because semantic search returns no menu match. Misspellings, synonyms, stale embeddings, and new menu items can create false negatives. In those situations, clarification or fallback search may be safer.

### 7. Define embedding synchronization before production

The menu embedding pipeline should handle:

* New items
* Renamed items
* Deleted or disabled items
* Location availability changes
* Modifier changes
* Account-level overrides

Use a stable canonical ID and version or update timestamp. Prefer event-driven updates where possible, with a periodic reconciliation job as backup.

Deleted items should be removed or marked inactive so they cannot continue appearing in results.

### 8. Distinguish three types of memory

The discussion appears to mix different concepts:

**Profile preferences**
Stable information such as preferred location, language, or commonly selected item.

**Session or conversational memory**
Information needed only during the current conversation.

**Interaction and audit logs**
What the user asked, what the system detected, what was suggested, and what the user selected.

These should not automatically feed one another. A user selecting a sandwich once should not become a permanent preference. Promotion to long-term preference should require repetition, an explicit user statement, or a confidence policy.

### 9. Keep the two BigQuery table purposes distinct

A reasonable separation would be:

**Interaction table**

* Original query
* Summarized intent
* Detected entities
* Candidate entities
* Clarification shown
* User selection
* Final outcome
* Timestamps and trace ID

**User preference table**

* Preference type
* Canonical value
* Confidence
* Evidence count
* Last confirmed time
* Source
* Expiry or decay policy

Avoid storing the full streaming response as the primary memory representation. Store concise structured summaries and retain raw traces only for debugging or audit purposes.

### 10. Batch clarification into one turn

The proposed “single-go” clarification is the right approach. The orchestrator should collect all unresolved required fields first, then ask one consolidated question.

For example:

> I found two possible locations and four sandwich items. Which location and sandwich should I use?

The system should not ask sequential questions unless the answer to the first question materially changes the remaining options.

### 11. Keep classification after enrichment, but impose a latency budget

Moving enrichment before classification makes sense because the classifier can use grounded entities instead of interpreting everything itself. However, every detector should have a strict timeout.

Optional enrichment should not block the whole request. Required detectors may block, but optional detectors should fail gracefully.

### 12. Add observability from the beginning

Track at least:

* Detector latency
* Grounding success rate
* Clarification rate
* User correction rate
* False short-circuit rate
* No-match rate
* Number of candidates per entity
* Percentage of defaults later changed by users
* Embedding freshness
* End-to-end latency added by enrichment

The most valuable quality metric may be **user correction rate after automatic resolution**. High correction rates indicate that confidence thresholds or grounding rules are too aggressive.

## Recommended implementation order

1. Finalize supported entity list and grounding sources.
2. Define the canonical entity and orchestrator-state schemas.
3. Define confidence, conflict, clarification, and short-circuit policies.
4. Implement language and authorization/location checks.
5. Implement menu-item grounding and embedding synchronization.
6. Add centralized ambiguity resolution.
7. Add batched human clarification.
8. Add structured interaction logging.
9. Add preference-learning rules.
10. Add streaming for user-facing events.
11. Expand to modifiers, quantities, day parts, and additional entities.

The architecture should not “kick overlapping entities down the road” completely. Full resolution can come later, but the schema must support multiple candidates and conflicts from the start; otherwise, the team may need to redesign every detector when ambiguity becomes unavoidable.

## Verdict

**Providing an entity list is the right approach—provided the list defines entity types and grounding rules, not every possible entity value.**

The disagreement appears to come from using “entity list” to mean two different things.

### What should be provided

Define the categories the system is expected to detect, such as:

* Menu item
* Menu category
* Modifier
* Quantity or size
* Location
* Date and time
* Day part
* Language

For every entity type, also define:

* Its authoritative data source
* Whether it is required or optional
* Whether it must be grounded
* Its account and location scope
* How ambiguity should be handled
* Whether the system may default or must clarify

This is necessary because the orchestrator needs a stable contract. Otherwise, the LLM may detect arbitrary concepts that downstream systems do not understand or cannot use.

### What should not be provided

Do not attempt to provide an exhaustive global list such as:

* Every burger name
* Every sandwich variation
* Every modifier
* Every possible menu item phrase
* Every possible person, place, or product name

That would be brittle, difficult to maintain, and contrary to how semantic detection is useful.

The LLM should be able to infer that:

* “Double cheeseburger” is probably a menu item
* “Large” is probably a size modifier
* “Tomorrow evening” contains time and day-part information

It should then ground those detected candidates against the account’s actual data.

## Why the entity-type list is important

### 1. It defines the system’s scope

An LLM can identify hundreds of possible semantic concepts. The application only needs a controlled subset.

For example, the phrase:

> Show sales of large burgers near Boston yesterday.

could contain:

* Menu item or category: burger
* Modifier: large
* Location: Boston
* Time: yesterday
* Intent: sales analysis

The entity list tells the system which parts should be extracted and stored in the enrichment state.

### 2. It creates a contract between product and engineering

Product must decide what behavior is supported. Engineering should not independently guess whether the system needs to detect:

* Menu categories
* Individual menu items
* Ingredients
* Dietary properties
* Brands
* Sizes
* Combos
* Locations
* Day parts

That decision affects schemas, APIs, clarification UX, test cases, and grounding sources.

### 3. It separates detection from grounding

Detection asks:

> Does “double cheeseburger” appear to be a menu item?

Grounding asks:

> Is there a matching item available for this account and location?

The LLM may successfully detect a menu item even when the business does not sell it. That does not mean detection failed. It means grounding returned no valid match.

### 4. It prevents unrestricted LLM behavior

Without a supported entity schema, the LLM might return inconsistent fields across queries:

```json
{
  "food": "burger"
}
```

then:

```json
{
  "product": "French fries"
}
```

then:

```json
{
  "dish_category": "pasta"
}
```

A predefined entity model ensures that these consistently return something like:

```json
{
  "entity_type": "menu_item_or_category",
  "raw_text": "burger",
  "grounded_candidates": []
}
```

### 5. It enables reliable clarification

The system cannot ask good clarification questions unless it knows which entity types are relevant.

For example:

> Did you mean the Double Cheeseburger, Bacon Burger, or Veggie Burger?

That clarification is only possible because “burger” was detected as a supported menu entity and then grounded to several account-specific candidates.

## The key distinction: category versus canonical value

“Burger” may be interpreted differently depending on the business data:

* A menu category
* A generic menu-item concept
* An exact menu item
* Part of several detailed menu-item names

The detector should not force a premature distinction. It can return a broader candidate first and let grounding determine what exists in the account’s data.

For example:

```json
{
  "entity_type": "menu",
  "raw_text": "burger",
  "possible_subtypes": ["menu_category", "menu_item"],
  "grounded_candidates": [
    "Double Cheeseburger",
    "Veggie Burger",
    "Chicken Burger"
  ],
  "resolution": "clarification_required"
}
```

## Recommended position for the team

The team should say:

> We are not asking product to enumerate every possible burger, sandwich, or menu-item value. We are asking product to define the entity categories the system is expected to understand and the business source against which each category must be grounded.

So the final position is:

* **Right:** Define a bounded list of supported entity types.
* **Right:** Ground detected values against account- and location-specific data.
* **Right:** Let the LLM detect natural-language variations and unseen phrases.
* **Wrong:** Maintain a universal exhaustive list of all possible entity values.
* **Wrong:** Allow the LLM to invent arbitrary entity types with no downstream contract.

The entity-type list does not restrict the LLM’s language understanding. It restricts the application’s output to concepts the system can reliably process.
