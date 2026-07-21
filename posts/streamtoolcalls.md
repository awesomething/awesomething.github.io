---
title: Stream Tool Calls
date: 2023-11-21
---

Yes, **But thread pooling should only wrap synchronous/blocking tool implementations**. For ADK streaming, keep the orchestration on the asyncio event loop.

Google ADK for Python already attempts to execute independent function-tool calls concurrently starting with version 1.10.0, so you may not need to implement your own parallel dispatcher. ([ADK][1])

### Why not replace `asyncio.gather()` entirely with a thread pool?

`asyncio.gather()` coordinates async operations without occupying OS threads while they wait for network I/O. A `ThreadPoolExecutor` is primarily useful when a tool:

* Uses a synchronous HTTP/database SDK
* Performs blocking file I/O
* Calls a library that has no async API
* Would otherwise block ADK’s streaming event loop

For genuinely async tools, sending them to threads usually adds overhead and complicates cancellation, exception propagation, context variables, and streaming.

A good mixed approach is:

```python
import asyncio
from collections.abc import Callable
from typing import Any


async def run_blocking_tool(
    tool: Callable[..., Any],
    *args: Any,
    **kwargs: Any,
) -> Any:
    # Runs the synchronous function in asyncio's thread pool.
    return await asyncio.to_thread(tool, *args, **kwargs)


async def run_tools() -> list[Any]:
    return await asyncio.gather(
        async_network_tool(),
        async_database_tool(),
        run_blocking_tool(sync_legacy_tool, "input"),
    )
```

### For streaming results as each tool completes

`gather()` waits until all calls finish before returning its combined result. To process completions incrementally, use `asyncio.as_completed()`:

```python
async def stream_tool_results():
    tasks = [
        asyncio.create_task(async_tool_a()),
        asyncio.create_task(async_tool_b()),
        asyncio.create_task(
            asyncio.to_thread(sync_tool_c)
        ),
    ]

    for completed in asyncio.as_completed(tasks):
        try:
            result = await completed
            yield result
        except Exception as exc:
            yield {"error": str(exc)}
```

This lets your application emit a tool result immediately after that particular call finishes. It does **not necessarily mean ADK or the model can consume partial tool responses mid-turn**; that depends on ADK’s event and runner flow. Usually each function-call response remains an atomic result, while your surrounding application can stream status and completion events.

For structured cancellation and error handling on modern Python, `TaskGroup` is preferable:

```python
async def run_parallel():
    async with asyncio.TaskGroup() as group:
        task_a = group.create_task(async_tool_a())
        task_b = group.create_task(asyncio.to_thread(sync_tool_b))

    return task_a.result(), task_b.result()
```

**Recommended architecture:** let ADK parallelize independent tool calls, define tools as `async def` when their dependencies support async I/O, and use `asyncio.to_thread()` only at the boundary around blocking synchronous code. Avoid running the whole ADK streaming loop inside a thread pool.

[1]: https://adk.dev/tools-custom/performance/?utm_source=chatgpt.com "Increase tool performance with parallel execution - Google ADK"


## Recommendations

1. **Preserve the existing streaming contract.**
   Keep the current `StreamingPayloadV2` JSON structure unchanged, especially required fields such as `user_id`, `session`, `event_type`, `event_name`, `content`, and metadata. Downstream consumers may rely on undocumented assumptions.

2. **Change execution, not the external payload.**
   Thread pooling or parallel tool execution can be introduced internally, but the resulting events should still pass through the existing serializer and validation layer before being streamed.

3. **Treat ordering as part of the contract.**
   Parallel execution may cause responses to arrive in a different order. Add fields such as `tool_call_id`, `sequence_number`, or `parent_event_id` within the existing extensible metadata section rather than changing established fields.

4. **Do not stream raw tool responses.**
   Tool outputs may differ in shape or contain non-JSON-serializable values. Normalize every result into the existing `content` field and retain the same event naming conventions.

5. **Add concurrency behind a feature flag.**
   Keep the current implementation as the default while enabling the new thread-pool or concurrent execution path for selected environments, sessions, or consumers.

6. **Version only when compatibility cannot be maintained.**
   Create a new API or payload version only when the required behavior fundamentally changes the schema, event semantics, ordering guarantees, or error handling. Keep the existing endpoint intact during migration.

7. **Define error and timeout events explicitly.**
   A failed parallel tool call should produce a valid existing-format event rather than terminate the entire stream. Decide whether other calls continue, are cancelled, or return partial results.

8. **Run contract and security testing again.**
   Even without changing the public JSON schema, concurrency changes can affect race conditions, data leakage between sessions, logging, cancellation, authorization context, and response ordering. Regression, load, and penetration testing are appropriate.

### Recommended architecture

```text
ADK tool calls
    ↓
Concurrent execution
    ↓
Result normalization
    ↓
Existing StreamingPayloadV2 builder
    ↓
Schema validation
    ↓
Existing streaming API
```

The safest approach is to **retain the current API and payload contract while replacing only the internal execution mechanism**. Introduce a new API version only if the new streaming behavior cannot be represented without changing existing semantics.
