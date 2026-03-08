
### Resources
- [Article](https://blog.algomaster.io/p/geohashing)
- [GeoHash Explorer](https://geohash.softeng.co/)

# Geohashing

## 1. The Problem Geohash Solves

Imagine we store millions of locations:

```
Driver A → (12.9716, 77.5946)
Driver B → (12.9610, 77.6387)
Driver C → (13.0010, 77.5000)
```

Now a user asks:

> Find all drivers within **5 km** of my location.

### Naive approach

Check distance with **every driver**.

Time complexity:

```
O(N)
```

If N = 10 million drivers → slow.

We need a way to **index locations** so that:

```
Nearby locations → similar keys
Far locations → different keys
```

This is exactly what **Geohash** does.

---

## 2. What is Geohash?

**Geohash = encode latitude + longitude into a short string**

Example:

```
Location: 12.9716, 77.5946

Geohash: tdr1v9
```

Important property:

```
Nearby locations share the same prefix
```

Example:

```
tdr1v9
tdr1v8
tdr1vb
```

These are all **close geographically**.

But something far away:

```
xn774c
```

Different prefix.

So we can store data like:

```
HashMap<GeohashPrefix, List<Location>>
```

---

## 3. Key Idea Behind Geohash

Geohash works by **recursively dividing the earth into grids**.

Process:

```
Earth
 └── Divide into halves (longitude)
      └── Divide into halves (latitude)
           └── Repeat...
```

Each division produces **bits**.

These bits become the **geohash string**.

---

## 4. Step 1 — Start With Coordinate Ranges

Latitude range:

```
-90 to +90
```

Longitude range:

```
-180 to +180
```

Suppose we encode:

```
Lat = 42.6
Lon = -5.6
```

---

## 5. Step 2 — Encode Longitude (Binary Search)
- If we go left during search: bit -> 0
- If we go right during search: bit -> 1
  
Start with:

```
Longitude range = [-180, 180]
```

Midpoint of -180 and 180:

```
0
```

Check where does longitude lie, on the left side or the right side of 0?

```
-5.6 < 0
```
-5.6 lies on the left side of 0, So bit = **0** (because we chose the left region of the search space)

New range:

```
[-180, 0]
```

---

Next midpoint:

```
-90
```

Check where does -5.6 lies, on the left side or right side of midpoint (-90)?

```
-5.6 > -90
```
-5.6 is larger than -90, so it lies on the right side of midpoint. So, Bit = **1** (because we chose the right region of the search space)

Range:

```
[-90, 0]
```

---

Next midpoint:

```
-45
```

Check:

```
-5.6 > -45
```

Bit = **1**

Range:

```
[-45, 0]
```

Continue this process.

This is **binary search on longitude**.

---

## 6. Step 3 — Encode Latitude

Latitude range:

```
[-90, 90]
```

Midpoint:

```
0
```

Check:

```
42.6 > 0
```

Bit = **1**

Range:

```
[0, 90]
```

---

Next midpoint:

```
45
```

Check:

```
42.6 < 45
```

Bit = **0**

Range:

```
[0, 45]
```

Continue.

---

## 7. Step 4 — Interleave Bits

We don't keep lat and lon separately.

We **interleave them**:

```
lon lat lon lat lon lat ...
```

Example:

```
Longitude bits: 0 1 1 0 0
Latitude bits : 1 0 1 1 1
```

Interleaving:

```
0 1 1 0 1 1 0 1 0 1
```

This creates **one binary number**.

---

## 8. Step 5 — Convert to Base32

Geohash does not store raw binary.

It converts every **5 bits → Base32 character**

Base32 alphabet:

```
0123456789bcdefghjkmnpqrstuvwxyz
```

Example:

```
01101 → d
```

So final geohash looks like:

```
u4pruydqqvj
```

Each character adds **more precision**.

---

## 9. Precision of Geohash

Longer geohash = smaller grid.

| Length | Area    |
| ------ | ------- |
| 3      | ~150 km |
| 5      | ~5 km   |
| 6      | ~1.2 km |
| 7      | ~150 m  |
| 8      | ~40 m   |

Example:

```
tdr1v
```

means a **~5km square**.

All points inside that square share the prefix.

---

## 10. Why Geohash Is Powerful

Because **prefix matching = spatial locality**

Example:

```
tdr1v9
tdr1v8
tdr1vb
```

All share prefix:

```
tdr1v
```

So DB query:

```
SELECT * 
FROM drivers
WHERE geohash LIKE 'tdr1v%'
```

Instantly returns nearby drivers.

This is **very fast with indexes**.

---

## 11. Handling "Nearby Search"

Problem:

Geohash grids are **squares**.

But radius search is **circle**.

Example:

```
User near border of grid
```

Nearby drivers may be in **neighboring grids**.

Solution:

Search **adjacent geohashes**.

Example:

```
Main cell
8 neighboring cells
```

```
[ ] [ ] [ ]
[ ] [X] [ ]
[ ] [ ] [ ]
```

So query:

```
tdr1v
tdr1u
tdr1t
tdr1w
...
```

Total ≈ **9 cells**.

---

## 12. Example System Design (Uber)

Drivers stored like:

```
DriverID → Geohash
```

Example table:

```
driver_id | geohash
--------------------
A         | tdr1v9
B         | tdr1v8
C         | tdr1vb
```

User requests ride:

1. Compute user geohash
2. Take prefix length = radius
3. Find **neighbor cells**
4. Query DB using prefix
5. Filter by **exact distance**

---

## 13. Complexity

Without geohash:

```
O(N)
```

With geohash:

```
O(k)
```

Where:

```
k = drivers in nearby cells
```

Usually **very small**.

---

## 14. Limitations (Important for Interviews)

### 1. Edge Problem

Points near border may be closer to **neighbor cell**.

Solution:

```
Search neighbors
```

---

### 2. Uneven Cell Shapes

Cells are **not perfect squares on Earth** because of spherical geometry.

But acceptable for most apps.

---

### 3. Precision Tradeoff

Too small:

```
Too many DB queries
```

Too large:

```
Too many results
```

So systems dynamically adjust precision.

---

## 15. Where Geohash Is Used

### Ride matching

Uber / Ola

### Food delivery

Swiggy / Zomato

### Nearby search

Google Maps

### Geo databases

MongoDB / Redis GEO

---

## 16. Geohash vs QuadTree


| Feature        | Geohash           | QuadTree       |
| -------------- | ----------------- | -------------- |
| Representation | String prefix     | Tree           |
| Storage        | Database friendly | In-memory      |
| Queries        | Prefix search     | Tree traversal |
| Scale          | Distributed DB    | Harder         |

So:

```
Geohash → production systems
QuadTree → algorithms / graphics
```

---

## 17. Interview One-Line Explanation

Good concise explanation:

> Geohash converts latitude and longitude into a Base32 string by recursively subdividing the Earth’s surface and interleaving latitude and longitude bits. Nearby locations share common prefixes, allowing efficient spatial indexing and prefix-based database queries.

---


# GeoHash Precision

## What Precision Means in Geohash

Precision = **number of characters in the geohash string**

Example:

```
tdr
tdr1
tdr1v
tdr1v9
```

Each additional character:

* divides the space further
* makes the cell **smaller**
* increases location **accuracy**

So:

```
more characters → smaller cell → higher precision
```

---

## Geohash Precision Table

These are the **commonly used levels** interviewers expect you to know.

| Length | Cell Width | Cell Height | Typical Use   |
| ------ | ---------- | ----------- | ------------- |
| 1      | ~5000 km   | ~5000 km    | continent     |
| 2      | ~1250 km   | ~625 km     | large country |
| 3      | ~156 km    | ~156 km     | region        |
| 4      | ~39 km     | ~19.5 km    | city          |
| 5      | ~4.9 km    | ~4.9 km     | neighborhood  |
| 6      | ~1.2 km    | ~0.6 km     | small area    |
| 7      | ~150 m     | ~150 m      | street        |
| 8      | ~38 m      | ~19 m       | building      |
| 9      | ~4.8 m     | ~4.8 m      | house         |
| 10     | ~1.2 m     | ~0.6 m      | room-level    |

Most real systems use:

```
precision 5–7
```

---

## Example

Take Bangalore coordinates:

```
Latitude: 12.9716
Longitude: 77.5946
```

Different precision levels:

```
t
td
tdr
tdr1
tdr1v
tdr1v9
tdr1v9h
```

Each step zooms into a **smaller geographic area**.

Example:

```
tdr        → ~150 km region
tdr1       → ~40 km area
tdr1v      → ~5 km area
tdr1v9     → ~1 km area
tdr1v9h    → ~150 m area
```

---

## Why Systems Choose Precision Carefully

If precision is **too small**:

```
cell size = 150 km
```

Query:

```
find drivers within 3 km
```

You will retrieve **thousands of drivers**.

Bad performance.

---

If precision is **too large**:

```
cell size = 40 meters
```

Now a 3 km search requires **hundreds of geohash cells**.

Also bad.

---

So systems pick a precision where:

```
cell size ≈ search radius
```

Example:

| Search Radius | Geohash Precision |
| ------------- | ----------------- |
| 20 km         | 4                 |
| 5 km          | 5                 |
| 1 km          | 6                 |
| 200 m         | 7                 |

---

## Why Width and Height Differ

Geohash cells are not perfect squares.

Example:

```
precision 6
width  ≈ 1.2 km
height ≈ 0.6 km
```

This happens because:

```
longitude and latitude bits alternate
```

So the grid stretches slightly.

---

## Typical Precision Used in Real Systems

Examples:

**Ride sharing (Uber / Ola)**

```
precision = 6
```

**Restaurant search**

```
precision = 6 or 7
```

**Geo analytics**

```
precision = 5
```

---

## Interview Answer (Concise)

If asked:

**“What are geohash precision levels?”**

Good answer:

> Geohash precision refers to the number of characters in the geohash string. Each additional character divides the geographic grid further, increasing spatial resolution. For example, precision 5 represents about a 5 km area, precision 6 about 1 km, and precision 7 about 150 meters. Systems choose precision based on the search radius to balance query efficiency and accuracy.

---



## **How do you choose geohash precision for a given search radius?**


### 1. The Core Problem

Geohash creates **grid cells**.

Example:

```
precision = 5 → cell ≈ 5km × 5km
precision = 6 → cell ≈ 1.2km × 0.6km
precision = 7 → cell ≈ 150m × 150m
```

But user queries are usually:

```
Find drivers within 3 km
Find restaurants within 2 km
Find users within 500 m
```

So the question becomes:

> **Which geohash length should we use for that radius?**

If we choose the wrong one, performance suffers.

---

### 2. If Precision Is Too Small (Bad)

Example:

```
radius = 2km
precision = 3
```

Cell size:

```
~150km
```

That means the DB query returns **huge numbers of results**.

Example:

```
SELECT * FROM drivers
WHERE geohash LIKE 'tdr%'
```

Maybe **50,000 drivers** come back.

Then the system must filter them all.

Bad performance.

---

### 3. If Precision Is Too Large (Also Bad)

Example:

```
radius = 5km
precision = 8
```

Cell size:

```
~40 meters
```

Your 5km radius now spans **thousands of cells**.

You must query **many geohashes**.

That also hurts performance.

---

### 4. Ideal Precision

The rule is:

> Choose a geohash cell size **close to the search radius**.

That way:

* few DB queries
* few extra results

---

### 5. Common Precision Table

This is the table most engineers memorize.

| Length | Cell Width | Cell Height |
| ------ | ---------- | ----------- |
| 1      | 5000 km    | 5000 km     |
| 2      | 1250 km    | 625 km      |
| 3      | 156 km     | 156 km      |
| 4      | 39 km      | 19.5 km     |
| 5      | 4.9 km     | 4.9 km      |
| 6      | 1.2 km     | 0.6 km      |
| 7      | 150 m      | 150 m       |
| 8      | 38 m       | 19 m        |

Important ones for interviews:

```
5 → ~5km
6 → ~1km
7 → ~150m
```

---

### 6. Example (Uber Style)

User wants:

```
drivers within 3km
```

Choose:

```
precision = 5
```

Because cell ≈ **5km**.

Then search:

```
center + 8 neighbors
```

Total cells:

```
9
```

This covers roughly:

```
~15km × 15km
```

Then filter exact distance.

---

### 7. Another Example

User query:

```
restaurants within 500m
```

Use:

```
precision = 6
```

Cell size:

```
~1km
```

So nearby cells cover enough area.

---

### 8. Typical Mapping Used in Systems

Real systems often do something like:

```
if radius > 2500m → precision 5
if radius > 600m  → precision 6
if radius > 75m   → precision 7
else              → precision 8
```

This keeps queries efficient.

---

### 9. What the System Actually Does

Example query:

```
Find drivers within 2km
```

System flow:

#### Step 1

Compute user geohash.

```
tdr1v
```

#### Step 2

Choose precision.

```
5
```

#### Step 3

Find neighbors.

```
tdr1v
tdr1u
tdr1t
tdr1w
tdr1x
...
```

Total:

```
9 cells
```


#### Step 4

Query DB

```
SELECT * 
FROM drivers
WHERE geohash IN (...)
```


#### Step 5

Filter by exact distance.

Using **Haversine distance**.



### 10. Why Exact Distance Is Still Needed

Geohash cells are **squares**.

But search area is **circle**.

Example:

```
+---------+
|         |
|   ( )   |
|         |
+---------+
```

Points in corners of square may be outside the circle.

So final step:

```
distance(driver, user) <= radius
```


### 11. Important Interview Insight

Good engineers mention this optimization:

> **Store multiple geohash precisions**

Example DB schema:

```
driver_id
geohash_5
geohash_6
geohash_7
```

Then queries can use the correct index.

This is often done in **Redis GEO** or **MongoDB geo indexing**.

---

### 12. Clean Interview Answer

If interviewer asks:

> How do you choose geohash precision?

Answer:

> The geohash precision determines the grid cell size. We choose a precision where the cell size is close to the search radius. For example, precision 5 corresponds to roughly 5km cells, precision 6 to around 1km, and precision 7 to about 150 meters. This ensures we retrieve nearby candidates efficiently while minimizing unnecessary results. After fetching candidates from the center and neighboring cells, we filter them using exact distance calculations.





