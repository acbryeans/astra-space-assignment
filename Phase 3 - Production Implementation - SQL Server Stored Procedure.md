# Phase 3: Production Implementation - SQL Server Stored Procedure

## Adapting for SQL Server

The prototype worked fine in SQLite, but production means SQL Server and all the T-SQL quirks that come with it. Had to rework several pieces to make this actually deployable in their environment.

## Production-Ready Stored Procedure

```sql
-- Agent assignment algorithm for production use
-- Parameterized for any customer profile

create procedure sp_GetRankedAgents
    @CustomerName nvarchar(100),
    @CommunicationMethod nvarchar(20),
    @LeadSource nvarchar(20), 
    @Destination nvarchar(50),
    @LaunchLocation nvarchar(100)
as
begin
    set nocount on;
    
    -- Validate inputs early to avoid debugging nightmares later
    -- These exact values come from analyzing the assignment_history table
    if @LeadSource not in ('Organic', 'Bought')
    begin
        raiserror('LeadSource must be either Organic or Bought', 16, 1);
        return;
    end
    
    if @CommunicationMethod not in ('Phone Call', 'Text')
    begin
        raiserror('CommunicationMethod must be Phone Call or Text', 16, 1);
        return;
    end
    
    -- Only 5 destinations in our booking data - anything else is a data entry error
    if @Destination not in ('Mars', 'Europa', 'Venus', 'Titan', 'Ganymede')
    begin
        raiserror('Invalid destination. Must be Mars, Europa, Venus, Titan, or Ganymede', 16, 1);
        return;
    end

    with 
    -- CTE 1: Pull together all historical data we need for scoring
    -- Using left joins to keep all agents even if they have zero bookings
    agent_performance as (
        select 
            sta.AgentID,
            sta.FirstName,
            sta.LastName,
            sta.AverageCustomerServiceRating,
            sta.YearsOfService,
            sta.DepartmentName,
            
            -- How good are they with organic vs bought leads specifically?
            -- NULL means they've never worked this lead type
            avg(case when ah.LeadSource = @LeadSource then sta.AverageCustomerServiceRating end) as lead_source_rating,
            
            -- Any track record with this destination? 
            avg(case when b.Destination = @Destination then sta.AverageCustomerServiceRating end) as destination_rating,
            
            -- Phone people vs text people - matters more than you'd think
            avg(case when ah.CommunicationMethod = @CommunicationMethod then sta.AverageCustomerServiceRating end) as communication_rating,
            
            -- Basic volume metrics for experience weighting
            count(b.BookingID) as total_bookings,
            count(case when b.BookingStatus = 'Confirmed' then 1 end) as confirmed_bookings,
            count(case when b.BookingStatus = 'Cancelled' then 1 end) as cancelled_bookings,
            
            -- Cancellation rate - need explicit float cast or SQL Server rounds to int
            case 
                when count(b.BookingID) > 0 
                then cast(count(case when b.BookingStatus = 'Cancelled' then 1 end) as float) / count(b.BookingID)
                else 0.0 
            end as cancellation_rate
            
        from space_travel_agents sta
        left join assignment_history ah on sta.AgentID = ah.AgentID
        left join bookings b on ah.AssignmentID = b.AssignmentID
        group by sta.AgentID, sta.FirstName, sta.LastName, sta.AverageCustomerServiceRating, 
                 sta.YearsOfService, sta.DepartmentName
    ),

    -- CTE 2: Normalize metrics and apply the weighted scoring formula
    scored_agents as (
        select *,
            -- Transform 2-18 year range to 1-5 scale so it plays nice with customer ratings
            1.0 + ((YearsOfService - 2.0) * 4.0 / 16.0) as normalized_service_years,
            
            -- Same deal for booking volume - saw max of 18 confirmed bookings in the data
            1.0 + ((confirmed_bookings - 0.0) * 4.0 / 18.0) as normalized_trip_volume,
            
            -- Here's the money shot - weighted scoring with 3.0 baseline for missing experience
            -- 3.0 is conservative since actual ratings range 3.3-5.0, keeps new agents in play
            (AverageCustomerServiceRating * 0.35) +                    -- Overall rating trumps everything
            (coalesce(lead_source_rating, 3.0) * 0.15) +               -- Organic vs bought matters
            (AverageCustomerServiceRating * 0.25) +                    -- Using overall for dept fit - departments aren't specialized enough
            ((1.0 + ((confirmed_bookings - 0.0) * 4.0 / 18.0)) * 0.10) + -- Volume proxy for experience
            ((1.0 + ((YearsOfService - 2.0) * 4.0 / 16.0)) * 0.05) +  -- Tenure baseline competency
            (coalesce(destination_rating, 3.0) * 0.10) as base_score   -- Destination specialization
            
        from agent_performance
    )

    -- Final output with cancellation penalty applied and full audit trail
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
        round(base_score * (1.0 - cancellation_rate), 3) as final_score,  -- Cancellation penalty applied here
        row_number() over (order by base_score * (1.0 - cancellation_rate) desc) as agent_rank,
        
        -- Include assignment context so we can trace decisions later
        -- Helpful when business asks "why did agent X get this customer?"
        @CustomerName as assigned_customer,
        @CommunicationMethod as customer_communication_method,
        @LeadSource as customer_lead_source,
        @Destination as customer_destination,
        @LaunchLocation as customer_launch_location,
        getdate() as assignment_timestamp
        
    from scored_agents
    order by final_score desc;
    
end
```

## Usage Example

```sql
exec sp_GetRankedAgents 
    @CustomerName = 'Sarah Johnson',
    @CommunicationMethod = 'Phone Call',
    @LeadSource = 'Organic', 
    @Destination = 'Europa',
    @LaunchLocation = 'Kennedy Space Center';
```

## Production Enhancements

**Error Handling**: Added validation for all the key parameters. Better to fail fast with a clear message than return garbage results.

**Audit Trail**: Every result set includes the customer parameters and timestamp. Critical for compliance and debugging assignment decisions later.

**SQL Server Compatibility**: 
- Used `nvarchar` for Unicode text handling
- `getdate()` instead of SQLite's datetime functions  
- Proper T-SQL stored procedure syntax with `set nocount on`
- Added explicit `float` casting for reliable division operations

**Destination Validation**: Since the data only has 5 valid destinations, added validation to catch typos or invalid requests early.

## Implementation Notes

The stored procedure returns all agents ranked from best to worst. In practice, you'd probably only assign the top-ranked agent, but returning the full list gives flexibility for business rules like "skip if top agent is unavailable" or "always show top 3 options to customer."

The algorithm handles missing experience gracefully - if an agent has never worked organic leads or Europa trips, they get a conservative 3.0 baseline score rather than being excluded entirely. This keeps new agents in the rotation while still favoring specialists.

Performance should be fine for the current data size (30 agents, ~450 assignments), but you'd want to add indexes on the join columns for larger datasets.