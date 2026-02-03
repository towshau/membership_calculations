# Analysis: `view_membership_costs`

## Overview
`view_membership_costs` is a complex PostgreSQL view that calculates membership costs, margins, and profitability metrics for gym memberships. It uses Common Table Expressions (CTEs) to break down the calculation into logical steps.

## View Structure

The view is built using **4 CTEs (Common Table Expressions)** that process data sequentially:

### 1. `base` CTE - Data Foundation
**Purpose**: Gathers all base data from related tables and performs initial calculations.

**Key Operations**:
- Joins `member_memberships` with:
  - `member_database` (member info)
  - `membership_types` (membership type details)
  - `member_newsale_metadata` (new sale metadata)
  - `member_renewal_meta` (renewal metadata)
- Calculates `sale_group_id` using `COALESCE(primary_membership_id, id)` to group related memberships
- Determines `is_primary` and `is_child` flags based on `primary_membership_id`
- Extracts base costs from `system_config` table for:
  - PERFORM sessions (`match_pattern = 'PERFORM'`)
  - VO2 sessions (`match_pattern = 'VO2'`)
  - RM (Results Manager) costs (`match_pattern = 'RM'`)
- Calculates `membership_weeks_true` from `test_duration` field:
  - Contains "3" → 12 weeks
  - Contains "6" → 26 weeks
  - Contains "12" → 52 weeks
  - Default → 26 weeks

**Key Fields Extracted**:
- Membership identifiers (id, member_id, membership_type_id)
- Dates (start_date, end_date)
- Values from metadata (nsm_value, rmm_value)
- Total sessions from metadata
- Gym location

### 2. `child_groups` CTE - Grouping Logic
**Purpose**: Aggregates child membership IDs for each sale group.

**Key Operations**:
- Groups by `sale_group_id`
- Creates an array of `child_membership_ids` ordered by membership_id
- This allows tracking all memberships that belong to the same sale group

### 3. `calc` CTE - Cost Calculations
**Purpose**: Performs the core cost calculations for each membership type.

**Key Calculations**:

#### Adjusted Sessions
- **Online Coaching**: `NULL` (no session count)
- **Flexible/Pack memberships**: Uses `total_sessions` from metadata
- **Standard memberships**: `session_frequency_per_week * membership_weeks_true`

#### PERFORM Cost
- **Online Coaching**: `0`
- **VO2 memberships**: `0` (VO2 has separate cost calculation)
- **Flexible/Pack**: `total_sessions * perform_base_cost`
- **Standard**: `(session_frequency_per_week * membership_weeks_true) * perform_base_cost`

#### VO2 Cost
- **Online Coaching**: `0`
- **VO2 memberships**: Uses `vo2_base_cost` with same session calculation logic
- **Other memberships**: `0`

#### RM Cost
- **Online Coaching**: `0`
- **Pack memberships**: `0`
- **Primary memberships only**: `rm_base_cost * membership_weeks_true`
- **Child memberships**: `0` (RM cost only applies to primary)

#### Membership Value
- Uses `COALESCE(nsm_value, rmm_value)` to get value from either new sale or renewal metadata
- Calculates `membership_value_ex_gst` by dividing by 1.1 (removing 10% GST)

### 4. `sale_group_totals` CTE - Aggregation
**Purpose**: Sums total costs across all memberships in a sale group.

**Key Operations**:
- Groups by `sale_group_id`
- Calculates `total_overall_cost = SUM(perform_cost + vo2_cost + rm_cost)`
- This ensures costs are aggregated at the sale group level (important for memberships with child memberships)

## Final SELECT - Output Columns

The final output includes:

### Identifiers
- `membership_id`, `member_id`, `member_name`
- `membership_type_id`, `membership_type_name`
- `sale_group_id`, `primary_membership_id`
- `is_primary`, `is_child` flags

### Dates & Location
- `start_date`, `end_date`
- `gym`, `gym_from_metadata`

### Metadata References
- `newsale_metadata`, `renewal_metadata`
- `newsale_meta_id`, `renewal_meta_id`
- `nsm_value`, `rmm_value`

### Cost Components
- `perform_base_cost`, `vo2_base_cost`, `rm_base_cost`
- `perform_cost`, `vo2_cost`, `rm_cost`
- `total_overall_cost` (aggregated at sale group level)

### Sessions & Duration
- `session_frequency_per_week`
- `total_sessions`, `adjusted_sessions`
- `membership_weeks_true`
- `child_membership_ids` (array)

### Financial Metrics
- `membership_value` (with GST)
- `membership_value_ex_gst` (without GST)
- `margin` = `membership_value_ex_gst - total_overall_cost`
- `margin_percent` = `1 - (total_overall_cost / membership_value_ex_gst)`

### Special Handling
- **Online Coaching**: `margin` and `margin_percent` are set to `NULL` (excluded from profitability calculations)

## Key Design Patterns

### 1. Sale Group Logic
The view handles memberships that can have child memberships (via `primary_membership_id`). This allows:
- Grouping related memberships together
- Aggregating costs at the sale group level
- Tracking which memberships are primary vs. child

### 2. Membership Type Specialization
Different membership types have different cost calculation rules:
- **Standard memberships**: Calculate based on frequency × weeks
- **Flexible/Pack**: Use total_sessions from metadata
- **VO2**: Separate VO2 cost calculation
- **Online Coaching**: Excluded from cost calculations

### 3. Metadata Priority
The view uses `COALESCE` to prioritize:
- New sale metadata over renewal metadata
- This ensures the most relevant pricing information is used

### 4. GST Handling
Membership values are stored with GST included, but margin calculations use values excluding GST (divided by 1.1).

## Evolution History

Based on migration history, the view has been updated several times:

1. **Initial Creation**: (Not found in current migrations - may have been created manually)
2. **20251208214803**: `update_view_membership_costs_flexible_pack_logic` - Added logic for Flexible/Pack memberships
3. **20251208215202**: `add_gym_column_to_view_membership_costs` - Added gym column
4. **20251208234840**: `add_newsale_renewal_meta_id_columns_to_view_membership_costs` - Added metadata ID columns
5. **20251208235040**: `add_gym_from_metadata_to_view_membership_costs` - Added gym from metadata
6. **20251209001110**: `exclude_online_coaching_from_cost_calculations` - Excluded online coaching from costs
7. **20251209001143**: `set_online_coaching_margin_to_null` - Set margins to NULL for online coaching

## Dependencies

### Tables Used
- `member_memberships` (primary table)
- `member_database`
- `membership_types`
- `member_newsale_metadata`
- `member_renewal_meta`
- `system_config` (for base costs)

### Key Relationships
- `member_memberships.member_id` → `member_database.id`
- `member_memberships.membership_type_id` → `membership_types.id`
- `member_memberships.newsale_metadata` → `member_newsale_metadata.id`
- `member_memberships.renewal_metadata` → `member_renewal_meta.id`
- `member_memberships.primary_membership_id` → `member_memberships.id` (self-referential)

## Business Logic Insights

1. **Cost Structure**: The view calculates three types of costs:
   - **PERFORM**: Per-session cost for standard training sessions
   - **VO2**: Per-session cost for VO2 testing sessions
   - **RM**: Weekly cost for Results Manager services (only on primary memberships)

2. **Profitability**: Margin is calculated as revenue (ex-GST) minus total costs, with percentage margin showing the profit margin ratio.

3. **Special Cases**: Online coaching memberships are explicitly excluded from cost and margin calculations, likely because they have a different cost structure.

4. **Sale Grouping**: The view supports complex sale structures where a primary membership can have child memberships, with costs aggregated at the group level.

## RM Cost Application Rules

### Current RM Cost Logic

The RM cost is calculated using the following SQL CASE statement logic:

```sql
CASE
    WHEN membership_type_name LIKE '%online%coaching%' THEN 0
    WHEN membership_type_name LIKE '%Pack%' THEN 0
    WHEN is_primary = true THEN rm_base_cost * membership_weeks_true
    ELSE 0
END AS rm_cost
```

### Rules Summary

**RM Cost = 0 (Excluded) when:**
- **Online Coaching memberships**: Any membership type name containing "online coaching" (case-insensitive)
- **Pack memberships**: Any membership type name containing "Pack" (case-insensitive)
- **Child memberships**: Any membership where `is_child = true` or `primary_membership_id IS NOT NULL`

**RM Cost Applies when:**
- **Primary membership**: `is_primary = true` (or `primary_membership_id IS NULL`)
- **Not Online Coaching**: Membership type does not contain "online coaching"
- **Not a Pack membership**: Membership type does not contain "Pack"

### Calculation Formula

When RM costs apply, the calculation is:
- **Formula**: `RM Cost = rm_base_cost × membership_weeks_true`
- **Current `rm_base_cost`**: $33.17 per week (retrieved from `system_config` table where `match_pattern = 'RM'`)

The `membership_weeks_true` is determined from the `test_duration` field:
- Contains "3" → 12 weeks
- Contains "6" → 26 weeks
- Contains "12" → 52 weeks
- Default → 26 weeks

### Examples from Data

Real-world examples showing how RM costs are applied:

- **6 Month SILVER - x2 (Primary)**: RM cost = $33.17 × 26 weeks = **$862.42**
- **12 Month GOLD - x3 (Primary)**: RM cost = $33.17 × 52 weeks = **$1,724.84**
- **3 Month SILVER - x2 (Primary)**: RM cost = $33.17 × 12 weeks = **$398.04**
- **6 Month Vo2 BASE - x 1 (Child)**: RM cost = **$0** (child membership, excluded)
- **Vo2 10-Pack (Pack)**: RM cost = **$0** (Pack membership, excluded)
- **Boxing 10-Pack (Pack)**: RM cost = **$0** (Pack membership, excluded)
- **Online Coaching – Program Only – 6 Months**: RM cost = **$0** (Online Coaching, excluded)
- **12 Month Vo2 BASE - x 1 (Child)**: RM cost = **$0** (child membership, excluded)

### Current Base Cost

The RM base cost is stored in the `system_config` table:
- **Config Key**: `offloor_rm`
- **Match Pattern**: `RM`
- **Current Value**: $33.17 per week

This value is retrieved via a subquery in the `base` CTE and used for all RM cost calculations.

## Potential Improvements

1. **Performance**: The view uses subqueries to fetch base costs from `system_config`. Consider joining this table instead.

2. **Maintainability**: The membership type matching uses `LIKE` patterns (`~~*`). Consider using a more explicit membership type categorization.

3. **Documentation**: Add comments to explain the business logic for each cost calculation type.

4. **Testing**: The view would benefit from test cases covering:
   - Standard memberships
   - Flexible/Pack memberships
   - VO2 memberships
   - Online coaching
   - Primary/child membership relationships

