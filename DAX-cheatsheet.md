# DAX Unified Cheat Sheet

A comprehensive quick-reference guide to essential DAX building blocks, combining conceptual depth with ready-to-run examples for efficient development and learning.

---

## Summary of Functions

| Category              | Function                                                        | Syntax                                                               |
| --------------------- | --------------------------------------------------------------- | -------------------------------------------------------------------- |
| **Filter**            | [CALCULATE](#calculate)                                         | `CALCULATE(<expression>, <filter1>, …)`                              |
|                       | [FILTER](#filter)                                               | `FILTER(<table>, <boolean-expression>)`                              |
| **Context**           | [ALL](#all)                                                     | `ALL(<table-or-column>)`                                             |
|                       | [ALLEXCEPT](#allexcept)                                         | `ALLEXCEPT(<table>, <column1>, …)`                                   |
|                       | [CALCULATETABLE](#calculatetable)                               | `CALCULATETABLE(<table>, <filter1>, …)`                              |
|                       | [USERELATIONSHIP](#userrelationship)                            | `USERELATIONSHIP(<col1>, <col2>)`                                    |
| **Iterator**          | [SUMX / AVERAGEX / …](#sumx--averagax---countx)                 | `SUMX(<table>, <expression>)`, etc.                                  |
|                       | [RELATEDTABLE](#relatedtable)                                   | `RELATEDTABLE(<related-table>)`                                      |
| **Aggregation**       | [SUM / AVERAGE / …](#sum---average---countrows---distinctcount) | `SUM(<column>)`, `COUNTROWS(<table>)`, `DISTINCTCOUNT(<column>)`     |
|                       | [DIVIDE](#divide)                                               | `DIVIDE(<numerator>, <denominator>, [<alternate>])`                  |
| **Lookup**            | [RELATED](#related)                                             | `RELATED(<column>)`                                                  |
| **Selection**         | [VALUES / SELECTEDVALUE](#values---selectedvalue)               | `VALUES(<column>)`, `SELECTEDVALUE(<column>, [<alternate>])`         |
| **Time-Intelligence** | [SAMEPERIODLASTYEAR](#sameperiodlastyear)                       | `SAMEPERIODLASTYEAR(<dates>)`                                        |
|                       | [TOTALYTD](#totalytd)                                           | `TOTALYTD(<expression>, <dates>[, <filter>])`                        |
| **Logical**           | [IF / SWITCH](#if---switch)                                     | `IF(<cond>, <then>, [<else>])`, `SWITCH(TRUE(), <cond1>, <res1>, …)` |
| **Row vs. Filter**    | [EARLIER / EARLIEST](#earlier---earliest)                       | `EARLIER(<column>[, <n>])`, `EARLIEST(<column>)`                     |
| **Variables**         | [VAR / RETURN](#var---return)                                   | `VAR x = … RETURN <expression>`                                      |

---

## Detailed Function Reference

### 1. Filter Functions

#### CALCULATE {#calculate}

**What it does:** Evaluates an expression in a modified filter context.
**Why it matters:** The cornerstone of context manipulation—lets you override slicers or apply new filters.

```dax
Sales West Region :=
CALCULATE(
  [Total Sales],
  'Geography'[Region] = "West"
)
```

#### FILTER {#filter}

**What it does:** Returns a table filtered by a Boolean condition.
**Why it matters:** Use inside iterators or CALCULATE to apply complex row-level filters.

```dax
Big Orders :=
FILTER(
  Sales,
  Sales[Amount] > 10000
)
```

### 2. Context-Control Functions

#### ALL {#all}

**What it does:** Removes all filters from the specified table or column.

```dax
All Products Sales :=
CALCULATE(
  [Total Sales],
  ALL(Products)
)
```

#### ALLEXCEPT {#allexcept}

**What it does:** Removes filters except those on specified columns.

```dax
Sales Excl Region :=
CALCULATE(
  [Total Sales],
  ALLEXCEPT(
    Geography,
    Geography[Country]
  )
)
```

#### CALCULATETABLE {#calculatetable}

**What it does:** Returns a filtered table, rather than a scalar.

```dax
Top 5 Products by Sales :=
CALCULATETABLE(
  TOPN(5, Products, [Total Sales], DESC),
  ALL(Products)
)
```

#### USERELATIONSHIP {#userrelationship}

**What it does:** Activates an inactive relationship within CALCULATE.

```dax
Promo Sales :=
CALCULATE(
  [Total Sales],
  USERELATIONSHIP(
    Calendar[Date],
    Promotion[Date]
  )
)
```

### 3. Iterator Functions

#### SUMX / AVERAGEX / COUNTX {#sumx--averagax---countx}

**What they do:** Row-by-row calculations over a table.

```dax
Sales Amount per Unit :=
SUMX(
  Sales,
  Sales[Quantity] * Sales[UnitPrice]
)
```

#### RELATEDTABLE {#relatedtable}

**What it does:** Returns the "many" side rows related to the current row.

```dax
Sales by Product :=
SUMX(
  RELATEDTABLE(Sales),
  Sales[Amount]
)
```

### 4. Aggregation Functions

#### SUM / AVERAGE / COUNTROWS / DISTINCTCOUNT {#sum---average---countrows---distinctcount}

**What they do:** Basic aggregations.

```dax
Order Count       := COUNTROWS(Sales)
Unique Customers  := DISTINCTCOUNT(Sales[CustomerID])
Average Price     := AVERAGE(Sales[UnitPrice])
```

#### DIVIDE {#divide}

**What it does:** Safe division with optional alternate result.

```dax
Sales per Order :=
DIVIDE(
  [Total Sales],
  [Order Count],
  0
)
```

### 5. Lookup & Selection

#### RELATED {#related}

**What it does:** Fetches a column value from a related table (many→one).

```dax
Product Category :=
RELATED(
  Product[CategoryName]
)
```

#### VALUES / SELECTEDVALUE {#values---selectedvalue}

**What they do:**

* **VALUES:** Returns distinct values of a column as a one-column table.
* **SELECTEDVALUE:** Returns the single selected value or an alternate.

```dax
Current Year :=
SELECTEDVALUE(
  Calendar[Year],
  0
)
```

### 6. Time-Intelligence Functions

#### SAMEPERIODLASTYEAR {#sameperiodlastyear}

**What it does:** Shifts dates by −1 year in the current selection.

```dax
Sales LY :=
CALCULATE(
  [Total Sales],
  SAMEPERIODLASTYEAR(Calendar[Date])
)
```

#### TOTALYTD {#totalytd}

**What it does:** Cumulates an expression from year-start to the current date.

```dax
YTD Sales :=
TOTALYTD(
  [Total Sales],
  Calendar[Date]
)
```

### 7. Logical Functions

#### IF / SWITCH {#if---switch}

**What they do:** Conditional branching.

```dax
Sales Category :=
SWITCH(
  TRUE(),
  [Total Sales] < 10000, "Low",
  [Total Sales] < 50000, "Medium",
  "High"
)
```

### 8. Row vs. Filter Context

#### EARLIER / EARLIEST {#earlier---earliest}

**What they do:**

* **EARLIER:** Reference the current row’s value in an outer iteration of nested iterators. Use when you need to compare against the row being processed.
* **EARLIEST:** Retrieve the first (earliest) value seen in the current row context chain. Use when you want the very first occurrence across contexts.

```dax
-- EARLIER example:
Running Total by Customer :=
SUMX(
  FILTER(
    Sales,
    Sales[CustomerID] = EARLIER(Sales[CustomerID])
      && Sales[Date] <= EARLIER(Sales[Date])
  ),
  Sales[Amount]
)

-- EARLIEST example:
First Sale Amount by Customer :=
VAR FirstSaleDate = EARLIEST(Sales[Date])
RETURN
CALCULATE(
  Sales[Amount],
  Sales[Date] = FirstSaleDate
)
```

### 9. Variables & Best-Practice Pattern

#### VAR / RETURN {#var---return}

**What it does:** Defines intermediate variables in measures for clarity and performance.

```dax
Sales Growth % :=
VAR Current = [Total Sales]
VAR Prior   = CALCULATE(
  [Total Sales],
  SAMEPERIODLASTYEAR(Calendar[Date])
)
RETURN
  DIVIDE(Current - Prior, Prior)
```

---

## Patterns & Best Practices

* **Context Transition:** Use CALCULATE inside an iterator to turn row-context into filter-context.
* **Safe Math:** Always use DIVIDE to handle zero denominators.
* **Variable Decomposition:** Break complex logic into VAR blocks for readability.
* **Relationship Control:** Use USERELATIONSHIP to leverage inactive model relationships.
* **Filter Overrides:** Combine CALCULATE + ALL/ALLEXCEPT to override slicers.
* **Top-N Tables:** `CALCULATETABLE(TOPN(...), ...)` for ranking scenarios.
* **Selection Capture:** Use SELECTEDVALUE for slicer‐driven logic.
* **Time Shifts:** Leverage SAMEPERIODLASTYEAR, TOTALYTD, etc., for date comparisons.

---

## Quick-Reference Table

| Function               | Example Snippet                                   |
| ---------------------- | ------------------------------------------------- |
| CALCULATE              | `CALCULATE([Sales], Region[Name] = "West")`       |
| FILTER                 | `FILTER(Orders, Orders[Qty] > 10)`                |
| ALL                    | `ALL(Products)`                                   |
| ALLEXCEPT              | `ALLEXCEPT(Geo, Geo[Country])`                    |
| CALCULATETABLE         | `CALCULATETABLE(TOPN(5,Products,[Sales]),ALL())`  |
| USERELATIONSHIP        | `USERELATIONSHIP(Date[Date],Promo[Date])`         |
| SUMX                   | `SUMX(Sales, Sales[Qty] * Sales[Price])`          |
| RELATEDTABLE           | `RELATEDTABLE(Sales)`                             |
| SUM / AVERAGE          | `SUM(Orders[Total]), AVERAGE(Orders[Total])`      |
| COUNTROWS / DISTINCT   | `COUNTROWS(Sales), DISTINCTCOUNT(Sales[CustID])`  |
| DIVIDE                 | `DIVIDE([Sales], [Orders], 0)`                    |
| RELATED                | `RELATED(Product[Cat])`                           |
| VALUES / SELECTEDVALUE | `VALUES(Date[Year]), SELECTEDVALUE(Date[Year],0)` |
| SAMEPERIODLASTYEAR     | `SAMEPERIODLASTYEAR(Date[Date])`                  |
| TOTALYTD               | `TOTALYTD([Sales], Date[Date])`                   |
| IF / SWITCH            | `SWITCH(TRUE(), ...)`                             |
| EARLIER / EARLIEST     | `EARLIER(Table[Col])`, `EARLIEST(Table[Col])`     |
| VAR / RETURN           | `VAR x = … RETURN x`                              |
