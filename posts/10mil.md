---
title: RAG Layer
date: 2026-05-09
---

## Meeting summary

The team discussed the current state and scope of the Payments help-center chatbot, which reuses much of the existing Marketing and Tech Ops chatbot infrastructure.

### Current status

* The application is functioning.
* Trenton has the development environment running and is close to completing QA infrastructure.
* Most backend and ingestion components already exist.
* The immediate priority is to complete development and QA—not to design the final production-scale architecture yet.

### Remaining technical work

* Review the application code and make a few content-filtering adjustments.
* Investigate metadata and ingestion issues caused by the move between GitHub tenants.
* Check the ingestion pipeline for silent failures.
* Validate Trenton’s infrastructure work from an application perspective.
* Help the front-end portal team connect to the backend APIs.
* Complete the load balancer-to-Apigee integration, including any required service account creation or allowlisting.
* Confirm whether Model Armor has been configured correctly.
* Create DNS records once dependencies are unblocked.
* Test model alternatives for response quality, cost, and deprecation risk.
* Complete the remaining ATO activities. Network and application penetration testing may be out of scope.

### Major unresolved issue: expected traffic

Two very different traffic estimates were discussed.

One estimate attributed to Sarisha projected:

* 10 million total users
* 2 million daily active users
* 100,000 concurrent users
* 10 queries per user per day
* Peaks approaching 10,000 requests per minute

At that scale, the current solution would require a substantially different production architecture and could generate model and infrastructure costs around $100,000 per month.

Mark strongly questioned those assumptions. His rough estimate was closer to:

* 100,000 portal visitors per month
* Approximately 10,000–30,000 chatbot users per month
* Potentially fewer than 100,000 chatbot users in an entire year

The traffic assumptions need to be validated before production architecture, capacity planning, or funding decisions can be made.

### Cost concerns

* Model inference is expected to be the largest operating expense.
* The ingestion process itself is likely inexpensive—potentially only a few dollars per day.
* The current full-repository synchronization could potentially be optimized to ingest only changes, but it is not believed to be a major cost driver.
* Production rollout should not proceed until the business confirms the actual audience, expected usage, and available operating budget.

### Agreed direction

Complete development and QA using the current approach. In parallel, validate the traffic assumptions with Sarisha and determine whether the business case supports the cost and architecture required for a production deployment.

### Action items

| Owner                        | Action                                                                                         |
| ---------------------------- | ---------------------------------------------------------------------------------------------- |
| Sar / business team      | Confirm the source and accuracy of the user, concurrency, and query-volume estimates.          |
| Mark / engineering           | Review application code, content filtering, metadata handling, and ingestion behavior.         |
| Trent                      | Finish QA infrastructure and confirm Model Armor and other infrastructure tasks are complete.  |
| Engineering / infrastructure | Resolve the Apigee and load-balancer connection, including service-account requirements.       |
| Front-end team               | Validate connectivity to the backend APIs.                                                     |
| Security / project team      | Complete remaining ATO activities and confirm penetration-testing scope.                       |
| Architecture / finance       | Produce a production design and cost estimate after realistic usage assumptions are confirmed. |
