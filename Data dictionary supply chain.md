# 📚 Data Dictionary
## Supply Chain & Transportation Analytics Dashboard

---

## Table of Contents
1. [Fact Tables](#fact-tables)
2. [Dimension Tables](#dimension-tables)
3. [Calculated Columns](#calculated-columns)
4. [Measures & KPIs](#measures--kpis)
5. [Data Quality Rules](#data-quality-rules)

---

# FACT TABLES

## FactTrips

**Description**: Core transactional table containing one record per trip/delivery event

**Grain**: One row per Trip ID

**Row Count**: ~45,000 records

**Refresh Frequency**: Daily at 6:00 AM

| Field Name | Data Type | Source | Definition | Sample Value | Business Rule |
|-----------|-----------|--------|-----------|--------------|----------------|
| **Trip_ID** | Integer (PK) | trips.xlsx | Unique identifier for each trip | 1001 | Non-null, unique |
| **Trip_Date** | Date | trips.xlsx | Date when trip commenced | 2024-03-15 | Range: 2022-01-01 to current |
| **Driver_ID** | Integer (FK) | trips.xlsx | Reference to driver | 42 | Must exist in DimDriver |
| **Truck_ID** | Integer (FK) | trips.xlsx | Reference to truck/vehicle | 18 | Must exist in DimTruck |
| **Customer_ID** | Integer (FK) | trips.xlsx | Reference to customer | 205 | Must exist in DimCustomer |
| **Route_ID** | Integer (FK) | trips.xlsx | Reference to route | 15 | Must exist in DimRoute |
| **Revenue** | Currency | trips.xlsx | Shipping revenue for trip | $1,250.00 | ≥ $0, NULL allowed |
| **Total_Miles** | Decimal | trips.xlsx | Distance traveled (miles) | 245.5 | > 0, NULL not allowed |
| **Planned_Duration_Hours** | Decimal | trips.xlsx | Planned trip duration | 8.0 | > 0, typically 4-12 |
| **Actual_Duration_Hours** | Decimal | trips.xlsx | Actual time taken | 8.4 | ≥ Planned_Duration |
| **Departure_Time** | DateTime | trips.xlsx | When trip started | 2024-03-15 06:30 | Same date as Trip_Date |
| **Arrival_Time** | DateTime | trips.xlsx | When trip completed | 2024-03-15 14:45 | ≥ Departure_Time |
| **Is_Late** | Boolean | trips.xlsx | Flag if trip was delayed | TRUE / FALSE | Based on promised vs actual |
| **Full_Load** | Boolean | trips.xlsx | Trip at capacity? | TRUE / FALSE | Y/N indicator |
| **Trip_Type** | Text | trips.xlsx | Classification of trip | "Express", "Scheduled", "LTL" | Predefined list |
| **Fuel_Consumed_Gallons** | Decimal | trips.xlsx | Fuel used in gallons | 42.3 | > 0 or NULL |
| **Status** | Text | trips.xlsx | Final trip status | "On Time", "Late", "Cancelled" | Predefined: 3 values |
| **Notes** | Text | trips.xlsx | Free-form trip notes | "Mechanical delay in Chicago" | Optional, NULL allowed |

---

## FactDeliveryEvents

**Description**: Detailed delivery event tracking (multiple events per trip)

**Grain**: One row per delivery event (pickup, waypoint, delivery)

**Row Count**: ~78,000 records

**Refresh Frequency**: Real-time / Hourly

| Field Name | Data Type | Source | Definition | Sample Value | Business Rule |
|-----------|-----------|--------|-----------|--------------|----------------|
| **Event_ID** | Integer (PK) | delivery_events.xlsx | Unique event identifier | 50001 | Non-null, unique |
| **Trip_ID** | Integer (FK) | delivery_events.xlsx | Reference to trip | 1001 | Must exist in FactTrips |
| **Event_Type** | Text | delivery_events.xlsx | Type of event | "Pickup", "Delivery", "Waypoint" | Predefined list |
| **Event_DateTime** | DateTime | delivery_events.xlsx | When event occurred | 2024-03-15 14:45:30 | ISO 8601 format |
| **Facility_ID** | Integer (FK) | delivery_events.xlsx | Warehouse/facility involved | 5 | Must exist in DimFacility |
| **Location_Latitude** | Decimal | delivery_events.xlsx | GPS latitude coordinate | 40.7128 | Range: -90 to +90 |
| **Location_Longitude** | Decimal | delivery_events.xlsx | GPS longitude coordinate | -74.0060 | Range: -180 to +180 |
| **Detention_Hours** | Decimal | delivery_events.xlsx | Hours of detention/delay | 2.5 | ≥ 0, NULL allowed |
| **Detention_Reason** | Text | delivery_events.xlsx | Why was truck detained? | "Dock congestion", "Weather", "Mechanical" | Predefined codes |
| **Planned_Time** | DateTime | delivery_events.xlsx | Scheduled time for event | 2024-03-15 14:00 | Before Event_DateTime if late |
| **Planned_vs_Actual_Mins** | Integer | delivery_events.xlsx | Variance in minutes | 45 | Can be positive (late) or negative (early) |
| **On_Time_Flag** | Boolean | delivery_events.xlsx | Event on time? | TRUE / FALSE | Planned_Time ≥ Event_DateTime |
| **Event_Status** | Text | delivery_events.xlsx | Completion status | "Completed", "In Progress", "Failed" | Predefined: 3 values |
| **Signature_Obtained** | Boolean | delivery_events.xlsx | Proof of delivery signed? | TRUE / FALSE | Required for final delivery |
| **Delivery_Exception** | Boolean | delivery_events.xlsx | Any delivery issues? | FALSE | Damage, wrong address, etc. |

---

## FactFuelPurchases

**Description**: Fuel transaction tracking for fleet vehicles

**Grain**: One row per fuel purchase transaction

**Row Count**: ~12,000 records

**Refresh Frequency**: Daily

| Field Name | Data Type | Source | Definition | Sample Value | Business Rule |
|-----------|-----------|--------|-----------|--------------|----------------|
| **Fuel_Purchase_ID** | Integer (PK) | fuel_purchases.xlsx | Unique purchase identifier | 10001 | Non-null, unique |
| **Purchase_Date** | Date | fuel_purchases.xlsx | When fuel purchased | 2024-03-15 | Must be ≤ current date |
| **Truck_ID** | Integer (FK) | fuel_purchases.xlsx | Vehicle being fueled | 18 | Must exist in DimTruck |
| **Fuel_Type** | Text | fuel_purchases.xlsx | Type of fuel | "Diesel", "Gasoline", "Electric" | Predefined list |
| **Quantity_Gallons** | Decimal | fuel_purchases.xlsx | Amount purchased | 42.5 | > 0 |
| **Unit_Price** | Currency | fuel_purchases.xlsx | Price per gallon | $3.45 | > 0 |
| **Total_Cost** | Currency | fuel_purchases.xlsx | Total transaction cost | $146.63 | = Quantity × Unit_Price |
| **Odometer_Start** | Decimal | fuel_purchases.xlsx | Odometer reading before trip | 145,230 | Increasing sequence |
| **Odometer_End** | Decimal | fuel_purchases.xlsx | Odometer reading after trip | 145,475 | > Odometer_Start |
| **Miles_Driven** | Decimal | fuel_purchases.xlsx | Distance on fuel | 245 | = Odometer_End - Odometer_Start |
| **Fuel_Efficiency_MPG** | Decimal | fuel_purchases.xlsx | Miles per gallon achieved | 6.2 | Miles_Driven / Quantity_Gallons |
| **Vendor** | Text | fuel_purchases.xlsx | Fuel station/supplier | "Shell", "Chevron", "BP" | Predefined vendor list |
| **Location_State** | Text | fuel_purchases.xlsx | State where fuel purchased | "CA", "TX", "NY" | US state abbreviation |
| **Fuel_Price_Index** | Decimal | fuel_purchases.xlsx | National fuel price index | 98.45 | Reference metric |

---

# DIMENSION TABLES

## DimDate

**Description**: Standard date dimension for time-based analysis

**Grain**: One row per calendar day

**Row Count**: 1,461 (4 years × 365.25)

**Coverage**: 2022-01-01 through 2025-12-31

| Field Name | Data Type | Definition | Sample Value | Notes |
|-----------|-----------|-----------|--------------|--------|
| **Date** | Date (PK) | Calendar date | 2024-03-15 | Format: YYYY-MM-DD |
| **Year** | Integer | Calendar year | 2024 | Range: 2022-2025 |
| **Quarter** | Text | Fiscal quarter | "Q1" | Format: Q1-Q4 |
| **Month** | Integer | Month number | 3 | Range: 1-12 |
| **Month_Name** | Text | Month full name | "March" | Capitalized |
| **Month_Short** | Text | Month abbreviation | "Mar" | 3 characters |
| **Week_Number** | Integer | ISO week number | 11 | Range: 1-52 |
| **Day_of_Week** | Text | Day name | "Friday" | Full name |
| **Day_Short** | Text | Day abbreviation | "Fri" | 3 characters |
| **Day_of_Month** | Integer | Day within month | 15 | Range: 1-31 |
| **Is_Weekend** | Boolean | Weekend flag | FALSE | Saturday/Sunday = TRUE |
| **Is_Holiday** | Boolean | US holiday flag | FALSE | Major holidays marked TRUE |
| **Holiday_Name** | Text | Holiday description | NULL / "Good Friday" | NULL if not holiday |
| **Season** | Text | Meteorological season | "Spring" | Spring, Summer, Fall, Winter |

---

## DimCustomer

**Description**: Customer master dimension

**Grain**: One row per unique customer

**Row Count**: 420 customers

| Field Name | Data Type | Definition | Sample Value | Business Rule |
|-----------|-----------|-----------|--------------|----------------|
| **Customer_ID** | Integer (PK) | Unique customer identifier | 205 | Non-null, unique |
| **Customer_Name** | Text | Legal company name | "First Group USA" | 50 chars max |
| **Customer_Segment** | Text | Business classification | "Enterprise", "Mid-Market", "SMB" | Predefined: 3 values |
| **Industry** | Text | Industry vertical | "Logistics", "Retail", "Manufacturing" | Predefined list |
| **Primary_State** | Text | Main operating state | "CA" | US state abbreviation |
| **Primary_City** | Text | Primary city | "Los Angeles" | 30 chars max |
| **Total_Employees** | Integer | Company employee count | 5000 | NULL allowed for smaller firms |
| **Annual_Revenue_USD** | Currency | Customer annual revenue | $500,000,000 | Approximate, NULL allowed |
| **Account_Manager** | Text | Assigned account manager | "John Smith" | 50 chars max |
| **Join_Date** | Date | When customer onboarded | 2020-01-15 | Must be ≤ current date |
| **Contract_Expiry_Date** | Date | Contract end date | 2025-12-31 | NULL if ongoing |
| **Status** | Text | Customer status | "Active", "Inactive", "Churned" | Predefined: 3 values |
| **Phone** | Text | Main contact phone | "(800) 555-0123" | Formatted |
| **Email** | Text | Primary contact email | "contact@firstgroup.com" | Valid email format |

---

## DimDriver

**Description**: Driver master dimension

**Grain**: One row per unique driver

**Row Count**: 285 drivers

| Field Name | Data Type | Definition | Sample Value | Business Rule |
|-----------|-----------|-----------|--------------|----------------|
| **Driver_ID** | Integer (PK) | Unique driver identifier | 42 | Non-null, unique |
| **Driver_Name** | Text | Full legal name | "Ahmed Salah" | 50 chars max |
| **License_Number** | Text | Commercial driver license | "DL123456789" | Unique, state-specific |
| **License_State** | Text | License issuing state | "CA" | US state abbreviation |
| **License_Class** | Text | CDL class | "A", "B", "C" | Determines vehicle class |
| **Date_of_Birth** | Date | Driver birth date | 1985-03-22 | Age typically 21-65 |
| **Hire_Date** | Date | When hired by company | 2018-06-15 | Must be ≤ current date |
| **Termination_Date** | Date | When employment ended | NULL / 2024-02-28 | NULL if active |
| **Years_Experience** | Integer | Years in trucking | 8 | ≥ 0, calculated |
| **Home_State** | Text | Driver's home state | "TX" | US state abbreviation |
| **Status** | Text | Employment status | "Active", "Inactive", "On Leave" | Predefined: 3 values |
| **Violations_Count** | Integer | Traffic violations YTD | 1 | ≥ 0, impacts ratings |
| **Accidents_Count** | Integer | Chargeable accidents YTD | 0 | ≥ 0, safety metric |
| **Average_Trip_Rating** | Decimal | Customer satisfaction | 4.7 | Range: 1.0-5.0 stars |
| **Safety_Certification** | Text | HAZMAT/Tanker cert | "HAZMAT" / NULL | Determines eligible loads |
| **Contact_Phone** | Text | Driver phone number | "(512) 555-0123" | Formatted |

---

## DimTruck

**Description**: Fleet vehicle master dimension

**Grain**: One row per unique vehicle

**Row Count**: 171 trucks

| Field Name | Data Type | Definition | Sample Value | Business Rule |
|-----------|-----------|-----------|--------------|----------------|
| **Truck_ID** | Integer (PK) | Unique vehicle identifier | 18 | Non-null, unique |
| **Unit_Number** | Text | Fleet display number | "TC-18" | 10 chars max |
| **Make** | Text | Vehicle manufacturer | "Volvo", "Peterbilt", "Freightliner" | Standard makes |
| **Model** | Text | Vehicle model name | "VNL 860" | 30 chars max |
| **Year** | Integer | Model year | 2021 | Range: 2015-2024 |
| **VIN** | Text | Vehicle Identification Number | "1HGCS21362H123456" | 17 chars, unique |
| **License_Plate** | Text | Registration plate | "CA-12ABC" | State-specific format |
| **Capacity_Lbs** | Integer | Max payload weight | 26000 | Typically 20K-80K lbs |
| **Capacity_Cubic_Ft** | Integer | Box volume | 2200 | Typically 1600-2600 cu ft |
| **Transmission** | Text | Type of transmission | "Manual", "Automatic" | Predefined: 2 values |
| **Purchase_Date** | Date | When acquired | 2021-03-15 | Must be ≤ current date |
| **Acquisition_Cost** | Currency | Purchase price | $145,000 | For depreciation tracking |
| **Current_Odometer** | Decimal | Total miles on vehicle | 245,000 | Updated per fuel purchase |
| **Service_Schedule** | Integer | Maintenance interval miles | 15000 | Oil change frequency |
| **Last_Service_Date** | Date | Most recent maintenance | 2024-02-28 | Must be ≤ current date |
| **Next_Service_Due** | Date | Scheduled next maintenance | 2024-05-15 | Calculated based on odometer |
| **Fuel_Type** | Text | Primary fuel | "Diesel" | Diesel, Gasoline, Electric |
| **Average_MPG** | Decimal | Fuel efficiency baseline | 6.2 | Miles per gallon target |
| **Status** | Text | Vehicle availability | "Active", "In Maintenance", "Retired" | Predefined: 3 values |
| **Cost_Per_Mile** | Currency | Operating cost per mile | $2.14 | Used for economic analysis |

---

## DimRoute

**Description**: Standard routes/lanes in network

**Grain**: One row per unique route

**Row Count**: 89 routes

| Field Name | Data Type | Definition | Sample Value | Business Rule |
|-----------|-----------|-----------|--------------|----------------|
| **Route_ID** | Integer (PK) | Unique route identifier | 15 | Non-null, unique |
| **Route_Name** | Text | Route description | "Seattle to Charlotte" | 50 chars max |
| **Origin_State** | Text | Departure state | "WA" | US state abbreviation |
| **Origin_City** | Text | Departure city | "Seattle" | 30 chars max |
| **Destination_State** | Text | Arrival state | "NC" | US state abbreviation |
| **Destination_City** | Text | Arrival city | "Charlotte" | 30 chars max |
| **Distance_Miles** | Decimal | Standard route mileage | 2350 | Round trip or one-way? |
| **Estimated_Hours** | Decimal | Planned transit time | 34.5 | 2350 miles ÷ 68 mph avg |
| **Frequency** | Text | How often run | "Daily", "Weekly", "As-Needed" | Predefined: 3 values |
| **Route_Type** | Text | Classification | "Regional", "Interstate", "Scheduled" | Predefined list |
| **Truck_Class_Required** | Text | Minimum truck type | "Class A", "Class B" | Size requirement |
| **Hazmat_Allowed** | Boolean | HAZMAT cargo permitted? | FALSE | TRUE if authorized |
| **Toll_Roads** | Boolean | Route uses toll highways? | TRUE | Affects cost |
| **Seasonal** | Boolean | Seasonal route? | FALSE | Active only certain times |
| **Peak_Season_Start** | Integer | Peak season start month | 5 | Month 1-12, NULL if non-seasonal |
| **Peak_Season_End** | Integer | Peak season end month | 8 | Month 1-12, NULL if non-seasonal |
| **Average_Delay_Pct** | Decimal | Historical delay rate | 0.4633 | 46.33% delays on this route |
| **Status** | Text | Route status | "Active", "Suspended", "Retired" | Predefined: 3 values |
| **Notes** | Text | Route specific notes | "Heavy truck restrictions on I-40" | Optional |

---

## DimFacility

**Description**: Warehouses, distribution centers, terminals

**Grain**: One row per facility

**Row Count**: 23 facilities

| Field Name | Data Type | Definition | Sample Value | Business Rule |
|-----------|-----------|-----------|--------------|----------------|
| **Facility_ID** | Integer (PK) | Unique facility identifier | 5 | Non-null, unique |
| **Facility_Name** | Text | Official facility name | "Chicago Distribution Center" | 50 chars max |
| **Facility_Type** | Text | Type of facility | "DC", "Hub", "Terminal", "Cross-dock" | Predefined list |
| **Address** | Text | Street address | "123 Industrial Blvd" | 50 chars max |
| **City** | Text | City location | "Chicago" | 30 chars max |
| **State** | Text | State location | "IL" | US state abbreviation |
| **Zip_Code** | Text | Postal code | "60601" | 5-digit US ZIP |
| **Latitude** | Decimal | GPS latitude | 41.8781 | Range: -90 to +90 |
| **Longitude** | Decimal | GPS longitude | -87.6298 | Range: -180 to +180 |
| **Operating_Hours** | Text | Hours of operation | "24/7" | Format varies |
| **Dock_Doors** | Integer | Number of loading doors | 12 | > 0, capacity metric |
| **Max_Capacity_Lbs** | Integer | Maximum weight capacity | 500000 | Total facility capacity |
| **Square_Footage** | Integer | Warehouse size | 75000 | Building area |
| **Manager_Name** | Text | Facility manager | "Jane Smith" | 50 chars max |
| **Phone** | Text | Main facility phone | "(312) 555-0123" | Formatted |
| **Average_Detention_Hours** | Decimal | Historical detention time | 6.4 | Performance metric |
| **Status** | Text | Operational status | "Active", "Closed", "Under Renovation" | Predefined: 3 values |

---

# CALCULATED COLUMNS

## In Power Query (M Language)

### FactTrips Additions

```
Trip_Duration_Calculated = 
Arrival_Time - Departure_Time
(Returns time duration)

Cost_Per_Mile = 
Total_Cost / Total_Miles
(Dollar cost per mile driven)

Is_Late_Calculated = 
Actual_Duration_Hours > Planned_Duration_Hours
(Boolean: TRUE if late, FALSE if on-time)

Trip_Month = 
Date.ToText([Trip_Date], "YYYY-MM")
(Extract year-month for grouping)
```

---

# MEASURES & KPIs

## Core KPIs

### 1. Total Revenue
```dax
Total_Revenue = SUM(FactTrips[Revenue])
```
- **Purpose**: Aggregate revenue across trips
- **Format**: Currency ($)
- **Filter Context**: Responds to date, customer, route filters

---

### 2. On-Time Delivery %
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
- **Purpose**: % of deliveries meeting scheduled time
- **Format**: Percentage (0-100%)
- **Target**: 90%

---

### 3. Delay Rate
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
- **Purpose**: % of trips delayed
- **Format**: Percentage
- **Industry Benchmark**: <30%

---

### 4. OTIF (On-Time In-Full) %
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
- **Purpose**: Metric combining on-time AND full delivery
- **Format**: Percentage
- **Target**: 85%+

---

### 5. Fuel Cost Per Mile
```dax
Fuel_Cost_Per_Mile = 
VAR TotalCost = SUM(FactFuelPurchases[Total_Cost])
VAR TotalMiles = SUM(FactTrips[Total_Miles])
RETURN
    DIVIDE(TotalCost, TotalMiles, 0)
```
- **Purpose**: Efficiency metric for fuel cost
- **Format**: Currency ($/mile)
- **Benchmark**: $0.80-$0.95

---

### 6. Average Trip Duration
```dax
Avg_Trip_Duration_Hours = 
AVERAGEX(FactTrips, FactTrips[Actual_Duration_Hours])
```
- **Purpose**: Mean hours per trip
- **Format**: Hours (decimal)
- **Baseline**: 8 hours

---

### 7. Detention Cost Impact
```dax
Detention_Cost = 
VAR TotalDetentionHours = SUM(FactDeliveryEvents[Detention_Hours])
VAR HourlyLaborCost = 6
RETURN
    TotalDetentionHours * HourlyLaborCost
```
- **Purpose**: Financial impact of delays
- **Format**: Currency ($)
- **Goal**: Minimize

---

### 8. Driver Utilization
```dax
Driver_Utilization = 
VAR TripsPerDriver = 
    DIVIDE(
        COUNTROWS(FactTrips),
        DISTINCTCOUNT(FactTrips[Driver_ID]),
        0
    )
RETURN
    TripsPerDriver
```
- **Purpose**: Trips per driver (productivity)
- **Format**: Number (decimal)
- **Target**: 25+ trips/month

---

# DATA QUALITY RULES

## Validation Checks

| Rule | Table | Column | Condition | Action |
|------|-------|--------|-----------|--------|
| Non-null PK | FactTrips | Trip_ID | Must not be null | Flag record |
| Unique PK | FactTrips | Trip_ID | No duplicates | Reject load |
| Date Range | FactTrips | Trip_Date | Between 2022-01-01 and today | Warning if outside |
| Referential Integrity | FactTrips | Driver_ID | Must exist in DimDriver | Flag orphan |
| Logical Check | FactTrips | Actual_Duration > Planned_Duration | Must be true for late flag | Auto-correct |
| Range Check | FactFuelPurchases | Fuel_Efficiency_MPG | Between 4.0 and 9.0 | Flag outlier |
| Format Check | DimCustomer | Phone | Matches (XXX) XXX-XXXX | Warning |
| Completeness | DimDriver | License_State | Must have value | Mandatory |

---

## Data Governance

### Ownership
- **Fact Tables**: Operations team (daily refresh)
- **Dimension Tables**: Master data team (weekly review)
- **Calculated Measures**: Analytics team (DAX ownership)

### Retention Policy
- **Transactional Data**: 3 years rolling
- **Summary Data**: 5 years archived
- **Personal Data**: GDPR compliance (redact if needed)

### SLAs
- **Refresh Latency**: ≤4 hours for fact data
- **Data Accuracy**: ≥99% validated records
- **Availability**: 99.5% dashboard uptime

---

## Contact for Data Questions

**Data Steward**: Analytics Team
**Email**: analytics@company.com
**Escalation**: Data Governance Committee

---

**Last Updated**: March 2026
**Version**: 2.0
