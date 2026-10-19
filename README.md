# graph_database

UK road-accident data (Department for Transport "Road Accidents and Safety Data",
2015) prepared for loading into a graph database such as Neo4j. The repository
contains the 2015 accident records plus the data-guide lookup tables that turn
the numeric codes in those records into readable labels. It holds data only: there
are no scripts or Cypher queries in the repo yet.

## Project structure

```
graph_database/
├── data/
│   ├── accidents.csv          # 140,056 accident records from 2015 (one row per accident)
│   └── lookups/               # 48 data-guide files (code -> label tables plus notes)
│       ├── Introduction.csv       # data-guide notes (see below)
│       ├── Export Variables.csv   # variables for accidents, vehicles and casualties
│       ├── Accident Severity.csv
│       ├── Police Force.csv
│       ├── Weather.csv
│       └── ...
├── .gitignore
└── README.md
```

## The dataset

### `data/accidents.csv`

One row per accident, 32 columns. `Accident_Index` is the unique key (the data
guide notes it also links to vehicle and casualty records, which are not in this
repo). Columns:

`Accident_Index, Location_Easting_OSGR, Location_Northing_OSGR, Longitude, Latitude,
Police_Force, Accident_Severity, Number_of_Vehicles, Number_of_Casualties, Date
(DD/MM/YYYY), Day_of_Week, Time (HH:MM), Local_Authority_(District),
Local_Authority_(Highway), 1st_Road_Class, 1st_Road_Number, Road_Type, Speed_limit,
Junction_Detail, Junction_Control, 2nd_Road_Class, 2nd_Road_Number,
Pedestrian_Crossing-Human_Control, Pedestrian_Crossing-Physical_Facilities,
Light_Conditions, Weather_Conditions, Road_Surface_Conditions,
Special_Conditions_at_Site, Carriageway_Hazards, Urban_or_Rural_Area,
Did_Police_Officer_Attend_Scene_of_Accident, LSOA_of_Accident_Location`

Notes from the data guide (`data/lookups/Introduction.csv`):

- The value `-1` means NULL or out of range.
- Longitude and latitude use WGS 1984.

### `data/lookups/`

Most files are two-column `code,label` tables, for example `Weather.csv`:

```
code,label
1,Fine no high winds
2,Raining no high winds
...
-1,Data missing or out of range
```

Exceptions:

- `Local Authority _Highway_.csv` uses the header `Code,Label` and its codes are
  ONS codes (e.g. `E09000020`), matching `Local_Authority_(Highway)`.
- `Introduction.csv` is free-text notes, not a table.
- `Export Variables.csv` lists the variables for accidents, vehicles and casualties
  in three columns.
- Several lookups (vehicle and casualty fields such as `Vehicle Type.csv` or
  `Casualty Class.csv`) describe columns that are not in `accidents.csv`.

Lookup files that match `accidents.csv` columns:

| accidents.csv column | Lookup file |
|---|---|
| Police_Force | `Police Force.csv` |
| Accident_Severity | `Accident Severity.csv` |
| Day_of_Week | `Day of Week.csv` |
| Local_Authority_(District) | `Local Authority _District_.csv` |
| Local_Authority_(Highway) | `Local Authority _Highway_.csv` |
| 1st_Road_Class / 2nd_Road_Class | `1st Road Class.csv` / `2nd Road Class.csv` |
| Road_Type | `Road Type.csv` |
| Junction_Detail | `Junction Detail.csv` |
| Junction_Control | `Junction Control.csv` |
| Pedestrian_Crossing-Human_Control | `Ped Cross _ Human.csv` |
| Pedestrian_Crossing-Physical_Facilities | `Ped Cross _ Physical.csv` |
| Light_Conditions | `Light Conditions.csv` |
| Weather_Conditions | `Weather.csv` |
| Road_Surface_Conditions | `Road Surface.csv` |
| Special_Conditions_at_Site | `Special Conditions at Site.csv` |
| Carriageway_Hazards | `Carriageway Hazards.csv` |
| Urban_or_Rural_Area | `Urban Rural.csv` |
| Did_Police_Officer_Attend_Scene_of_Accident | `Police Officer Attend.csv` |

File names contain spaces (and `_` where the original sheet names had brackets or
slashes), so quote them in shell commands and URL-encode them (`%20`) in
`LOAD CSV` URLs.

## Loading into Neo4j

The repo has no load scripts; the following is a starting point.

1. Copy the `data/` folder into your Neo4j `import` directory (for Neo4j Desktop:
   open the DBMS's folder, then `import/`), so the files are at
   `import/accidents.csv` and `import/lookups/...`:

   ```bash
   cp -r data/* "<NEO4J_HOME>/import/"
   ```

2. Load accidents as nodes:

   ```cypher
   CREATE CONSTRAINT accident_id IF NOT EXISTS
   FOR (a:Accident) REQUIRE a.id IS UNIQUE;

   LOAD CSV WITH HEADERS FROM 'file:///accidents.csv' AS row
   CALL {
     WITH row
     MERGE (a:Accident {id: row.Accident_Index})
     SET a.date = row.Date,
         a.time = row.Time,
         a.longitude = toFloat(row.Longitude),
         a.latitude = toFloat(row.Latitude),
         a.vehicles = toInteger(row.Number_of_Vehicles),
         a.casualties = toInteger(row.Number_of_Casualties),
         a.speedLimit = toInteger(row.Speed_limit),
         a.severityCode = row.Accident_Severity,
         a.weatherCode = row.Weather_Conditions,
         a.policeForceCode = row.Police_Force
   } IN TRANSACTIONS OF 10000 ROWS;
   ```

   (On Neo4j 4.x use `USING PERIODIC COMMIT 10000` before `LOAD CSV` instead of
   `CALL { ... } IN TRANSACTIONS`.)

3. Load a lookup table and link it, e.g. weather:

   ```cypher
   LOAD CSV WITH HEADERS FROM 'file:///lookups/Weather.csv' AS row
   MERGE (:Weather {code: row.code, label: row.label});

   MATCH (a:Accident), (w:Weather {code: a.weatherCode})
   MERGE (a)-[:HAD_WEATHER]->(w);
   ```

   Repeat for the other lookups in the table above, remembering that
   `Local Authority _Highway_.csv` uses `row.Code` / `row.Label`.

Running `LOAD CSV` against `accidents.csv` (about 19 MB) is fine for a local
Neo4j instance; the lookup files are a few hundred bytes each.
