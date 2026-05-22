---
title: LOcaTioN
date: 2024-12-15
---

Here’s a cleaned-up summary of what was said:

The main point is that we should **build on the existing location-access model rather than reinvent it**, but there are some real complications.

`show all locations` seems unused or irrelevant today. `location_id` is intended to represent the **single location the user is actively looking at**, so when a user asks something like “What are my sales at this location?”, the system can filter the query to that location.

The hard part is that access is not one-dimensional. A user may have access to **sales data across many locations**, but only have access to **payroll data for some locations**. Because of that, narrowing the location list correctly would require first understanding what kind of question the user is asking: sales, payroll, or something else. That likely means an extra LLM classification step before the main flow, which could add latency.

One possible product discussion is whether there are common cases we can optimize for. For example, if a store manager only has sales and payroll access for one location, maybe we can automatically limit the location context to that one location and ignore the rest of the account’s locations when building the location list.

Overall concern: this will likely affect a lot of users, especially store managers, so it is worth discussing with Product before deciding how much complexity to take on.

Here’s the cleaned-up version of the added discussion:

The expected workflow is that the NLM query already has access to the **account ID** and, where applicable, the **location ID**. These are already available in the current flow, so the proposal is to reuse them rather than introduce a completely new mechanism.

If the user does **not** provide a location in the natural-language question, the system should use the user’s **allowed location list**. If the user does provide a location hint, for example, “What are the sales for my Pune location?”, then an LLM call can be used to match that location mention against the locations the user has access to.

Once the location is resolved, the generated SQL should be inspected before execution to ensure that the required filters are present. The query should always include the **account ID from the token** and the resolved/allowed **location IDs**. If those required values are missing, the query should not execute and should instead exit gracefully.

The current safety mechanism is largely handled through **TVFs**. The team has effectively replaced direct PTC access with **TVF-backed PTCs** that require `account_id` and `location_id` as parameters. Because those parameters are mandatory, the query cannot be invoked without the appropriate access constraints.

The initially agreed workflow did **not** rely heavily on the JWT `location_id`, because that field supports only a single integer value. It cannot represent multiple locations. A value like `0` generally means “all locations,” which is mainly associated with admin or super-user accounts. For non-admin users, relying on the JWT `location_id` is not sufficient for RBAC because their permissions may involve multiple allowed locations or more granular access.

Henry’s point was that although the JWT `location_id` has limitations, it can still be useful for the most common querying scenarios and should probably be used where it makes sense. However, the broader location-access problem still needs product input, because permissions appear to be defined at a very granular level, almost at the query level.

A possible final framing:

> Use the existing account/location access model as the foundation. Resolve location from the user query only when the user explicitly hints at one; otherwise, fall back to the allowed location list. Enforce access at execution time through TVFs that require account and location parameters, and do not execute any generated SQL unless those filters are present. The JWT `location_id` can help in common cases, but it should not be the sole source of truth for RBAC because it only supports a single value or “all.”
