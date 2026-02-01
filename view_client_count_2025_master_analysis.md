# Analysis: `view_client_count_2025_master`

## Overview
`view_client_count_2025_master` is a PostgreSQL view designed to provide a **master list of active clients** for 2025. It uses `DISTINCT ON` to ensure each member appears only once, selecting their most recent active membership. The view includes comprehensive coach assignment information (primary coach, programming coach, and handoff coach).

## View Structure

The view is a **single SELECT statement** with:
- **DISTINCT ON** clause for deduplication
- **Multiple LEFT JOINs** to enrich membership data with coach names
- **Filtering conditions** to include only active, valid memberships

## SQL Breakdown

### DISTINCT ON Clause
```sql
SELECT DISTINCT ON (mm.member_name) ...
ORDER BY mm.member_name, mm.start_date DESC
```

**Purpose**: Ensures each member appears only once in the result set.

**How it works**:
- `DISTINCT ON (mm.member_name)` groups rows by member name
- `ORDER BY mm.member_name, mm.start_date DESC` ensures the most recent membership (by start_date) is selected for each member
- This handles cases where a member has multiple memberships - only the most recent one is included

**Business Logic**: When a member has multiple active memberships, the view prioritizes the most recently started membership.

### Data Source - Base Table
**Primary Table**: `member_memberships` (aliased as `mm`)

This table contains:
- Membership records with start/end dates
- Member identifiers and names
- Coach assignments (coach_id, programming_coach_id, handoff_coach_id)
- Status and journey stage information
- Gym location

### JOIN Operations

The view performs **3 LEFT JOINs** to enrich membership data with coach names:

#### 1. Primary Coach Join
```sql
LEFT JOIN staff_database coach ON (mm.coach_id = coach.id)
```
- **Purpose**: Gets the name of the primary Results Manager (RM) coach
- **Output**: `coach.coach_name`
- **Why LEFT JOIN**: Some memberships may not have a primary coach assigned

#### 2. Programming Coach Join
```sql
LEFT JOIN staff_database programming_coach ON (mm.programming_coach_id = programming_coach.id)
```
- **Purpose**: Gets the name of the programming coach responsible for training programs
- **Output**: `programming_coach.coach_name AS programming_coach_name`
- **Why LEFT JOIN**: Not all memberships require a programming coach

#### 3. Handoff Coach Join
```sql
LEFT JOIN staff_database handoff_coach ON (mm.handoff_coach_id = handoff_coach.id)
```
- **Purpose**: Gets the name of the handoff coach (typically assigned during onboarding)
- **Output**: `handoff_coach.coach_name AS handoff_coach_name`
- **Why LEFT JOIN**: Handoff coaches are temporary assignments during the onboarding period

**Why LEFT JOINs?**: All three coach assignments are optional, so LEFT JOINs ensure memberships without assigned coaches still appear in the results.

### Filtering Conditions (WHERE Clause)

The view applies **3 filtering conditions** to ensure only valid, active memberships are included:

#### 1. Active Membership Filter
```sql
mm.end_date > CURRENT_DATE
```
- **Purpose**: Only includes memberships that haven't expired
- **Logic**: End date must be in the future
- **Business Rule**: Expired memberships are excluded from active client counts

#### 2. Exclude "No Sale" Memberships
```sql
mm.journey_stage IS DISTINCT FROM 'no_sale'::journey_stage_type
```
- **Purpose**: Excludes memberships where no sale was completed
- **Logic**: Uses `IS DISTINCT FROM` to handle NULL values correctly (unlike `<>` which would exclude NULLs)
- **Business Rule**: Prospects who didn't convert to members are excluded

#### 3. Exclude "F&F" (Friends & Family) Status
```sql
mm.status IS DISTINCT FROM 'f&f'::text
```
- **Purpose**: Excludes Friends & Family memberships from client counts
- **Logic**: Uses `IS DISTINCT FROM` for NULL-safe comparison
- **Business Rule**: F&F memberships are typically discounted or complimentary and shouldn't count toward regular client metrics

**Why `IS DISTINCT FROM`?**: 
- Standard `<>` operator returns NULL when comparing with NULL values
- `IS DISTINCT FROM` treats NULL as a distinct value, allowing proper filtering
- This ensures memberships with NULL status or journey_stage are handled correctly

### Output Columns

The view returns the following columns:

#### Membership Identifiers
- `mm.id` - Membership record ID
- `mm.member_name` - Member's name (used for DISTINCT ON grouping)

#### Dates
- `mm.start_date` - Membership start date
- `mm.end_date` - Membership end date

#### Location & Status
- `mm.gym` - Gym location (e.g., "Bridge Street", "Bligh Street", "Collin Street")
- `mm.status` - Membership status (e.g., "active", "indefinite_hold", "f&f")
- `mm.journey_stage` - Member's journey stage (e.g., "new_member", "renewed_member", "expired")

#### Coach Assignments (IDs)
- `mm.coach_id` - Primary Results Manager coach ID
- `mm.programming_coach_id` - Programming coach ID
- `mm.handoff_coach_id` - Handoff/onboarding coach ID

#### Coach Assignments (Names)
- `coach.coach_name` - Primary RM coach name
- `programming_coach.coach_name AS programming_coach_name` - Programming coach name
- `handoff_coach.coach_name AS handoff_coach_name` - Handoff coach name

## Key Design Patterns

### 1. Deduplication Strategy
**Pattern**: `DISTINCT ON` with `ORDER BY`

**Why this approach?**:
- Members can have multiple active memberships (e.g., primary + secondary)
- For client counting purposes, each member should appear only once
- Most recent membership is typically the most relevant

**Alternative approaches considered**:
- `GROUP BY` with `MAX(start_date)` - More complex, requires subqueries
- Window functions with `ROW_NUMBER()` - More verbose
- `DISTINCT ON` is the most efficient for this use case

### 2. Coach Name Enrichment
**Pattern**: Multiple LEFT JOINs to the same table with different aliases

**Why this approach?**:
- All coach assignments reference the same `staff_database` table
- Different aliases allow joining the same table multiple times
- LEFT JOINs ensure memberships without coach assignments still appear

**Performance consideration**: 
- All joins use indexed foreign keys (`coach_id`, `programming_coach_id`, `handoff_coach_id`)
- LEFT JOINs are efficient when foreign keys are indexed

### 3. NULL-Safe Filtering
**Pattern**: `IS DISTINCT FROM` instead of `<>`

**Why this matters**:
- Some memberships may have NULL `status` or `journey_stage`
- Standard `<>` operator returns NULL when comparing with NULL
- `IS DISTINCT FROM` properly handles NULL values in comparisons

**Example**:
```sql
-- Standard operator (problematic with NULLs)
WHERE status <> 'f&f'  -- Excludes rows where status IS NULL

-- NULL-safe operator (correct)
WHERE status IS DISTINCT FROM 'f&f'  -- Includes rows where status IS NULL
```

### 4. Active Membership Definition
**Pattern**: Date-based filtering with `CURRENT_DATE`

**Business logic**:
- Membership is "active" if `end_date > CURRENT_DATE`
- This is evaluated at query time, so results are always current
- No need for status flags that might get out of sync

## Related Views

The database contains several related views for different client counting scenarios:

### `view_client_count_expired`
- Similar structure to `view_client_count_2025_master`
- Filters for `journey_stage = 'expired'`
- Used for tracking expired memberships

### `view_member_count_per_coach`
- Aggregates client counts by coach
- Includes metrics like:
  - Total active clients
  - Indefinite hold clients
  - New sale clients
  - First 30 days clients
  - Handoff members
  - RM capacity remaining

### `view_test_client_count`
- Simpler version for testing
- Uses different filtering logic
- Includes window function for total count

## Business Logic Insights

### 1. Client Counting Rules
The view implements specific business rules for what counts as an "active client":
- ✅ Must have an active membership (end_date in future)
- ✅ Must have completed a sale (not "no_sale")
- ✅ Must not be Friends & Family status
- ✅ Only most recent membership per member is counted

### 2. Coach Assignment Hierarchy
The view tracks three types of coach assignments:
- **Primary Coach (RM)**: Ongoing relationship manager
- **Programming Coach**: Responsible for training program design
- **Handoff Coach**: Temporary assignment during onboarding

This allows tracking of:
- Who is currently managing the client
- Who designed their program
- Who onboarded them

### 3. Membership Deduplication
When a member has multiple active memberships:
- Only the most recent (by start_date) is included
- This prevents double-counting in client metrics
- Ensures accurate client counts for reporting

## Dependencies

### Tables Used
- `member_memberships` (primary table)
  - Contains membership records with dates, status, and coach assignments
- `staff_database` (joined 3 times)
  - Contains staff/coach information
  - Provides coach names for all three coach types

### Key Relationships
- `member_memberships.coach_id` → `staff_database.id`
- `member_memberships.programming_coach_id` → `staff_database.id`
- `member_memberships.handoff_coach_id` → `staff_database.id`

### Enums Used
- `journey_stage_type` - Enum for member journey stages
  - Values include: 'no_sale', 'expired', 'new_member', 'renewed_member', etc.

## Evolution History

Based on migration history, related views have been updated:

1. **20251204235640**: `update_view_member_count_exclude_primary_membership`
   - Likely updated to exclude primary memberships from counts (to avoid double-counting)

2. **20251205000644**: `fix_handoff_coach_counting_logic`
   - Fixed logic for counting handoff coaches
   - May have affected how handoff coaches are included in counts

3. **20251208063719**: `sync_member_coach_from_handoff_coach`
   - Added logic to sync primary coach from handoff coach
   - May have affected coach assignments in this view

**Note**: The specific creation date of `view_client_count_2025_master` is not found in the migration history. It may have been:
- Created manually via SQL
- Created in a migration not currently tracked
- Created as part of a larger migration

## Use Cases

This view is likely used for:

1. **Client Reporting**: Generating lists of active clients for management reports
2. **Coach Workload Analysis**: Understanding which clients are assigned to which coaches
3. **Client Count Metrics**: Calculating total active client counts for KPIs
4. **Dashboard Displays**: Showing active client lists in business intelligence tools
5. **Client Segmentation**: Filtering clients by gym, status, or journey stage

## Performance Considerations

### Indexes That Help
- Index on `member_memberships.end_date` - Speeds up active membership filter
- Index on `member_memberships.member_name` - Helps with DISTINCT ON
- Index on `member_memberships.start_date` - Helps with ORDER BY
- Foreign key indexes on coach_id columns - Speeds up JOINs

### Potential Optimizations
1. **Materialized View**: If this view is queried frequently, consider making it a materialized view with refresh logic
2. **Partial Index**: Create a partial index on active memberships:
   ```sql
   CREATE INDEX idx_active_memberships 
   ON member_memberships (member_name, start_date DESC)
   WHERE end_date > CURRENT_DATE 
     AND journey_stage IS DISTINCT FROM 'no_sale'
     AND status IS DISTINCT FROM 'f&f';
   ```
3. **Covering Index**: Include frequently selected columns in indexes to avoid table lookups

## Potential Improvements

1. **Add Member ID**: Consider including `member_id` in addition to `member_name` for more reliable identification
2. **Add Membership Type**: Include membership type information for segmentation
3. **Add Created/Updated Timestamps**: Track when records were created/updated
4. **Add Comments**: Document the business logic in view comments
5. **Consider Materialized View**: If performance is an issue, materialize with refresh strategy
6. **Add Index Hints**: If query planner isn't optimal, consider index hints

## Example Queries

### Count Active Clients by Gym
```sql
SELECT 
    gym,
    COUNT(*) as active_clients
FROM view_client_count_2025_master
GROUP BY gym
ORDER BY active_clients DESC;
```

### List Clients by Coach
```sql
SELECT 
    coach_name,
    COUNT(*) as client_count,
    array_agg(member_name) as clients
FROM view_client_count_2025_master
WHERE coach_name IS NOT NULL
GROUP BY coach_name
ORDER BY client_count DESC;
```

### Find Clients Without Primary Coach
```sql
SELECT *
FROM view_client_count_2025_master
WHERE coach_name IS NULL;
```
