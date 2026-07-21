---
title: ML Anomaly Engine
date: 2024-12-15
---

You have a clear three-part flow:

1. **Alert engine** detects a meaningful anomaly.
2. **Notification hub** presents and prioritizes alerts.
3. **Chat experience** explains what happened, why it happened, and what the merchant should do next.

The next step is to define the product rules and user experience around each part.

## Follow-up questions for the product team

### Alert engine

* Which metrics should generate alerts beyond net revenue: ticket count, average ticket, item sales, channel performance, labor cost, refunds, cancellations, or service time?
* Should alerts be created for both declines and unexpected increases?
* Is a fixed threshold, such as a 15% decline, appropriate for every merchant and metric?
* Should the baseline be the previous week, the same day last week, the same period last year, a rolling average, or forecasted performance?
* How long must an anomaly persist before an alert is generated?
* What minimum revenue or order volume is required to avoid alerts caused by small numbers?
* Should seasonality, holidays, promotions, closures, and unusual operating hours be considered?
* Should merchants be able to configure thresholds and metrics?
* How do we prevent repeated alerts for the same ongoing issue?
* What makes an alert critical, high priority, or informational?

### Notification hub

* Should notifications be ranked by financial impact, percentage change, urgency, or merchant preference?
* What information should be visible before the merchant opens an alert?
* Should alerts be grouped by location, metric, channel, or incident?
* Can merchants mark alerts as read, resolved, dismissed, or not useful?
* Should the hub show only active alerts or also historical and resolved alerts?
* Should merchants receive alerts outside the product through email, push, SMS, or in-app messages?
* How frequently should notifications be delivered: immediately, in a daily digest, or both?
* What happens when multiple metrics are symptoms of the same issue?

### Chat and explanation experience

* What questions should the chat answer automatically when an alert is opened?
* Should the initial response explain what changed, where it changed, likely drivers, and recommended actions?
* How much evidence should be shown to support the explanation?
* Should merchants be able to drill down by location, channel, daypart, and item?
* Should the assistant distinguish between confirmed causes and possible causes?
* What actions can merchants take directly from the chat?
* Should the chat collect feedback such as “helpful,” “not relevant,” or “expected change”?
* How should the experience respond when there is not enough data to explain the anomaly?

## Product recommendations

### 1. Avoid using only fixed percentage thresholds

A 15% decline may be important for one merchant but normal for another. Use a combination of:

* Percentage change
* Absolute financial impact
* Statistical deviation from expected performance
* Duration of the issue
* Data volume and confidence

For example, a 20% revenue decline representing $50 may be less important than a 7% decline representing $8,000.

### 2. Begin with a narrow set of high-value alerts

Start with alerts that are easy to understand and likely to drive action:

* Net revenue decline
* Ticket-count decline
* Average-ticket decline
* Major item sales decline
* Channel underperformance
* Unusually high refunds or cancellations
* Labor cost increasing while revenue declines

This will make the first version easier to validate before adding more operational metrics.

### 3. Build alerts around incidents, not individual metrics

Revenue, ticket count, and item sales may all decline because of one underlying event. Instead of sending three separate notifications, combine them into one incident:

> Revenue declined 18%, primarily due to lower delivery orders and reduced sales of the top chicken sandwich.

This reduces notification fatigue and creates a more useful merchant experience.

### 4. Make every alert answer four questions

Each alert should clearly communicate:

* **What happened?**
* **Where and when did it happen?**
* **Why did it likely happen?**
* **What should the merchant do next?**

An example:

> Net revenue at Location A was 17% below its normal Tuesday level between 11 a.m. and 2 p.m. The decline was mainly driven by 24% fewer delivery orders and lower chicken sandwich availability. Review delivery-channel availability and item stock levels.

### 5. Separate detection from explanation

The alert engine should determine that something unusual happened. A separate diagnostic layer should identify likely contributing factors. This makes the system easier to improve and avoids forcing the alert model to explain every anomaly itself.

### 6. Add merchant feedback early

Allow merchants to identify alerts as:

* Useful
* Expected
* Not relevant
* Incorrect
* Already resolved

This feedback can help tune thresholds, improve prioritization, and personalize future alerts.

### 7. Define success metrics for the feature

Measure more than alert volume. Useful product metrics may include:

* Percentage of alerts opened
* Percentage of alerts investigated in chat
* Time from alert to merchant action
* Percentage of alerts marked useful
* Dismissal or false-positive rate
* Repeat engagement with the notification hub
* Revenue or operational impact after action
* Reduction in time required to identify business issues

## Recommended MVP flow

For the first version, the system could:

1. Monitor net revenue, ticket count, and average ticket.
2. Compare performance with an appropriate historical baseline.
3. Trigger only when both percentage change and financial impact exceed thresholds.
4. Group related changes into one alert.
5. Display the alert in the notification hub with severity and estimated impact.
6. Open a pre-populated chat explanation when clicked.
7. Show the main drivers by location, channel, daypart, and item.
8. Recommend one or two practical next steps.
9. Ask the merchant whether the alert was useful or expected.

The key product decision is not simply **“What percentage decline should trigger an alert?”** It is **“Which changes are important, unusual, explainable, and actionable enough to interrupt the merchant?”**


You are converging on a sensible **layered anomaly-detection architecture**, rather than relying on a single model.

The main approaches discussed are:

1. **Statistical rules:** Bollinger-style bands or deviations beyond 1.5–2.5 standard deviations.
2. **Isolation Forest:** lightweight multivariate anomaly detection.
3. **Forecast-based detection:** compare actual performance with an XGBoost or LightGBM forecast and expected range.
4. **Ensemble logic:** combine signals to improve confidence and reduce false positives.

## Recommended architecture

### Layer 1: Statistical guardrails

Start with transparent rules that are inexpensive and easy to explain:

* Actual versus moving average or EMA
* Standardized deviation or z-score
* Absolute financial impact
* Minimum transaction volume
* Persistence across multiple periods

For example:

> Alert when net revenue is more than two standard deviations below its expected level, the estimated impact exceeds $500, and the condition persists for two intervals.

One caution: standard deviation works best when the historical distribution is relatively stable. Restaurant data can be skewed by holidays, promotions, closures, and large orders. Consider testing **median absolute deviation** or empirical percentile bands alongside standard deviation.

### Layer 2: Isolation Forest

Isolation Forest is appropriate for detecting unusual combinations such as:

* Revenue decreased
* Ticket count remained stable
* Average ticket decreased
* Refunds increased
* One channel declined sharply
* Labor hours stayed elevated

It is lightweight, but it does **not forecast the expected value**. It answers:

> “Is this observation unusual compared with historical observations?”

It does not directly answer:

> “What should revenue have been today?”

That is why combining it with statistical or forecast-based methods is useful.

### Layer 3: Forecast residuals

A forecasting model can predict:

* Expected net revenue
* Expected ticket count
* Expected average ticket
* Expected channel or item volume
* Lower and upper prediction bounds

The anomaly signal then becomes:

[
\text{Residual}=\text{Actual}-\text{Forecast}
]

An alert is generated when actual performance falls outside the prediction interval and the impact is material.

This approach is likely more accurate later, but it introduces additional requirements around data quality, model lifecycle, retraining, monitoring, and reproducibility.

## Do not select a different algorithm for every merchant initially

Automatically selecting Isolation Forest for one merchant and XGBoost for another may sound robust, but it can create:

* Inconsistent alert behavior
* Harder debugging
* More difficult explainability
* Greater operational complexity
* More complicated product messaging
* Higher validation and monitoring costs

For an MVP, use a **consistent detection framework** and vary parameters by merchant or merchant cohort.

For example:

* Low-volume merchants: robust statistical bands
* Medium-volume merchants: bands plus Isolation Forest
* High-volume merchants: bands, Isolation Forest, and forecasting

This provides sophistication where the data supports it without requiring thousands of unique architectures.

## A practical ensemble

Each detector can produce a normalized score:

* Statistical deviation score
* Isolation Forest anomaly score
* Forecast residual score
* Financial-impact score
* Persistence score

Then create an alert confidence score:

[
\text{Alert score}
==================

w_1S_{\text{statistical}}
+
w_2S_{\text{isolation}}
+
w_3S_{\text{forecast}}
+
w_4S_{\text{impact}}
+
w_5S_{\text{persistence}}
]

You do not necessarily need to alert only when every model agrees. A useful policy could be:

* **High severity:** multiple detectors agree and financial impact is high.
* **Medium severity:** one strong signal with meaningful impact.
* **Low severity:** unusual behavior that requires monitoring but not interruption.

## Batch training without permanently hosting models

The proposed batch approach is reasonable:

1. Pull a defined historical training window.
2. Generate features.
3. Train the model.
4. Create forecasts for the next one or two weeks.
5. Store forecasts and prediction bounds.
6. Store model and dataset metadata.
7. Remove the live compute resources.
8. Compare incoming actuals against stored forecasts.

You may not need to keep every model loaded in Vertex AI. However, you should not simply destroy everything after forecasting.

Persist a compact reproducibility package containing:

* Merchant or cohort identifier
* Training start and end dates
* Feature definitions
* Data version or snapshot identifier
* Model type
* Model version
* Hyperparameters
* Random seed
* Code or container version
* Forecast output
* Prediction intervals
* Model evaluation metrics
* Training timestamp
* Schema version

For higher-value merchants or disputed alerts, retaining the serialized model artifact may also be worthwhile. Model storage is generally much cheaper than continuously hosting a prediction endpoint.

## Explainability recommendation

The concern about losing the original model and dataset context is valid. The notification must be explainable using the information available **when the prediction was created**, not whatever the current menu or dataset looks like later.

However, LIME may not be the most practical primary explanation layer here.

Use two types of explanation:

### Model explanation

For forecasting models such as XGBoost or LightGBM:

* SHAP feature contributions
* Important seasonal variables
* Channel, daypart, promotion, and location effects
* Prediction interval and confidence

For Isolation Forest, explanations are less intuitive. You can identify which features had the largest standardized deviation, but that is not always the same as explaining the model’s anomaly score.

### Business-driver explanation

This is often more useful to merchants:

> Revenue was $2,100 below expected. Approximately $1,300 of the decline came from delivery, $500 from fewer chicken-sandwich orders, and $300 from increased refunds.

This driver decomposition can be produced separately from the anomaly model. The model decides **whether something unusual occurred**; the diagnostic layer explains **which business dimensions contributed to it**.

## Important follow-up questions

### Detection strategy

* What is the minimum viable detector for launch?
* Are you detecting sudden drops, sustained deterioration, unusual combinations, or all three?
* Should increases also generate alerts?
* How will scores from different algorithms be normalized?
* Does one model need to agree with another before an alert is shown?
* How will alert severity be calculated?

### Population and modeling

* Is there enough history for every merchant?
* What is the minimum ticket volume required for merchant-specific modeling?
* Can merchants be grouped by size, category, geography, or operating pattern?
* Should models run per merchant, per location, or per merchant-location-channel?
* How should new merchants without sufficient history be handled?
* How frequently should models or baselines be refreshed?

### Data reproducibility

* Can the source tables support time-travel or versioned snapshots?
* What happens when menu items are renamed, removed, or recategorized?
* Will feature definitions be versioned?
* How long must predictions, explanations, and training metadata be retained?
* Do merchants need to see the explanation weeks after the original alert?

### Product packaging

The final question about whether this should be sold as a feature is important:

* Is basic anomaly detection available to all merchants?
* Are forecasting and advanced explanations premium capabilities?
* Is pricing per merchant, location, alert volume, or subscription tier?
* Does the feature provide enough measurable value to justify a paid tier?
* Will paid customers expect configurable metrics, thresholds, and delivery channels?
* What service-level expectations follow from charging for the feature?

## Product recommendation

Use a tiered model:

**Core capability**

* Revenue and ticket-count alerts
* Statistical or robust deviation bands
* Basic notification hub
* Simple driver breakdown

**Advanced capability**

* Multivariate Isolation Forest
* Forecast-based anomaly detection
* Custom sensitivity
* Item, channel, and labor alerts
* Advanced explanations
* Alert history and benchmarking
* Email, SMS, or push delivery

This allows broad adoption of the basic feature while making the more expensive forecasting and explanation capabilities commercially sustainable.

The most defensible MVP is therefore:

> **Robust statistical bands plus materiality and persistence rules, supported by Isolation Forest as a secondary signal. Add forecast-based models only for merchants with sufficient data and demonstrated product value.**
