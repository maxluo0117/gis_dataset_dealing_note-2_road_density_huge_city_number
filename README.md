# GIS Dataset Dealing Note #2: Road Density Calculation for 101 Japanese Cities

> **Case #2:** Calculating road density for approximately 101 Japanese cities from a nationwide OpenStreetMap road network.

---

## 1. Why I Did This

This is another GIS processing problem I encountered during my Master's research.

For my cross-city analysis, I needed a consistent road density indicator for approximately **101 Japanese cities**, with city boundaries defined using the **2015 GHSL Urban Centre dataset**.

Calculating road density for one city is fairly straightforward:

```text
city boundary → clip roads → calculate road length → divide by area
```

The problem appeared when I tried to repeat the same operation for more than 100 cities using a **nationwide Japanese OpenStreetMap road dataset**.

My first instinct was to load the entire road network and simply loop through every city.

That works in principle, but it quickly became inefficient in practice. The nationwide road layer is large, while each iteration only needs a very small part of it.

So the main challenge of this case was actually not the road-density formula itself.

It was:

> **How can I repeatedly extract and process small local road networks from a nationwide dataset without keeping unnecessary spatial data in memory?**

This eventually led me to a relatively simple processing strategy:

```text
bbox → local extraction → precise clipping → calculation → release memory
```

---

## 2. Data

The workflow uses two main spatial datasets:

- **2015 GHSL Urban Centre boundaries** — used to define the study areas;
- **OpenStreetMap road network for Japan** — used to calculate road density.

The original OSM data was downloaded in `.osm.pbf` format from [OpenStreetMap](https://www.openstreetmap.org/).

For the actual Python processing, however, I converted the road network into **GeoPackage (`.gpkg`)** format.

---

## 3. Why I Used GeoPackage

At first, I considered working directly with the original `.osm.pbf` data.

For repeated GeoPandas operations, however, I found it more convenient to first prepare the road network as a GIS-ready vector dataset.

Following the approach described in Reference [1], I converted the Japanese OSM road network into:

```text
.osm.pbf → .gpkg
```

Using `.gpkg` also made it easier for me to inspect the data in GIS software and reuse the same processed road layer without repeating the OSM conversion step.

For this project, I therefore treated the GeoPackage as my main road-network database.

---

## 4. A Computing Problem I Did Not Expect

One thing I underestimated at the beginning was how much the **computing environment** would matter.

My initial attempts were made directly in Windows. For smaller city-level datasets this was fine, but processing the nationwide road network repeatedly became unstable in my local environment.

Instead of continuing to fight with the same setup, I moved the workflow to:

- **Visual Studio Code**
- **WSL2**
- **Linux-based Python environment**

This did not change the GIS methodology itself, but it made the large-scale processing much more stable on my machine.

It was also a useful reminder that sometimes the bottleneck in spatial analysis is not the algorithm — it is simply how much data I am asking the computer to hold and move around.

---

## 5. The Main Processing Strategy

The key idea is to avoid processing the entire Japanese road network for every city.

Instead, each city is handled independently.

For every city, the workflow is roughly:

```text
City boundary
     ↓
Bounding box
     ↓
Read only nearby roads
     ↓
Clip roads to actual city boundary
     ↓
Calculate road length
     ↓
Calculate road density
     ↓
Delete temporary data
     ↓
Next city
```

### Step 1 — Get the City Bounding Box

For each GHSL Urban Centre, I first obtain its bounding box:

```python
minx, miny, maxx, maxy = city.geometry.total_bounds
```

The bounding box is not the final study boundary.

I use it as a **cheap first spatial filter**.

Instead of asking GeoPandas to work with the entire nationwide road network, I first tell it:

> Only give me the roads somewhere around this city.

---

### Step 2 — Extract Only the Local Road Network

The next step is to read or select only roads that fall within the city's bounding-box extent.

Conceptually:

```text
Nationwide roads
      ↓
bbox filter
      ↓
Small local road subset
```

This was the most important change in the workflow.

Most Japanese cities only occupy a very small fraction of the nationwide dataset, so loading everything for every iteration wastes both memory and processing time.

The bounding-box filter reduces the dataset before any more expensive geometric operation is performed.

---

### Step 3 — Clip Roads to the Actual Urban Boundary

A bounding box is rectangular, while an actual urban boundary is obviously not.

So after the initial extraction, I perform a more precise geometric clip:

```python
local_roads = gpd.clip(local_roads, city_boundary)
```

I think of these two operations as serving different purposes:

```text
bbox filtering = computational shortcut
gpd.clip()      = actual spatial boundary
```

The first step makes the dataset smaller.

The second step makes the spatial result accurate.

Keeping these two purposes separate helped me understand why using both operations was more efficient than immediately clipping the nationwide dataset.

---

### Step 4 — Calculate Road Length

After clipping, I calculate the length of the road geometries:

```python
local_roads["road_length"] = local_roads.geometry.length
```

The total road length for a city can then be calculated using:

```python
total_road_length = local_roads["road_length"].sum()
```

One important detail here is the **coordinate reference system (CRS)**.

Because `.length` calculates distance using the coordinates of the geometry, the road data needs to be projected into an appropriate projected CRS before calculating metric road lengths.

Otherwise, the resulting values may not represent meaningful distances in meters.

---

### Step 5 — Calculate Road Density

For each city, the basic road-density indicator is:

```text
road_density = total_road_length / city_area
```

The exact unit depends on how road length and city area are represented.

For example:

```text
km of road / km² of urban area
```

This produces a comparable road-network intensity indicator across the approximately 101 urban centres.

---

### Step 6 — Release Temporary Data

This step looks almost too simple to mention, but it became important when repeating the workflow more than 100 times.

Once the calculation for one city is complete, I no longer need its temporary road subset.

So I explicitly remove temporary objects before moving to the next city:

```python
del local_roads
```

and, when necessary:

```python
import gc
gc.collect()
```

The logic is basically:

```text
load small → calculate → save result → delete → repeat
```

rather than:

```text
load everything → keep everything → hope memory survives
```

For this particular task, that simple change in thinking made the workflow much easier to manage.

---

## 6. What I Learned From This Case

The road-density calculation itself turned out to be the easiest part.

The more interesting problem was **how to scale a familiar GIS operation from one city to more than 100 cities**.

### Do the Cheap Spatial Filter First

My main takeaway from this case was that the order of spatial operations matters.

It is technically possible to directly run a precise geometric clip against a very large dataset.

But if a simple bounding-box filter can eliminate most irrelevant features first, there is little reason to make the expensive operation process everything.

So I gradually settled on:

```text
coarse spatial filter first → precise geometry operation second
```

This is a very simple idea, but it became useful in several later GIS workflows.

---

### Nationwide Data Does Not Mean Nationwide Processing

Another thing I learned was to separate the **extent of the source dataset** from the **extent of each computation**.

My source data covers all of Japan.

But my calculation at any given moment only concerns one city.

Those are two very different things.

Once I started treating the nationwide road network as a database from which I repeatedly extract small local subsets, the workflow became much more manageable.

---

### Memory Management Is Part of the GIS Workflow

Before working with larger spatial datasets, I mostly thought about memory problems as a programming issue.

In this project, I started to see them as part of the **spatial-analysis design** itself.

The geometry type, spatial extent, file format, order of spatial filters, and size of temporary GeoDataFrames all affect how practical a GIS workflow is.

In other words, optimizing a spatial workflow is not always about writing more complicated code.

Sometimes it is simply about **not loading data that I do not need**.

---

## 7. GIS Operations Used

I am keeping this section as a quick reference for myself.

| Function / Property | Where I Used It |
| :--- | :--- |
| `gpd.read_file()` | Read city and road datasets |
| `.total_bounds` | Obtain city bounding boxes |
| `gpd.clip()` | Precisely clip roads to urban boundaries |
| `.to_crs()` | Project geometries before distance calculations |
| `.geometry.length` | Calculate road-segment lengths |
| `.geometry.area` | Calculate city area |
| `.sum()` | Aggregate road length |
| `del` | Remove temporary GeoDataFrames |
| `gc.collect()` | Request garbage collection when needed |

---

## 8. About This Note

This case started as a very simple task: calculate road density for a group of cities.

What made it interesting was that the approach I would normally use for **one city** did not scale very comfortably to **101 cities and a nationwide road dataset**.

Most of my time therefore went into reorganizing the processing logic rather than changing the actual road-density calculation.

A large part of the implementation was developed through my own testing and debugging, with **Gemini** helping me think through some of the coding and performance issues.

I would describe this repository less as a polished GIS tutorial and more as a record of how I solved one practical problem in my research.

There are probably more efficient ways to do some of these operations, and suggestions are always welcome.



## Reference

[1] 竹やぶ大好き (2020).  
https://zenn.dev/akioz/articles/2f42d0db9800a7
