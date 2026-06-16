# Building a GIS Data Pipeline — Learning Summary

A reference guide built up step by step. The running example: a public dataset of
**point locations** (e.g. bike-share stations) that we clean, reproject, join to
**neighborhood polygons**, and export as both a **map** and a **queryable layer**.

Stack: **GeoPandas** (table + geometry workhorse), **Shapely** (individual geometries),
**PostGIS** (durable spatial storage + SQL).

---

## The five stages

A pipeline is a one-directional flow; each stage hands a well-defined product to the next.

| Stage | Job | "Done" looks like |
|-------|-----|-------------------|
| **1. Ingest** | Get raw data in the door (file, API, DB) | Data loaded, in some readable form |
| **2. Clean / Transform** | Make it trustworthy and consistent | Valid geometries, one known CRS, no junk rows |
| **3. Store** | Put it somewhere durable and queryable | Data in a spatial DB (or organized files) |
| **4. Analyze** | Answer the actual question | Joined / aggregated results |
| **5. Output** | Deliver to humans or machines | A map, an exported layer, an API |

### Cross-cutting principles
- **Each stage is independently re-runnable.** Tweak cleaning logic without re-downloading.
- **Raw data is sacred** — never overwrite it. `data/raw/` (read-only) → `data/processed/` (derived).
- **The CRS is the thing that bites beginners.** Most pipeline bugs are silent CRS mismatches, not crashes.

---

## Key terms (defined on first use)

- **CRS (Coordinate Reference System)** — how coordinate numbers map to real positions on Earth.
- **EPSG code** — numeric id for a CRS. **EPSG:4326** = lat/lon in degrees (GPS / WGS84). Memorize it.
- **Geographic CRS** — degrees (e.g. 4326); "1 unit" is a different real distance depending on location.
- **Projected CRS** — flattens Earth onto a plane, units in meters; needed for area/distance.
- **UTM** — family of projected CRSs, 60 zones, meters; great default for city/regional work.
- **Point / LineString / Polygon** — geometry types: a location / a path / an area.
- **GeoDataFrame** — pandas DataFrame with a `geometry` column of Shapely objects.
- **Valid geometry** — obeys its type's rules (a polygon ring doesn't self-intersect / "bowtie").
- **Spatial join** — match rows by *geometric relationship* (point-in-polygon), not a shared key/id.
- **Spatial predicate** — the relationship tested: `within`, `intersects`, `contains`, `touches`, `crosses`.
- **PostGIS** — PostgreSQL spatial extension; adds `geometry` type and `ST_*` functions.
- **SRID** — PostGIS's stored CRS id (same number as the EPSG code). Mismatches error loudly.
- **Spatial index (GIST)** — prunes candidates by bounding box before the exact geometry test.
- **Bounding box (envelope)** — min/max x,y rectangle around a geometry; cheap pre-filter.
- **Choropleth** — a map shading each area by a data value.
- **GeoPackage (.gpkg)** — modern single-file spatial format (a SQLite DB); replaces Shapefile.
- **Vector vs raster** — discrete features as geometry vs a grid of cells (imagery, elevation).

---

## Stage 1 — Ingest

A CSV is **not** spatial yet; build geometry explicitly and **declare** the CRS.

```python
import pandas as pd, geopandas as gpd

df = pd.read_csv("data/raw/stations.csv")                 # plain pandas, nothing spatial
points = gpd.GeoDataFrame(
    df,
    geometry=gpd.points_from_xy(df["longitude"], df["latitude"]),  # x=LON first!
    crs="EPSG:4326",                                       # ASSERT source CRS (no move)
)

boundaries = gpd.read_file("data/raw/neighborhoods.geojson")  # self-describing format
```

- `points_from_xy(x, y)` takes **longitude first** — humans say "lat, lon"; geo uses (x, y).
- `crs="EPSG:4326"` **asserts** metadata, it does **not** move points. (CSV has no embedded CRS.)
- Spatial files (GeoJSON/Shapefile/GPKG) carry their own CRS, so `read_file` reads it automatically.

**Always inspect before trusting:** `.shape`, `.crs`, `.head()`, and `.plot()`.
Dots clustered at (0,0) = "Null Island" (missing coords → 0); dots spread but wrong = lon/lat swap.

---

## Stage 2 — Clean / Transform

Order: **(a) drop bad rows → (b) repair geometry → (c) reproject.**

```python
# 2a. filter while values are plain numbers (cheaper, simpler than inspecting geometry)
df = df.dropna(subset=["longitude", "latitude"])
df = df[df["latitude"].between(-90, 90) & df["longitude"].between(-180, 180)]

# 2b. repair invalid polygons (self-intersections etc.)
boundaries["geometry"] = boundaries.make_valid()

# 2c. reproject BOTH to one common projected CRS (meters) because we'll measure area
target = points.estimate_utm_crs()        # picks the right UTM zone, e.g. EPSG:32618
points = points.to_crs(target)
boundaries = boundaries.to_crs(target)
```

- **`.crs =` (declare, moves nothing) vs `.to_crs()` (convert, recomputes coords).** Mixing these up is the classic silent disaster.
- **Topology (inside/touches) is unit-free; measurement (area/distance/density) needs a projected CRS.** We reproject because we measure, not because we join.
- Confirm: `points.crs == boundaries.crs` is `True`, and a plot shows points sitting on polygons.

---

## Stage 3 — Store (PostGIS)

```python
from sqlalchemy import create_engine, text
engine = create_engine("postgresql://gis:gis@localhost:5432/gisdb")
# one-time on the DB: CREATE EXTENSION postgis;

points.to_postgis("stations", engine, if_exists="replace", index=False)
boundaries.to_postgis("neighborhoods", engine, if_exists="replace", index=False)

with engine.connect() as conn:
    conn.execute(text("CREATE INDEX IF NOT EXISTS idx_neigh_geom "
                      "ON neighborhoods USING GIST (geometry);"))
    conn.execute(text("CREATE INDEX IF NOT EXISTS idx_stations_geom "
                      "ON stations USING GIST (geometry);"))
    conn.commit()
```

- `if_exists="replace"` makes the stage **idempotent** — re-run = same clean state. `"append"` would duplicate rows (3 runs → triple counts).
- `to_postgis` stores the CRS as an **SRID**; PostGIS **errors loudly** on mismatched SRIDs — a gift, turning the silent CRS bug into an obvious one.
- **Always add a GIST spatial index** — cheap bbox pre-filter, then exact test on survivors.

---

## Stage 4 — Analyze (spatial join + aggregation)

```python
joined = gpd.sjoin(points, boundaries[["neighborhood_name", "geometry"]],
                   how="left", predicate="within")     # left preserves unmatched points
counts = joined.groupby("neighborhood_name").size().reset_index(name="station_count")

result = boundaries.merge(counts, on="neighborhood_name", how="left")
result["station_count"] = result["station_count"].fillna(0)         # empty hoods → 0
result["area_km2"] = result.geometry.area / 1_000_000               # m² only because UTM
result["density"] = result["station_count"] / result["area_km2"]
```

Same analysis in PostGIS SQL:

```sql
SELECT n.neighborhood_name,
       COUNT(s.geometry) AS station_count,
       ST_Area(n.geometry) / 1000000 AS area_km2,
       COUNT(s.geometry) / (ST_Area(n.geometry) / 1000000) AS density
FROM neighborhoods n
LEFT JOIN stations s ON ST_Within(s.geometry, n.geometry)
GROUP BY n.neighborhood_name, n.geometry;
```

- `how="left"` / `LEFT JOIN` keeps unmatched rows — the null count is a **data-quality signal** (points outside all polygons).
- The meaningful `area_km2` depends on **two stage-2 decisions**: reprojection to UTM (gives meters) **and** `make_valid()` (accurate area).
- **Tradeoff:** GeoPandas for small data / MVP / interactive; PostGIS for large data, durable storage, concurrent SQL querying. *Prototype in GeoPandas, productionize in PostGIS.*

---

## Stage 5 — Output (map + queryable layer)

```python
import matplotlib.pyplot as plt
fig, ax = plt.subplots(figsize=(10, 10))
result.plot(column="density", cmap="viridis", legend=True,         # density, NOT count
            legend_kwds={"label": "Stations per km²"},
            edgecolor="white", linewidth=0.3, ax=ax)
points.plot(ax=ax, color="red", markersize=1, alpha=0.4)
ax.set_title("Station density by neighborhood"); ax.set_axis_off()
fig.savefig("data/processed/density_map.png", dpi=200, bbox_inches="tight")

result.to_file("data/processed/neighborhood_density.gpkg", driver="GPKG")
```

- **Color by density, not count** — count rewards big neighborhoods for being big; density normalizes by area.
- **`viridis`** is perceptually uniform + colorblind-safe; avoid rainbow/`jet`.
- **GeoPackage > Shapefile** — Shapefile is a fragile 4+ file bundle, 10-char column truncation, 2 GB cap.
- Three output forms, three audiences: **PNG** (humans), **GPKG** (GIS tools / handoff), **PostGIS table** (apps / live SQL).

---

## Tying it together — one re-runnable pipeline

```python
def ingest():
    df = pd.read_csv("data/raw/stations.csv")
    boundaries = gpd.read_file("data/raw/neighborhoods.geojson")
    return df, boundaries

def clean(df, boundaries):
    df = df.dropna(subset=["longitude", "latitude"])
    df = df[df["latitude"].between(-90, 90) & df["longitude"].between(-180, 180)]
    points = gpd.GeoDataFrame(df, geometry=gpd.points_from_xy(df.longitude, df.latitude),
                              crs="EPSG:4326")
    boundaries["geometry"] = boundaries.make_valid()
    target = points.estimate_utm_crs()
    return points.to_crs(target), boundaries.to_crs(target)

def store(points, boundaries, engine):
    points.to_postgis("stations", engine, if_exists="replace", index=False)
    boundaries.to_postgis("neighborhoods", engine, if_exists="replace", index=False)

def analyze(points, boundaries):
    joined = gpd.sjoin(points, boundaries[["neighborhood_name", "geometry"]],
                       how="left", predicate="within")
    counts = joined.groupby("neighborhood_name").size().reset_index(name="station_count")
    result = boundaries.merge(counts, on="neighborhood_name", how="left")
    result["station_count"] = result["station_count"].fillna(0)
    result["area_km2"] = result.geometry.area / 1_000_000
    result["density"] = result["station_count"] / result["area_km2"]
    return result

def output(result, points):
    result.to_file("data/processed/neighborhood_density.gpkg", driver="GPKG")
    # ... map code from stage 5 ...

df, boundaries = ingest()
points, boundaries = clean(df, boundaries)
store(points, boundaries, engine)
result = analyze(points, boundaries)
output(result, points)
```

Each function is one stage — testable in isolation, re-runnable without touching the others.

---

## Hand-off contract — what must be right for the next stage

1. **Ingest** → data loaded *with a known/declared CRS*.
2. **Clean** → both layers in the **same projected CRS** + **valid geometry**. *(Highest-leverage hand-off.)*
3. **Store** → **idempotent** + **spatially indexed**.
4. **Analyze** → correct counts (including 0s) + meaningful areas.
5. **Output** → right encoding for the audience (density not count; GeoPackage not Shapefile).

> **The one idea to keep:** the CRS decision in stage 2 is load-bearing for stages 3, 4, and 5.

---

## Carry-anywhere checklist

- **Ingest:** Is it spatial yet? Build geometry, declare CRS. Inspect `.shape/.crs/.head()` and **plot it**.
- **Clean:** Filter bad values as plain numbers. `make_valid()`. **Reproject to one common CRS** (meters if measuring).
- **`.crs =` vs `.to_crs()`:** declare vs convert — don't mix them up.
- **Store:** idempotent writes (`replace`) + **always add a GIST index**.
- **Analyze:** `how="left"` early to *see* unmatched rows. Topology is unit-free; measurement needs meters.
- **Output:** normalize before mapping (density > count). GeoPackage > Shapefile. Raw read-only, derived in `processed/`.
- **Across all:** if a join returns empty/null, **suspect CRS first**.

---

## Where to grow next

1. **Incremental / scheduled updates** — upserts + watermarks instead of replace-all; schedule with cron/Airflow/Prefect.
2. **Raster data** — `rasterio` / `rioxarray`; "zonal statistics" is the raster cousin of the spatial join.
3. **Bigger-than-memory** — `dask-geopandas`, DuckDB spatial extension, or keep it all in PostGIS.
