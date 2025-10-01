---
layout: post
title: "Newsvendor Model in BigQuery Capacity Management"
date: 2025-10-01
categories: bigquery finops capacity
tags: [bigquery, finops, capacity, newsvendor]
---

When talking about optimizing **BigQuery slots**, one of the decision-making tools I applied was the **Newsvendor Model**. It may sound like just another statistical method, but in practice, it’s one of the clearest ways to understand cost-performance tradeoffs when slot usage is uncertain or random.

---

## What is the Newsvendor Model?

At its core, the Newsvendor model is about finding the **optimal balance point** between two competing risks:

* **Overage cost (Co):** Paying for too many resources that go unused (idle committed slots).
* **Underage cost (Cu):** Paying more for on-demand autoscale slots or facing degraded performance.

It answers: *“Where should I set my baseline slot capacity so that I minimize both wasted cost and risk of slowdowns?”*

---

## Why I Used It in BigQuery Capacity Management

In BigQuery capacity planning, we don’t just want to look at the **average slot usage** — averages can mislead. We need to consider:

* Idle cost when provisioning too high.
* Autoscale (burst) cost when provisioning too low.

This is where the **Newsvendor model** comes in.

---

## Example Computation

**Step 1: Identify Costs**

* Committed slot cost (Co) = **$0.036 per slot-hour**
* Autoscale slot cost = **$0.06 per slot-hour**
* Underage cost (Cu) = difference = **0.06 − 0.036 = $0.024**

**Step 2: Compute Critical Ratio (CR)**
[ CR = \frac{Cu}{Cu + Co} = \frac{0.024}{0.024 + 0.036} = 0.4 ]

**Step 3: Find Percentile from Demand Distribution**

* CR = 0.4 → corresponds to the **40th percentile** of historical slot usage.
* From the cumulative probability table, this mapped to **~4500 slots**.

---

## Comparison with Brute Sweep

* Brute force sweep (cost-only) suggested **4000 slots** with the lowest total cost (~190.4).
* Newsvendor model (CR = 0.4) recommended **4500 slots**, slightly higher cost (~193.2) but more balanced against bursts.
---

## Insights

By looking at the **Newsvendor model**:

* I was able to detect where **autoscale risk** becomes significant.
* I can see how **idle slots** affect cost if I overprovision.
* I have a **quantitative rationale** for recommending **4500 slots** instead of relying only on averages or brute sweep (if you are more cost oriented, go with brute sweep).

---

### Summary

With **CR = 0.4**, the Newsvendor model recommended **4500 slots** as the optimal baseline. This method gave me a structured, data-driven way to balance **committed cost efficiency** with **autoscale performance risk** in BigQuery FinOps.
