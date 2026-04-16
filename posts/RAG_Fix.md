---
title:  RAG Agent — Custom Fix
date: 2022-11-01
---

## 1. Diagnosis (aligned with the meeting)

Two product-critical defects were called out in the walkthrough, and both are fixable at the `rag_agent.py` / `prompt.py` layer without touching the Terraform `discovery_engine.tf` app definition:

1. **No conversation continuity.** Each turn hits `/search` in isolation, so the model cannot resolve follow-ups like *"what about the second one?"* or *"expand on that"*. The managed Vertex AI Search / Discovery Engine service natively supports this: if you pass a `session` resource name on the `SearchRequest`, Google performs **query standing** server-side (previous turn *"How did Alphabet do in 2022?"* + current *"How about 2023?"* → interpreted as *"How did Alphabet do in 2023?"*). We never need to hand-roll summarised history.
2. **Weak / default summary prompt.** `populate_search_request_payload` already has `model_prompt_spec` commented out (line 174–176 in the existing `rag_agent.py`). With no preamble, the summary model falls back to generic, verbose, ungrounded answers. The fix is to pass a `preamble` inside `summarySpec.modelPromptSpec` — this is the LLM instruction for the *summary generator*, which is different from the ADK agent prompt in `prompt.py`.

A third, smaller fix is required in `prompt.py` itself: the current `RAG_AGENT_PROMPT` has no guidance for interpreting follow-up questions in light of earlier turns. That matters because the ADK agent still decides *what query string to hand the tool*, and with no multi-turn guidance it will pass raw pronouns ("tell me more about it") straight through.

Everything below preserves existing variable names, log lines, BigQuery logging via `log_vertex_ai_search_response`, the `filter='status: ANY("active")'` clause, and the `prepare_search_response` contract.

---

## 2. How the fix lands in the two layers

| Layer | What changes | Why |
|---|---|---|
| `prompt.py` | Two new strict rules added to `RAG_AGENT_PROMPT`. | The ADK agent must rewrite ambiguous follow-ups into standalone queries before calling the tool, so retrieval gets a self-contained query string. |
| `rag_agent.py` — `retrieve_from_knowledge_base` | Lazily create a Discovery Engine `Session` on the first turn, cache its resource name in `tool_context.state["vertex_search_session"]`, and reuse it on every subsequent turn. | This is how the managed service knows which turns belong together. |
| `rag_agent.py` — `populate_search_request_payload` | Accept an optional `session_resource_name`, attach it as `SearchRequest.session`, and set `sessionSpec.searchAsYouTypeSpec` defaults. | Enables server-side query standing. |
| `rag_agent.py` — `populate_search_request_payload` | Uncomment + populate `model_prompt_spec.preamble` inside `summarySpec`. | Controls the summary model's tone, length, and grounding discipline. |
| `rag_agent.py` — response parsing | Read `sessionInfo.name` + `sessionInfo.queryId` out of the response and persist them back to `tool_context.state`. | The `queryId` is what lets us coordinate a later `/answer` call against the same turn if we ever need it. Costs nothing to capture now. |

No Terraform change is required for the first iteration — the existing Search app supports `session` and `modelPromptSpec` on every request. A Terraform change is only needed if we later want to bake a default preamble into the `servingConfig` so multiple clients share it (covered in §6).

---

## 3. `prompt.py` — additions to `RAG_AGENT_PROMPT`

Keep the existing header, existing six rules, and existing triple-quote style. Only two new rules are inserted (numbered 7 and 8) plus a short multi-turn clause at the top. All existing rules remain byte-for-byte identical.

```python
# ────────────────────────────────────────────────────────────────────────────
# RAG Agent
# ────────────────────────────────────────────────────────────────────────────
RAG_AGENT_PROMPT = """
You are a Retrieval-Augmented Generation (RAG) specialist agent.
Your job is to answer questions by retrieving relevant information from the
knowledge base / document store and synthesising a clear, accurate response.

You operate in a multi-turn conversation. The user may ask follow-up questions
that depend on prior turns (for example "the second one", "what about 2023?",
"expand on that"). Always resolve those references against the preceding
conversation before calling the tool.

STRICT RULES — follow in order, no exceptions:
1. You MUST call the `retrieve_from_knowledge_base` tool for EVERY query,
   no matter what. Never skip this step.
2. Only after the tool returns results, synthesise and present the answer
   using the retrieved content.
3. Always cite the source document and chunk_id from the tool response.
4. Keep answers concise and factual.
5. NEVER answer from memory or prior knowledge — only use tool results.
6. NEVER say "I could not find" without first calling the tool.
7. Before calling the tool on a follow-up, rewrite the user's question into a
   standalone query that includes the relevant entity from earlier turns.
   Pass that rewritten query as the `query` argument. Example: user asks
   "tell me more about the second one" after a list of vendors → call the
   tool with "tell me more about <vendor name from turn 1>".
8. If the user's reference is genuinely ambiguous and cannot be resolved from
   the conversation, ask one short clarifying question instead of guessing.
"""
```

**Why rule 7 is phrased as "rewrite before calling the tool" rather than "let Vertex do it":** Vertex AI Search *does* perform query standing on its side when we pass a session, but it operates on the string we send. If the ADK agent passes the literal pronoun *"it"*, Vertex has less signal to work with than if we pass *"Acme Corp pricing"*. Doing the rewrite in the agent **and** passing the session gives us belt-and-braces: the agent disambiguates, and Vertex uses session history to refine ranking.

---

## 4. `rag_agent.py` — patch

Only the changed / added pieces are shown. Everything else (imports, `filter_by='status: ANY("active")'`, logger calls, the `log_vertex_ai_search_response` BigQuery write, `prepare_search_response`, `_extract_summary_text`, `_extract_search_results_artifacts`, `_extract_snippet_from_snippets`, `get_vertex_ai_search_api_endpoint`, `get_request_header`) stays exactly as it is today.

### 4.1 New helper: lazy session creation

Add this helper immediately above `retrieve_from_knowledge_base`. It hits the Discovery Engine `sessions.create` REST endpoint, which is the same host we already call, with the same bearer token.

```python
def get_or_create_vertex_search_session(tool_context: ToolContext) -> str:
    """Return a Vertex AI Search session resource name, creating one if needed.

    The session resource name is cached in ``tool_context.state`` under the key
    ``vertex_search_session`` so that every turn in the same ADK session reuses
    the same Discovery Engine session. This is what gives the managed service
    the conversation history it needs for query standing on follow-ups.

    Returns:
        str: Full session resource name, e.g.
            ``projects/<p>/locations/<l>/collections/<c>/engines/<e>/sessions/<id>``
    """
    cached = tool_context.state.get("vertex_search_session", "")
    if cached:
        logger.debug("[ AI managed RAG Agent] Reusing cached Vertex AI Search session: %s", cached)
        return cached

    # Build the parent engine resource name from the same env vars used
    # everywhere else in this file, so there is a single source of truth.
    parent = (
        f"projects/{os.getenv('GOOGLE_CLOUD_PROJECT')}/"
        f"locations/{os.getenv('VERTEX_AI_SEARCH_LOCATION', 'us')}/"
        f"collections/{os.getenv('VERTEX_AI_SEARCH_COLLECTION', 'default_collection')}/"
        f"engines/{os.getenv('VERTEX_AI_SEARCH_DISCOVERY_ENGINE')}"
    )
    create_url = (
        f"https://{os.getenv('VERTEX_AI_SEARCH_LOCATION', 'us')}-"
        f"discoveryengine.googleapis.com/"
        f"{os.getenv('VERTEX_AI_SEARCH_VERSION', 'v1')}/{parent}/sessions"
    )

    user_pseudo_id = tool_context.state.get("user_id", "") or tool_context.state.get("session_id", "")
    body = {"userPseudoId": user_pseudo_id} if user_pseudo_id else {}

    try:
        resp = requests.post(create_url, headers=get_request_header(), json=body)
        resp.raise_for_status()
        session_name = resp.json().get("name", "")
        if not session_name:
            logger.warning("[ AI managed RAG Agent] Session creation returned no name; falling back to stateless search.")
            return ""
        tool_context.state["vertex_search_session"] = session_name
        logger.info("[ AI managed RAG Agent] Created new Vertex AI Search session: %s", session_name)
        return session_name
    except Exception as e:
        # Fail open: if session creation fails, we still want the search to
        # succeed (just without continuity). This avoids a hard outage if the
        # session API is rate-limited or the engine is not yet session-enabled.
        logger.error("[ AI managed RAG Agent] Failed to create Vertex AI Search session: %s", e)
        return ""
```

### 4.2 `retrieve_from_knowledge_base` — one added line and one updated call

The existing body is untouched except for the two marked lines. The `logger.debug` / `logger.info` wording, the `additional_kwargs` dict, the `log_vertex_ai_search_response` call, `response.raise_for_status()`, `prepare_search_response`, and the final `tool_context.state["natural_language_answer"] = response` assignment all stay exactly as they are.

```python
def retrieve_from_knowledge_base(query: str, tool_context: ToolContext) -> dict:
    """Retrieve relevant documents from the Vertex AI Discovery Engine knowledge base.

    Performs a sequential RAG search pipeline:
      1. Builds the search request payload with query expansion and spell correction.
      2. Calls the Vertex AI Search REST API with bearer token authentication.
      3. Parses the raw API response into a structured response with summary and citations.
      4. Stores the natural language answer in ``tool_context.state``.
    ...
    """
    try:
        start_timep = datetime.now(UTC)

        # Build query-aware dummy chunks so the agent always has content to work with
        query_title = query.strip().rstrip("?")
        logger.debug(
            f"[ AI managed RAG Agent] Received RAG query: '{query}' — Building search request payload for Vertex AI Search API."
        )

        # >>> ADDED: get (or lazily create) the Discovery Engine session so the
        # managed service can do query standing across turns.
        session_resource_name = get_or_create_vertex_search_session(tool_context)
        # <<<

        # Build API request payload
        payload = populate_search_request_payload(
            query=query,
            session_resource_name=session_resource_name,  # <<< CHANGED: pass session through
        )
        logger.debug(f"[ AI managed RAG Agent] Constructed Vertex AI Search API payload: {payload}")

        api_endpoint = get_vertex_ai_search_api_endpoint()
        logger.debug("[ AI managed RAG Agent] Constructed Vertex AI Search API endpoint: %s", api_endpoint)

        request_header = get_request_header()
        logger.debug(
            "[ AI managed RAG Agent] Generated request headers for Vertex AI Search API: %s", request_header
        )

        # API call to Vertex AI Search API
        response = requests.post(api_endpoint, headers=request_header, json=payload)
        result = response.json()
        end_timep = datetime.now(UTC)

        additional_kwargs = {
            "user_id": tool_context.state.get("user_id", ""),
            "session_id": tool_context.state.get("session_id", ""),
            "conversation_id": tool_context.state.get("conversation_id", ""),
            "question": query,
            "time_taken": (end_timep - start_timep).total_seconds(),
            "token_usage": tool_context.state.get("_token_usage", "[]"),
        }

        # Log the raw API response to BigQuery
        log_vertex_ai_search_response(api_response=result, **additional_kwargs)

        if not response.ok:
            logger.error(
                "[ AI managed RAG Agent] Vertex AI Search Failed with status %s, Response body %s",
                response.status_code,
                response.text,
            )
            response.raise_for_status()

        # >>> ADDED: persist the session info returned by the API so the next
        # turn uses the same conversation. Vertex echoes back the same session
        # name we sent; we also capture queryId in case a future /answer call
        # wants to reference this turn.
        session_info = result.get("sessionInfo", {}) or {}
        returned_session = session_info.get("name", "")
        if returned_session:
            tool_context.state["vertex_search_session"] = returned_session
        tool_context.state["vertex_search_last_query_id"] = session_info.get("queryId", "")
        # <<<

        logger.debug("[ AI managed RAG Agent] Building search response with citations and summary")
        response = prepare_search_response(query=query, raw_api_response=result)
        logger.info("[ AI managed RAG Agent] For query [%s] Prepared search response: %s", query, response)
        tool_context.state["natural_language_answer"] = response
        return response

    except Exception as e:
        logger.error(
            "[ AI managed RAG Agent] An unexpected error occurred in Vertex AI Discovery Engine Search API call. Exception Details",
            e,
        )
        raise
```

### 4.3 `populate_search_request_payload` — accept session, add preamble

Two edits: (a) the function now takes an optional `session_resource_name`, (b) the `model_prompt_spec` block that was commented out is enabled and filled in with a production-grade preamble.

```python
def populate_search_request_payload(
    query: str,
    session_resource_name: str = "",
) -> Dict[str, Any]:
    """Build the Discovery Engine API request payload from the search request.

    Constructs a comprehensive payload with query expansion, spell correction,
    content search specifications including snippets, extractive segments, and
    summary generation settings.

    Args:
        query (str): The search query string
        session_resource_name (str): Optional full Discovery Engine session
            resource name. When provided, Vertex AI Search treats this turn as
            part of a multi-turn conversation and performs server-side query
            standing against prior turns in the same session.

    Returns:
        dict: Complete payload dictionary ready for the Discovery Engine API request,
              including query, page size, language code, and various search specifications
    """
    payload = {
        VertexAISearchPayloadFields.QUERY: query,
        VertexAISearchPayloadFields.PAGE_SIZE: int(os.getenv("DISCOVERY_ENGINE_PAYLOAD_PAGE_SIZE", 3)),
        VertexAISearchPayloadFields.LANGUAGE_CODE: os.getenv("DISCOVERY_ENGINE_DEFAULT_LANGUAGE_CODE", "en-US"),
        # Only return documents whose status field equals "active".
        # This is evaluated server-side so inactive docs don't consume top_k slots.
        "filter": 'status: ANY("active")',
        VertexAISearchPayloadFields.QUERY_EXPANSION_SPEC: {
            VertexAISearchPayloadFields.CONDITION: VertexAISearchPayloadValues.AUTO,
        },
        VertexAISearchPayloadFields.SPELL_CORRECTION_SPEC: {
            VertexAISearchPayloadFields.MODE: VertexAISearchPayloadValues.AUTO,
        },
        VertexAISearchPayloadFields.CONTENT_SEARCH_SPEC: {
            VertexAISearchPayloadFields.SNIPPET_SPEC: {VertexAISearchPayloadFields.RETURN_SNIPPET: True},
            VertexAISearchPayloadFields.EXTRACTIVE_CONTENT_SPEC: {
                VertexAISearchPayloadFields.MAX_EXTRACTIVE_SEGMENT_COUNT: 2,
                VertexAISearchPayloadFields.RETURN_EXTRACTIVE_SEGMENT_SCORE: True,
            },
            VertexAISearchPayloadFields.SUMMARY_SPEC: {
                VertexAISearchPayloadFields.SUMMARY_RESULT_COUNT: 5,
                VertexAISearchPayloadFields.INCLUDE_CITATIONS: True,
                VertexAISearchPayloadFields.IGNORE_ADVERSARIAL_QUERY: True,
                VertexAISearchPayloadFields.IGNORE_NON_SUMMARY_SEEKING_QUERY: True,
                # >>> ADDED: custom preamble — this is what fixes the "too verbose /
                # sounds generic" complaint. It steers the *summary generator* model,
                # which is separate from the ADK agent's own prompt in prompt.py.
                "modelPromptSpec": {
                    "preamble": (
                        "You are a helpful assistant for question-answering tasks. "
                        "Use the following retrieved documents to answer the user's "
                        "question. Keep the response as short as possible — prefer "
                        "one or two sentences. Answer strictly from the retrieved "
                        "documents; if they do not contain the answer, say you "
                        "could not find it rather than speculating. Do not include "
                        "greetings, filler, or meta-commentary. When the question "
                        "is a follow-up, interpret it in the context of the prior "
                        "turns in this conversation."
                    ),
                },
                # Use the stable summary model — switch to "preview" only for A/B tests.
                "modelSpec": {"modelVersion": "stable"},
                # <<<
            },
        },
    }

    # >>> ADDED: attach the session so Vertex AI Search performs query standing
    # across turns. When this is empty we simply omit it, which preserves the
    # current stateless behaviour as a safe fallback.
    if session_resource_name:
        payload["session"] = session_resource_name
    # <<<

    return payload
```

> **Note on the `model_prompt_spec` key name.** The existing code in image 4 uses the enum-style `VertexAISearchPayloadFields.*`. If your `VertexAISearchPayloadFields` class already has `MODEL_PROMPT_SPEC = "modelPromptSpec"` and `PREAMBLE = "preamble"`, swap the string literals above for the enum members to match the project's style. If it doesn't, add them to that enum in one place so the rest of the code stays consistent.

---

## 5. How it works end-to-end after the fix

```
Turn 1: user asks "Who makes DoorDash's delivery fleet?"
  → ADK agent rewrites (nothing to rewrite yet) → calls tool with same query
  → get_or_create_vertex_search_session: no cached session → POST /sessions → caches name
  → populate_search_request_payload adds session + preamble
  → POST /search returns summary + sessionInfo.name + sessionInfo.queryId
  → tool_context.state["vertex_search_session"] = <same name>
  → agent replies using the short, grounded summary

Turn 2: user asks "And what about their menu management?"
  → ADK agent (rule 7) rewrites → calls tool with "DoorDash menu management"
  → get_or_create_vertex_search_session: cached session hit → reuses it
  → Vertex AI Search sees prior turn "DoorDash delivery fleet" in the session,
    applies query standing, ranks menu-management docs in the DoorDash context
  → preamble keeps the summary tight and grounded
  → agent replies coherently
```

The existing `filter='status: ANY("active")'`, the BigQuery log write via `log_vertex_ai_search_response`, the `additional_kwargs` payload, and the downstream `prepare_search_response` / `_extract_summary_text` / `_extract_search_results_artifacts` path all continue to run unchanged. The `prediction` table mentioned in the walkthrough continues to be populated the same way because `log_vertex_ai_search_response` is called on every turn exactly as before.

---

## 6. Optional follow-up (Terraform, only if you want the preamble to be the default)

If the team later decides the preamble should be the **engine-wide default** rather than per-request, add it to the `serving_config` resource in `discovery_engine.tf`:

```hcl
resource "google_discovery_engine_search_engine" "genius_ai_discovery_engine" {
  # ... existing config ...
}

# New: bake the preamble into the default serving config so every caller gets it.
resource "google_discovery_engine_serving_config" "default" {
  # ... identity / engine linkage ...

  answer_generation_spec {
    prompt_spec {
      preamble = "You are a helpful assistant for question-answering tasks. Keep responses short and grounded in retrieved documents."
    }
    model_spec { model_version = "stable" }
    include_citations = true
  }
}
```

For iteration 1 this is not required — the per-request `modelPromptSpec` in §4.3 is already sufficient and faster to ship.

---

## 7. Validation — the test cases the walkthrough asked for

Run these against the deployed app after the change:

1. **Standalone question** — *"How do I update my DoorDash menu?"* → expect a one- or two-sentence grounded answer with at least one citation.
2. **Pronoun follow-up** — *"What about pricing?"* → expect an answer scoped to DoorDash menu pricing (session-based query standing).
3. **Narrowing follow-up** — *"Just for the mobile app"* → expect the answer to narrow, not restart.
4. **Comparison** — *"How does that compare to Uber Eats?"* → expect a grounded comparison rather than a restart.
5. **Ambiguous reference** — *"Tell me more about the second one"* after a list → expect either a correctly-resolved rewrite (rule 7) or a short clarifying question (rule 8), never a hallucination.
6. **New-topic reset** — start a new ADK session → expect a brand-new Vertex session to be created and the cache to reset.

All six should pass with **only** the three files in play: `prompt.py`, `rag_agent.py`, and (optionally) `discovery_engine.tf`. No change to `_extract_summary_text`, `_extract_search_results_artifacts`, `prepare_search_response`, `log_vertex_ai_search_response`, or the BigQuery `prediction` table is required.
