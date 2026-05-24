# Step 1 — Operational definition
 "A green space is any contiguous vegetated parcel accessible to the public, ≥500 sqm in area, with NDVI ≥0.3 in peak season." This definition does three things — it sets a minimum area filter (kills noise), sets a vegetation threshold (makes satellite detection calibratable), and excludes private gardens (prevents double-counting).
# Step 2 — expand the OSM tag query, don't rely on defaults
The default QGIS QuickOSM green space preset queries leisure=park only. Requirement is to add every tag that in Indian cities gets used for parks that aren't called parks. The full query:
leisure = park OR garden OR nature_reserve OR recreation_ground
landuse = grass OR village_green OR recreation_ground OR forest
amenity = community_centre  (filter only if polygon area > 500 sqm)
boundary = protected_area
There are parks that someone pinned but never digitised in OSM. Run a separate query for nodes with the same tags, buffer 40m, flag as geometry=point_origin
# Step 3 — Extraction of NDVI patches (for ground-truth)
Physics doesn't lie. Sentinel-2 Band 8 (NIR) and Band 4 (Red), 10m resolution, peak monsoon season (August) gives the clearest signal. On Google Earth Engine, this is about 15 lines of Python via the earthengine-api. Threshold NDVI at 0.3, run connectedPixelCount to drop patches smaller than 50 pixels (~500 sqm), then vectorize with reduceToVectors. This gives every significant vegetated patch in the city, regardless of OSM or municipal records.
# Step 4 — Getting the municipal layer (for ground truth)
Municipal park registers for most Indian cities exist in two forms: a PDF list (dig it out, geocode by name), or an official GIS layer (sometimes on Bhuvan, sometimes in the Smart Cities data portal). For cities without a digital layer, an RTI (Right to Information) request to the municipal corporation or a manual survey by any other legal authority can help to list down public parks.
# Step 5 — Spatial union and source counting in Python

Python code:

import geopandas as gpd

osm   = gpd.read_file("osm_parks.gpkg").assign(src_osm=1)</br>
ndvi  = gpd.read_file("ndvi_patches.gpkg").assign(src_ndvi=1)</br>
muni  = gpd.read_file("municipal_parks.gpkg").assign(src_muni=1)</br>

> union all three, tag each polygon with how many sources it appears in

merged = gpd.overlay(osm, ndvi, how='union')</br>
merged = gpd.overlay(merged, muni, how='union')</br>
merged['source_count'] = (
    merged['src_osm'].fillna(0) +
    merged['src_ndvi'].fillna(0) +
    merged['src_muni'].fillna(0)
).astype(int)</br>
confirmed = merged[merged['source_count'] >= 2]</br>
flagged   = merged[merged['source_count'] == 1]</br>
</br>
confirmed goes directly into your analysis dataset. flagged gets a manual imagery audit pass — open it in QGIS over a Google Satellite XYZ tile and classify each polygon as accept/reject in 30 minutes.
# Step 6 — Stratified sample audit to measure your actual error rate
Create a 500m × 500m fishnet grid over the city boundary. Classify each cell by income zone (use census ward-level data as proxy). Randomly sample 5% of cells within each income stratum — this ensures coverage in poor neighbourhoods separately, not letting high-coverage rich areas mask the problem. For each sampled cell, manually count ground-truth parks using high-res imagery. Compare to your dataset. Omission rate is calculated as (missed_parks / total_real_parks). If it's below 5% across all strata, the job is efficiently done..
