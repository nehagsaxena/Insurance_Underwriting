# AAS to Databricks SQL Conversion

## Overview

This conversion transforms the Azure Analysis Services (AAS) tabular model `Sales Model.bim` into Databricks SQL.

## Generated Files

| File | Description |
|------|-------------|
| `databricks_conversion.sql` | Main SQL script with tables, views, and measures |
| `complex_dax_analysis.sql` | Analysis of complex DAX patterns with conversion strategies |
| `CONVERSION_README.md` | This documentation file |

## Model Summary

### Source Model
- **Name**: Sales Model
- **Original Database**: IKEA-DW (Azure SQL)
- **Compatibility Level**: 1500
- **Schema**: BV (Business Views)

### Converted Components

#### Dimension Tables (20+)
| Table | Description |
|-------|-------------|
| `dim_date` | Calendar/Hijri date dimension |
| `dim_time` | Time dimension (hour/minute) |
| `dim_store` | Store locations |
| `dim_item` | Products/Items |
| `dim_home_furnishing_business` | HFB product categories |
| `dim_product_area` | Product areas |
| `dim_agent_code` | Order agent codes |
| `dim_cashier` | Cashiers/Employees |
| `dim_delivery_location` | Geographic locations |
| `dim_order_type` | Order types |
| `dim_order_status` | Order statuses |
| `dim_sales_channel` | Sales channels |
| `dim_currency` | Currencies |
| `dim_transaction_type` | Transaction types |
| ... | (and more) |

#### Fact Tables (5+)
| Table | Description |
|-------|-------------|
| `fact_sales` | Main sales transactions |
| `fact_sales_ly` | Last year sales (for YoY) |
| `fact_visits` | Store visitor counts |
| `fact_sales_budget` | Sales budgets |
| `fact_item_movement` | Inventory movements |
| `fact_payment` | Payment transactions |

#### Views for Measures (15+)
| View | DAX Measure Equivalent |
|------|----------------------|
| `v_sales_measures_simple` | Sales Amount, Cost, Quantity, etc. |
| `v_transactions` | Transaction counts |
| `v_store_visits` | Store visit counts |
| `v_calculated_measures` | Average Check, Price Per Item, etc. |
| `v_sales_yoy` | Year-over-Year comparisons |
| `v_visitors` | Visitor metrics |
| `v_budget` | Budget measures |

## Conversion Complexity Analysis

### Simple Conversions (Direct Translation)
- `SUM()` aggregations
- `DISTINCTCOUNT()` -> `COUNT(DISTINCT ...)`
- `DIVIDE()` -> `CASE WHEN ... THEN ... / ... END`
- Basic calculated columns

### Medium Complexity
- `CALCULATE()` with filter modifications -> SQL `WHERE` + subqueries
- `SUMX()` with `RELATED()` -> SQL `JOIN` + aggregation
- Basic time intelligence using pre-calculated LY tables

### Complex (Requires Special Handling)
| Pattern | Challenge | Solution |
|---------|-----------|----------|
| `ALLSELECTED()` | Context removal | Window functions |
| Complex time intelligence | Multi-level date matching | Pre-calculated mapping tables |
| Bidirectional relationships | Filter propagation | Multiple JOINs / views |
| Dynamic date comparisons | Leap year adjustments | UDFs |

## Deployment Steps

### 1. Create Schema
```sql
CREATE SCHEMA IF NOT EXISTS sales_analytics;
USE sales_analytics;
```

### 2. Deploy Tables
Run the DDL statements from `databricks_conversion.sql` in order:
1. Dimension tables first
2. Then fact tables

### 3. Load Data
Create ETL pipelines to load data from source systems into Delta tables.

### 4. Deploy Views
Run the view creation statements.

### 5. Validate
Run data quality checks from `complex_dax_analysis.sql`.

## Key Differences: DAX vs SQL

| DAX Concept | SQL Equivalent |
|-------------|----------------|
| Filter context | WHERE clause / JOIN conditions |
| Row context | Current row in aggregation |
| CALCULATE() | Subquery with different WHERE |
| RELATED() | JOIN to dimension |
| RELATEDTABLE() | JOIN with GROUP BY |
| ALLSELECTED() | Window function OVER() |
| Time Intelligence | Date dimension JOINs / UDFs |
| Measures | Views or pre-aggregated tables |
| Calculated Columns | Generated columns or ETL |

## Semantic Layer Recommendations

Since Databricks SQL doesn't provide a semantic layer like AAS, consider:

1. **Unity Catalog** - For governance and discoverability
2. **dbt** - For metric definitions and documentation
3. **Power BI on Databricks** - For end-user semantic layer
4. **Databricks AI/BI Dashboards** - For self-service analytics

## Measures Not Converted (Need Manual Review)

1. Marketing campaign measures (complex joins)
2. Customer insights percentages
3. OTIF delivery metrics
4. Out of Stock calculations
5. Forecast vs Actual comparisons

## Performance Considerations

1. **Partitioning**: Fact tables partitioned by `dt_srgt` (date)
2. **Z-Ordering**: Consider Z-ORDER on frequently filtered columns
3. **Caching**: Use aggregate tables for dashboard performance
4. **Liquid Clustering**: Consider for frequently updated tables

## Contact

For questions about specific DAX measure conversions, refer to the detailed analysis in `complex_dax_analysis.sql`.
