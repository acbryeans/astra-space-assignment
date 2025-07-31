# Phase 1: Exploration and Planning

## Problem Analysis

The core requirement is building a real-time SQL algorithm that takes customer parameters and returns agents ranked from best to worst match. The inputs are customer name, communication method, lead source, destination, and launch location. While the concept appears straightforward, the implementation requires careful consideration of multiple factors.

## Environment Setup

I selected SQLiteOnline.com for development - it provides quick setup without local installation requirements. All three tables loaded successfully:
- assignment_history (the historical assignments)  
- bookings (outcomes and revenue)
- space_travel_agents (agent profiles and ratings)

## Data Exploration

Initial analysis focused on understanding the data structure and distribution patterns:

```sql
-- Checking available launch locations
select distinct LaunchLocation from bookings;
-- Returns 6 different launch points

-- Analyzing destination variety
select distinct Destination from bookings;  
-- Limited to 5 destinations

-- Verifying agent count
select count(distinct AgentID) from space_travel_agents;

-- Examining booking distribution to identify potential workload imbalances
select 
    sta.AgentID as agent_id,
    count(b.BookingID) as bookings
from bookings as b
left join assignment_history as ah
    on b.AssignmentID = ah.AssignmentID
left join space_travel_agents as sta
    on ah.AgentID = sta.AgentID
group by sta.AgentID;
-- Distribution is relatively balanced: 10-18 bookings per agent
```

The data shows relatively balanced performance across agents with no clear outliers in booking volume. This eliminates the simple approach of defaulting to highest-volume agents and requires a more sophisticated scoring mechanism.

## Algorithm Strategy 

Selected a weighted scoring approach over conditional logic structures. Previous experience with nested if-statement systems has shown they become difficult to maintain and debug. The scoring system provides transparency in ranking decisions and allows for parameter tuning.

## Scoring Formula

Based on data pattern analysis, the following weighted formula was developed:

```
Total Score = (Customer Service Rating � 35%) + 
              (Lead Source Experience � 15%) + 
              (Category Experience � 25%) + 
              (Trip Volume Score � 10%) + 
              (Years of Service � 5%) + 
              (Destination Experience � 10%)

Final Score = Total Score � (1 - Cancellation Rate)
```

## Why These Weights?

**Customer Service Rating (35%)** - Primary performance indicator. Customer satisfaction directly correlates with business success and retention.

**Category Experience (25%)** - Department specialization matters. Each division (Luxury Voyages, Interplanetary Sales, Premium Bookings) requires distinct expertise and approach.

**Lead Source Experience (15%)** - Organic and purchased leads require different handling strategies. Organic leads typically have higher intent, while bought leads need more nurturing.

**Destination Experience (10%)** - Limited weight due to only 5 available destinations. Most agents develop familiarity across all locations relatively quickly.

**Trip Volume (10%)** - Experience proxy normalized to account for factors outside agent control (market conditions, assignment luck).

**Years of Service (5%)** - Foundation competency measure. Establishes baseline proficiency but diminishing returns after initial learning curve.

The cancellation rate applies as a final multiplier rather than additive penalty - an agent with high cancellation rates represents increased risk regardless of other strengths.

## Implementation Decisions

All metrics normalize to a 5-point scale for consistency with existing customer service ratings. This standardization simplifies calculations and improves result interpretation.

The cancellation penalty uses multiplication rather than subtraction - a 20% cancellation rate reduces the effective score to 80% of its base value. This proportional approach better reflects business risk than linear deduction.

Destination and launch location weights remain low due to limited variety in the dataset. With only 5 destinations, agents typically gain exposure across all locations within a reasonable timeframe.

## Production Considerations

A production implementation would require stakeholder collaboration to validate these weights against business priorities. The current weightings derive from data analysis and industry best practices, but organizational objectives may warrant different emphasis. A/B testing would provide empirical validation of the scoring model's effectiveness. The weighted approach facilitates easy adjustment without structural changes to the algorithm.