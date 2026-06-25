---
title: ML Discuss
date: 2025-01-01
---

## Approaches

Let's frame Sales Forecasting as Problem #1, but the "first approach" should not stop there. Based on the conversation, the broader Phase 1 approach could be:

> Run scheduled analytical checks across sales, labor, tickets, dayparts, item performance, and channels; compare results against expected baselines; then surface meaningful exceptions as alerts or dashboard insights.

So instead of building only a forecast engine, you build an **Insight + Alerting Foundation**.

### First Approach Problems Besides Sales Forecasting

#### 1. Sales Forecasting

**Problem:** Can we predict expected sales for a vendor/location for a future day or period?

**Example:** "Expected sales for Wednesday are $8,400 based on prior Wednesdays, seasonality, holidays, and recent trends."

This is the core forecasting problem.

#### 2. Labor Efficiency / Staffing Variance

**Problem:** Are labor hours aligned with expected or actual sales volume?

This is important because the owner-facing queries already include staffing and labor hours.

**Example insight:** Labor hours were 12% higher than expected yesterday while sales were 8% below expected.

This could identify overstaffing, understaffing, or inefficient scheduling.

**Inputs:**

- Labor hours
- Sales
- Ticket count
- Daypart
- Business type
- Historical labor-to-sales ratio

**Possible metrics:**

- Labor cost as % of sales
- Sales per labor hour
- Tickets per labor hour
- Labor hours by daypart

**Why this belongs in Phase 1:** It uses internal data, creates clear business value, and does not require external APIs.

#### 3. Daypart Performance Analysis

**Problem:** Which part of the day is driving the sales change?

Dayparts are configurable time windows like breakfast, lunch, dinner, late night, etc.

**Example insight:** Lunch sales were down 18% compared to the prior four Wednesdays, while dinner was normal.

This is more actionable than saying total sales were down.

**Inputs:**

- Transaction timestamp
- Vendor-configured dayparts
- Sales
- Ticket count
- Average ticket size
- Order channel

**Possible metrics:**

- Sales by daypart
- Tickets by daypart
- Average ticket size by daypart
- Labor hours by daypart
- Item performance by daypart

**Why this belongs in Phase 1:** It helps explain where the issue happened without requiring advanced forecasting.

#### 4. Labor-to-Daypart Alignment

**Problem:** Was staffing aligned with actual demand during each daypart?

This combines the labor conversation with daypart analysis.

**Example insight:** Breakfast had 22% fewer tickets than expected, but labor hours were unchanged from the normal Wednesday average.

Or: Dinner sales exceeded expectations by 15%, but labor coverage was 10% below normal.

**Inputs:**

- Labor hours by time window
- Sales by daypart
- Ticket count by daypart
- Historical averages

**Possible metrics:**

- Sales per labor hour by daypart
- Tickets per labor hour by daypart
- Labor variance vs demand variance

**Why this is useful:** It gives the owner a specific operational problem, not just a sales observation.

#### 5. Order Channel Mix Analysis

**Problem:** Did sales shift between in-store, online, DoorDash, Uber Eats, or other channels?

This came up directly in the conversation. You likely have this data somewhere already.

**Example insight:** Total sales were flat, but third-party delivery increased 24% while in-store sales declined 15%.

That changes the interpretation completely.

**Inputs:**

- Order channel
- Sales
- Ticket count
- Average ticket size
- Daypart
- Vendor/location

**Possible metrics:**

- Sales by channel
- Tickets by channel
- Average ticket size by channel
- Channel share %
- Channel growth/decline

**Why this belongs in Phase 1:** It is internal, likely already captured, and can explain sales movement better than total revenue alone.

#### 6. Ticket Count vs. Average Ticket Size Analysis

**Problem:** Did revenue change because fewer people ordered, or because people spent more or less per order?

**Example insight:** Revenue was up 6%, but ticket count was down 11%. The increase was driven by higher average ticket size, not more volume.

**Inputs:**

- Ticket count
- Ticket total
- Sales
- Average ticket size
- Daypart
- Channel

**Possible metrics:**

- Tickets
- Average ticket size
- Revenue per ticket
- Ticket count variance
- Ticket size variance

**Why this matters:** Sales alone can hide the real issue. A business may be making more revenue but serving fewer customers.

#### 7. Item-Level Performance Detection

**Problem:** Which menu items are meaningfully up or down?

**Example insight:** Burger sales were down 28% yesterday compared to the normal Wednesday dinner average.

**Inputs:**

- Item sales
- Quantity sold
- Revenue by item
- Daypart
- Channel
- Historical item performance

**Possible metrics:**

- Item quantity sold
- Item revenue
- Item share of sales
- Item trend vs baseline

**Phase 1 caution:** This is useful, but only if item naming and mapping are reliable. If item data is messy, this may need to be limited to top-selling items first.

#### 8. Ingredient or Plate-Level Monitoring

**Problem:** Did usage or sales of a key ingredient/plate deviate from normal?

This was mentioned in the discussion, but it may be harder than item-level analysis.

**Example insight:** Chicken-based menu items were down 19% yesterday compared to the recent Wednesday average.

**Inputs:**

- Menu item sales
- Ingredient mapping
- Plate/category mapping
- Historical usage or sales

**Phase 1 caution:** Only include this if the ingredient-to-item mapping already exists. Otherwise, this becomes a data normalization project.

#### 9. Scheduled Query Automation / Insight Detection

This may actually be the best way to describe the first approach.

**Problem:** Can we automate known business questions and run them on a schedule?

Instead of asking an LLM to freely analyze everything, define a set of trusted analytical checks.

**Example scheduled checks:**

- Was yesterday's sales materially above or below expected?
- Which daypart caused the variance?
- Did labor hours align with sales?
- Did ticket count drop?
- Did average ticket size increase or decrease?
- Did sales shift by order channel?
- Did any top items significantly underperform?
- Were there unusual staffing patterns?

**Output:**

Three alerts generated for Wednesday morning:

- Tuesday sales were 14% below expected.
- Lunch ticket count was down 21%.
- Labor hours remained flat despite lower demand.

This keeps Phase 1 practical and avoids becoming an open-ended analytics science project.

### Recommended Phase 1 "First Approach" List

I would take this to Sai as a focused set of problems:

| Priority | Problem | Phase 1 Fit |
|---|---|---|
| 1 | Sales forecasting | Yes |
| 2 | Labor efficiency / staffing variance | Yes |
| 3 | Daypart performance analysis | Yes |
| 4 | Ticket count vs average ticket size | Yes |
| 5 | Order channel mix analysis | Yes, if data is available |
| 6 | Scheduled insight / alert generation | Yes |
| 7 | Item-level performance detection | Maybe |
| 8 | Ingredient / plate monitoring | Maybe later |
| 9 | Weather impact | Phase 2 |
| 10 | Local events / sports / road closures | Phase 2 |

### Best Way to Position It

I would not say Phase 1 is only "forecasting."

I would say:

> Phase 1 should build an internal-data-driven insight engine that combines simple forecasting, variance detection, labor analysis, daypart breakdowns, ticket analytics, and channel mix analysis. The system should run scheduled checks, compare actual performance against expected baselines, and surface meaningful alerts to the owner dashboard.

That gives you something more useful than a generic analytics dashboard, but still keeps the scope realistic.

### Suggested "First Approach Problems" Wording

**First Approach Problems**

1. **Sales Forecasting** — Predict expected sales for a vendor/location using historical sales, day of week, seasonality, holidays, and recent trends.
2. **Labor Efficiency and Staffing Variance** — Identify whether labor hours were aligned with sales volume, ticket count, and daypart demand. Surface cases where labor appears high relative to sales or low relative to demand.
3. **Daypart Performance Analysis** — Break sales and ticket activity into configured vendor dayparts such as breakfast, lunch, dinner, and late night. Identify which daypart contributed most to overperformance or underperformance.
4. **Ticket Count vs. Average Ticket Size Analysis** — Determine whether sales changes were caused by changes in transaction volume, average ticket size, or both.
5. **Order Channel Mix Analysis** — Analyze sales and ticket trends across channels such as in-store, online, DoorDash, Uber Eats, and other ordering sources where available.
6. **Scheduled Insight and Alert Generation** — Run predefined analytical checks on a scheduled basis and generate alerts when performance materially deviates from expected baselines.
7. **Item-Level Performance Detection** — Identify top menu items that are significantly overperforming or underperforming compared to historical expectations.
8. **Ingredient or Plate-Level Monitoring** — Where reliable item-to-ingredient or plate mapping exists, detect unusual changes in ingredient or plate-level sales patterns.

### Recommended Phase 1 Focus

Phase 1 should focus on internal data sources first: sales, tickets, labor, dayparts, order channels, holidays, and item performance. External signals such as weather, sports, local events, road closures, and supply chain disruptions should be treated as future enhancements unless reliable data already exists internally.

## Details

Based on the discussion, I'd recommend reframing the epic around building a reliable sales forecasting foundation first, then layering external signals later. The team is correctly identifying that some of the proposed Phase 1 items are disproportionately expensive relative to their value.

### What I Heard in the Conversation

There are really two categories of inputs:

**1. Internal signals (high value, low complexity)**

- Historical sales
- Day of week
- Hour of day
- Business type (pub, breakfast restaurant, fine dining, etc.)
- Region/location
- Holidays
- Average ticket size
- Labor costs
- Operational metrics

**2. External signals (potentially valuable, high complexity)**

- Weather APIs
- Sporting events
- Road closures
- Competitor activity
- Local events

The concern raised is valid: weather, sports schedules, and local events are all highly location-specific and require significant data engineering before they add meaningful predictive power.

### First: Clarify "Ticket"

In restaurant and hospitality systems, a ticket almost always means an order/check/transaction.

**Examples:**

| Metric | Meaning |
|---|---|
| Ticket count | Number of orders |
| Average ticket size | Average revenue per order |
| Ticket total | Dollar amount of a specific order |
| Covers | Number of guests served |

A table of four people could generate:

- One ticket ($120)
- Two tickets ($60 each)
- Four tickets ($30 each)

So average ticket size does not directly indicate guest count.

If guest count matters, restaurants usually track:

- Covers
- Party size
- Guests served

The team should verify exactly what the source system means by "ticket."

### Recommended V1 Scope

I would narrow the epic to five core systems:

#### 1. Sales Forecasting Engine

Forecast:

- Daily sales
- Weekly sales
- Hourly sales (if data supports it)

Inputs:

- Historical sales
- Day of week
- Month
- Holiday flags
- Business type
- Region

#### 2. Seasonality Detection

Automatically identify patterns such as:

- Fridays +18%
- Saturdays +32%
- Memorial Day weekend +25%
- January slump -12%

This is actionable and explainable.

#### 3. Business Segmentation Layer

Group customers by:

- Restaurant type
- Region
- Operating hours
- Service style

Examples:

- Pacific Northwest Pub — strong weekend sales, strong evening sales
- Breakfast Cafe — morning concentration, weekday traffic

This allows models to share learnings across similar businesses.

#### 4. Ticket Analytics

Track:

- Ticket count
- Average ticket size
- Revenue per ticket
- Ticket growth trends

This often explains sales changes better than raw revenue.

**Example:**

| Month | Tickets | Avg Ticket | Revenue |
|---|---|---|---|
| Jan | 1,000 | $20 | $20,000 |
| Feb | 900 | $25 | $22,500 |

Sales increased despite fewer orders.

#### 5. Explainability Layer

Instead of simply saying:

> Forecast = $12,000

Provide:

> Expected sales are 11% above normal because:
> - Friday effect (+7%)
> - Holiday weekend (+3%)
> - Seasonal trend (+1%)

Business users trust this much more.

### Move to Phase 2

I'd push all of these out:

**Weather**
Reason: Location-dependent, data quality issues, complex feature engineering. Unless you're forecasting for a single chain, weather is expensive.

**Sports Schedules**
Reason: Only impacts businesses near venues, requires venue mapping, highly bespoke. Great enhancement later, not foundational.

**Road Closures**
Reason: Hard to source consistently, sparse impact.

**Competitor Activity**
Reason: Difficult to obtain reliable data, difficult to quantify.

**Local Events**
Reason: Valuable but operationally expensive, event coverage varies widely by market.

### My Suggested Epic Objective

Instead of:

> "Build a forecasting platform using 9 different prediction approaches"

Use:

> "Build an explainable sales forecasting platform that predicts revenue, ticket volume, and ticket size using internal operational data and seasonality patterns."

That scope is achievable by October, produces business value quickly, and creates a foundation where weather, sports, and event APIs can later be added as incremental signal sources rather than core dependencies.

## Follow-Up Questions

### Forecasting vs. Previous-Day Analytics

- Are we trying to forecast future sales, explain yesterday's performance, or support both?
- For Phase 1, should the primary forecast be daily, with hourly or daypart analysis treated as supporting detail?
- When a user logs in Wednesday morning, should the system show: Tuesday performance analysis, Wednesday forecast, or both?
- Is the "three alerts when the user logs in" experience meant to be based on actuals from the prior day or forecasted risk for the current/future day?
- Should we rename part of this epic from "forecasting" to "sales performance insights" if the first version is primarily retrospective?

### Time Grain and Daypart Logic

- What is the minimum viable time grain for Phase 1: daily, daypart, or hourly?
- Do we have reliable timestamped transaction data to support morning/afternoon/evening analysis?
- How should dayparts be defined across different business types?
  - Breakfast
  - Lunch
  - Dinner
  - Late night
- Should dayparts be configurable by business type or standardized across all customers?
- Do we need same-day intra-day forecasting, such as "based on sales through 10 a.m., today is trending 12% below expected"?

### Alerting and Thresholds

- What types of alerts should be generated in Phase 1?
  - Overall sales decline
  - Item-level decline
  - Ingredient usage anomaly
  - Order channel shift
  - Ticket count decline
  - Average ticket size change
- Should alert thresholds be fixed, like 10%, or adaptive based on historical volatility?
- Should we compare against a rolling average, an exponential moving average, the same weekday average, or a true forecast baseline?
- What is the expected sensitivity of alerts?
  - Fewer, higher-confidence alerts
  - More alerts with possible noise
- Should users be able to configure alert thresholds?

### Baseline and Forecasting Method

- What should the baseline be for comparison?
  - Same day last week
  - Four-week rolling average
  - Exponential moving average
  - Same weekday over prior N weeks
  - Forecasted expected value
- Are we comfortable starting with an exponential moving average or similar lightweight model before introducing a more advanced forecasting model?
- What level of lag is acceptable in the baseline calculation?
- Should the model account for holidays and observed holidays in Phase 1?
- Should CPI/economic data be included now if those models already exist internally, or deferred until the core sales baseline is stable?

### Scope of "Forecasting"

- What does "forecasting" mean for this epic?
  - Predicting tomorrow's sales
  - Predicting the rest of today
  - Predicting next week
  - Detecting when yesterday deviated from expectation
- What forecast horizon matters most to the business user?
  - Remainder of day
  - Tomorrow
  - Next 7 days
  - Next 30 days
- Should Phase 1 include at least one forward-looking forecast so the product does not become only an analytics dashboard?
- What is the minimum forecast output that would feel valuable to customers?
- Should we separate the work into: performance analysis, anomaly detection, forecasting, recommendations?

### User Experience

- Who is the primary user for the alert experience?
  - Owner
  - Store manager
  - Regional manager
  - Operations analyst
- What decisions should the user be able to make from the alert?
  - Adjust staffing
  - Review menu performance
  - Check inventory
  - Investigate channel issues
  - Plan promotions
- What should an alert include besides the metric change?
  - What changed
  - Compared to what
  - Likely reason
  - Recommended action
- Should the system provide an explanation for each alert?
- Should alerts be ranked by severity or business impact?

### Item, Ingredient, and Menu-Level Analysis

- What item-level data is available today?
- Is ingredient-level tracking reliable enough to support alerts?
- How do we map ingredients to menu items or plates?
- Are item names standardized across locations, or will we need normalization?
- Should item-level performance be Phase 1, or should we start with sales, ticket count, and average ticket size?

### Order Channel Analysis

- Which order channels are available in the source data?
  - In-store
  - Online ordering
  - DoorDash
  - Uber Eats
  - Grubhub
  - Catering
- Are third-party delivery channels consistently labeled across customers?
- Should channel mix changes be part of Phase 1 alerts?
- Can we distinguish between lost demand and channel shift?
- Should channel-level forecasting be separate from total sales forecasting?

### External Data Signals

- Are weather, sports, and local events truly Phase 1 requirements, or optional future enhancements?
- Do we already have a weather provider in use anywhere in the company?
- If weather is included later, do we need current weather, historical weather, forecasted weather, or all three?
- Do we have location/address data accurate enough to map businesses to weather, venues, and local events?
- Without physical addresses, how would we determine whether a sports event, road closure, or local event is relevant to a customer?

### Local Events and Sports

- Should local events only apply to businesses with known proximity to venues?
- Do we need a flag for businesses that are event-sensitive, such as food trucks or stadium-adjacent restaurants?
- Do we have a way to identify food trucks today?
- Should event-based forecasting be excluded until we have location enrichment?
- What is the minimum data needed before sports or local events become feasible?

### Customer and Location Data

- Do we have customer location data at all?
- Do we have store-level address, city, ZIP, latitude/longitude, or only merchant-level information?
- Is location data clean and standardized?
- Can one customer have multiple locations with different sales patterns?
- Do we know the business type for each customer, or does that need to be inferred?

### Phase 1 Boundary

- What is the real Phase 1 deliverable: alerting dashboard, previous-day performance analysis, daily sales forecast, or all of the above?
- What should explicitly be out of scope for Phase 1?
- What can we build now that creates a foundation for more advanced forecasting later?
- Which capabilities are required for the October target, and which are stretch goals?
- What would make this feel like a forecasting product rather than just business analytics?

### My Recommended Key Questions to Ask First

The most important ones are:

1. Are we forecasting the future, explaining the past, or doing both?
2. Should Phase 1 be daily forecasting first, with daypart/hourly analysis later?
3. When the user logs in, should they see yesterday's anomalies, today's forecast, or both?
4. What is the baseline comparison: rolling average, exponential moving average, same weekday history, or forecasted expected value?
5. Do we have store-level location/address data?
6. Are weather, sports, and local events confirmed out of Phase 1?
7. What data do we actually have for ticket count, average ticket size, item sales, ingredients, and order channels?
8. What decision should the user make after seeing an alert?

My strongest recommendation would be to force a distinction between forecasting and retrospective anomaly detection. Right now, the conversation is blending the two, and that could create scope confusion.
