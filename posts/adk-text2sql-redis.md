---
title: Low-Latency Text-to-SQL with Google ADK + Redis Semantic Layer
date: 2026-01-03
---

## Architecture Overview

Two ADK agents share a single Redis semantic layer. The **Simple Agent** handles straightforward queries with shallow retrieval. The **Complex Agent** handles escalated queries using deep retrieval, skeleton matching, and focused context assembly.

**Target latency:** 8-15 seconds end-to-end (down from 40-60s)

```
User Query
    │
    ▼
┌─────────────────────┐
│   Simple Agent       │──── shallow retrieval (top 5) ──┐
│   (Gemini Flash)     │                                  │
└─────────┬───────────┘                                   │
          │ escalate if complex                           │
          ▼                                               │
┌─────────────────────┐                                   │
│   Complex Agent      │                                  │
│   (Gemini Pro)       │                                  │
│                      │                                  │
│  1. Semantic Cache   │◄── cache hit? return instantly   │
│  2. Skeleton Match   │◄── match? slot-fill only         │
│  3. Deep Retrieval   │◄── top 10-15 fragments           │
│  4. LLM Generation   │                                  │
└─────────┬───────────┘                                   │
          │                                               │
          ▼                                               │
┌─────────────────────────────────────────────────────────┤
│                REDIS (Single Instance)                   │
│                                                          │
│  Index 1: semantic_layer    (40K lines, embedded)        │
│  Index 2: sql_skeletons     (20-30 templates, embedded)  │
│  Index 3: query_cache       (semantic cache of results)  │
│  Sessions: ADK session state                             │
└──────────────────────────────────────────────────────────┘
```

---

## 1. Redis Index Setup

### 1.1 Install Dependencies

```bash
pip install google-adk redisvl redis adk-redis sentence-transformers
```

### 1.2 Semantic Layer Index

This is the single source of truth both agents query. Your 40K-line JSON gets
chunked into per-table/per-column fragments, enriched with descriptions, and
embedded.

```python
# redis_setup.py

import json
from redisvl.index import SearchIndex
from redisvl.schema import IndexSchema
from redisvl.utils.vectorize import HFTextVectorizer

REDIS_URL = "redis://localhost:6379"

# Use a small, fast embedding model for low latency
vectorizer = HFTextVectorizer(model="sentence-transformers/all-MiniLM-L6-v2")
VECTOR_DIM = 384

# --- SEMANTIC LAYER INDEX ---
semantic_layer_schema = IndexSchema.from_dict({
    "index": {
        "name": "semantic_layer",
        "prefix": "schema:",
        "storage_type": "json"
    },
    "fields": [
        {"name": "table_name",   "type": "tag"},
        {"name": "domain",       "type": "tag"},       # finance, marketing, logistics, etc.
        {"name": "object_type",  "type": "tag"},       # table, column, metric, join
        {"name": "description",  "type": "text"},
        {"name": "ddl_fragment", "type": "text"},
        {"name": "join_info",    "type": "text"},
        {"name": "business_logic", "type": "text"},
        {
            "name": "embedding",
            "type": "vector",
            "attrs": {
                "algorithm": "hnsw",
                "dims": VECTOR_DIM,
                "distance_metric": "cosine",
                "datatype": "float32",
                "m": 16,
                "ef_construction": 200
            }
        }
    ]
})

semantic_index = SearchIndex(
    schema=semantic_layer_schema,
    redis_url=REDIS_URL
)
semantic_index.create(overwrite=True)
```

### 1.3 Ingest Your 40K-Line Semantic Layer

```python
# ingest_semantic_layer.py

def chunk_semantic_layer(semantic_layer_json: dict) -> list[dict]:
    """
    Break your 40K-line JSON into per-table fragments.
    Each fragment gets an enriched description for embedding.

    Your JSON likely has structure like:
    {
      "tables": [
        {
          "name": "orders",
          "domain": "finance",
          "columns": [...],
          "description": "...",
          "joins": [...],
          "business_rules": [...]
        }
      ]
    }
    """
    fragments = []

    for table in semantic_layer_json.get("tables", []):
        # --- Table-level fragment ---
        col_names = [c["name"] for c in table.get("columns", [])]
        enriched_desc = (
            f"Table: {table['name']}. "
            f"{table.get('description', '')} "
            f"Columns: {', '.join(col_names)}. "
            f"Domain: {table.get('domain', 'general')}."
        )

        fragments.append({
            "table_name": table["name"],
            "domain": table.get("domain", "general"),
            "object_type": "table",
            "description": enriched_desc,
            "ddl_fragment": build_ddl(table),         # your DDL builder
            "join_info": json.dumps(table.get("joins", [])),
            "business_logic": json.dumps(table.get("business_rules", [])),
        })

        # --- Column-level fragments for important columns ---
        for col in table.get("columns", []):
            if col.get("is_metric") or col.get("is_dimension"):
                col_desc = (
                    f"Column {col['name']} in table {table['name']}. "
                    f"{col.get('description', '')} "
                    f"Type: {col.get('data_type', 'unknown')}. "
                    f"Domain: {table.get('domain', 'general')}."
                )
                fragments.append({
                    "table_name": table["name"],
                    "domain": table.get("domain", "general"),
                    "object_type": "column",
                    "description": col_desc,
                    "ddl_fragment": f"{col['name']} {col.get('data_type', '')}",
                    "join_info": "",
                    "business_logic": json.dumps(col.get("business_rules", [])),
                })

    return fragments


def ingest(semantic_layer_path: str):
    with open(semantic_layer_path) as f:
        raw = json.load(f)

    fragments = chunk_semantic_layer(raw)

    # Embed all descriptions in batch
    descriptions = [f["description"] for f in fragments]
    embeddings = vectorizer.embed_many(descriptions, as_buffer=True)

    # Attach embeddings
    for frag, emb in zip(fragments, embeddings):
        frag["embedding"] = emb

    # Load into Redis
    semantic_index.load(fragments, id_field="table_name")
    print(f"Loaded {len(fragments)} fragments into semantic_layer index")


if __name__ == "__main__":
    ingest("semantic_layer.json")
```

### 1.4 SQL Skeleton Index

```python
# skeleton_setup.py

skeleton_schema = IndexSchema.from_dict({
    "index": {
        "name": "sql_skeletons",
        "prefix": "skeleton:",
        "storage_type": "json"
    },
    "fields": [
        {"name": "skeleton_id",     "type": "tag"},
        {"name": "description",     "type": "text"},
        {"name": "sql_template",    "type": "text"},
        {"name": "slots",           "type": "text"},       # JSON string
        {"name": "complexity_tags",  "type": "tag"},
        {
            "name": "embedding",
            "type": "vector",
            "attrs": {
                "algorithm": "hnsw",
                "dims": VECTOR_DIM,
                "distance_metric": "cosine",
                "datatype": "float32"
            }
        }
    ]
})

skeleton_index = SearchIndex(
    schema=skeleton_schema,
    redis_url=REDIS_URL
)
skeleton_index.create(overwrite=True)


# --- Load your skeletons ---
SKELETONS = [
    {
        "skeleton_id": "simple_aggregation",
        "description": (
            "Aggregate a single metric grouped by one dimension "
            "with an optional date range filter"
        ),
        "sql_template": (
            "SELECT {dimension}, {agg_func}({metric}) AS {alias}\n"
            "FROM {table}\n"
            "WHERE {date_column} BETWEEN '{start_date}' AND '{end_date}'\n"
            "GROUP BY {dimension}\n"
            "ORDER BY {alias} DESC"
        ),
        "slots": json.dumps({
            "dimension": "column to group by",
            "agg_func": "SUM, COUNT, AVG, etc.",
            "metric": "numeric column to aggregate",
            "alias": "output column name",
            "table": "source table",
            "date_column": "date/timestamp column",
            "start_date": "range start (YYYY-MM-DD)",
            "end_date": "range end (YYYY-MM-DD)"
        }),
        "complexity_tags": "aggregation,date_filter"
    },
    {
        "skeleton_id": "yoy_comparison",
        "description": (
            "Year-over-year or period-over-period comparison of a metric "
            "grouped by a dimension using CASE WHEN for two date windows"
        ),
        "sql_template": (
            "SELECT {dimension},\n"
            "  SUM(CASE WHEN {date_col} BETWEEN '{curr_start}' AND '{curr_end}'\n"
            "       THEN {metric} END) AS current_period,\n"
            "  SUM(CASE WHEN {date_col} BETWEEN '{prev_start}' AND '{prev_end}'\n"
            "       THEN {metric} END) AS prior_period,\n"
            "  ROUND(\n"
            "    (SUM(CASE WHEN {date_col} BETWEEN '{curr_start}' AND '{curr_end}'\n"
            "          THEN {metric} END) -\n"
            "     SUM(CASE WHEN {date_col} BETWEEN '{prev_start}' AND '{prev_end}'\n"
            "          THEN {metric} END)) /\n"
            "    NULLIF(SUM(CASE WHEN {date_col} BETWEEN '{prev_start}' AND '{prev_end}'\n"
            "          THEN {metric} END), 0) * 100, 2\n"
            "  ) AS pct_change\n"
            "FROM {table}\n"
            "GROUP BY {dimension}\n"
            "ORDER BY current_period DESC"
        ),
        "slots": json.dumps({
            "dimension": "column to group by",
            "date_col": "date/timestamp column",
            "metric": "numeric column to aggregate",
            "table": "source table",
            "curr_start": "current period start",
            "curr_end": "current period end",
            "prev_start": "prior period start",
            "prev_end": "prior period end"
        }),
        "complexity_tags": "time_comparison,aggregation,case_when"
    },
    {
        "skeleton_id": "top_n_with_join",
        "description": (
            "Top N entities by an aggregate metric with a JOIN "
            "between two tables and an optional qualifying filter"
        ),
        "sql_template": (
            "SELECT {entity_col}, {agg_func}({metric}) AS agg_value\n"
            "FROM {table_a}\n"
            "JOIN {table_b} ON {table_a}.{join_key_a} = {table_b}.{join_key_b}\n"
            "WHERE {filter_col} = '{filter_val}'\n"
            "GROUP BY {entity_col}\n"
            "ORDER BY agg_value DESC\n"
            "LIMIT {n}"
        ),
        "slots": json.dumps({
            "entity_col": "entity column to rank",
            "agg_func": "aggregation function",
            "metric": "numeric column",
            "table_a": "primary table",
            "table_b": "joined table",
            "join_key_a": "join key in table_a",
            "join_key_b": "join key in table_b",
            "filter_col": "filter column",
            "filter_val": "filter value",
            "n": "number of results"
        }),
        "complexity_tags": "join,aggregation,top_n,filter"
    },
    {
        "skeleton_id": "multi_join_report",
        "description": (
            "Report combining data from three or more tables with "
            "multiple JOINs, aggregation, and ordering"
        ),
        "sql_template": (
            "SELECT\n"
            "  {select_cols}\n"
            "FROM {base_table}\n"
            "JOIN {table_2} ON {join_cond_1}\n"
            "JOIN {table_3} ON {join_cond_2}\n"
            "WHERE {where_clause}\n"
            "GROUP BY {group_cols}\n"
            "HAVING {having_clause}\n"
            "ORDER BY {order_cols}\n"
            "LIMIT {limit}"
        ),
        "slots": json.dumps({
            "select_cols": "comma-separated select expressions",
            "base_table": "primary FROM table",
            "table_2": "second table",
            "table_3": "third table",
            "join_cond_1": "first join condition",
            "join_cond_2": "second join condition",
            "where_clause": "filter conditions",
            "group_cols": "group by columns",
            "having_clause": "having filter (use 1=1 if none)",
            "order_cols": "order by expressions",
            "limit": "row limit"
        }),
        "complexity_tags": "multi_join,aggregation,having"
    },
    {
        "skeleton_id": "subquery_exists",
        "description": (
            "Filter rows from a main table where a condition exists "
            "or does not exist in a correlated subquery"
        ),
        "sql_template": (
            "SELECT {select_cols}\n"
            "FROM {main_table} m\n"
            "WHERE {exists_or_not} (\n"
            "  SELECT 1 FROM {sub_table} s\n"
            "  WHERE s.{sub_key} = m.{main_key}\n"
            "    AND {sub_condition}\n"
            ")\n"
            "ORDER BY {order_cols}"
        ),
        "slots": json.dumps({
            "select_cols": "columns to return",
            "main_table": "outer query table",
            "exists_or_not": "EXISTS or NOT EXISTS",
            "sub_table": "subquery table",
            "sub_key": "subquery join key",
            "main_key": "outer query join key",
            "sub_condition": "additional subquery filter",
            "order_cols": "order by expression"
        }),
        "complexity_tags": "subquery,correlated,exists"
    }
]


def ingest_skeletons():
    descriptions = [s["description"] for s in SKELETONS]
    embeddings = vectorizer.embed_many(descriptions, as_buffer=True)

    for skeleton, emb in zip(SKELETONS, embeddings):
        skeleton["embedding"] = emb

    skeleton_index.load(SKELETONS, id_field="skeleton_id")
    print(f"Loaded {len(SKELETONS)} skeletons")


if __name__ == "__main__":
    ingest_skeletons()
```

### 1.5 Semantic Cache Index

```python
# cache_setup.py

from redisvl.extensions.llmcache import SemanticCache

query_cache = SemanticCache(
    name="sql_query_cache",
    redis_url=REDIS_URL,
    vectorizer=vectorizer,
    distance_threshold=0.08   # cosine distance — lower = stricter
                              # 0.08 ≈ 0.92 cosine similarity
)
```

---

## 2. Retrieval Pipeline (Shared by Both Agents)

This is the core latency-reduction engine. Both agents call into it; they
differ only in `retrieval_depth`.

```python
# retrieval.py

import json
from redisvl.query import VectorQuery, FilterQuery
from redisvl.query.filter import Tag

# Already initialized from redis_setup.py
# semantic_index, skeleton_index, query_cache, vectorizer


def retrieve_schema(
    query: str,
    domain: str | None = None,
    retrieval_depth: int = 5,
) -> list[dict]:
    """
    Three-stage retrieval against the semantic layer.

    Stage 1: Tag-based coarse filter (microseconds)
    Stage 2: Vector search within filtered set (1-5ms)
    Stage 3: Return top-K fragments

    Args:
        query:            Natural language user query
        domain:           Optional domain filter (finance, marketing, etc.)
        retrieval_depth:  Number of fragments to return
                          Simple agent: 5, Complex agent: 10-15
    """
    query_embedding = vectorizer.embed(query)

    # Build filter
    filter_expr = None
    if domain:
        filter_expr = Tag("domain") == domain

    vector_query = VectorQuery(
        vector=query_embedding,
        vector_field_name="embedding",
        return_fields=[
            "table_name", "domain", "object_type",
            "description", "ddl_fragment", "join_info",
            "business_logic"
        ],
        num_results=retrieval_depth,
        filter_expression=filter_expr
    )

    results = semantic_index.query(vector_query)
    return results


def match_skeleton(query: str, threshold: float = 0.10) -> dict | None:
    """
    Check if the query matches a pre-computed SQL skeleton.

    Returns the skeleton dict if similarity > threshold, else None.
    threshold=0.10 in cosine distance ≈ 0.90 cosine similarity
    """
    query_embedding = vectorizer.embed(query)

    vector_query = VectorQuery(
        vector=query_embedding,
        vector_field_name="embedding",
        return_fields=[
            "skeleton_id", "description",
            "sql_template", "slots"
        ],
        num_results=1
    )

    results = skeleton_index.query(vector_query)

    if results and float(results[0].get("vector_distance", 1.0)) < threshold:
        return results[0]

    return None


def check_cache(query: str) -> str | None:
    """
    Check semantic cache for a previously generated SQL.
    Returns cached SQL string or None.
    """
    hits = query_cache.check(prompt=query)
    if hits:
        return hits[0]["response"]
    return None


def store_in_cache(query: str, sql: str):
    """Store a successful query-SQL pair in the semantic cache."""
    query_cache.store(prompt=query, response=sql)
```

---

## 3. ADK Tool Definitions

```python
# tools.py

import json
from google.adk.tools import FunctionTool


def get_schema_simple(input: dict) -> dict:
    """
    Retrieve relevant schema fragments for simple queries.
    Shallow retrieval: top 5 fragments.
    """
    query = input.get("query", "")
    domain = input.get("domain", None)

    fragments = retrieve_schema(
        query=query,
        domain=domain,
        retrieval_depth=5
    )

    return {
        "schema_context": format_fragments(fragments),
        "fragment_count": len(fragments)
    }


def get_schema_deep(input: dict) -> dict:
    """
    Deep retrieval for complex queries.
    Returns top 10-15 fragments including join info and business logic.
    """
    query = input.get("query", "")
    domain = input.get("domain", None)
    escalation_reason = input.get("escalation_reason", "")

    # Primary retrieval — more fragments
    fragments = retrieve_schema(
        query=query,
        domain=domain,
        retrieval_depth=15
    )

    # Second pass — if escalation mentions joins or multi-table,
    # also retrieve by referenced table names
    if "join" in escalation_reason.lower() or "multi" in escalation_reason.lower():
        table_names = extract_table_names(fragments)
        for tname in table_names:
            join_fragments = retrieve_schema(
                query=f"joins and relationships for {tname}",
                domain=domain,
                retrieval_depth=3
            )
            fragments.extend(join_fragments)

    # Deduplicate
    seen = set()
    unique = []
    for f in fragments:
        key = f.get("table_name", "") + f.get("object_type", "")
        if key not in seen:
            seen.add(key)
            unique.append(f)

    return {
        "schema_context": format_fragments(unique),
        "fragment_count": len(unique)
    }


def find_sql_skeleton(input: dict) -> dict:
    """
    Search for a matching SQL skeleton template.
    If found, returns the template and slots for the LLM to fill.
    """
    query = input.get("query", "")

    skeleton = match_skeleton(query, threshold=0.10)

    if skeleton:
        return {
            "match_found": True,
            "skeleton_id": skeleton.get("skeleton_id", ""),
            "description": skeleton.get("description", ""),
            "sql_template": skeleton.get("sql_template", ""),
            "slots": skeleton.get("slots", "{}"),
            "distance": skeleton.get("vector_distance", "")
        }

    return {"match_found": False}


def check_sql_cache(input: dict) -> dict:
    """Check if a semantically similar query has been answered before."""
    query = input.get("query", "")
    cached = check_cache(query)

    if cached:
        return {"cache_hit": True, "sql": cached}

    return {"cache_hit": False}


def execute_sql(input: dict) -> dict:
    """Execute SQL against the database and return results."""
    sql = input.get("sql", "")
    # YOUR DATABASE EXECUTION LOGIC HERE
    # results = db.execute(sql)
    return {"status": "executed", "sql": sql, "rows": []}


def save_to_cache(input: dict) -> dict:
    """Store a successful query-SQL pair in the semantic cache."""
    query = input.get("query", "")
    sql = input.get("sql", "")
    store_in_cache(query, sql)
    return {"status": "cached"}


# --- Helpers ---

def format_fragments(fragments: list[dict]) -> str:
    """Format retrieved fragments into a compact context string."""
    lines = []
    for f in fragments:
        lines.append(f"### {f.get('table_name', 'unknown')} ({f.get('object_type', '')})")
        lines.append(f"Description: {f.get('description', '')}")
        if f.get("ddl_fragment"):
            lines.append(f"DDL: {f['ddl_fragment']}")
        if f.get("join_info") and f["join_info"] != "[]":
            lines.append(f"Joins: {f['join_info']}")
        if f.get("business_logic") and f["business_logic"] != "[]":
            lines.append(f"Business Rules: {f['business_logic']}")
        lines.append("")
    return "\n".join(lines)


def extract_table_names(fragments: list[dict]) -> list[str]:
    """Pull unique table names from fragments for secondary retrieval."""
    return list(set(f.get("table_name", "") for f in fragments if f.get("table_name")))


# --- Wrap as ADK FunctionTools ---

get_schema_simple_tool = FunctionTool(get_schema_simple)
get_schema_deep_tool = FunctionTool(get_schema_deep)
find_skeleton_tool = FunctionTool(find_sql_skeleton)
check_cache_tool = FunctionTool(check_sql_cache)
execute_sql_tool = FunctionTool(execute_sql)
save_cache_tool = FunctionTool(save_to_cache)
```

---

## 4. ADK Agent Definitions

### 4.1 Simple Agent

```python
# agents/simple_agent.py

from google.adk.agents import Agent

simple_agent = Agent(
    name="simple_sql_agent",
    model="gemini-2.0-flash",          # fast model for simple queries
    tools=[
        get_schema_simple_tool,
        check_cache_tool,
        execute_sql_tool,
        save_cache_tool,
    ],
    instruction="""You are a SQL generation agent for simple, single-table queries.

WORKFLOW (follow this order every time):
1. Call check_sql_cache with the user's query.
   - If cache_hit is true, return the cached SQL immediately.
2. Call get_schema_simple to retrieve relevant schema.
3. Generate the SQL query using ONLY the retrieved schema context.
4. Call execute_sql to run the query.
5. Call save_to_cache with the query and successful SQL.
6. Return the results to the user.

ESCALATION RULES — transfer to complex_sql_agent if ANY of these apply:
- Query requires JOINs across 2+ tables
- Query involves year-over-year or period comparisons
- Query has subqueries, CTEs, or window functions
- Query references calculated metrics not in a single column
- Query is ambiguous and needs business logic interpretation
- You are not confident the schema context is sufficient

When escalating, include:
- The original user query
- Your reason for escalating
- Any domain you identified (finance, marketing, etc.)

SQL RULES:
- Generate {your_sql_dialect} dialect only
- Never use SELECT *
- Always include column aliases
- Date formats: YYYY-MM-DD
"""
)
```

### 4.2 Complex Agent

```python
# agents/complex_agent.py

from google.adk.agents import Agent

complex_agent = Agent(
    name="complex_sql_agent",
    model="gemini-2.5-pro",            # reasoning model for complex queries
    tools=[
        check_cache_tool,
        find_skeleton_tool,
        get_schema_deep_tool,
        execute_sql_tool,
        save_cache_tool,
    ],
    instruction="""You are an expert SQL generation agent for complex analytical queries.

WORKFLOW (follow this exact order — do NOT skip steps):

STEP 1 — CACHE CHECK
  Call check_sql_cache with the user's query.
  If cache_hit is true → return the cached SQL. Done.

STEP 2 — SKELETON MATCH
  Call find_sql_skeleton with the user's query.
  If match_found is true → proceed to STEP 3 with the skeleton.
  If match_found is false → proceed to STEP 3 without a skeleton.

STEP 3 — SCHEMA RETRIEVAL
  Call get_schema_deep with:
    - query: the user's original query
    - domain: if identified from the escalation context
    - escalation_reason: why the simple agent escalated

STEP 4 — SQL GENERATION

  IF a skeleton was matched in Step 2:
    You have a sql_template with {slot_name} placeholders.
    Fill each slot using the schema context from Step 3.
    DO NOT restructure the template. Only fill slots.

    Your output format for skeleton fills:
    ```
    SKELETON USED: {skeleton_id}
    SLOTS FILLED:
    - dimension = region_name
    - metric = order_total_usd
    - ...

    FINAL SQL:
    SELECT ...
    ```

  IF no skeleton was matched:
    Generate SQL from scratch using the schema context.
    Think step by step:
    a) Identify which tables are needed
    b) Determine join conditions from the join_info fields
    c) Apply business_logic rules from the context
    d) Write the SQL

STEP 5 — EXECUTE AND CACHE
  Call execute_sql. If it fails, review the error and retry once.
  On success, call save_to_cache with the query and SQL.

CONSTRAINTS:
- Generate {your_sql_dialect} dialect only
- Never use SELECT *
- Always include column aliases
- Use the schema context as your ONLY source of truth for table/column names
- Do NOT hallucinate table or column names not in the context
- Date formats: YYYY-MM-DD
- If schema context seems insufficient, say so rather than guessing
"""
)
```

### 4.3 Coordinator Agent (Orchestrator)

```python
# agents/coordinator.py

from google.adk.agents import Agent

coordinator = Agent(
    name="sql_coordinator",
    model="gemini-2.0-flash",
    sub_agents=[simple_agent, complex_agent],
    instruction="""You are a routing coordinator for SQL query generation.

Your ONLY job is to route the user's query to the right agent:

- Route to simple_sql_agent for:
  Single table queries, basic filters, simple aggregations,
  straightforward SELECT statements.

- Route to complex_sql_agent for:
  Multi-table JOINs, period comparisons, subqueries, CTEs,
  window functions, calculated metrics, ambiguous business questions.

When routing to complex_sql_agent, always include:
1. The original user query
2. Why you think it's complex
3. Any domain hints (finance, marketing, logistics, etc.)

Do NOT attempt to generate SQL yourself. Route immediately.
"""
)
```

---

## 5. Runner and Session Setup

```python
# main.py

from google.adk.runners import Runner
from adk_redis.sessions import (
    RedisWorkingMemorySessionService,
    RedisWorkingMemorySessionServiceConfig,
)
from agents.coordinator import coordinator

# --- Session persistence in Redis ---
session_config = RedisWorkingMemorySessionServiceConfig(
    api_base_url="http://localhost:8088",  # Redis Agent Memory Server
    default_namespace="text2sql_app",
    model_name="gemini-2.0-flash",
    context_window_max=8000,
)
session_service = RedisWorkingMemorySessionService(config=session_config)

# --- Runner ---
runner = Runner(
    agent=coordinator,
    app_name="text2sql",
    session_service=session_service,
)


async def handle_query(user_id: str, session_id: str, query: str):
    """Entry point for a user query."""
    session = await session_service.get_session(
        app_name="text2sql",
        user_id=user_id,
        session_id=session_id
    )

    if not session:
        session = await session_service.create_session(
            app_name="text2sql",
            user_id=user_id,
            session_id=session_id
        )

    response = await runner.run_async(
        user_id=user_id,
        session_id=session.id,
        new_message=query
    )

    return response
```

---

## 6. Mining Skeletons from Query Logs

Run this once against your historical query logs to bootstrap the skeleton
store. Then re-run periodically (weekly/monthly) to catch new patterns.

```python
# mine_skeletons.py

"""
Extract SQL skeletons from your query logs.

Input:  List of successful (question, sql) pairs from logs.
Output: Deduplicated skeleton templates with slot definitions.
"""

import re
from collections import Counter


def sql_to_skeleton(sql: str) -> str:
    """
    Strip literals and specific identifiers from SQL to get structural skeleton.

    'SELECT region, SUM(revenue) FROM orders WHERE date > '2024-01-01' GROUP BY region'
    becomes:
    'SELECT {col}, {agg}({col}) FROM {table} WHERE {col} > {literal} GROUP BY {col}'
    """
    # Replace string literals
    skeleton = re.sub(r"'[^']*'", "{literal}", sql)
    # Replace numbers
    skeleton = re.sub(r"\b\d+\.?\d*\b", "{num}", skeleton)
    # Normalize whitespace
    skeleton = re.sub(r"\s+", " ", skeleton).strip()
    return skeleton


def cluster_skeletons(query_log: list[dict]) -> list[dict]:
    """
    Group queries by skeleton structure.
    Returns the most common patterns with example queries.

    query_log: [{"question": "...", "sql": "..."}, ...]
    """
    skeleton_groups = {}

    for entry in query_log:
        skel = sql_to_skeleton(entry["sql"])

        if skel not in skeleton_groups:
            skeleton_groups[skel] = {
                "skeleton": skel,
                "count": 0,
                "examples": []
            }

        skeleton_groups[skel]["count"] += 1
        if len(skeleton_groups[skel]["examples"]) < 3:
            skeleton_groups[skel]["examples"].append(entry)

    # Sort by frequency
    sorted_groups = sorted(
        skeleton_groups.values(),
        key=lambda x: x["count"],
        reverse=True
    )

    return sorted_groups[:30]  # Top 30 patterns


def generate_skeleton_entry(group: dict) -> dict:
    """
    Use an LLM to generate a natural language description and
    proper slot definitions from example queries.

    In production, call your LLM here. For bootstrapping, you can
    do this manually for the top 20-30 patterns.
    """
    return {
        "skeleton_id": f"pattern_{hash(group['skeleton']) % 10000:04d}",
        "description": "",           # Fill via LLM or manually
        "sql_template": "",          # Parameterize the skeleton
        "slots": "{}",              # Define slot names and types
        "complexity_tags": "",
        "frequency": group["count"],
        "examples": group["examples"]
    }


if __name__ == "__main__":
    # Load your query logs
    # query_log = load_query_logs()
    # patterns = cluster_skeletons(query_log)
    # for p in patterns:
    #     print(f"[{p['count']}x] {p['skeleton'][:100]}...")
    #     for ex in p['examples']:
    #         print(f"  Q: {ex['question']}")
    pass
```

---

## 7. Latency Budget Breakdown

| Step | Component | Expected Latency |
|------|-----------|-----------------|
| 1 | Coordinator routes query | 1-2s |
| 2 | Semantic cache check | 2-5ms |
| 3 | Skeleton match (vector search) | 2-5ms |
| 4 | Coarse domain filter (tag lookup) | <1ms |
| 5 | Deep schema retrieval (HNSW) | 3-10ms |
| 6a | LLM: slot-fill (skeleton hit) | 2-4s |
| 6b | LLM: generate from scratch (no skeleton) | 5-10s |
| 7 | SQL execution | 1-3s |
| 8 | Cache store | 1-2ms |
| **Total (skeleton hit)** | | **5-10s** |
| **Total (no skeleton)** | | **8-15s** |
| **Total (cache hit)** | | **1-2s** |

---

## 8. Key Configuration Knobs

| Parameter | Recommended Start | Tune When |
|-----------|------------------|-----------|
| Semantic cache threshold | 0.08 distance (≈0.92 similarity) | Wrong SQL served from cache → raise to 0.05 |
| Skeleton match threshold | 0.10 distance (≈0.90 similarity) | Wrong template selected → raise to 0.07 |
| Simple agent retrieval depth | 5 fragments | Simple queries failing → raise to 8 |
| Complex agent retrieval depth | 15 fragments | Complex queries failing → raise to 20 |
| HNSW M parameter | 16 | >10K fragments → raise to 32 |
| HNSW ef_construction | 200 | Low recall → raise to 400 |
| HNSW ef_runtime | 100 | Latency too high → lower to 50 |
| Cache TTL | 24 hours | Stale results → lower; stable schema → raise |
| Embedding model | all-MiniLM-L6-v2 (384d) | Poor retrieval → try bge-small-en-v1.5 (384d) |

---

I’d suggest positioning the Friday demo around **three messages**:

1. **Semantic layer improves correctness**
2. **Latency improves by reducing unnecessary work**
3. **Metadata/feedback improves the model over time**

### 1. Demo story: with vs without semantic layer

Use two simple chatbots or notebooks side by side.

Same user question:

> “What are my net sales for March?”

**Without semantic layer:**
The model sees raw tables and guesses. It may write something like:

```sql
SELECT SUM(net_revenue)
FROM transactions
WHERE month = 'March';
```

This may be wrong because the business definition of net sales is not simply `net_revenue`.

**With semantic layer:**
The model has access to business definitions:

```text
Net Sales = Gross Sales - Refunds
Gross Sales = SUM(transaction_amount)
Refunds = SUM(refund_amount)
```

So it generates:

```sql
SELECT 
  SUM(transaction_amount) - SUM(refund_amount) AS net_sales
FROM sales
WHERE month = 'March';
```

The point to land:

> “The semantic layer prevents the model from guessing business logic. It gives the AI certified definitions, relationships, and terminology.”

### 2. Keep the sample data tiny

Use maybe three tables:

**sales**

| order_id | merchant_id | month | transaction_amount |
| -------- | ----------- | ----- | ------------------ |
| 1        | M001        | March | 100                |
| 2        | M001        | March | 200                |

**refunds**

| refund_id | order_id | refund_amount |
| --------- | -------- | ------------- |
| R1        | 2        | 50            |

**semantic_metrics**

| metric_name | definition                              | sql_logic                                    |
| ----------- | --------------------------------------- | -------------------------------------------- |
| net_sales   | gross sales minus refunds               | SUM(transaction_amount) - SUM(refund_amount) |
| gross_sales | total transaction amount before refunds | SUM(transaction_amount)                      |
| refunds     | total refunded amount                   | SUM(refund_amount)                           |

Then show:

* Without semantic layer: answer = 300 or wrong metric
* With semantic layer: answer = 250 and explains formula

### 3. Latency demo: show before and after

For latency, don’t try to solve everything in the demo. Show the **approach**.

You can say  AI latency is improved by reducing how much the model has to reason over and by avoiding repeated work.

The key latency levers are:

| Area                 | Latency issue                      | Improvement                                         |
| -------------------- | ---------------------------------- | --------------------------------------------------- |
| Metadata retrieval   | Model sees too much schema/context | Retrieve only relevant domain metadata              |
| Semantic definitions | Recomputed every time              | Cache metric definitions and joins                  |
| SQL generation       | Large model does every step        | Use smaller/faster model for routing/classification |
| Query execution      | Raw table scans                    | Use pre-aggregated tables/materialized views        |
| Repeated questions   | Same or similar query reruns       | Cache previous SQL/results                          |
| User experience      | User waits for full answer         | Stream intermediate response/status                 |

Suggested demo flow:

**Notebook A: unoptimized**

```text
User question
→ Load all metadata
→ Send large prompt to model
→ Generate SQL
→ Execute SQL
→ Return answer
```

Show total latency: maybe 8–12 seconds.

**Notebook B: optimized**

```text
User question
→ Classify domain: Sales
→ Retrieve only Sales semantic layer
→ Reuse cached metric definition
→ Generate SQL
→ Execute optimized query
→ Return answer
```

Show total latency: maybe 2–4 seconds.

Even if the timing is simulated, make it clear it is illustrative.

### 4. What to say when asked: “How are we improving latency?”

Use this answer:

> “We are improving latency by reducing the amount of context the model needs, routing the question to the right domain, caching reusable semantic definitions, using optimized SQL or pre-aggregated data where possible, and measuring latency at each step. The goal is not just to make the model faster, but to remove unnecessary work before the model is called.”

Then add:

> “The semantic layer also helps latency because the model does not need to infer business logic from raw tables every time. It can directly use certified definitions like net sales, refunds, active merchant, churn, or approval rate.”

### 5. Metadata and model improvement story

This is important. I would avoid saying “retrain” too strongly unless you actually plan to fine-tune. Say **metadata-driven improvement** first.

The system should capture:

* User question
* Domain selected
* Metric selected
* Tables used
* Generated SQL
* Whether SQL executed successfully
* Final answer
* User feedback
* Corrected query if the answer was wrong
* Latency breakdown

Then this becomes a feedback loop:

```text
User questions + generated SQL + feedback
→ evaluation dataset
→ better semantic definitions
→ better examples/few-shot prompts
→ better routing
→ possible fine-tuning later
```

Suggested phrasing:

> “We are building metadata not just as documentation, but as training and evaluation fuel. Every question, SQL query, selected metric, failure, correction, and latency measurement becomes part of the improvement loop. Initially this helps us improve prompts, semantic definitions, and routing. Over time, once we have enough high-quality examples, it can support fine-tuning or model retraining.”

### 6. Recommended Friday demo structure

I’d propose this sequence:

**Part 1 — Business problem**
“AI can answer data questions, but without business context it may use the wrong metric.”

**Part 2 — Side-by-side demo**
Ask: “What are my net sales?”
Show wrong/ambiguous answer without semantic layer.
Show correct answer with semantic layer.

**Part 3 — Latency demo**
Show baseline flow vs optimized flow.
Break latency into retrieval, model, SQL, and response time.

**Part 4 — Metadata improvement loop**
Show how every interaction improves future accuracy and speed.

**Part 5 — Close with roadmap**
“Next we expand certified metrics, add feedback capture, benchmark latency, and prioritize high-value domains.”

### 7. Strong closing message

Use this:

> “The semantic layer improves accuracy because the AI uses certified business definitions instead of guessing. The latency work improves speed because we reduce unnecessary context, cache what is reusable, and route questions more intelligently. The metadata layer then creates a feedback loop so  AI gets better over time.”


## 9. Reference Links

### Core Repos
- **ADK + Redis integration:** [redis-developer/adk-redis](https://github.com/redis-developer/adk-redis)
- **RedisVL (Python SDK):** [redis/redis-vl-python](https://github.com/redis/redis-vl-python)
- **Redis AI resources:** [redis-developer/redis-ai-resources](https://github.com/redis-developer/redis-ai-resources)
- **Semantic caching tutorial:** [redis-developer/semantic-caching-with-redis-langcache](https://github.com/redis-developer/semantic-caching-with-redis-langcache)

### Text-to-SQL Patterns
- **DAIL-SQL (skeleton matching):** [Charles-Tian/nl2sql](https://github.com/Charles-Tian/nl2sql)
- **Text-to-SQL architectures for BigQuery:** [arunpshankar/LLM-Text-to-SQL-Architectures](https://github.com/arunpshankar/LLM-Text-to-SQL-Architectures)
- **NL2SQL handbook:** [HKUSTDial/NL2SQL_Handbook](https://github.com/HKUSTDial/NL2SQL_Handbook)
- **Awesome Text2SQL:** [eosphoros-ai/Awesome-Text2SQL](https://github.com/eosphoros-ai/Awesome-Text2SQL)
- **PremSQL (end-to-end):** [premAI-io/premsql](https://github.com/premAI-io/premsql)
- **AgentSM paper (trajectory reuse):** [arxiv.org/abs/2601.15709](https://arxiv.org/abs/2601.15709)

### Blogs
- [Google Cloud: Techniques for Improving Text-to-SQL](https://cloud.google.com/blog/products/databases/techniques-for-improving-text-to-sql)
- [Building a Semantic Layer for GenBI](https://medium.com/@kapildevkhatik2/text-to-sql-is-finally-production-ready-building-a-semantic-layer-for-genbi-0127c1127574)
- [Text-to-SQL Patterns for BigQuery](https://medium.com/google-cloud/architectural-patterns-for-text-to-sql-leveraging-llms-for-enhanced-bigquery-interactions-59756a749e15)
- [Redis: What is Semantic Caching](https://redis.io/blog/what-is-semantic-caching/)
- [Redis: Cache Optimization Strategies](https://redis.io/blog/guide-to-cache-optimization-strategies/)
- [Low-Latency Table Retrieval with Small Embedding Models](https://medium.com/@brijeshrn/schema-conditioned-small-embedding-models-for-low-latency-table-retrieval-in-cross-domain-4095993a8420)
- [ADK Multi-Agent Patterns](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/)

### Courses
- [DeepLearning.AI: Semantic Caching for AI Agents](https://learn.deeplearning.ai/courses/semantic-caching-for-ai-agents/) (free, uses Redis + RedisVL)
- [DeepLearning.AI: Improving Accuracy of LLM Applications](https://www.deeplearning.ai/short-courses/improving-accuracy-of-llm-applications/) (text-to-SQL + LoRA)
- [DeepLearning.AI: LLMOps](https://www.deeplearning.ai/short-courses/llmops/) (BigQuery + fine-tuning pipeline)
