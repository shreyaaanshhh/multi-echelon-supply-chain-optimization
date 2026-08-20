# Multi-Echelon Supply Chain Network Optimization — Project Report

## 1. Project Aim

The goal of this project is to solve a real, practical supply chain question:

> **Given a set of plants (production/distribution nodes) with limited capacity, and a set of customers with known demand, how should shipments be allocated from plants to customers so that total logistics cost is minimized, while respecting each plant's capacity limits?**

This is a classic **Linear Programming (LP)** problem, formally known as a **transportation/network flow problem**. Instead of solving it with invented or textbook numbers, this project uses a real-world Kaggle dataset ("Supply Chain Logistics Problem") to make the exercise reflect the kind of messy, imperfect data an actual supply chain analyst works with.

The broader intent behind the project is to demonstrate three things to a recruiter or interviewer:
1. Ability to translate a real business problem into a mathematical optimization model.
2. Ability to clean, join, and reconcile messy real-world data from multiple sources.
3. Ability to make and clearly justify simplifying assumptions where data is incomplete or ambiguous — a routine and expected part of real analytics work.

---

## 2. Dataset Overview

Source: Kaggle — "Supply Chain Logistics Problem" (Excel workbook, multiple sheets).

Sheets used:
- **OrderList** (9,215 rows) — individual customer orders: Order ID, Order Date, Origin Port, Carrier, Customer, Plant Code, Destination Port, Unit quantity, Weight.
- **FreightRates** — shipping cost data by Carrier, Origin Port, Destination Port, and weight bracket (`minm_wgh_qty`, `max_wgh_qty`), giving a `rate` per unit.
- **WhCapacities** — Plant ID and Daily Capacity (production/handling capacity per plant).
- **WhCosts** — Plant-level handling cost per unit (not directly used in the current model; considered for a future cost-layer refinement).
- **PlantPorts** — mapping of which plants connect to which ports (structural reference; not directly used once the network was simplified — see Section 3).

---

## 3. Structural Decision: Why Plant → Customer (Not a Deeper Multi-Echelon Chain)

Initial plan was a deeper network: Supplier → Factory → Warehouse → City. On exploring the data:

- `OrderList["Destination Port"]` had only **one unique value** (a single port). This meant "Port" was not a real decision-making node in the network — every shipment funnels through the same port regardless of plant or customer.
- The real destination diversity was in the `Customer` column (46 unique customers).
- `Plant Code` and `WhCapacities`' `Plant ID` referred to the same set of entities — there was no genuinely separate "Warehouse" layer distinct from "Plant" in this dataset.

**Decision:** Rather than inventing an artificial warehouse layer not supported by the data (which would look fabricated in an interview), the project uses a **Plant → Customer** structure, and instead pursues depth and realism: real freight-rate-based costing, a demand forecasting layer (planned), and a disruption/resilience scenario (planned) — see Section 8.

---

## 4. Data Cleaning Pipeline — Step by Step with Reasoning

### 4.1 Problem: No Direct Cost Data

`OrderList` tells us *what* was shipped (plant, customer, quantity, weight) but not *what it cost*. `FreightRates` has cost, but indexed by **Carrier + Origin Port + Destination Port + Weight bracket**, not by order or by customer. A join was required to attach a real cost to each order.

### 4.2 Merge Step

```python
merged = order_list.merge(
    freight_rates,
    left_on=["Carrier", "Origin Port", "Destination Port"],
    right_on=["Carrier", "orig_port_cd", "dest_port_cd"],
    how="inner"
)
```
This is a SQL-JOIN-equivalent operation. `how="inner"` was used deliberately — only orders with a genuine match in `freight_rates` are retained, since orders without a cost match cannot be priced and would need to be dropped anyway.

**Finding:** Only carriers `V444_0` and `V444_1` in `order_list` had corresponding entries in `freight_rates`. A third carrier, `V44_3`, appeared in `order_list` (854 orders, ~9.3% of total) but had **no matching entries at all** in `freight_rates`.

**Investigation performed (not skipped):** Before dropping these orders, `order_list["Carrier"].value_counts()` was run to confirm the scale of the issue (854 out of 9,215 orders) and to confirm this wasn't a data-processing bug on our end — `freight_rates` genuinely has no data for `V44_3` under any weight bracket. This was treated as a **genuine source-data limitation**, not an error to "fix," and those 854 orders were excluded, with the reasoning documented rather than silently dropped.

### 4.3 Weight-Bracket Filter

A single Carrier + Origin + Destination combination in `freight_rates` can have multiple rows, each valid for a different weight range (e.g., 0–50 kg at one rate, 50–100 kg at another). The merge above matches an order to *all* weight-bracket rows for its route, so a filter is needed to keep only the bracket the order's actual weight falls into:

```python
final = merged[(merged["Weight"] >= merged["minm_wgh_qty"]) & (merged["Weight"] <= merged["max_wgh_qty"])]
```

This reduced the working set and also **surfaced a second data gap**: of the 8,361 orders with a matching carrier, only 6,991 had a weight that actually fell inside any defined bracket in `freight_rates`. The remaining ~1,370 orders (carrier-matched but no valid weight bracket) were excluded for the same reason as above — no valid rate could be assigned to them.

### 4.4 Deduplication (Overlapping Weight Brackets)

Some orders still matched **more than one** weight-bracket row (overlapping bracket definitions in the source data — e.g., a weight of 70 falling inside both a 60–70 and a 65–75 bracket). To resolve this without arbitrarily picking a row:

```python
final_sorted = final.sort_values("rate")
final_dedup = final_sorted.drop_duplicates(subset="Order ID", keep="first")
```

Sorting by `rate` ascending before deduplicating means the retained row for each `Order ID` is always the **cheapest available valid rate** — a defensible, explainable rule (a company shipping via a given carrier/route would reasonably use whichever listed rate applies most favorably, and this avoids arbitrary row selection).

**Result: `final_dedup` — 6,991 clean, uniquely-priced orders.** This is the working dataset for all downstream calculations.

Coverage summary: 6,991 / 9,215 = **~75.9%** of raw orders retained with a verified real freight rate. This is the number that should be quoted in documentation/resume, not the earlier 90.7% carrier-match figure, since the weight-bracket step removed further rows after that initial match.

---

## 5. Cost Matrix Construction

```python
plant_customer_cost = final_dedup.groupby(["Plant Code", "Customer"])["rate"].mean().reset_index()
```

For each unique (Plant, Customer) pair that appears in the cleaned order history, the average shipping rate across all matching orders is computed. This produces a **68-row cost matrix** — i.e., 68 real, data-derived Plant→Customer shipping lanes with an associated per-unit cost. This directly replaces the manually-typed `cost = {...}` dictionary used in the earlier Tropicsun practice exercise, but is now sourced from real transactional data rather than invented.

Grouping by two columns (`groupby(["Plant Code", "Customer"])`) rather than one bucket the data more finely — each unique combination of plant and customer becomes its own group, and `.mean()` collapses potentially many individual order-level rates for that specific lane into one representative number.

---

## 6. Demand Aggregation

```python
demand = final_dedup.groupby("Customer")["Unit quantity"].sum().reset_index()
```

Total ordered quantity per customer, summed across their (cleaned, cost-matched) orders. This produced **42 unique customers** with demand data (fewer than the 46 total customers in the raw data, because some customers' orders were entirely among the excluded/unmatched rows).

**Important design decision:** demand was computed from `final_dedup` (the cleaned dataset), not from the raw `order_list`. This was deliberate — if demand were computed from all 9,215 raw orders, the model would include demand for customers or quantities for which no valid cost exists, making the optimization problem unsolvable or requiring further patchwork assumptions. Using the same cleaned dataset for both cost and demand keeps the model internally consistent.

Total demand across all 42 customers: **24,294,442 units**.

---

## 7. Capacity Mismatch — Discovery, Diagnosis, and Resolution

### 7.1 The Problem

`WhCapacities` gives a `Daily Capacity` figure per plant (e.g., PLANT03 = 1013, PLANT09 = 11). Summed across the 6 plants that also appear in the cost matrix, total available capacity was only **2,194 units**, compared to total demand of **24,294,442 units** — a mismatch of roughly 11,000x.

This was diagnosed methodically:
- First, order dates were checked (`Order Date.min()` / `.max()`) to rule out a "daily vs. total period" unit confusion — both returned the same date (2013-05-26), so the order data represents a single-day snapshot, ruling out the need to multiply daily capacity by a number of days.
- With that ruled out, the conclusion was that `Daily Capacity`, as recorded in the source file, is likely expressed in a different unit or scale than `Unit quantity` in the order data (e.g., batches, pallets, or some other aggregation) — a common inconsistency in real, loosely-documented datasets.

### 7.2 Resolution: Proportional Capacity Scaling

Rather than inventing an arbitrary absolute capacity number (which would be an unjustifiable fabrication), each plant's **relative share** of total capacity was preserved and rescaled to match the actual demand volume:

```python
wh_subset = wh_capacities[wh_capacities["Plant ID"].isin(plants_in_cost)].copy()
wh_subset["capacity_share"] = wh_subset["Daily Capacity"] / wh_subset["Daily Capacity"].sum()
wh_subset["scaled_capacity"] = wh_subset["capacity_share"] * total_demand * 1.15
```

- `capacity_share`: each plant's proportion of total capacity among the 6 relevant plants (e.g., PLANT03 = 46.17% of total capacity — the largest single plant).
- `scaled_capacity`: that same proportion applied to total demand, with a **15% buffer** (`* 1.15`) added so the model has some slack rather than being exactly tight (which can cause solver difficulties or an unrealistically rigid allocation).

This preserves the *relative* sizing information that is present in the real data (which plants are bigger or smaller relative to each other) while making the *absolute* numbers usable for the optimization. This assumption is explicitly flagged as an assumption, to be stated plainly in the final README and interview discussion: **"Daily Capacity values appear to be in a different unit/scale than order quantities; capacities were rescaled proportionally to align with total observed demand, preserving each plant's relative share."**

### 7.3 Resulting Scaled Capacities (6 plants)

| Plant ID | Daily Capacity (raw) | Capacity Share | Scaled Capacity |
|---|---|---|---|
| PLANT16 | 457 | 20.8% | ~5,819,482 |
| PLANT12 | 209 | 9.5% | ~2,661,426 |
| PLANT09 | 11 | 0.5% | ~140,751 |
| PLANT03 | 1013 | 46.2% | ~12,899,640 |
| PLANT13 | 490 | 22.3% | ~6,239,707 |
| PLANT08 | 14 | 0.6% | ~178,277 |

PLANT03 dominates capacity share (46%); PLANT09 and PLANT08 are minor contributors (~0.5–0.6% each). This means the optimization is expected to route the bulk of shipments through PLANT03 and PLANT13, with PLANT09/PLANT08 handling only small volumes — a pattern the final results should be checked against as a sanity check.

---

## 8. Current Status and Remaining Work

**Completed:**
- Data cleaning and merging pipeline (OrderList + FreightRates), with documented, justified exclusions at each filtering step.
- Real, data-derived cost matrix (68 Plant–Customer lanes).
- Real, data-derived demand aggregation (42 customers).
- Capacity reconciliation via proportional scaling, with the underlying assumption explicitly documented.

**Not yet done:**
1. **Build and solve the actual PuLP LP model** — decision variables (units shipped per Plant–Customer pair), objective function (minimize total cost), constraints (plant capacity ≤ scaled_capacity, customer demand met), using the cost/demand/capacity tables built above.
2. **Demand forecasting layer (ML)** — apply linear regression on historical order data to forecast demand, feeding predictions into the LP model as an input (this has been committed to on the resume and needs to be built to match).
3. **Disruption/resilience scenario** — simulate a plant capacity shock (e.g., PLANT03 going offline or reduced) and quantify the resulting cost/feasibility impact, comparing mitigation strategies (safety stock vs. dual-sourcing).
4. **Final README** — a polished, complete write-up of the project for GitHub, written once the model and results are finalized (a lighter "progress log" has been maintained in the interim).

---

## 9. Key Assumptions and Limitations (for transparency in interviews)

- Orders shipped via carrier `V44_3` (9.3% of raw orders) were excluded due to a complete absence of matching freight rate data for that carrier.
- A further ~15% of carrier-matched orders were excluded because their weight did not fall within any defined rate bracket.
- Net usable data: **6,991 of 9,215 orders (~75.9%)**.
- Cost per Plant–Customer lane is an **average** of all matching cleaned orders for that lane, not a single canonical rate — real rates likely vary by shipment size, time, and other factors not modeled here.
- Plant capacity figures were **proportionally rescaled** due to an apparent unit mismatch with order quantities; absolute capacity values are therefore illustrative/derived, not the literal source-file numbers.
- The network structure is two-stage (Plant → Customer); a Port-level intermediate stage exists in the raw data but was not treated as a decision variable, since only one port appears in the destination data.
