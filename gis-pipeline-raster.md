# Building a GIS Data Pipeline — Raster Data

A companion to [gis-pipeline-vector.md](gis-pipeline-vector.md), which covered the **vector**
pipeline (points → polygons → spatial join). That pipeline answers *"where are the features?"*.
**Raster** answers *"what's the value at every location?"* — a **grid of cells**, each holding a
number (elevation, temperature, land-cover class, satellite reflectance).

The **same five stages apply**; only the tools and the failure modes change. The running example:
an **elevation raster (DEM)** that we mask, reproject, and reduce to **mean elevation per
neighborhood** — reusing the same neighborhood polygons produced by the vector guide.

Stack: **rasterio** (low-level read/write), **rioxarray** (labeled arrays — the high-level
workhorse, raster's GeoPandas), **rasterstats** (zonal statistics — the raster↔vector bridge).

---

## The five stages (raster)

A pipeline is a one-directional flow; each stage hands a well-defined product to the next.

| Stage | Vector (guide) | Raster equivalent |
|-------|----------------|-------------------|
| **1. Ingest** | `read_csv` + build geometry + declare CRS | `rioxarray.open_rasterio` — GeoTIFF is self-describing |
| **2. Clean / Transform** | dropna, `make_valid()`, `to_crs()` | mask NoData, `reproject` (**resamples!**), align grids |
| **3. Store** | `to_postgis` + GIST index | write a **Cloud-Optimized GeoTIFF** (tiling + overviews) |
| **4. Analyze** | `sjoin` + `groupby` | **zonal statistics** — the spatial-join cousin |
| **5. Output** | PNG + GPKG + PostGIS table | PNG + COG + per-zone stats table |

### Cross-cutting principles (unchanged from the vector guide)
- **Each stage is independently re-runnable.** Tweak cleaning without re-downloading.
- **Raw data is sacred** — `data/raw/` (read-only) → `data/processed/` (derived).
- **The CRS is the thing that bites beginners.** Most bugs are silent CRS/grid mismatches, not crashes.

---

## Key terms (defined on first use)

- **Raster / grid / cell (pixel)** — a regular grid; each cell holds one value per **band**.
- **Band** — one layer of the grid (a DEM has 1 band; RGB imagery has 3; satellite scenes many).
- **Transform (affine)** — the 6 numbers mapping pixel `(row, col)` → world `(x, y)`. Raster's "geometry"; lose it and the grid floats in space.
- **NoData** — sentinel value (e.g. `-9999`, `NaN`) meaning "no measurement here". Raster's `dropna`.
- **Resolution** — real-world size of one cell (e.g. 30 m). A 30 m grid can't answer a 5 m question.
- **Resampling** — inventing new cell values when the grid changes (reproject/resize): `nearest`, `bilinear`, `cubic`.
- **Zonal statistics** — summarize raster cells that fall inside each vector polygon (mean/min/max/sum). The raster cousin of the spatial join.
- **COG (Cloud-Optimized GeoTIFF)** — a GeoTIFF with internal **tiling** + **overviews**; lets a client read one tile without downloading the whole file. Raster's durable, queryable format.
- **Overviews (pyramids)** — pre-computed downsampled copies baked into the file for fast zoomed-out reads.

> A raster is defined by **CRS + transform + NoData** the way a vector layer is defined by **CRS + geometry**.

---

## Stage 1 — Ingest

A GeoTIFF is **self-describing** (like GeoJSON, unlike CSV) — it carries CRS, transform, and NoData.
So ingest is a single read, closer to `read_file` than to building geometry by hand.

```python
import rioxarray

dem = rioxarray.open_rasterio("data/raw/elevation.tif", masked=True)  # masked=True → NoData becomes NaN
```

- `masked=True` reads the file's declared NoData straight into `NaN` — opt in here and stage 2 is half done.
- **Always inspect before trusting:** `dem.rio.crs`, `dem.rio.transform()`, `dem.rio.nodata`, `dem.shape`, and `dem.plot()`.
- **The visual tell:** if NoData isn't masked, the plot shows giant flat blocks of `-9999` that crush the real color range — the raster version of "Null Island".

---

## Stage 2 — Clean / Transform

Order: **(a) mask NoData → (b) reproject (= resample) → (c) align grids.**

```python
# 2a. mask NoData so it can't poison statistics (skip if masked=True already did it)
dem = dem.where(dem != dem.rio.nodata)

# 2b. reproject to the SAME projected CRS as the vector layers (meters).
#     Raster reprojection RESAMPLES — you must pick a method that matches the data type.
from rasterio.enums import Resampling
dem = dem.rio.reproject("EPSG:32618", resampling=Resampling.bilinear)   # continuous → bilinear/cubic
#                                                  Resampling.nearest    # categorical → nearest (never average class IDs!)

# 2c. (only when combining two rasters) snap one onto the other's exact grid
# dem = dem.rio.reproject_match(other_raster)
```

- **Reproject resamples.** Unlike vector `to_crs()` (which just recomputes exact coordinates), raster reprojection must **invent new cell values** on the new grid — hence a resampling choice.
- **Resampling method follows data type:** `bilinear`/`cubic` for continuous (elevation, temperature); `nearest` for categorical (land-cover classes). Averaging class IDs is silent corruption.
- **Grid alignment is stricter than CRS matching.** For raster↔raster math, cells must *coincide*, not just share a CRS — that's what `reproject_match` guarantees.

---

## Stage 3 — Store (COG)

The durable, queryable raster format is the **Cloud-Optimized GeoTIFF** — tiling + overviews are
the raster analogue of a GIST index: structure that lets a reader avoid scanning everything.

```python
dem.rio.to_raster(
    "data/processed/dem_utm.tif",
    driver="COG",                 # tiled + overviews baked in
    compress="deflate",
)
```

- **COG over plain GeoTIFF** — internal tiles + overview pyramids let clients fetch one map tile or a coarse zoom without reading gigabytes.
- Writing to `data/processed/` keeps **raw read-only** — same rule as the vector guide.
- *(PostGIS can store rasters via `postgis_raster`, but COGs on object storage are the more common modern pattern.)*

---

## Stage 4 — Analyze (zonal statistics)

The headline. **Zonal statistics is the structural twin of the spatial join + groupby**: instead of
"count points within each neighborhood," compute "mean/min/max of raster cells within each neighborhood."
This is exactly where **raster meets vector** — reuse the polygons from the vector guide as the zones.

```python
import geopandas as gpd
from rasterstats import zonal_stats

zones = gpd.read_file("data/processed/neighborhood_density.gpkg")  # the vector pipeline's output
zones = zones.to_crs(dem.rio.crs)                                  # SAME CRS — the trap is still the trap

stats = zonal_stats(zones, "data/processed/dem_utm.tif",
                    stats=["mean", "min", "max"],
                    nodata=dem.rio.nodata)                         # exclude NoData from the math

zones["mean_elev"] = [s["mean"] for s in stats]
```

- **The raster↔vector bridge lives here** — the neighborhood polygons are the zones; the two halves of GIS plug together.
- **NoData must be excluded** or every mean is dragged toward `-9999`. (Masking in stage 2 + `nodata=` here are belt-and-suspenders.)
- **If the stats come back all-null, suspect CRS first** — same instinct as an empty spatial join.

---

## Stage 5 — Output

Three forms, three audiences — mirroring the vector guide.

```python
import matplotlib.pyplot as plt

# 1. PNG for humans — choropleth of the zonal result (reuse the vector map machinery)
fig, ax = plt.subplots(figsize=(10, 10))
zones.plot(column="mean_elev", cmap="viridis", legend=True,
           legend_kwds={"label": "Mean elevation (m)"}, ax=ax)
ax.set_title("Mean elevation by neighborhood"); ax.set_axis_off()
fig.savefig("data/processed/elevation_map.png", dpi=200, bbox_inches="tight")

# 2. COG — the cleaned raster, for GIS tools / web tiles (written in stage 3)
# 3. GPKG / PostGIS — the per-zone stats table, for apps / live SQL
zones.to_file("data/processed/neighborhood_elevation.gpkg", driver="GPKG")
```

- **PNG** (humans) · **COG** (GIS tools / tile servers) · **stats table** (apps / SQL) — same trio as the vector output.
- **`viridis`**, perceptually uniform + colorblind-safe, applies to raster renders too; avoid rainbow/`jet`.

---

## Tying it together — one re-runnable pipeline

```python
import rioxarray, geopandas as gpd
from rasterio.enums import Resampling
from rasterstats import zonal_stats

def ingest():
    return rioxarray.open_rasterio("data/raw/elevation.tif", masked=True)

def clean(dem, target_crs="EPSG:32618"):
    dem = dem.where(dem != dem.rio.nodata)
    return dem.rio.reproject(target_crs, resampling=Resampling.bilinear)

def store(dem, path="data/processed/dem_utm.tif"):
    dem.rio.to_raster(path, driver="COG", compress="deflate")
    return path

def analyze(cog_path, dem):
    zones = gpd.read_file("data/processed/neighborhood_density.gpkg").to_crs(dem.rio.crs)
    stats = zonal_stats(zones, cog_path, stats=["mean", "min", "max"], nodata=dem.rio.nodata)
    zones["mean_elev"] = [s["mean"] for s in stats]
    return zones

def output(zones):
    zones.to_file("data/processed/neighborhood_elevation.gpkg", driver="GPKG")
    # ... map code from stage 5 ...

dem = ingest()
dem = clean(dem)
cog = store(dem)
zones = analyze(cog, dem)
output(zones)
```

Each function is one stage — testable in isolation, re-runnable without touching the others.

---

## Hand-off contract — what must be right for the next stage

1. **Ingest** → raster loaded *with CRS, transform, and NoData known*.
2. **Clean** → NoData masked + **same projected CRS as the zones**, resampled with a **type-appropriate method**. *(Highest-leverage hand-off.)*
3. **Store** → COG (tiled + overviews).
4. **Analyze** → zonal stats with NoData excluded; zones in the raster's CRS.
5. **Output** → PNG (humans) + COG (tools) + stats table (apps).

> **The one idea to keep:** raster reprojection *resamples* — the method must match whether the data is continuous or categorical.

---

## Carry-anywhere checklist

- **Ingest:** open with `masked=True`; inspect `.rio.crs / .rio.transform() / .rio.nodata` and **plot it**.
- **Clean:** mask NoData *before any statistic*. Reproject = resample → **`bilinear` continuous, `nearest` categorical**. `reproject_match` to align grids for raster math.
- **Store:** write a **COG** (tiling + overviews) — the raster "index".
- **Analyze:** **zonal statistics** = spatial-join cousin; reuse vector polygons as zones; exclude NoData.
- **Resolution is destiny** — you can't recover detail finer than the cell size.
- **Across all:** CRS still bites — if zonal stats come back empty/null, **suspect CRS first**.

---

## Bigger-than-memory note

Rasters are dense and large, so the "bigger-than-memory" concern bites raster hardest: use
`rioxarray` + `dask` for chunked processing, and lean on **COG overviews** to read coarse versions
cheaply instead of loading full resolution.
