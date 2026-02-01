# Guide: `view_monthly_active_client_count`

## Overview

This view provides **historical monthly snapshots** of active client counts going back to October 2022 (the earliest data in your system). It generates month-end snapshots and counts active clients using the same criteria as `view_client_count_2025_master`.

## How It Works

### Approach: Month-End Snapshots

The view uses a **time-series approach**:

1. **Generates Month-End Dates**: Creates a series of month-end dates from the earliest membership start date to the current month
2. **Snapshot Logic**: For each month-end date, determines which memberships were active on that specific date
3. **Deduplication**: Uses the same logic as `view_client_count_2025_master` - only counts the most recent membership per member
4. **Filtering**: Applies the same filters (excludes "no_sale" and "f&f" status)

### SQL Structure

```sql
WITH month_ends AS (
    -- Generate month-end dates
    SELECT (DATE_TRUNC('month', month_date) + INTERVAL '1 month' - INTERVAL '1 day')::date AS month_end_date
    FROM generate_series(...)
),
active_memberships_by_month AS (
    -- For each month-end, find active memberships
    SELECT ..., ROW_NUMBER() OVER (PARTITION BY month_end_date, member_name ORDER BY start_date DESC) as rn
    FROM month_ends
    INNER JOIN member_memberships ON (start_date <= month_end_date AND end_date > month_end_date)
    WHERE ...
)
SELECT month_end_date, COUNT(*) as active_client_count, ...
FROM active_memberships_by_month
WHERE rn = 1  -- Only most recent membership per member
GROUP BY month_end_date
```

## Output Columns

| Column | Description |
|--------|-------------|
| `month_end_date` | The last day of the month (e.g., 2025-01-31) |
| `active_client_count` | Total number of distinct active clients on that date |
| `gym_count` | Number of unique gyms with active clients |
| `unique_coaches` | Number of unique coaches with active clients |
| `bridge_street_count` | Active clients at Bridge Street (if applicable) |
| `bligh_street_count` | Active clients at Bligh Street (if applicable) |
| `collin_street_count` | Active clients at Collin Street (if applicable) |

## Usage Examples

### 1. Get Full Historical Trend

```sql
SELECT 
    month_end_date,
    active_client_count,
    LAG(active_client_count) OVER (ORDER BY month_end_date) as previous_month_count,
    active_client_count - LAG(active_client_count) OVER (ORDER BY month_end_date) as month_over_month_change
FROM view_monthly_active_client_count
ORDER BY month_end_date;
```

### 2. Get Last 12 Months

```sql
SELECT 
    month_end_date,
    active_client_count
FROM view_monthly_active_client_count
WHERE month_end_date >= CURRENT_DATE - INTERVAL '12 months'
ORDER BY month_end_date DESC;
```

### 3. Calculate Growth Rate

```sql
SELECT 
    month_end_date,
    active_client_count,
    ROUND(
        100.0 * (active_client_count - LAG(active_client_count) OVER (ORDER BY month_end_date)) 
        / NULLIF(LAG(active_client_count) OVER (ORDER BY month_end_date), 0),
        2
    ) as growth_rate_percent
FROM view_monthly_active_client_count
ORDER BY month_end_date;
```

### 4. Year-over-Year Comparison

```sql
SELECT 
    EXTRACT(MONTH FROM month_end_date) as month_number,
    EXTRACT(YEAR FROM month_end_date) as year,
    active_client_count,
    LAG(active_client_count) OVER (
        PARTITION BY EXTRACT(MONTH FROM month_end_date) 
        ORDER BY EXTRACT(YEAR FROM month_end_date)
    ) as previous_year_count
FROM view_monthly_active_client_count
WHERE EXTRACT(YEAR FROM month_end_date) IN (2024, 2025)
ORDER BY month_number, year;
```

### 5. Get Current Month Count

```sql
SELECT 
    month_end_date,
    active_client_count
FROM view_monthly_active_client_count
WHERE month_end_date = (
    SELECT (DATE_TRUNC('month', CURRENT_DATE) + INTERVAL '1 month' - INTERVAL '1 day')::date
);
```

### 6. Find Peak Month

```sql
SELECT 
    month_end_date,
    active_client_count
FROM view_monthly_active_client_count
WHERE active_client_count = (SELECT MAX(active_client_count) FROM view_monthly_active_client_count);
```

## Key Features

### ✅ Automatic Date Range
- Automatically starts from the earliest membership date in your database
- Extends to the current month
- No manual date range configuration needed

### ✅ Consistent Filtering
- Uses the same business rules as `view_client_count_2025_master`:
  - Excludes "no_sale" journey stages
  - Excludes "f&f" status
  - Only counts most recent membership per member

### ✅ Performance Optimized
- Uses `ROW_NUMBER()` window function for efficient deduplication
- Indexes on `start_date` and `end_date` help with date range filtering

## Data Range

Based on your data:
- **Earliest**: October 2022 (3 active clients)
- **Latest**: Current month
- **Total Months**: ~40+ months of history

## Monthly Growth Trend (Sample)

From the data:
- **Oct 2022**: 3 clients
- **Dec 2023**: 177 clients
- **Dec 2024**: 294 clients
- **Jan 2026**: 388 clients (peak)
- **Feb 2026**: 359 clients

## Comparison with `view_client_count_2025_master`

| Feature | `view_client_count_2025_master` | `view_monthly_active_client_count` |
|--------|--------------------------------|-----------------------------------|
| **Purpose** | Current active clients | Historical monthly snapshots |
| **Time Period** | Current date only | All historical months |
| **Output** | Individual client records | Aggregated counts per month |
| **Use Case** | List current clients | Trend analysis, reporting |

## Performance Considerations

### Indexes That Help
- Index on `member_memberships.start_date`
- Index on `member_memberships.end_date`
- Composite index: `(start_date, end_date, journey_stage, status)`

### Query Performance
- The view uses `generate_series` which is efficient
- Window functions (`ROW_NUMBER`) are optimized in PostgreSQL
- For very large datasets, consider materializing the view

### Materialized View Option

If you need faster queries and can accept slightly stale data:

```sql
CREATE MATERIALIZED VIEW view_monthly_active_client_count_materialized AS
SELECT * FROM view_monthly_active_client_count;

CREATE UNIQUE INDEX ON view_monthly_active_client_count_materialized (month_end_date);

-- Refresh monthly (add to cron job)
REFRESH MATERIALIZED VIEW view_monthly_active_client_count_materialized;
```

## Troubleshooting

### Issue: Gym counts showing 0
**Solution**: Check the actual gym name values in your `member_memberships.gym` column. The view currently filters for exact matches like 'Bridge Street', 'Bligh Street', 'Collin Street'. You may need to update the FILTER clauses to match your actual gym names.

### Issue: Count seems too high/low
**Solution**: Verify the filtering logic matches your business rules. Check:
- Are "no_sale" memberships being excluded?
- Are "f&f" memberships being excluded?
- Is the deduplication working correctly?

### Issue: Missing months
**Solution**: The view generates months from the earliest `start_date` to current. If you see gaps, it means no memberships were active during those months (which is normal).

## Future Enhancements

Potential improvements you could add:

1. **Breakdown by Journey Stage**: Add counts by journey_stage (new_member, renewed_member, etc.)
2. **Coach-Level Metrics**: Add per-coach counts per month
3. **Retention Metrics**: Calculate retention rates month-over-month
4. **Churn Analysis**: Track clients who dropped off
5. **Cohort Analysis**: Track cohorts by their start month

## Related Views

- `view_client_count_2025_master` - Current active clients (detailed)
- `view_client_count_expired` - Expired memberships
- `view_member_count_per_coach` - Coach-level aggregations
