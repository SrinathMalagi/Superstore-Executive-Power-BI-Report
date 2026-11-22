# Superstore Executive BI Report — Complete DAX Measures Reference

This document catalogs all DAX measures powering the three-page **Superstore Executive BI Report**:
1. **Executive Sales & Profitability Overview**
2. **Product Performance & Discount Impact**
3. **Operations, Shipping & Returns Analysis**

---

## 1. Core Sales, Profit & Margin Measures

These foundational measures aggregate revenue, profitability, order volume, and unit counts across the `Orders` fact table.

* **`Total Sales`**: Sums total gross sales revenue across the current filter context.
* **`Total Profit`**: Sums net profit earned after discounts and costs.
* **`Profit Margin %`**: Calculates net profit margin safely using `DIVIDE` to prevent division-by-zero errors.
* **`Total Orders`**: Counts unique customer orders using `DISTINCTCOUNT` on `Order ID` (since a single order can contain multiple line items).
* **`Total Quantity`**: Sums total product units sold across all order lines.

```dax
Total Sales =
SUM ( Orders[Sales] )

Total Profit =
SUM ( Orders[Profit] )

Profit Margin % =
DIVIDE (
    [Total Profit],
    [Total Sales],
    0
)

Total Orders =
DISTINCTCOUNT ( Orders[Order ID] )

Total Quantity =
SUM ( Orders[Quantity] )
```

---

## 2. Year-over-Year (YoY) Time Intelligence Measures

These measures use `SAMEPERIODLASTYEAR` against the `Calendar` date table and `HASONEVALUE` so YoY variance badges display cleanly only when a single year is selected in the report slicers.

* **`Sales PY` / `Profit PY`**: Shifts the active date filter back exactly one year to retrieve prior-year baseline performance.
* **`Sales YoY %` / `Profit YoY %`**: Computes percentage growth or decline compared to the prior year when a single year is filtered.
* **`Profit Margin YoY %`**: Measures the percentage-point change in profit margin year-over-year (`Current Margin - Prior Margin`).
* **`Orders YoY %` / `Quantity YoY %`**: Tracks year-over-year percentage growth in distinct order volume and total units shipped.

```dax
Sales PY =
CALCULATE (
    [Total Sales],
    SAMEPERIODLASTYEAR ( 'Calendar'[Date] )
)

Sales YoY % =
VAR CurrentSales = [Total Sales]
VAR PriorSales = [Sales PY]
RETURN
    IF (
        HASONEVALUE ( 'Calendar'[Year] ),
        DIVIDE ( CurrentSales - PriorSales, PriorSales )
    )

Profit PY =
CALCULATE (
    [Total Profit],
    SAMEPERIODLASTYEAR ( 'Calendar'[Date] )
)

Profit YoY % =
VAR CurrentProfit = [Total Profit]
VAR PriorProfit = [Profit PY]
RETURN
    IF (
        HASONEVALUE ( 'Calendar'[Year] ),
        DIVIDE ( CurrentProfit - PriorProfit, PriorProfit )
    )

Profit Margin YoY % =
VAR CurrentMargin = [Profit Margin %]
VAR PriorMargin =
    CALCULATE (
        [Profit Margin %],
        SAMEPERIODLASTYEAR ( 'Calendar'[Date] )
    )
RETURN
    IF (
        HASONEVALUE ( 'Calendar'[Year] ),
        CurrentMargin - PriorMargin
    )

Orders YoY % =
VAR CurrentOrders = [Total Orders]
VAR PriorOrders =
    CALCULATE (
        [Total Orders],
        SAMEPERIODLASTYEAR ( 'Calendar'[Date] )
    )
RETURN
    IF (
        HASONEVALUE ( 'Calendar'[Year] ),
        DIVIDE ( CurrentOrders - PriorOrders, PriorOrders )
    )

Quantity YoY % =
VAR CurrentQty = [Total Quantity]
VAR PriorQty =
    CALCULATE (
        [Total Quantity],
        SAMEPERIODLASTYEAR ( 'Calendar'[Date] )
    )
RETURN
    IF (
        HASONEVALUE ( 'Calendar'[Year] ),
        DIVIDE ( CurrentQty - PriorQty, PriorQty )
    )
```

---

## 3. Product Performance & Discount Impact Measures

These measures evaluate how promotional discounting impacts product margins and revenue concentration on Page 2.

* **`Avg Discount %`**: Calculates the average discount rate applied across order line items.
* **`Avg Product Base Margin`**: Computes the average baseline margin before order-level discounting.
* **`Discounted Sales`**: Isolates revenue generated strictly from orders where a discount (`Discount > 0`) was applied.
* **`Discounted Orders`**: Counts distinct orders that received a promotional discount.

```dax
Avg Discount % =
AVERAGE ( Orders[Discount] )

Avg Product Base Margin =
AVERAGE ( Orders[Product Base Margin] )

Discounted Sales =
CALCULATE (
    [Total Sales],
    Orders[Discount] > 0
)

Discounted Orders =
CALCULATE (
    [Total Orders],
    Orders[Discount] > 0
)
```

---

## 4. Operations, Shipping & Returns Measures

These measures track fulfillment speed, shipping expenses, and revenue/profit leakage from returned orders on Page 3.

* **`Total Shipping Cost`**: Sums total outbound freight and shipping expenses incurred across orders.
* **`Avg Days to Ship`**: Uses `AVERAGEX` and `DATEDIFF` to compute the average number of days between `Order Date` and `Ship Date` at the line-item level.
* **`Returned Orders`**: Counts distinct orders that have a matching return record in the `Returns` table (`NOT ( ISBLANK ( Returns[Status] ) )`).
* **`Return Rate %`**: Calculates the percentage of total orders that resulted in a return (`Returned Orders / Total Orders`).
* **`Lost Sales (Returns)`**: Quantifies gross sales revenue tied to returned orders.
* **`Lost Profit (Returns)`**: Quantifies net profit lost due to returned orders.

```dax
Total Shipping Cost =
SUM ( Orders[Shipping Cost] )

Avg Days to Ship =
AVERAGEX (
    Orders,
    DATEDIFF ( Orders[Order Date], Orders[Ship Date], DAY )
)

Returned Orders =
CALCULATE (
    DISTINCTCOUNT ( Orders[Order ID] ),
    NOT ( ISBLANK ( Returns[Status] ) )
)

Return Rate % =
DIVIDE (
    [Returned Orders],
    [Total Orders],
    0
)

Lost Sales (Returns) =
CALCULATE (
    [Total Sales],
    NOT ( ISBLANK ( Returns[Status] ) )
)

Lost Profit (Returns) =
CALCULATE (
    [Total Profit],
    NOT ( ISBLANK ( Returns[Status] ) )
)
```