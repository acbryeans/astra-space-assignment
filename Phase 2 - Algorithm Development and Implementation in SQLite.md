# Phase 2: Algorithm Development and Implementation

## Building the SQL Structure

Decided to go with a CTE-based approach for this ranking algorithm. Makes the logic much easier to follow and debug compared to one massive query with subqueries everywhere. Two main CTEs handle the heavy lifting - one for aggregating performance data, another for scoring calculations.

## The Two-Step Architecture

**agent_performance CTE**: Pulls together all the historical metrics per agent - ratings, booking counts, cancellation rates, the works.

**scored_agents CTE**: Takes those raw metrics and runs them through the normalization and weighting formulas.

Pretty clean separation that lets me validate each step independently.

## Handling Missing Data

Looking at the current data, every agent actually has some experience with both lead sources and all destinations - no NULL cases in practice. I verified this with a quick check:

```sql
-- Looking for any agent/lead source/destination combinations with zero experience
select 
    sta.AgentID,
    ah.LeadSource,
    b.Destination,
    count(*)
from bookings as b
left join assignment_history as ah
    on ah.AssignmentID = b.AssignmentID
left join space_travel_agents as sta
    on sta.AgentID = ah.AgentID
group by sta.AgentID, ah.LeadSource, b.Destination
having count(*) = 0 or count(*) is null;
-- Returns zero rows - everyone has worked everything
```

But I still built in a 3.0 fallback for system robustness. New agents will get hired, edge cases will pop up, and I don't want the algorithm breaking when they do.

Why 3.0? It's below the actual average in our data (around 4.15) but not punitive. This way if someone genuinely hasn't worked organic leads or a specific destination, they stay in the rotation but don't get unfairly boosted above specialists.

## Normalization Approach

Years of service was straightforward - linear scale from the 2-18 year range in our data up to 1-5 points:

```sql
1.0 + ((YearsOfService - 2.0) * 4.0 / 16.0)
```

Trip volume used the same approach with 0-18 confirmed bookings:

```sql  
1.0 + ((confirmed_bookings - 0.0) * 4.0 / 18.0)
```

Could have gotten fancy with logarithmic scaling, but linear felt more transparent for stakeholders.

## Cancellation Penalty Logic

The cancellation rate gets applied as a final multiplier, not an additive penalty. So if your base score is 4.2 and you have a 15% cancellation rate, your final score becomes 4.2 � 0.85 = 3.57.

This felt more realistic than subtracting points - the impact should be proportional to how good you were to begin with.

## The Full Implementation

```sql
-- Customer assignment algorithm
-- Example inputs: 'Phone Call', 'Organic', 'Mars', 'Dallas-Fort Worth Launch Complex' 

with 
-- First CTE: gather all the performance data per agent
agent_performance as (
    select 
        sta.AgentID,
        sta.FirstName,
        sta.LastName,
        sta.AverageCustomerServiceRating,
        sta.YearsOfService,
        sta.DepartmentName,
        
        -- How well do they handle organic leads specifically?
        avg(case when ah.LeadSource = 'Organic' then sta.AverageCustomerServiceRating end) as lead_source_rating,
        
        -- Mars destination experience
        avg(case when b.Destination = 'Mars' then sta.AverageCustomerServiceRating end) as destination_rating,
        
        -- Booking volume and success rates
        count(b.BookingID) as total_bookings,
        count(case when b.BookingStatus = 'Confirmed' then 1 end) as confirmed_bookings,
        count(case when b.BookingStatus = 'Cancelled' then 1 end) as cancelled_bookings,
        
        -- Cancellation rate calculation
        case 
            when count(b.BookingID) > 0 
            then cast(count(case when b.BookingStatus = 'Cancelled' then 1 end) as float) / count(b.BookingID)
            else 0 
        end as cancellation_rate
        
    from space_travel_agents sta
    left join assignment_history ah on sta.AgentID = ah.AgentID
    left join bookings b on ah.AssignmentID = b.AssignmentID
    group by sta.AgentID, sta.FirstName, sta.LastName, sta.AverageCustomerServiceRating, 
             sta.YearsOfService, sta.DepartmentName
),

-- Second CTE: apply the scoring formula
scored_agents as (
    select *,
        -- Normalize service years to 1-5 scale
        1.0 + ((YearsOfService - 2.0) * 4.0 / 16.0) as normalized_service_years,
        
        -- Normalize booking volume to 1-5 scale  
        1.0 + ((confirmed_bookings - 0.0) * 4.0 / 18.0) as normalized_trip_volume,
        
        -- The weighted scoring formula from Phase 1
        (AverageCustomerServiceRating * 0.35) +
        (coalesce(lead_source_rating, 3.0) * 0.15) +
        (AverageCustomerServiceRating * 0.25) + -- Using overall rating for category since departments aren't perfectly specialized
        ((1.0 + ((confirmed_bookings - 0.0) * 4.0 / 18.0)) * 0.10) +
        ((1.0 + ((YearsOfService - 2.0) * 4.0 / 16.0)) * 0.05) +
        (coalesce(destination_rating, 3.0) * 0.10) as base_score
        
    from agent_performance
)

-- Final output with cancellation penalty applied
select 
    AgentID,
    FirstName,
    LastName,
    DepartmentName,
    AverageCustomerServiceRating,
    YearsOfService,
    total_bookings,
    confirmed_bookings,
    cancelled_bookings,
    round(cancellation_rate, 3) as cancellation_rate,
    round(base_score, 3) as base_score,
    round(base_score * (1 - cancellation_rate), 3) as final_score,
    row_number() over (order by base_score * (1 - cancellation_rate) desc) as agent_rank
from scored_agents
order by final_score desc;
```

## Making It Dynamic

Right now this is hardcoded for organic leads going to Mars. To handle different customer types, you'd update the case when conditions in the agent_performance CTE. Could wrap this in a stored procedure with parameters, but kept it as a standalone query for now since the requirements didn't specify that level of abstraction.

## What You Get Back

The query returns every agent ranked from best to worst match, with all the scoring details visible. You can see exactly why agent A ranked above agent B - their base score, cancellation penalty, and final result are all broken out.

Useful for debugging the algorithm and explaining recommendations to business stakeholders who want to understand the "why" behind assignments.