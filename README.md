# SQL++ demo
## Introduction and prerequisites
Create a free Couchbase Capella account, spin up a cluster and import the `travel-sample` dataset.

Navigate to `Data Tools -> Query`

### Select the name, IATA code, and ICAO code for airlines located in the United States, limiting the result to 5 entries.
```sql
SELECT name, iata, icao
FROM airline
WHERE country = "United States"
LIMIT 5;
```

### Count the number of airports in each country, ordering the results by the airport count in descending order and limiting to the top 5 countries.
```sql
SELECT country, COUNT(*) AS airport_count
FROM airport
GROUP BY country
ORDER BY airport_count DESC
LIMIT 5;
```

### In the route keyspace, flatten the schedule array to get details of the flights on Monday (1).
```sql
SELECT route.sourceairport, route.destinationairport, sched.flight, sched.utc
FROM route
UNNEST schedule sched
WHERE  sched.day = 1
LIMIT 3;
```

### List only airports in Toulouse which have routes starting from them, and nest details of the routes.
```sql
SELECT *
FROM airport a
  INNER NEST route r
  ON a.faa = r.sourceairport
WHERE a.city = "Toulouse"
ORDER BY a.airportname;
```

### Calculate the number of airlines per country using a window function, showing the distribution of airlines across different countries.
```sql
SELECT a.name, a.country, 
       COUNT(*) OVER (PARTITION BY a.country) AS country_airline_count
FROM airline a
ORDER BY country_airline_count DESC
LIMIT 10;
```

### Retrieve airport names and geo-coordinates, filtering based on latitude and longitude ranges.
```sql
SELECT ap.name, ap.geo.lat, ap.geo.lon
FROM airport ap
WHERE ap.geo.lat BETWEEN 40 AND 50
  AND ap.geo.lon BETWEEN -80 AND -70
LIMIT 5;
```

### Join airline and route on the IATA code, groups by airline name, and counts the number of routes per airline.
```sql
SELECT a.name AS airline, COUNT(r.airline) AS route_count
FROM `travel-sample`.inventory.airline a
JOIN `travel-sample`.inventory.route r ON r.airline = a.iata
WHERE a.callsign IS NOT MISSING
GROUP BY a.name
ORDER BY route_count DESC
LIMIT 5;
```

### Use pattern matching to find airlines whose names start with "A" or contain "Air" with a "B" prefix.
```sql
SELECT a.name, a.icao
FROM `travel-sample`.inventory.airline a
WHERE (a.name LIKE "A%"
       OR REGEXP_CONTAINS(a.name, "^[Bb].*[Aa]ir"))
ORDER BY a.name
LIMIT 10;
```

### Categorize airports by region based on their country and counts the number of airports in each region.
```sql
SELECT ap.name, ap.country,
       CASE 
         WHEN ap.country IN ["United States", "Canada", "Mexico"] THEN "North America"
         WHEN ap.country IN ["United Kingdom", "France", "Germany", "Spain", "Italy"] THEN "Europe"
         WHEN ap.country IN ["China", "Japan", "India"] THEN "Asia"
         ELSE "Other Regions"
       END AS region,
       COUNT(*) AS airport_count
FROM `travel-sample`.inventory.airport ap
GROUP BY ap.name, ap.country, region
ORDER BY airport_count DESC
LIMIT 10;
```

### Calculate the distance between each hotel and New York City using geo-coordinates.
```sql
SELECT h.name AS hotel_name, 
       h.address.city AS city,
       h.geo.lat AS latitude, 
       h.geo.lon AS longitude,
       ROUND(
         DEGREES(ACOS(
           SIN(RADIANS(h.geo.lat)) * SIN(RADIANS(40.7128)) + 
           COS(RADIANS(h.geo.lat)) * COS(RADIANS(40.7128)) * 
           COS(RADIANS(h.geo.lon - (-74.0060)))
         )) * 69.09
       ) AS miles_from_nyc
FROM `travel-sample`.inventory.hotel h
WHERE h.geo IS NOT MISSING
  AND h.geo.lat IS NOT MISSING
  AND h.geo.lon IS NOT MISSING
ORDER BY miles_from_nyc
LIMIT 10;
```

### The CTE (TopAirports) filters countries with more than 50 airports. The main query joins these countries to airlines and aggregates airline names per country.
```sql
WITH TopAirports AS (
  SELECT ap.country, COUNT(*) AS airport_count
  FROM `travel-sample`.inventory.airport ap
  GROUP BY ap.country
  HAVING COUNT(*) > 50
)
SELECT ta.country, ta.airport_count, 
       ARRAY_AGG(a.name) AS airlines_in_country
FROM TopAirports ta
JOIN `travel-sample`.inventory.airline a ON ta.country = a.country
GROUP BY ta.country, ta.airport_count
ORDER BY ta.airport_count DESC
LIMIT 5;
```

### One hop (direct) connections (edges) from node LAX to node JFK, and show the properties of the edge (flight, airline) and the connected nodes (airports).
```sql
SELECT 
    src.faa AS source_airport_code,
    src.name AS source_airport_name,
    dest.faa AS destination_airport_code,
    dest.name AS destination_airport_name,
    r.airline AS airline_code,
    a.name AS airline_name,
    r.flight AS flight_number
FROM `travel-sample`.inventory.route r
JOIN `travel-sample`.inventory.airport src ON r.sourceairport = src.faa
JOIN `travel-sample`.inventory.airport dest ON r.destinationairport = dest.faa
JOIN `travel-sample`.inventory.airline a ON r.airline = a.iata
WHERE src.faa = "LAX"
  AND dest.faa = "JFK";
```

Nodes: Airports (src and dest)  
Edge: Route (r) connecting source to destination  
Edge Properties: Airline, flight number  
Traversal: From LAX (source node) to JFK (destination node) via a direct route (edge)  
Enrichment: Joins with the airline collection to show airline details

### A two-hop graph traversal query that finds all possible connections from LAX to JFK with exactly one stopover.
```sql
SELECT 
    src.faa AS departure_airport_code,
    src.name AS departure_airport_name,
    r1.airline AS first_airline_code,
    a1.name AS first_airline_name,
    r1.flight AS first_flight_number,
    mid.faa AS stopover_airport_code,
    mid.name AS stopover_airport_name,
    r2.airline AS second_airline_code,
    a2.name AS second_airline_name,
    r2.flight AS second_flight_number,
    dest.faa AS arrival_airport_code,
    dest.name AS arrival_airport_name
FROM `travel-sample`.inventory.route r1
JOIN `travel-sample`.inventory.airport src ON r1.sourceairport = src.faa
JOIN `travel-sample`.inventory.airport mid ON r1.destinationairport = mid.faa
JOIN `travel-sample`.inventory.airline a1 ON r1.airline = a1.iata
JOIN `travel-sample`.inventory.route r2 ON mid.faa = r2.sourceairport
JOIN `travel-sample`.inventory.airport dest ON r2.destinationairport = dest.faa
JOIN `travel-sample`.inventory.airline a2 ON r2.airline = a2.iata
WHERE src.faa = "LAX"
  AND dest.faa = "JFK"
  AND mid.faa != src.faa
  AND mid.faa != dest.faa
  AND mid.country = "United States" -- Filter stopovers to US airports
ORDER BY mid.name
LIMIT 10;
```

Nodes: Airports (src, mid, dest)  
Edges: Routes (r1, r2) connecting the airports  
Edge 1: LAX → Stopover  
Edge 2: Stopover → JFK

Traversal Logic:  
Start at node LAX  
Find all outgoing edges (routes) from LAX  
Follow each edge to an intermediate node (stopover airport)  
From each intermediate node, find all outgoing edges to JFK
Return the complete path with details of both edges
