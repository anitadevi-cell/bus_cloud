# API Comparison and Benchmark Analysis Module

## Objective

Implement LOCAL API response comparison and benchmark analysis inside the existing Bus application root project.

Do NOT create:

- new application
- new solution
- new API project

Integrate everything inside the existing project structure.

---

## LOCAL API

```text
http://localhost:21910/api/detail/List
```

---

## Purpose

Analyze and benchmark API responses supplier-wise and route-wise.

Generate:

- supplier-wise response analysis
- supplier-wise response time
- bus count analysis
- fare comparison
- inventory comparison
- missing supplier report
- missing bus report
- duplicate bus analysis
- API performance report
- request/response logs
- benchmark report
- CSV/Excel/JSON reports

---

## Add Files Inside Existing Project

### Services

```text
/Services/ApiComparisonService.cs
/Services/SupplierBenchmarkService.cs
```

### Utilities

```text
/Utilities/ApiResponseComparer.cs
/Utilities/ResponseTimeTracker.cs
```

### Models

```text
/Models/ComparisonResult.cs
/Models/SupplierAnalysisResult.cs
/Models/PerformanceMetrics.cs
```

### Reports

```text
/Reports/ComparisonReportGenerator.cs
```

---

## API Requests To Execute

### REQUEST 1

```json
{
  "sourceId": "130",
  "destinationId": "122",
  "date": "25-05-2026",
  "key": "dsasa4gfdg4543gfdg6ghgf45325gfd",
  "version": "1",
  "isVrl": "False",
  "isVolvo": "False",
  "IsAndroidIos_Hit": false,
  "agentCode": "",
  "CountryCode": "IN",
  "Sid": "",
  "Vid": "",
  "snapApp": "Emt",
  "sourceName": "Pune",
  "destinationName": "Bangalore",
  "isInventory": "0",
  "TraceId": ""
}
```

### REQUEST 2

```json
{
  "sourceId": "733",
  "destinationId": "757",
  "date": "23-05-2026",
  "key": "dsasa4gfdg4543gfdg6ghgf45325gfd",
  "version": "1",
  "isVrl": "False",
  "isVolvo": "False",
  "IsAndroidIos_Hit": false,
  "agentCode": "",
  "CountryCode": "IN",
  "Sid": "",
  "Vid": "",
  "snapApp": "Emt",
  "sourceName": "Delhi",
  "destinationName": "Manali",
  "isInventory": "0",
  "TraceId": ""
}
```

### REQUEST 3

```json
{
  "sourceId": "122",
  "destinationId": "124",
  "date": "24-05-2026",
  "key": "dsasa4gfdg4543gfdg6ghgf45325gfd",
  "version": "1",
  "isVrl": "False",
  "isVolvo": "False",
  "IsAndroidIos_Hit": false,
  "agentCode": "",
  "CountryCode": "IN",
  "Sid": "",
  "Vid": "",
  "snapApp": "Emt",
  "sourceName": "Bengaluru",
  "destinationName": "Hyderabad",
  "isInventory": "0",
  "TraceId": ""
}
```

---

## Analyze These Parameters

### API Level Analysis

Measure:

- Total response time
- API execution time
- Response size
- Status code
- Serialization time
- Deserialization time
- Timeout
- Exceptions

### Supplier Level Analysis

Analyze:

- Supplier name
- Supplier response time
- Supplier bus count
- Supplier availability
- Missing suppliers
- Extra suppliers
- Failed suppliers
- Supplier mismatch
- Supplier inventory mismatch

### Bus Level Analysis

Compare:

- BusId
- TravelName
- BusType
- Fare
- AvailableSeats
- BoardingPoints
- DroppingPoints
- CancellationPolicy
- DepartureTime
- ArrivalTime
- Duration
- InventoryType

### Validation Checks

Validate:

- Null fields
- Empty responses
- Invalid datetime
- Duplicate buses
- Negative fare
- Zero seats
- Invalid supplier response

---

## Technical Requirements

Use:

- existing dependency injection
- existing logging
- existing models where possible
- HttpClient
- async/await
- Stopwatch
- Newtonsoft.Json
- LINQ
- Parallel processing

Add:

- retry handling
- timeout handling
- exception handling
- supplier-wise timing capture

---

## Logging Requirements

Store logs inside existing Logs folder.

Log:

- Request JSON
- API response JSON
- Response time
- Supplier-wise metrics
- Exception details
- Missing suppliers
- Missing buses
- Benchmark summary

---

## Generate Reports

Create reports inside project root.

```text
/Reports/ApiComparisonReport.xlsx
/Reports/ApiComparisonReport.csv
/Reports/ApiComparisonReport.json
/Reports/ApiComparisonSummary.txt
```

---

## Report Tables

### TABLE 1: API PERFORMANCE

| Route | API Time | Payload Size | Status |
| ----- | -------- | ------------ | ------ |

### TABLE 2: SUPPLIER ANALYSIS

| Supplier | Bus Count | Avg Response Time | Missing Buses |
| -------- | --------- | ----------------- | ------------- |

### TABLE 3: INVENTORY ANALYSIS

| Supplier | BusId | Seats | InventoryType |
| -------- | ----- | ----- | ------------- |

### TABLE 4: FARE ANALYSIS

| Supplier | BusId | Fare |
| -------- | ----- | ---- |

### TABLE 5: ERROR REPORT

| Supplier | Error | Route |
| -------- | ----- | ----- |

---

## Expected Output

Generate:

- production-ready code
- reusable comparison services
- supplier-wise benchmark analyzer
- API response analyzer
- report generator
- CSV export utility
- Excel export utility
- JSON benchmark output
- exception handling
- detailed logging
- maintainable clean architecture

Do NOT create a new application.

Integrate everything inside the existing Bus application root project.
