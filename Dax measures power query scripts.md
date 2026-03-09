# 🔧 DAX Measures & Power Query Scripts
## Supply Chain Transportation Analytics Dashboard

---

## Table of Contents
1. [DAX Measures](#dax-measures)
2. [Power Query Transformations](#power-query-transformations)
3. [How to Import](#how-to-import)
4. [Best Practices](#best-practices)

---

# DAX MEASURES

## Copy all measures below and paste into Power BI Data Model

### KPI Measures

#### 1. Total Revenue
```dax
Total_Revenue = 
    SUM(FactTrips[Revenue])
```

#### 2. Total Orders/Trips
```dax
Total_Orders = 
    COUNTROWS(FactTrips)
```

#### 3. Unique Customers
```dax
Unique_Customers = 
    DISTINCTCOUNT(FactTrips[Customer_ID])
```

#### 4. Active Trucks
```dax
Active_Trucks = 
    CALCULATE(
        DISTINCTCOUNT(FactTrips[Truck_ID]),
        DimTruck[Status] = "Active"
    )
```

---

### Performance Metrics

#### 5. On-Time Delivery %
```dax
OnTime_Delivery_Pct = 
VAR TotalDeliveries = COUNTROWS(FactDeliveryEvents)
VAR OnTimeCount = 
    CALCULATE(
        COUNTROWS(FactDeliveryEvents),
        FactDeliveryEvents[On_Time_Flag] = TRUE()
    )
RETURN
    DIVIDE(OnTimeCount, TotalDeliveries, BLANK())
```

**How to use**: Format as Percentage, use in KPI card
**Related**: Delay_Rate (complement metric)
**Dependencies**: FactDeliveryEvents[On_Time_Flag]

---

#### 6. Delay Rate %
```dax
Delay_Rate = 
VAR TotalTrips = COUNTROWS(FactTrips)
VAR LateTrips = 
    CALCULATE(
        COUNTROWS(FactTrips),
        FactTrips[Is_Late] = TRUE()
    )
RETURN
    DIVIDE(LateTrips, TotalTrips, 0)
```

**How to use**: Inverse of OnTime_Delivery_Pct
**Format**: Percentage, Conditional formatting (red if >45%)
**Calculation**: (Late trips) / (Total trips)

---

#### 7. OTIF (On-Time In-Full) %
```dax
OTIF_Pct = 
VAR OTIFTrips = 
    CALCULATE(
        COUNTROWS(FactTrips),
        FactTrips[Status] = "On Time",
        FactTrips[Full_Load] = TRUE()
    )
VAR TotalTrips = COUNTROWS(FactTrips)
RETURN
    DIVIDE(OTIFTrips, TotalTrips, 0)
```

**Business meaning**: Strictest metric (requires BOTH on-time AND full)
**Target**: ≥85%
**Where to use**: Executive dashboard KPI

---

#### 8. Average Trip Duration Hours
```dax
Avg_Trip_Duration_Hours = 
    AVERAGEX(FactTrips, FactTrips[Actual_Duration_Hours])
```

**Format**: Number with 1 decimal place (e.g., 8.2h)
**Baseline**: 8 hours
**Use in**: Driver & Fleet Performance dashboard

---

#### 9. Average Delay (Variance from Plan)
```dax
Avg_Delay_Minutes = 
VAR AllTrips = FactTrips
VAR VarianceMinutes = 
    AVERAGEX(
        AllTrips,
        (FactTrips[Actual_Duration_Hours] - FactTrips[Planned_Duration_Hours]) * 60
    )
RETURN
    VarianceMinutes
```

**Interpretation**: 
- Negative = early on average
- Positive = late on average
- Example: 24 minutes = avg 24 min late

---

### Cost Analytics

#### 10. Total Fuel Cost
```dax
Total_Fuel_Cost = 
    SUM(FactFuelPurchases[Total_Cost])
```

**Format**: Currency ($)
**Use in**: Fuel & Cost Analytics dashboard
**Filter by**: Truck, Driver, Date range

---

#### 11. Fuel Cost Per Mile
```dax
Fuel_Cost_Per_Mile = 
VAR TotalCost = SUM(FactFuelPurchases[Total_Cost])
VAR TotalMiles = SUM(FactTrips[Total_Miles])
RETURN
    DIVIDE(TotalCost, TotalMiles, 0)
```

**Formula**: Total fuel cost ÷ Total miles driven
**Format**: Currency (e.g., $0.87/mile)
**Benchmark**: $0.80-$0.95 (varies by market)
**Optimization**: Compare by driver to identify inefficiency

---

#### 12. Fuel Efficiency (MPG)
```dax
Avg_Fuel_Efficiency_MPG = 
VAR TotalMiles = SUM(FactFuelPurchases[Miles_Driven])
VAR TotalGallons = SUM(FactFuelPurchases[Quantity_Gallons])
RETURN
    DIVIDE(TotalMiles, TotalGallons, 0)
```

**Target**: 6.0-6.5 MPG (varies by truck model)
**Poor performance**: <5.5 MPG (flag for maintenance)
**Use in**: Fleet performance analysis

---

#### 13. Detention Hours (Total)
```dax
Total_Detention_Hours = 
    SUM(FactDeliveryEvents[Detention_Hours])
```

**Impact**: Each hour costs ~$6 in labor
**High detention**: Indicates facility/operational issues
**Where to use**: OTIF & Detention dashboard

---

#### 14. Detention Cost Impact
```dax
Detention_Cost = 
VAR TotalDetentionHours = SUM(FactDeliveryEvents[Detention_Hours])
VAR HourlyLaborCost = 6  -- $6 per hour labor cost
RETURN
    TotalDetentionHours * HourlyLaborCost
```

**Business impact**: Shows $ cost of delays
**Format**: Currency ($)
**Example**: 8,000 detention hours × $6 = $48,000 cost

---

#### 15. Cost Per Mile (Total Operating)
```dax
Total_Cost_Per_Mile = 
VAR TotalCost = 
    [Total_Fuel_Cost] + 
    [Detention_Cost]
VAR TotalMiles = SUM(FactTrips[Total_Miles])
RETURN
    DIVIDE(TotalCost, TotalMiles, 0)
```

**Comprehensive metric**: Includes fuel + detention
**Use for**: Overall profitability analysis

---

### Trend & Comparison Measures

#### 16. Revenue vs Prior Year (YoY)
```dax
Revenue_YoY_Growth = 
VAR CurrentYear = YEAR(TODAY())
VAR PreviousYear = CurrentYear - 1
VAR CurrentYearRevenue = 
    CALCULATE(
        [Total_Revenue],
        YEAR(FactTrips[Trip_Date]) = CurrentYear
    )
VAR PreviousYearRevenue = 
    CALCULATE(
        [Total_Revenue],
        YEAR(FactTrips[Trip_Date]) = PreviousYear
    )
VAR Growth = CurrentYearRevenue - PreviousYearRevenue
RETURN
    DIVIDE(Growth, PreviousYearRevenue, 0)
```

**Format**: Percentage (e.g., +12.3%)
**Use in**: Trend analysis, executive reporting
**Visualization**: KPI card with trend

---

#### 17. Revenue vs Budget
```dax
Revenue_vs_Budget = 
VAR ActualRevenue = [Total_Revenue]
VAR BudgetedRevenue = 7500000  -- Annual budget
RETURN
    DIVIDE(ActualRevenue, BudgetedRevenue, 0)
```

**Format**: Percentage (100% = on budget)
**Variance**: >100% = above budget (good)
**Use case**: Monthly performance tracking

---

#### 18. On-Time Delivery vs Target
```dax
OnTime_vs_Target = 
VAR ActualOnTime = [OnTime_Delivery_Pct]
VAR TargetOnTime = 0.90  -- 90% target
RETURN
    DIVIDE(ActualOnTime - TargetOnTime, TargetOnTime, 0)
```

**Interpretation**:
- Negative = Missing target
- Zero = On target
- Positive = Exceeding target

---

### Driver & Fleet Metrics

#### 19. Trips Per Driver
```dax
Trips_Per_Driver = 
    DIVIDE(
        COUNTROWS(FactTrips),
        DISTINCTCOUNT(FactTrips[Driver_ID]),
        0
    )
```

**Productivity metric**: How many trips per driver
**High performers**: 25+ trips/month
**Use in**: Driver performance leaderboard

---

#### 20. Average Truck Utilization %
```dax
Truck_Utilization_Pct = 
VAR FullLoadTrips = 
    CALCULATE(
        COUNTROWS(FactTrips),
        FactTrips[Full_Load] = TRUE()
    )
VAR TotalTrips = COUNTROWS(FactTrips)
RETURN
    DIVIDE(FullLoadTrips, TotalTrips, 0)
```

**Interpretation**: % of trips at full capacity
**Target**: >85% (higher = more efficient)
**Optimization**: Consolidate shipments to improve

---

#### 21. Driver Safety Score
```dax
Driver_Safety_Score = 
VAR AccidentCount = SUM(DimDriver[Accidents_Count])
VAR ViolationCount = SUM(DimDriver[Violations_Count])
VAR TotalDrivers = DISTINCTCOUNT(FactTrips[Driver_ID])
VAR SafetyMetric = 100 - (AccidentCount * 10) - (ViolationCount * 5)
RETURN
    SafetyMetric / TotalDrivers
```

**Scale**: 0-100 (higher is better)
**Flagged**: <70 requires retraining
**Use in**: Safety dashboards

---

### Customer & Route Analytics

#### 22. Top Customers (Revenue)
```dax
Top_Customers_Count = 
    SUMPRODUCT(
        (FactTrips[Customer_ID] <> BLANK()) * 1
    )
```

**Alternative approach**: Use visual ranking instead
**Better metric**: Create calculated dimension for top 10

---

#### 23. Route Performance (On-Time %)
```dax
Route_OnTime_Pct = 
VAR RouteFacts = 
    FILTER(
        FactTrips,
        FactTrips[Route_ID] = SELECTED(DimRoute[Route_ID])
    )
VAR OnTimeCount = 
    CALCULATE(
        COUNTROWS(RouteFacts),
        FactTrips[Is_Late] = FALSE()
    )
VAR TotalCount = COUNTROWS(RouteFacts)
RETURN
    DIVIDE(OnTimeCount, TotalCount, 0)
```

**Use in**: Route Performance dashboard
**Identifies**: Problem routes for optimization

---

#### 24. Worst Performing Routes (Bottom 10)
```dax
Route_Delay_Rate = 
VAR LateTrips = 
    CALCULATE(
        COUNTROWS(FactTrips),
        FactTrips[Is_Late] = TRUE()
    )
VAR TotalTrips = COUNTROWS(FactTrips)
RETURN
    DIVIDE(LateTrips, TotalTrips, 0)
```

**Sort**: Descending to show worst first
**Action**: Rerouting, additional time allocation

---

## Advanced Measures

#### 25. Forecast: Revenue (Next 12 Months)
```dax
Revenue_Forecast_12M = 
VAR HistoricalAvg = 
    AVERAGEX(
        ALL(DimDate),
        CALCULATE([Total_Revenue], DimDate)
    )
VAR ForecastMonths = 12
RETURN
    HistoricalAvg * ForecastMonths
```

**Note**: Simple average - upgrade to Prophet/ARIMA for production
**Use in**: Planning & forecasting dashboard
**Limitation**: Assumes stable environment

---

#### 26. Anomaly Detection (Cost Spike)
```dax
Cost_Anomaly_Flag = 
VAR CurrentCost = [Total_Fuel_Cost]
VAR AvgCost = 
    AVERAGEX(
        ALL(DimDate),
        CALCULATE([Total_Fuel_Cost], DimDate)
    )
VAR StdDeviation = 
    STDEV.P([Total_Fuel_Cost])
VAR ZScore = DIVIDE(CurrentCost - AvgCost, StdDeviation, 0)
RETURN
    IF(ABS(ZScore) > 2, "ANOMALY", "NORMAL")
```

**Flags**: Unusual costs (2+ std deviations)
**Action**: Investigate spikes
**Use in**: Cost monitoring dashboard

---

# POWER QUERY TRANSFORMATIONS

## Import & Paste into Power Query Editor

### 1. Fact Trips Data Transformation

```m
let
    Source = Excel.Workbook(File.Contents("C:\Data\trips.xlsx")),
    Sheet1 = Source{[Item="Sheet1"]}[Data],
    
    // Promote Headers
    Headers = Table.PromoteHeaders(Sheet1),
    
    // Change Data Types
    Types = Table.TransformColumnTypes(Headers, {
        {"Trip_ID", Int64.Type},
        {"Trip_Date", type date},
        {"Driver_ID", Int64.Type},
        {"Truck_ID", Int64.Type},
        {"Customer_ID", Int64.Type},
        {"Route_ID", Int64.Type},
        {"Revenue", Currency.Type},
        {"Total_Miles", Decimal.Type},
        {"Planned_Duration_Hours", Decimal.Type},
        {"Actual_Duration_Hours", Decimal.Type},
        {"Departure_Time", type datetime},
        {"Arrival_Time", type datetime},
        {"Is_Late", Logical.Type},
        {"Full_Load", Logical.Type},
        {"Trip_Type", Text.Type},
        {"Fuel_Consumed_Gallons", Decimal.Type},
        {"Status", Text.Type},
        {"Notes", Text.Type}
    }),
    
    // Create Calculated Columns
    AddTripDuration = Table.AddColumn(
        Types, 
        "Trip_Duration_Calculated", 
        each Duration.TotalHours([Arrival_Time] - [Departure_Time])
    ),
    
    AddCostPerMile = Table.AddColumn(
        AddTripDuration, 
        "Cost_Per_Mile", 
        each if [Total_Miles] > 0 then [Revenue] / [Total_Miles] else null
    ),
    
    AddMonthKey = Table.AddColumn(
        AddCostPerMile,
        "Month_Key",
        each Date.ToText([Trip_Date], "YYYY-MM")
    ),
    
    // Filter out nulls and invalid records
    FilterNull = Table.SelectRows(
        AddMonthKey, 
        each [Trip_ID] <> null and [Total_Miles] > 0
    ),
    
    // Sort by date
    SortByDate = Table.Sort(FilterNull, {{"Trip_Date", Order.Ascending}})
    
in
    SortByDate
```

**Apply**: Right-click data source → Edit Query → Paste this

---

### 2. Dimension: Date Table

```m
let
    // Generate dates from 2022-01-01 to 2025-12-31
    Source = List.Dates(
        #date(2022, 1, 1), 
        365*4,  -- 4 years worth of days
        #duration(1, 0, 0, 0)
    ),
    
    Table = Table.FromList(Source, Splitter.SplitByNothing()),
    
    Type = Table.TransformColumnTypes(Table, {{"Column1", type date}}),
    
    Renamed = Table.RenameColumns(Type, {{"Column1", "Date"}}),
    
    // Add year/month/day components
    Year = Table.AddColumn(Renamed, "Year", each Date.Year([Date]), Int64.Type),
    Quarter = Table.AddColumn(Year, "Quarter", each "Q" & Text.From(Date.QuarterOfYear([Date])), Text.Type),
    Month = Table.AddColumn(Quarter, "Month", each Date.Month([Date]), Int64.Type),
    MonthName = Table.AddColumn(Month, "Month_Name", each Text.Proper(Date.ToText([Date], "mmmm")), Text.Type),
    MonthShort = Table.AddColumn(MonthName, "Month_Short", each Text.Upper(Text.Start(Date.ToText([Date], "mmm"), 3)), Text.Type),
    
    Week = Table.AddColumn(MonthShort, "Week_Number", each Date.WeekOfYear([Date]), Int64.Type),
    DayOfWeek = Table.AddColumn(Week, "Day_of_Week", each Date.DayOfWeekName([Date]), Text.Type),
    DayShort = Table.AddColumn(DayOfWeek, "Day_Short", each Text.Upper(Text.Start([Day_of_Week], 3)), Text.Type),
    DayOfMonth = Table.AddColumn(DayShort, "Day_of_Month", each Date.Day([Date]), Int64.Type),
    
    IsWeekend = Table.AddColumn(DayOfMonth, "Is_Weekend", each [Day_of_Week] = "Saturday" or [Day_of_Week] = "Sunday", Logical.Type),
    
    // Add Holiday flag (US holidays)
    IsHoliday = Table.AddColumn(IsWeekend, "Is_Holiday", each 
        (Date.Month([Date]) = 1 and Date.Day([Date]) = 1) or  -- New Year
        (Date.Month([Date]) = 12 and Date.Day([Date]) = 25) or  -- Christmas
        (Date.Month([Date]) = 7 and Date.Day([Date]) = 4),  -- Independence Day
        Logical.Type
    ),
    
    Season = Table.AddColumn(IsHoliday, "Season", each
        if Date.Month([Date]) >= 3 and Date.Month([Date]) <= 5 then "Spring"
        else if Date.Month([Date]) >= 6 and Date.Month([Date]) <= 8 then "Summer"
        else if Date.Month([Date]) >= 9 and Date.Month([Date]) <= 11 then "Fall"
        else "Winter",
        Text.Type
    )

in
    Season
```

**Create new query**: Home → New Source → Blank Query → Paste code

---

### 3. Dimension: Customer

```m
let
    Source = Excel.Workbook(File.Contents("C:\Data\customers.xlsx")),
    Sheet = Source{[Item="Sheet1"]}[Data],
    
    Headers = Table.PromoteHeaders(Sheet),
    
    Types = Table.TransformColumnTypes(Headers, {
        {"Customer_ID", Int64.Type},
        {"Customer_Name", Text.Type},
        {"Customer_Segment", Text.Type},
        {"Industry", Text.Type},
        {"Primary_State", Text.Type},
        {"Primary_City", Text.Type},
        {"Total_Employees", Int64.Type},
        {"Annual_Revenue_USD", Currency.Type},
        {"Account_Manager", Text.Type},
        {"Join_Date", type date},
        {"Contract_Expiry_Date", type date},
        {"Status", Text.Type},
        {"Phone", Text.Type},
        {"Email", Text.Type}
    }),
    
    RemoveDuplicates = Table.Distinct(Types, {"Customer_ID"}),
    
    FilterActive = Table.SelectRows(
        RemoveDuplicates,
        each [Status] = "Active" or [Status] = "Inactive"
    )
    
in
    FilterActive
```

---

### 4. Merge Fact + Dimension (Enrichment)

```m
let
    FactTrips = #"FactTrips",
    DimDriver = #"DimDriver",
    
    // Left Outer Join on Driver_ID
    Merged = Table.NestedJoin(
        FactTrips,
        {"Driver_ID"},
        DimDriver,
        {"Driver_ID"},
        "DriverDetails",
        JoinKind.LeftOuter
    ),
    
    // Expand selected columns from driver
    Expanded = Table.ExpandTableColumn(
        Merged,
        "DriverDetails",
        {"Driver_Name", "License_State", "Safety_Certification"},
        {"Driver_Name", "License_State", "Safety_Certification"}
    )
    
in
    Expanded
```

---

# HOW TO IMPORT

## Step-by-Step in Power BI

### Import DAX Measures

1. **Open Power BI Desktop**
2. **Go to**: Home → View → Calculations → New Measure
3. **Copy-paste measure code** into formula bar
4. **Press Enter**
5. **Repeat for all 26 measures**

**Alternative (Faster)**:
- Advanced Editor → Paste all measures at once
- Format: Separate measures with blank line

### Import Power Query Scripts

1. **Home → Get Data → More** (or Transform Data)
2. **New Source → Blank Query**
3. **Home → Advanced Editor**
4. **Paste M code**
5. **Load table**

**Validate**: 
- Check row counts match expected
- Verify column names
- Test filters/calculations

---

# BEST PRACTICES

## DAX Tips

✅ **Use VAR for readability**
```dax
GOOD: VAR Count = COUNTROWS(...) RETURN Count
BAD: COUNTROWS(FILTER(...))
```

✅ **Always use DIVIDE for safety**
```dax
GOOD: DIVIDE(Numerator, Denominator, 0)
BAD: Numerator / Denominator  -- #DIV/0! errors
```

✅ **Filter context matters**
```dax
-- This measure responds to page filters
[Total_Revenue] = SUM(FactTrips[Revenue])

-- This ignores filters
[All_Time_Revenue] = CALCULATE(SUM(FactTrips[Revenue]), ALL(FactTrips))
```

---

## Performance Tips

⚡ **Minimize calculated columns** - Use measures instead
⚡ **Avoid FILTER on large tables** - Use CALCULATE with context
⚡ **Index foreign keys** - Speeds up joins
⚡ **Aggregate at source** - Don't bring raw data to PBI

---

**Copy-paste ready! Happy modeling! 📊**
