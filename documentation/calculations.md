# 📐 Power BI Calculations & DAX Measures

This document lists all calculated columns and DAX measures used in the Power BI model,
grouped by analytical purpose.

---

## 1️⃣ Core Consumption Metrics

### 1. Total Consumption (kWh)
```DAX
Total Consumption (kWh) =
SUM ( HalfHourly_Aggregated[TotalConsumption] )
```
### 2. % of Total Consumption
```DAX 
% of Total Consumption =
DIVIDE(
    SUM(HalfHourly_Aggregated[TotalConsumption]),
    CALCULATE(
        SUM(HalfHourly_Aggregated[TotalConsumption]),
        ALL(HalfHourly_Aggregated)
    )
)
```
### 3. Distinct MPANs
```DAX
Distinct MPANs =
DISTINCTCOUNT ( HalfHourly_Aggregated[MPAN] )
```
### 4. Avg Daily Consumption per MPAN
```DAX
Avg Daily Consumption per MPAN =
VAR TotalDays =
    DISTINCTCOUNT ( HalfHourly_Aggregated[Date] )
RETURN
    DIVIDE (
        [Total Consumption (kWh)],
        [Distinct MPANs] * TotalDays,
        0
    )
```

## 2️⃣ Date & Time Intelligence
### 5. DayOfWeek
```DAX
DayOfWeek =
FORMAT(HalfHourly_Aggregated[Date], "dddd")
```
### 6. DayOfWeekNumber
```DAX
DayOfWeekNumber =
WEEKDAY(HalfHourly_Aggregated[Date], 2)
```
### 7. Month
```DAX
Month =
FORMAT(HalfHourly_Aggregated[Date], "MMM")
```
### 8. TimeCategory
```DAX
TimeCategory =
SWITCH(
    TRUE(),
    HalfHourly_Aggregated[Hour] >= 0 && HalfHourly_Aggregated[Hour] < 6, "Night (0–6h)",
    HalfHourly_Aggregated[Hour] >= 6 && HalfHourly_Aggregated[Hour] < 12, "Morning (6–12h)",
    HalfHourly_Aggregated[Hour] >= 12 && HalfHourly_Aggregated[Hour] < 18, "Afternoon (12–18h)",
    HalfHourly_Aggregated[Hour] >= 18 && HalfHourly_Aggregated[Hour] < 24, "Evening (18–24h)",
    "Unknown"
)
```

## 3️⃣ Peak vs Off-Peak Analysis
### 9. Peak Consumption (07–22)
```DAX
Peak Consumption (07-22) =
CALCULATE (
    [Total Consumption (kWh)],
    FILTER (
        HalfHourly_Aggregated,
        HalfHourly_Aggregated[Hour] >= 7 &&
        HalfHourly_Aggregated[Hour] < 22
    )
)
```
### 10. Off-Peak Consumption
```DAX
OffPeak Consumption =
[Total Consumption (kWh)] - [Peak Consumption (07-22)]
```
### 11. PeakUsage%
```DAX
PeakUsage% =
VAR PeakSum =
    CALCULATE(
        SUM( HalfHourly_Aggregated[TotalConsumption] ),
        FILTER(
            HalfHourly_Aggregated,
            HalfHourly_Aggregated[Hour] >= 7 &&
            HalfHourly_Aggregated[Hour] <= 22
        )
    )
VAR TotalAll =
    SUM( HalfHourly_Aggregated[TotalConsumption] )
RETURN
    IF( TotalAll = 0, 0, DIVIDE( PeakSum, TotalAll ) )
```
### 12. OffPeak%
```DAX
OffPeak% =
VAR OffPeakSum =
    CALCULATE(
        SUM( HalfHourly_Aggregated[TotalConsumption] ),
        FILTER(
            HalfHourly_Aggregated,
            NOT(
                HalfHourly_Aggregated[Hour] >= 7 &&
                HalfHourly_Aggregated[Hour] <= 22
            )
        )
    )
VAR TotalAll =
    SUM( HalfHourly_Aggregated[TotalConsumption] )
RETURN
    IF( TotalAll = 0, 0, DIVIDE( OffPeakSum, TotalAll ) )
```
### 13. Percent Peak Usage
```DAX
Percent Peak Usage =
DIVIDE(
    [Peak Consumption (07-22)],
    [Total Consumption (kWh)],
    0
)
```
### 14. Peak_OffPeak
```DAX
Peak_OffPeak =
IF(
    HalfHourly_Aggregated[Hour] >= 7 &&
    HalfHourly_Aggregated[Hour] < 22,
    "Peak Hours",
    "Off-Peak Hours"
)
```
### 15. Peak Hour
```DAX
Peak Hour =
VAR HourlyTable =
    SUMMARIZE(
        HalfHourly_Aggregated,
        HalfHourly_Aggregated[Hour],
        "TotalConsumption",
        SUM(HalfHourly_Aggregated[TotalConsumption])
    )
VAR PeakRow =
    TOPN(1, HourlyTable, [TotalConsumption], DESC)
RETURN
    MAXX(PeakRow, HalfHourly_Aggregated[Hour])
```
### 16. Peak Hour Label
```DAX
Peak Hour Label =
VAR PeakHr = [Peak Hour]
RETURN
    FORMAT(TIME(PeakHr, 0, 0), "HH:mm")
```

## 4️⃣ Region Mapping & Data Enrichment
### 17. Final_Region
```DAX 
Final_Region =
IF(
    ISBLANK(ECOES_Clean[DDP_Region]),
    ECOES_Clean[Region],
    ECOES_Clean[DDP_Region]
)
```
### 18. Display_Region
```DAX
Display_Region =
IF(
    ISBLANK(ECOES_Clean[Final_Region]),
    "Unknown / Unmapped",
    ECOES_Clean[Final_Region]
)
```
### 19. Region_Mapped
```DAX
Region_Mapped =
VAR r = RELATED(ECOES_Clean[Display_Region])
RETURN
    IF(
        ISBLANK(r),
        "Unknown / Unmapped",
        r
    )
```

## 5️⃣ Dynamic Insight & Storytelling Measures
### 20. Top Class Insight
```DAXTop Class Insight =
VAR SummaryTable =
    SUMMARIZE(
        ECOES_Clean,
        ECOES_Clean[ProfileClass],
        "TotalConsumption",
        SUM(HalfHourly_Aggregated[TotalConsumption])
    )
VAR TopRow =
    TOPN(1, SummaryTable, [TotalConsumption], DESC)
VAR TopClass =
    MAXX(TopRow, ECOES_Clean[ProfileClass])
VAR TopClassConsumption =
    CALCULATE(
        SUM(HalfHourly_Aggregated[TotalConsumption]),
        ECOES_Clean[ProfileClass] = TopClass
    )
VAR TotalConsumptionAll =
    SUM(HalfHourly_Aggregated[TotalConsumption])
VAR Pct =
    DIVIDE(TopClassConsumption, TotalConsumptionAll)
RETURN
    "Class " & TopClass & " contributes " &
    FORMAT(Pct, "0.0%") & " of total consumption"
```
### 21. Top ProfileClass %
```DAX 
Top ProfileClass % =
VAR SummaryTable =
    SUMMARIZE(
        ECOES_Clean,
        ECOES_Clean[ProfileClass],
        "TotalConsumption",
        SUM(HalfHourly_Aggregated[TotalConsumption])
    )
VAR TopRow =
    TOPN(1, SummaryTable, [TotalConsumption], DESC)
VAR TopClass =
    MAXX(TopRow, ECOES_Clean[ProfileClass])
VAR TopClassConsumption =
    CALCULATE(
        SUM(HalfHourly_Aggregated[TotalConsumption]),
        ECOES_Clean[ProfileClass] = TopClass
    )
VAR TotalConsumptionAll =
    SUM(HalfHourly_Aggregated[TotalConsumption])
RETURN
    DIVIDE(TopClassConsumption, TotalConsumptionAll)
```
### 22. Top Region Insight
```DAX 
Top Region Insight =
VAR SummaryTable =
    SUMMARIZE(
        ECOES_Clean,
        ECOES_Clean[Region],
        "TotalConsumption",
        SUM(HalfHourly_Aggregated[TotalConsumption])
    )
VAR TopRow =
    TOPN(1, SummaryTable, [TotalConsumption], DESC)
VAR TopRegion =
    MAXX(TopRow, ECOES_Clean[Region])
VAR TopRegionConsumption =
    CALCULATE(
        SUM(HalfHourly_Aggregated[TotalConsumption]),
        ECOES_Clean[Region] = TopRegion
    )
VAR TotalConsumptionAll =
    SUM(HalfHourly_Aggregated[TotalConsumption])
VAR Pct =
    DIVIDE(TopRegionConsumption, TotalConsumptionAll)
RETURN
    TopRegion & " region contributes " &
    FORMAT(Pct, "0.0%") & " of total consumption"
```
### 23. Top Supplier Insight
```DAX 
Top Supplier Insight =
VAR SummaryTable =
    SUMMARIZE(
        ECOES_Clean,
        ECOES_Clean[SupplierID],
        "TotalConsumption",
        SUM(HalfHourly_Aggregated[TotalConsumption])
    )
VAR TopRow =
    TOPN(1, SummaryTable, [TotalConsumption], DESC)
VAR TopSupplier =
    MAXX(TopRow, ECOES_Clean[SupplierID])
VAR TopSupplierConsumption =
    CALCULATE(
        SUM(HalfHourly_Aggregated[TotalConsumption]),
        ECOES_Clean[SupplierID] = TopSupplier
    )
VAR TotalConsumptionAll =
    SUM(HalfHourly_Aggregated[TotalConsumption])
VAR Pct =
    DIVIDE(TopSupplierConsumption, TotalConsumptionAll)
RETURN
    TopSupplier & " supplies " &
    FORMAT(Pct, "0.0%") & " of total consumption"
```


