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


Part 2

Hello everyone, I want to walk through the proposed location permission fix in simple terms.

The issue is not that users are missing data, and it is not that the reporting system is broken. The issue is that our permission check is too broad for a common use case.

Today, some users only have access to one location. That is normal and expected. But in some report flows, our current logic effectively asks, “Does this user have permission across all locations?” before allowing the query.

That means even if a user has valid permission for their own location, they can still be blocked with a “table access denied” error.

The proposed fix changes that logic for single-location users.

Instead of asking, “Does this user have permission everywhere?” we ask, “Does this user have permission for the specific location they are trying to view?”

So if a user is viewing one location, we check that exact location. If they have permission for it, we allow the report. If they do not, we block them.

For users viewing all locations, nothing changes. That flow still uses the stricter all-location validation, because viewing all locations is a broader access case and should remain more restricted.

So the behavior becomes more accurate without becoming more permissive.

Why does this matter?

It fixes a frustrating product issue where legitimate users were denied access to reports they should be able to see. For example, a restaurant manager may only be allowed to see one store. That is valid access. The system should not treat that manager as if they are trying to access the entire business.

The new logic better matches the product expectation:

If I have access to my location, I should be able to see my location’s reports.

If I do not have the required sales permission, I should be blocked.

And if I am trying to view all locations, the stricter access checks still apply.

This fix is focused specifically on the permission-denied category of reporting issues. Sonia’s testing found several categories of issues, but this change addresses cases where a user has valid access to one or a few locations and is incorrectly blocked because the system checks the full account or full location set.

It does not fix unrelated issues, such as test accounts with no sales data in BigQuery, semantic-layer example query gaps, or questions that map to tables we have not fully modeled yet. Those are separate follow-up items.

The value of this fix is that it removes false access-denied blockers for users who do have valid location-level permission.

The fix also addresses a related single-location time zone edge case. Date-based reporting will now use the selected location’s time zone instead of treating the user’s full location list as ambiguous.

From a security perspective, this does not open up data across all locations. The report query only receives the location the user is allowed to view. We are not giving the model or query engine a broader list of locations. We are narrowing the check to the user’s actual location context.

So this is not loosening security. It is correcting an overly strict check that was blocking valid single-location users.

The expected impact is fewer false “table access denied” errors for users who have partial but valid location permissions.


Yes. The strongest point is to **separate the minimal permission fix from the broader product behavior change**.

The current PR sounds safe if it only says: **when the request is already scoped to one selected location, validate permission for that location only.**
The bigger discussion is: **should the assistant default to the clicked-in location instead of all locations?** That is a product behavior change and should probably be documented.

You can use this:

A good way to frame this is that there are two separate decisions here.

First, there is the narrow permission fix. If the user is already operating in a single-location context, we should not validate them as if they are trying to access every location. We should check whether they have the required permission for the selected location. If yes, allow the report. If no, block it. That is the current fix, and it does not make access more permissive because the query still only receives the selected location.

Second, there is a broader product behavior question: when a user asks a general reporting question, should we default to all allowed locations, or should we default to the location they are currently clicked into? That is separate from the permission bug fix. If we change that default behavior, then yes, we should document it and probably align with product, support, and anyone responsible for the user guide.

For Brian’s concern about trade-offs, I agree there is a real trade-off. If we continue to default every broad question to all locations, then partial-permission users may still get blocked when they ask broad questions involving restricted report types. But I would not want us to silently reduce the report scope without making that clear to the user. A safer product experience would be either:

“One or more locations are not available for this report. Please select a specific location.”

Or:

“You only have permission to view this report for Location A. Do you want to run the report for Location A?”

That avoids exposing unauthorized data, and it also avoids giving the user a partial answer while making them think it covers the whole business.

For the current PR, my recommendation would be to keep the scope narrow: fix the false denial when the selected context is one location. All-location requests should continue to use the stricter validation. That gives us an immediate improvement without changing the entire reporting philosophy.

Then, as a follow-up product decision, we can decide whether the assistant should default to the clicked-in location, all accessible locations, or ask the user to clarify when permissions differ by location.

The important safety requirement is that the permission check and the query scope must match. If we validate the user for one location, the query should only receive that one location. We should not validate narrowly and then query broadly.

So my answer is: I support the proposed fix for single-location context. I also agree that changing the default from “all locations” to “current location” is a bigger UX/product decision and should be called out in the guide or release notes.

A few strong points you can raise in the discussion:

**1. Clarify that the PR is not solving every reporting issue.**
It only fixes false “table access denied” cases caused by overly broad location permission validation.

**2. Do not silently downgrade “all locations” to “one location” unless product agrees.**
That could confuse users because they may think they are seeing all-business results when they are only seeing one location.

**3. Best safe behavior for partial permissions:**
If the user asks broadly and only has permission for some locations, either ask them to pick a location or clearly state the narrowed scope.

**4. Permission and query scope must always match.**
If we check permission for Location A, then only Location A should be sent to the query engine.

**5. Defaulting to the clicked-in location is reasonable, but it is a UX change.**
That should be documented because users may currently expect broader analysis.

**6. The fix should fail closed.**
If location context is ambiguous, missing, or contains multiple locations, use the stricter validation or ask for clarification rather than guessing.

A concise answer you can say live:

> I think the current PR is safe as long as it only applies when the user is clearly in a single-location context. We are not expanding access; we are aligning the permission check with the actual location being queried. But I agree there is a separate product question around default behavior. If we want to default to the clicked-in location instead of all locations, that should be documented and aligned with product because it changes user expectations.


Yes — the answer should be:

**We should not always treat the default as “all locations the user has access to” if the user’s session is already scoped to a specific location.**

A cleaner way to explain it:

I think we should distinguish between the user’s full accessible location list and the effective location scope for the current report request.

The user may technically have access to multiple locations, but if their current session or UI context is scoped to one selected location, then the report request should use that selected location as the effective scope.

So in that case, we should not default the query to all locations the user has access to. We should validate and query only against the selected location.

If the user explicitly asks for all locations, or if there is no selected location context, then we can fall back to the broader “all accessible locations” behavior and apply the stricter permission validation.

So the precedence should probably be:

1. If the user explicitly asks for a specific location, use that location and validate permission for it.
2. If the session has a selected location ID, use that selected location and validate permission for it.
3. If the user asks for all locations, use all accessible locations and apply stricter validation.
4. If the scope is ambiguous, either use the current selected location or ask the user to clarify, depending on the product decision.

That way, the permission check matches the actual query scope. We avoid false “table access denied” errors, but we also avoid accidentally giving broader access than intended.

The key point to say live is:

> The default should be based on the **effective report context**, not always the user’s full location access list.

So if the session is already filtered by `locationId`, then the default for that report should be that location. The user’s full accessible location list still matters for validating what they *could* access, but it should not automatically become the query scope.


Here’s a brief summary you can send to A team:

Hi all,

I wanted to summarize the reporting access issue we discussed today and the current direction.

We have been investigating cases where users receive “table access denied” errors when asking reporting questions. One identified issue is that our current permission validation can be too broad for users who only have access to a specific location, or who have limited permissions such as access to only certain report categories like tips, payroll, or sales.

The initial proposed fix was focused on a narrower case: when the user is clearly scoped to a single location, we would validate their permission against that selected location instead of checking whether they have permission across all locations. This would help reduce false access-denied errors for legitimate single-location or location-level users, while still keeping stricter validation in place for true all-location requests.

However, during the discussion, we agreed that the larger issue may require a broader requirements pass. The core question is not only how to fix the single-location permission check, but also how the reporting experience should behave when users have partial permissions across locations or report categories.

For example, we need to clarify:

* Should the assistant default to the user’s currently selected location or all locations the user can access?
* If a user has permission for only some report types, such as tips but not payroll, how should the assistant determine intent before running the query?
* If the user asks a broad question but only has permission for part of the requested data, should we narrow the scope, ask for clarification, or block the request?
* How do we make sure the permission check always matches the actual query scope?

The current PR may still be useful for the specific single-location false-denial scenario, but we should treat it as part of a larger product and permission-model discussion rather than the complete solution.

The proposed next step is for Autumn and me to take a step back, define the broader requirements, and then review the approach with Luke and the team. I will also sync with the dev team so we are aligned before we move forward or send out a final recommendation.

Thanks,
Henry

A good shorter version to say internally is:

> The single-location fix is valid, but the bigger issue is broader than one permission check. We need requirements around location scope, report-category permissions, and how the assistant should behave when a user has partial access. The safest next step is to align on the expected product behavior before treating the current PR as the full solution.
