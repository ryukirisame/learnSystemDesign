
### Resources
- [Article](https://blog.algomaster.io/p/geohashing)
- [GeoHash Explorer](https://geohash.softeng.co/)

# Geohashing

# 1. The Problem Geohash Solves

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

# 2. What is Geohash?

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

# 3. Key Idea Behind Geohash

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

# 4. Step 1 — Start With Coordinate Ranges

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

# 5. Step 2 — Encode Longitude (Binary Search)

Start with:

```
Longitude range = [-180, 180]
```

Midpoint:

```
0
```

Check:

```
-5.6 < 0
```

So bit = **0**

New range:

```
[-180, 0]
```

---

Next midpoint:

```
-90
```

Check:

```
-5.6 > -90
```

Bit = **1**

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

# 6. Step 3 — Encode Latitude

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

# 7. Step 4 — Interleave Bits

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

# 8. Step 5 — Convert to Base32

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

# 9. Precision of Geohash

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

# 10. Why Geohash Is Powerful

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

# 11. Handling "Nearby Search"

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

# 12. Example System Design (Uber)

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

# 13. Complexity

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

# 14. Limitations (Important for Interviews)

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

# 15. Where Geohash Is Used

### Ride matching

Uber / Ola

### Food delivery

Swiggy / Zomato

### Nearby search

Google Maps

### Geo databases

MongoDB / Redis GEO

---

# 16. Geohash vs QuadTree

You asked this earlier, so here's the **interview difference**:

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

# 17. Interview One-Line Explanation

Good concise explanation:

> Geohash converts latitude and longitude into a Base32 string by recursively subdividing the Earth’s surface and interleaving latitude and longitude bits. Nearby locations share common prefixes, allowing efficient spatial indexing and prefix-based database queries.

---



Those are the parts that **really impress interviewers**.
