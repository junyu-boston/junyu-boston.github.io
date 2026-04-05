---
title: "Measures in Power BI and Databricks: Why They Exist"
date: 2026-04-05
categories: [Data Engineering]
tags: [power-bi, databricks, dax, semantic-layer, analytics, data-modeling]
---

Most people learn Power BI or Databricks by connecting to data and dragging fields into visuals. That works — until the numbers start looking wrong. A total that does not add up. A percentage that gives the same value in every row. A chart that shows revenue correctly at the monthly level but breaks at the daily level.

The root cause is almost always the same: they are treating stored data as if it were the answer. Measures exist because the answer depends on context, and context changes every time the user interacts with a visual.

### The core problem

Data is stored at the lowest useful grain. A sales table has one row per transaction. An events table has one row per click. An inventory table might have one row per product per day. That is fine for storage. It is useless for a chart.

A chart showing monthly revenue does not want individual transactions. It wants a sum of transactions, grouped by month, filtered to whatever the user has selected. A chart showing market share wants that sum divided by a different sum, also filtered, possibly from a different table.

The question a visual asks is almost never "give me these rows." It is "compute this value for this slice of the data, under these conditions."

That computation — defined once, evaluated dynamically against whatever context the visual provides — is what a measure is.

### Why columns cannot do this job

The first instinct is to add a calculated column. Revenue per row? Multiply quantity by price and store it. Done.

That works for row-level math. It fails the moment you need anything that crosses rows.

A calculated column is evaluated once, at data load time, in **row context** — it sees exactly one row and nothing else. It cannot sum other rows. It cannot respond to a slicer. It cannot change its answer based on which month is selected in the report filter. Once stored, its value is fixed.

A measure is evaluated at **query time**, every time a visual renders, in **filter context** — the full set of active filters applied by whatever the user is looking at. Change the slicer, redraw the visual, apply a cross-filter from another chart — the filter context changes and the measure recomputes.

The distinction sounds academic. It is not. If you build anything beyond simple row arithmetic on calculated columns, the numbers will be wrong in ways that are hard to diagnose.

### Power BI: DAX and evaluation context

In Power BI, measures are written in DAX (Data Analysis Expressions). The language was designed specifically around the idea of evaluation context.

A minimal measure:

```dax
Total Revenue = SUM(Sales[Amount])
```

This looks trivial. It does more than it appears. When placed in a matrix with months as rows, it computes the sum for each month. When a slicer selects only one region, it sums only that region's transactions. The measure did not change. The filter context did.

More complex:

```dax
Revenue Margin % =
DIVIDE(
    SUM(Sales[Amount]) - SUM(Sales[Cost]),
    SUM(Sales[Amount])
)
```

This computes margin as a fraction, safely (no divide-by-zero). Still context-aware — it recomputes for each visual cell.

The hard part comes when you need to **escape** the filter context on purpose. Suppose you want each row of a matrix to show its revenue as a share of the total, ignoring whatever region filter is applied:

```dax
Revenue % of All Regions =
DIVIDE(
    SUM(Sales[Amount]),
    CALCULATE(SUM(Sales[Amount]), ALL(Sales[Region]))
)
```

`CALCULATE()` is the mechanism for modifying filter context. It evaluates its first argument under a modified context — here, with the Region filter removed via `ALL()`. This is where DAX gets powerful and where most confusion enters.

The key insight: **you are not writing a formula that runs once. You are writing a description of a computation that the engine will execute under varying conditions, many times, across many visual cells.**

Understanding context — what filters are active, what table relationships exist, what dimensions are in the visual — is the entire skill of DAX.

### Databricks: the same problem, different architecture

Databricks takes a different path to the same destination.

The traditional approach in Databricks SQL is to write the aggregation in the query itself. You are not defining a reusable measure — you are writing the full answer each time.

```sql
SELECT
    region,
    SUM(amount) AS total_revenue,
    SUM(amount - cost) / NULLIF(SUM(amount), 0) AS margin_pct
FROM sales
WHERE order_date BETWEEN '2026-01-01' AND '2026-03-31'
GROUP BY region
```

This is honest and explicit. The problem is that it does not scale across an organization. If twenty dashboards each define "Revenue" with slightly different filters, different table joins, or different handling of returns, the numbers will disagree. And they will, because SQL does not force a single definition.

The answer Databricks introduced is the **metrics layer** — centralized, reusable business logic living in Unity Catalog.

In Databricks AI/BI, you define metrics declaratively on top of a semantic layer. A metric specifies which table it draws from, what the aggregation is, how it handles nulls, and what dimensions it supports. Every dashboard that references "Revenue" pulls from the same definition. Change the definition in one place and every downstream visual reflects the change.

```yaml
# Conceptual Unity Catalog metric definition
name: total_revenue
table: catalog.sales.transactions
aggregation: SUM(amount)
filters:
  - status != 'returned'
dimensions:
  - region
  - product_category
  - order_month
```

The engine evaluates this against whatever dimensional context the dashboard requests — identical in principle to how a DAX measure evaluates against filter context, even though the implementation is entirely different.

### Why visuals specifically need this

A visual has two properties that make measures necessary: **grain** and **interactivity**.

**Grain**: A bar chart showing monthly revenue is at monthly grain. The underlying data is at transaction grain. Something has to bridge those levels. If you pre-aggregate at load time, you lose the ability to drill down. If you aggregate at render time against a reusable definition, you get both levels for free.

**Interactivity**: A dashboard filter changes the answer. A cross-highlight from another visual changes the answer. A drill-through changes the answer. A measure handles this cleanly because it does not hold a value — it holds a computation that runs fresh each time. A pre-computed column or a hardcoded query cannot respond to user interaction.

The broader point: **the visual is a question-asking machine**. Every time the user interacts, they are asking "what is this value for this slice?" A measure is the reusable answer function. Without it, every interaction requires either a new pre-aggregated dataset or a new query — neither of which scales.

### The semantic layer: the actual concept

Both Power BI and Databricks are solving the same architectural problem under different names: the **semantic layer**.

Raw data describes what happened. The semantic layer describes what things *mean* — what "Revenue" means to this business, how "Active Users" is counted, what "Margin" includes or excludes. Measures are the atomic unit of the semantic layer.

The value of centralizing business logic here is not just correctness. It is **trust**. When the CFO and the head of sales see the same Revenue number with the same definition, debates about "your numbers vs my numbers" disappear. The number is the number because the definition is shared.

#### Is Power BI's "semantic model" the same thing?

Yes. Microsoft renamed what used to be called a "dataset" to **semantic model** around 2023. It is their specific implementation of the semantic layer concept — tables and relationships, DAX measures, hierarchies, descriptions, and row-level security all bundled into one publishable artifact.

When you publish a semantic model to the Power BI service, other reports connect to it as a shared live dataset. They do not each carry their own copy of the logic. Change a measure definition once, every downstream report reflects it. That is the semantic layer working as intended.

#### How the two platforms differ architecturally

Power BI packages the semantic layer as a **single coherent object**. You build it in Power BI Desktop, publish it, and everyone connects to it. Tightly integrated, quick to stand up, limited flexibility outside the Microsoft ecosystem.

Databricks assembles the semantic layer from **modular pieces**:

| Layer | What it is |
| --- | --- |
| Delta tables in Unity Catalog | Raw and curated data storage |
| Gold layer views | Pre-joined, pre-cleaned tables ready for analysis |
| Unity Catalog metric definitions | Named, reusable business logic (AI/BI metrics layer) |
| Genie / AI/BI dashboards | Visualization and natural language query on top |

More flexible, works across any tool that connects to Databricks SQL, but requires more engineering to assemble and govern.

The tooling differs substantially. The goal is identical: define business logic once, enforce it everywhere, let visuals query against definitions rather than raw tables.

### The practical summary

If you are building in Power BI: write explicit measures for anything beyond simple column arithmetic. Never build ratios, running totals, comparisons, or period-over-period logic into calculated columns. Use `CALCULATE()` when you need to modify context deliberately.

If you are building in Databricks: do not define "Revenue" seven different ways across seven notebooks. Use the metrics layer in Databricks AI/BI or enforce a shared SQL view that everything references. Treat the semantic layer as infrastructure, not an afterthought.

In both tools, the underlying rule is the same: keep business logic in one place, make it context-aware, and let the visualization engine ask questions against it. That is what measures are for.

---

### Appendix: the essence of CALCULATE()

The practical summary above says "use `CALCULATE()` when you need to modify context deliberately." That deserves unpacking, because `CALCULATE()` is the most important and most misunderstood function in DAX.

Every measure runs inside a filter context set by the visual — whatever rows the slicer, report filter, or matrix header has narrowed down to. Most DAX functions operate within that context and cannot change it. `CALCULATE()` is the only function that can redefine the filter context before evaluating an expression.

The signature is simple:

```dax
CALCULATE(expression, filter1, filter2, ...)
```

It evaluates `expression` under a new context built by taking the current context and applying the filter arguments on top. That single mechanism covers three distinct needs.

**Adding a filter** — lock the result to a specific slice regardless of what the user selected:

```dax
North Revenue = CALCULATE(SUM(Sales[Amount]), Sales[Region] = "North")
```

Even if the slicer has "East" selected, this measure always returns North revenue. The visual's region filter was replaced by the one inside `CALCULATE()`.

**Removing a filter** — widen the calculation beyond what the visual sees:

```dax
Revenue % of All Regions =
DIVIDE(
    SUM(Sales[Amount]),
    CALCULATE(SUM(Sales[Amount]), ALL(Sales[Region]))
)
```

`ALL(Sales[Region])` strips the region filter from the denominator entirely. The numerator still respects the slicer. The denominator ignores it. This is the standard pattern for any share-of-total metric.

**Shifting a filter** — replace a filter with a transformed version of itself:

```dax
Prior Year Revenue =
CALCULATE(
    SUM(Sales[Amount]),
    SAMEPERIODLASTYEAR(Calendar[Date])
)
```

The matrix row might show 2025, but the date filter context is replaced with the equivalent 2024 period. The visual does not know this happened — it just receives a number.

#### The behavior most people miss

`CALCULATE()` filter arguments **replace** the existing filter on that column — they do not stack on top of it.

If a slicer has "East" selected and a measure says `CALCULATE(..., Sales[Region] = "North")`, the result is North revenue. The slicer is gone. East does not narrow it further.

If you want the opposite behavior — intersect with the existing slicer rather than replace it — use `KEEPFILTERS()`:

```dax
CALCULATE(SUM(Sales[Amount]), KEEPFILTERS(Sales[Region] = "North"))
```

Now the slicer is respected. If "East" is selected, no rows match both East and North — the result is blank. If "North" is selected, you get North revenue. The filter inside `CALCULATE()` intersects with whatever the user chose rather than overriding it.

Knowing which behavior you want — replacement or intersection — is a decision you have to make consciously. DAX defaults to replacement. Most people assume intersection. That mismatch is behind a large fraction of DAX bugs.

#### Why CALCULATE() exists at all

DAX was built for columnar storage engines that scan entire columns at once. The filter context is how the engine knows which rows to include in any given scan. `CALCULATE()` is the mechanism to programmatically redefine that inclusion set from within a measure.

Without it, any computation requiring two different views of the data simultaneously — current period vs prior period, this category vs all categories, cumulative total vs point-in-time value — would be impossible to express. It is not a convenience function. It is the foundation that makes analytical measures possible at all.
