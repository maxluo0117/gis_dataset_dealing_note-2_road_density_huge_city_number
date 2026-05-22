# gis_dataset_dealing_note-2_road_density_huge_city_number

This repository describes a workflow for calculating road density indicators for approximately 101 Japanese cities using the 2015 GHSL Urban Centre boundaries. The project mainly focuses on large-scale geospatial processing, memory-efficient vector GIS workflows, and road network density computation based on OpenStreetMap (OSM) data.

---

# 1. Highlights
- Applied GeoPandas-based road density calculations (`geometry.length`, `gpd.clip`) to large-scale multi-city processing workflows.
- Processed nationwide Japanese road network data for approximately 101 cities.
- Utilized bounding-box spatial filtering and local extraction strategies to reduce memory usage during computation.
- Converted large-scale OSM road network data into GeoPackage (`.gpkg`) format for efficient GIS processing.

---

# 2. Workflow
## (1) Using GeoPackage (`.gpkg`) for large-scale GIS processing
This project primarily uses GeoPackage (`.gpkg`) files because they are more efficient and stable for handling large vector geospatial datasets compared with directly processing raw `.osm.pbf` files.
First, the Japanese OSM (https://www.openstreetmap.org/) road network data was downloaded as `.osm.pbf` format.  
Then, following the method introduced in Reference [1], the road network data was converted into GeoPackage (`.gpkg`) format for subsequent spatial analysis.

---

## (2) Computing environment
Processing large-scale geospatial datasets directly on Windows was unstable in my local environment. Therefore, the workflow was executed using:
- Visual Studio Code
- WSL2 (Windows Subsystem for Linux)
- Linux-based Python GIS environment
This setup provided better stability for large-scale spatial computation.

---

## (3) Core processing strategy
The main workflow can be summarized as:
```text
bbox -> local extraction -> release memory
```
For each city:
1. Obtain the city's bounding box (`bbox`)
2. Extract only the local road network from the nationwide road dataset
3. Perform precise geometric clipping using the actual city boundary
4. Calculate road length density
5. Release temporary data from memory
This strategy significantly improves memory efficiency when processing large-scale nationwide OSM datasets.

---
# 3. Notes
This project was mainly developed through vibe coding with assistance from Gemini. Suggestions and improvements are always welcome!

---
# 4. Future Work
The next step is to apply the same workflow to Chinese cities and evaluate computational efficiency, memory usage
(The computation is currently ongoing.)

---

# References
[1] 竹やぶ大好き 2020.12.18  https://zenn.dev/akioz/articles/2f42d0db9800a7
