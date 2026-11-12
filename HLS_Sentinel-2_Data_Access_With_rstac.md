# Conversation
- conversationId: ae032dad-0194-52a3-99c9-6f4925ab5a3e
- endpoint: anthropic
- title: HLS Sentinel-2 Data Access With rstac
- exportAt: 13:10:46 GMT+0100 (Central European Standard Time)

## Options
- presetId: null
- model: claude-opus-5.5
- resendFiles: true
- maxContextTokens: 828400
- endpoint: anthropic
- endpointType: undefined
- title: HLS Sentinel-2 Data Access With rstac

## History
**User**
I want to access the Harmonized landsat sentinel 2 collection of the microsoft planetary computer. The collection is "hls2-s30". I want data for the month May 2025. I want to use the rstac package in the programming language R.


**Claude**
<details>
<summary>Thinking</summary>

I'm putting together R code with rstac to query the HLS2 S30 collection on Planetary Computer for May 2025, setting the STAC endpoint and search parameters.

Writing the rstac query code...

Still writing the query code...

Next I'm covering the wrap-up: a note on the roughly hour-long SAS token expiry and re-signing, using gdalcubes for a data cube, decoding the Fmask bits, and the reflectance scale factor and nodata value. I also want to confirm that the public data needs no API key for signing.
</details>


**Claude**
# Accessing HLS2 S30 on the Microsoft Planetary Computer with `rstac`

The workflow is to connect to the STAC API, search the `hls2-s30` collection for May 2025, sign the items, and then read or download the assets.

## 1. Install and load packages

```r
install.packages(c("rstac", "terra"))   # terra is optional, for reading rasters

library(rstac)
library(terra)
```

## 2. Connect and inspect the collection

```r
# Planetary Computer STAC endpoint
pc <- stac("https://planetarycomputer.microsoft.com/api/stac/v1")

# Metadata of the collection
hls_col <- pc |>
  collections("hls2-s30") |>
  get_request()

hls_col
```

## 3. Search for May 2025

HLS data is global. Searching the whole month without a spatial filter would return a very large number of items, so it's best to define an area of interest. The example below uses an area around Munich. Replace it with your own.

```r
# Area of interest: xmin, ymin, xmax, ymax (WGS84 lon/lat)
bbox <- c(11.3, 47.9, 11.8, 48.3)

items <- pc |>
  stac_search(
    collections = "hls2-s30",
    bbox        = bbox,
    datetime    = "2025-05-01T00:00:00Z/2025-05-31T23:59:59Z",
    limit       = 100
  ) |>
  ext_query("eo:cloud_cover" < 50) |>   # optional cloud filter
  post_request() |>
  items_fetch()                          # fetch all pages

items
items_length(items)
```

You can remove the `ext_query()` line if you want all scenes regardless of cloud cover.

## 4. Sign the items

Asset URLs on the Planetary Computer must be signed with a SAS token before you can access them. No API key is required. The tokens expire after a while, so re-run this step if access fails later.

```r
items <- items_sign(items, sign_fn = sign_planetary_computer())
```

## 5. Explore the results

```r
# Available asset names (bands)
items_assets(items)

# Acquisition dates
items_datetime(items)

# Cloud cover of each scene
items_reap(items, field = c("properties", "eo:cloud_cover"))

# Optional: footprints as an sf object
# items_sf <- items_as_sf(items)
```

The S30 assets typically include:

- **Spectral bands:** `B01`–`B12` and `B8A`
- **Quality mask:** `Fmask`
- **Angle bands:** `SAA`, `SZA`, `VAA`, `VZA`

Use `items_assets()` to confirm the exact names.

## 6a. Read data directly (without downloading)

```r
# Signed URLs with the /vsicurl/ prefix for GDAL
red_urls <- assets_url(items, asset_names = "B04", append_gdalvsi = TRUE)
nir_urls <- assets_url(items, asset_names = "B08", append_gdalvsi = TRUE)

# Read the first scene
red <- rast(red_urls[1])
nir <- rast(nir_urls[1])

# Example: NDVI (HLS reflectance is scaled by 0.0001)
ndvi <- (nir - red) / (nir + red)
plot(ndvi, main = "NDVI - HLS S30, May 2025")
```

## 6b. Or download the files

```r
dir.create("hls_may2025", showWarnings = FALSE)

assets_download(
  items,
  asset_names = c("B02", "B03", "B04", "B08", "Fmask"),
  output_dir  = "hls_may2025",
  overwrite   = TRUE
)
```

## Tips

- **Limit the search area.** HLS tiles are 109.8 km MGRS tiles, and global queries for a full month can return many thousands of items.
- **Mask clouds.** Use the `Fmask` band, which is bit-encoded (cloud, cloud shadow, snow, water, etc.).
- **Build data cubes.** For time series or mosaicking multiple tiles, the `gdalcubes` package works well with rstac item collections via `stac_image_collection()`.
- **Handle expired tokens.** If you get HTTP 403 errors, sign the items again with `items_sign()`.


**User**
How can I now use gdalcubes to access only the specified aoi (not the whole tile)


**Claude**
<details>
<summary>Thinking</summary>

I'm planning the gdalcubes workflow for the AOI crop: loading the STAC items as an image collection with the needed asset names, setting up a cube view with the extent in a projected CRS like UTM, then selecting bands, masking clouds with Fmask, computing NDVI, reducing over time, and writing out or plotting the result. I'm also working out how to pass the signed URLs through the collection call.

Next I'm checking how the signed URLs get read, since gdalcubes adds the /vsicurl/ prefix on its own and the tokens will need refreshing. For the AOI I'm reprojecting the bounding box to UTM with sf, and I'm working through how to express the Fmask bit values for `image_mask`.

I'm trying to pin down how the bit-encoded Fmask values work with `image_mask`, since gdalcubes may not support bitwise operations in `apply_pixel` expressions. I'm considering using `image_mask` with the `bits` and `values` arguments instead, but I'm not sure how it compares the extracted bits.

I'm checking how the bit values in `image_mask` are compared, since the bits I select get ANDed with the band value before matching. For Fmask that would mean selecting cloud and shadow bits and matching against the resulting values, but I'm not sure yet whether the match is on the extracted bits or the full value.

Settling on invert=TRUE with values=0 over bits 1 to 3 seems more robust, since zero stays zero whether or not the AND result gets shifted. Adding the snow bit is optional, and I still need to pick the time aggregation (P1M or P5D) and set gdalcubes_options(parallel=4) plus the GDAL vsicurl options.

Now I'm checking the GDAL settings for reading the remote files, and noting that HLS reflectance uses a 0.0001 scale while the NDVI ratio doesn't depend on it. I'm also thinking about the nodata values and the fact that gdalcubes only pulls the blocks it needs from the COGs, so only the AOI gets read.

Next I'm setting the date range to May 2025 and considering an optional property filter. I'll also derive the time extent from the items' timeline and then write the code.
</details>


**Claude**
# Cropping HLS S30 to Your AOI with `gdalcubes`

gdalcubes reads only the pixels that fall inside the **cube view** you define. HLS files are Cloud-Optimized GeoTIFFs, so only the parts of each tile that overlap your AOI are fetched over the network, not the whole 110 × 110 km tile.

The key is to define the cube view's spatial extent from your AOI, in a projected CRS (UTM).

## 1. Packages and settings

```r
library(rstac)
library(gdalcubes)
library(sf)

# Number of parallel worker processes
gdalcubes_options(parallel = 4)

# GDAL settings that speed up remote COG access
gdalcubes_set_gdal_config("GDAL_DISABLE_READDIR_ON_OPEN", "EMPTY_DIR")
gdalcubes_set_gdal_config("CPL_VSIL_CURL_ALLOWED_EXTENSIONS", ".tif")
gdalcubes_set_gdal_config("GDAL_HTTP_MAX_RETRY", "5")
gdalcubes_set_gdal_config("GDAL_HTTP_RETRY_DELAY", "1")
```

## 2. Search and sign (as before)

```r
bbox <- c(xmin = 11.3, ymin = 47.9, xmax = 11.8, ymax = 48.3)

items <- stac("https://planetarycomputer.microsoft.com/api/stac/v1") |>
  stac_search(
    collections = "hls2-s30",
    bbox        = bbox,
    datetime    = "2025-05-01T00:00:00Z/2025-05-31T23:59:59Z",
    limit       = 100
  ) |>
  ext_query("eo:cloud_cover" < 50) |>
  post_request() |>
  items_fetch() |>
  items_sign(sign_fn = sign_planetary_computer())
```

## 3. Create the image collection

```r
assets <- c("B02", "B03", "B04", "B08", "Fmask")

col <- stac_image_collection(
  items$features,
  asset_names = assets
)

col
```

`stac_image_collection()` automatically adds the `/vsicurl/` prefix to the signed URLs.

## 4. Transform the AOI to UTM

HLS tiles are in UTM. Munich lies in UTM zone 32N (EPSG:32632). Choose the zone that matches your AOI.

```r
aoi_wgs84 <- st_as_sfc(st_bbox(bbox, crs = 4326))
aoi_utm   <- st_transform(aoi_wgs84, 32632)
bb        <- st_bbox(aoi_utm)
bb
```

## 5. Define the cube view

This is the step that restricts processing to the AOI.

```r
v <- cube_view(
  srs        = "EPSG:32632",
  extent     = list(
    left   = bb["xmin"], right = bb["xmax"],
    bottom = bb["ymin"], top   = bb["ymax"],
    t0     = "2025-05-01", t1  = "2025-05-31"
  ),
  dx         = 30, dy = 30,   # HLS native resolution
  dt         = "P1M",         # one time step for the whole month
  aggregation = "median",     # combine multiple scenes per time step
  resampling  = "bilinear"
)

v
```

You can adjust the temporal settings to your needs:

- `dt = "P1D"` with `aggregation = "first"` gives (almost) individual scenes.
- `dt = "P5D"` or `"P1M"` gives composites.

## 6. Cloud mask from `Fmask`

`Fmask` is bit-encoded. The relevant bits are:

| Bit | Meaning |
|-----|---------|
| 1 | cloud |
| 2 | adjacent to cloud/shadow |
| 3 | cloud shadow |
| 4 | snow/ice |

The mask below keeps only pixels where all of these bits are 0:

```r
cloud_mask <- image_mask("Fmask", bits = c(1, 2, 3, 4), values = 0, invert = TRUE)
```

With `invert = TRUE`, every pixel whose selected bits are *not* 0 is masked.

## 7. Build the cube and compute something

Example: a cloud-free median NDVI composite for May 2025 over the AOI.

```r
ndvi_cube <- raster_cube(col, v, mask = cloud_mask) |>
  select_bands(c("B04", "B08")) |>
  apply_pixel("(B08 - B04) / (B08 + B04)", names = "NDVI")

plot(ndvi_cube, zlim = c(-0.2, 1), key.pos = 1,
     col = hcl.colors(100, "Greens", rev = TRUE))
```

NDVI is a ratio, so the 0.0001 reflectance scale factor cancels out. For true reflectance values, multiply the bands by 0.0001.

An RGB composite works the same way:

```r
raster_cube(col, v, mask = cloud_mask) |>
  select_bands(c("B02", "B03", "B04")) |>
  plot(rgb = 3:1, zlim = c(0, 1500))
```

## 8. Export the result

```r
# GeoTIFF (one file per time step)
write_tif(ndvi_cube, dir = "hls_ndvi", prefix = "ndvi_")

# Or NetCDF
write_ncdf(ndvi_cube, "hls_ndvi_may2025.nc")

# Continue in terra
library(terra)
ndvi_rast <- rast(list.files("hls_ndvi", full.names = TRUE))
```

## Daily time series instead of a composite

```r
v_daily <- cube_view(view = v, dt = "P1D", aggregation = "first")

ts_cube <- raster_cube(col, v_daily, mask = cloud_mask) |>
  select_bands(c("B04", "B08")) |>
  apply_pixel("(B08 - B04) / (B08 + B04)", names = "NDVI")

plot(ts_cube, zlim = c(-0.2, 1))
```

Days without an acquisition will be empty (NA).

## Notes

- **Choose the right UTM zone.** If your AOI spans several UTM zones, pick one; gdalcubes reprojects all tiles on the fly.
- **HTTP 403 errors** mean the SAS token has expired. Re-run `items_sign()` and recreate the image collection.
- **Speed.** The first run can be slow because of remote reads. Writing the cube to disk once (`write_ncdf`) and working from that file speeds up further analysis.


**User**
I want to download the raw data without the computation. If I define the cube with dt = "P1D" I get the daily data. However I get a lot of empty rasters for the days where there are no images. How can I only download the days where I have data in the cube?


**Claude**
<details>
<summary>Thinking</summary>

I'm weighing two ways to pull raw data only for the days that have acquisitions: looping over the unique acquisition dates to build a one-day cube for each, or trying an irregular time dimension in gdalcubes.

I'm now checking whether gdalcubes has a stack_cube option that handles irregular time, but I'm not sure it does, so the loop over dates stays the safer route. I'm also planning to keep the raw values by using "first" aggregation and "near" resampling, since the bit-coded Fmask has to stay near.

I'm noting that the pack argument can set the output to int16 with nodata -9999, which fits both the HLS bands and the 0 to 255 range of Fmask. I'm also planning to check each day's output afterward for all-NA files at tile or swath edges and remove them, with a helper for that.

I'm moving on to build the date list from the item datetimes, taking the day part and keeping each day once, and then I'll set up an output folder and loop over those dates to write one cube per day.

Next I'm writing the cube view and the write step for each date, with the spatial extent set to the bounding box and the time extent set to that single day.

I'm writing the output settings as int16 with nodata -9999, then setting up a loop that removes empty daily files by checking each raster for any valid cells with terra's global function.

I'm also weighing whether to check that the dates come only from scenes that passed the cloud filter, and whether the daily boundary in UTC is fine. Two items from adjacent tiles on the same day would get mosaicked with the first non-NA value, which is what I want.
</details>


**Claude**
# Downloading Only the Days with Acquisitions

gdalcubes always builds a **regular** time axis. With `dt = "P1D"`, every day between `t0` and `t1` becomes a time slice, including days without images. The solution is to **get the actual acquisition dates from the STAC items** and create **one single-day cube per date**.

Empty days cost almost nothing to read, because gdalcubes doesn't download anything when no image overlaps. The real problem is only the empty output files, and this approach avoids them.

## 1. Setup (as before)

```r
library(rstac)
library(gdalcubes)
library(sf)

gdalcubes_options(parallel = 4)
gdalcubes_set_gdal_config("GDAL_DISABLE_READDIR_ON_OPEN", "EMPTY_DIR")
gdalcubes_set_gdal_config("CPL_VSIL_CURL_ALLOWED_EXTENSIONS", ".tif")
gdalcubes_set_gdal_config("GDAL_HTTP_MAX_RETRY", "5")
gdalcubes_set_gdal_config("GDAL_HTTP_RETRY_DELAY", "1")

bbox <- c(xmin = 11.3, ymin = 47.9, xmax = 11.8, ymax = 48.3)

items <- stac("https://planetarycomputer.microsoft.com/api/stac/v1") |>
  stac_search(
    collections = "hls2-s30",
    bbox        = bbox,
    datetime    = "2025-05-01T00:00:00Z/2025-05-31T23:59:59Z",
    limit       = 100
  ) |>
  ext_query("eo:cloud_cover" < 50) |>
  post_request() |>
  items_fetch() |>
  items_sign(sign_fn = sign_planetary_computer())

assets <- c("B02", "B03", "B04", "B08", "B8A", "B11", "B12", "Fmask")
col <- stac_image_collection(items$features, asset_names = assets)

# AOI in UTM
bb <- st_bbox(st_transform(st_as_sfc(st_bbox(bbox, crs = 4326)), 32632))
```

## 2. Get the unique acquisition dates

```r
acq_dates <- items_datetime(items) |>
  substr(1, 10) |>
  unique() |>
  sort()

acq_dates
```

Several items can share the same day, for example when your AOI overlaps two MGRS tiles. These are mosaicked into one raster for that day.

## 3. Loop over the dates and write one file per day

To keep the data raw, the cube uses these settings:

- **`resampling = "near"`** keeps the original values. This is essential for the bit-encoded `Fmask` band.
- **`aggregation = "first"`** means no averaging. Overlapping tiles from the same day are simply mosaicked.
- **No mask and no `apply_pixel()`**, so no computation is applied.
- **`pack`** keeps the original integer data type (HLS uses int16 with nodata −9999) instead of writing doubles. This also keeps file sizes small.

```r
out_dir <- "hls_s30_raw_may2025"
dir.create(out_dir, showWarnings = FALSE)

pack <- list(type = "int16", scale = 1, offset = 0, nodata = -9999)

for (d in acq_dates) {
  message("Processing ", d)

  v_day <- cube_view(
    srs    = "EPSG:32632",
    extent = list(
      left   = unname(bb["xmin"]), right = unname(bb["xmax"]),
      bottom = unname(bb["ymin"]), top   = unname(bb["ymax"]),
      t0 = d, t1 = d
    ),
    dx = 30, dy = 30,
    dt = "P1D",
    aggregation = "first",
    resampling  = "near"
  )

  raster_cube(col, v_day) |>
    write_tif(dir = out_dir, prefix = "HLS_S30_", pack = pack, COG = TRUE)
}

list.files(out_dir)
# e.g. "HLS_S30_2025-05-03.tif" "HLS_S30_2025-05-08.tif" ...
```

Each file is a multi-band GeoTIFF clipped to your AOI, containing all selected assets.

## 4. Optional: remove days that are empty within the AOI

A tile's footprint can intersect your bbox while the AOI itself falls outside the satellite swath, giving an all-NA raster. You can remove such files afterwards:

```r
library(terra)

for (f in list.files(out_dir, pattern = "\\.tif$", full.names = TRUE)) {
  r <- rast(f)
  if (global(r[[1]], "notNA")$notNA == 0) {
    message("Removing empty file: ", basename(f))
    file.remove(f)
  }
}
```

## 5. Read the data back

```r
files <- list.files(out_dir, pattern = "\\.tif$", full.names = TRUE)
r <- rast(files[1])
names(r)                 # band names
plotRGB(r, r = "B04", g = "B03", b = "B02", stretch = "lin")
```

To scale to surface reflectance, multiply the spectral bands by 0.0001. `Fmask` stays as is.

## Notes

- **Expired tokens:** For long loops, the SAS tokens may expire, which shows up as HTTP 403 errors. If that happens, re-run `items_sign()` and `stac_image_collection()`, then continue the loop with the remaining dates.
- **Fully raw, unmosaicked scenes:** If you want each scene as its own file without any mosaicking, loop over the items instead of the dates. For each item, crop its assets with GDAL directly, e.g. `sf::gdal_utils("warp", source = url, destination = out, options = c("-te", ..., "-t_srs", "EPSG:32632"))`.


**User**
Can I also do it with the slice_time function from gdalcubes?


**Claude**
<details>
<summary>Thinking</summary>

I'm checking how slice_time works in gdalcubes: it pulls out one time step from a cube, so the plan is to build a daily cube for the month and slice out each acquisition date before writing it to a tif. Since the slicing is lazy, only the chosen slice gets read. I'm also looking at whether write_tif handles a cube that has just one time step.

I'm checking that the dates fall within the cube and line up with the dt grid. For finding acquisition dates, I'm weighing whether the image collection itself can list them, but the STAC item datetimes look like the simpler route. I also want to confirm slice_time accepts a date string like "2025-05-03", and that write_tif already produces one file per time slice. select_time(cube, t) might be even better, since it keeps only the chosen irregular time slices so one write_tif call covers them all.

I'm recalling that select_time() exists in gdalcubes 0.6.0 for picking out irregular time slices, so I'll mention it as an option, with a note about the version. The output filenames will carry the date label, and the AOI-empty check still applies.
</details>


**Claude**
# Using `slice_time()` Instead of Building One Cube per Day

Yes. Instead of creating a new cube view for every date, you define **one daily cube for the whole month** and extract only the days with acquisitions using `slice_time()`. Because gdalcubes evaluates lazily, only the selected slice is read and written. The empty days are never processed.

## 1. Setup (unchanged)

```r
library(rstac)
library(gdalcubes)
library(sf)

gdalcubes_options(parallel = 4)
gdalcubes_set_gdal_config("GDAL_DISABLE_READDIR_ON_OPEN", "EMPTY_DIR")
gdalcubes_set_gdal_config("CPL_VSIL_CURL_ALLOWED_EXTENSIONS", ".tif")

bbox <- c(xmin = 11.3, ymin = 47.9, xmax = 11.8, ymax = 48.3)

items <- stac("https://planetarycomputer.microsoft.com/api/stac/v1") |>
  stac_search(
    collections = "hls2-s30",
    bbox        = bbox,
    datetime    = "2025-05-01T00:00:00Z/2025-05-31T23:59:59Z",
    limit       = 100
  ) |>
  ext_query("eo:cloud_cover" < 50) |>
  post_request() |>
  items_fetch() |>
  items_sign(sign_fn = sign_planetary_computer())

assets <- c("B02", "B03", "B04", "B08", "B8A", "B11", "B12", "Fmask")
col <- stac_image_collection(items$features, asset_names = assets)

bb <- st_bbox(st_transform(st_as_sfc(st_bbox(bbox, crs = 4326)), 32632))

# Unique acquisition dates from the STAC metadata
acq_dates <- sort(unique(substr(items_datetime(items), 1, 10)))
```

## 2. Define one daily cube for the whole month

```r
v <- cube_view(
  srs    = "EPSG:32632",
  extent = list(
    left   = unname(bb["xmin"]), right = unname(bb["xmax"]),
    bottom = unname(bb["ymin"]), top   = unname(bb["ymax"]),
    t0 = "2025-05-01", t1 = "2025-05-31"
  ),
  dx = 30, dy = 30,
  dt = "P1D",
  aggregation = "first",
  resampling  = "near"
)

cube <- raster_cube(col, v)
cube   # 31 time steps, but nothing has been read yet
```

## 3. Slice and write only the acquisition dates

```r
out_dir <- "hls_s30_raw_may2025"
dir.create(out_dir, showWarnings = FALSE)

pack <- list(type = "int16", scale = 1, offset = 0, nodata = -9999)

for (d in acq_dates) {
  message("Writing ", d)
  cube |>
    slice_time(datetime = d) |>
    write_tif(dir = out_dir, prefix = "HLS_S30_", pack = pack, COG = TRUE)
}

list.files(out_dir)
```

`slice_time()` accepts either a date (`datetime = "2025-05-03"`) or a time index (`it = 3`). The date must fall on the cube's time grid. With `t0 = "2025-05-01"` and `dt = "P1D"`, every calendar day in May is valid.

## Alternative: `select_time()` (one call, no loop)

Recent gdalcubes versions (≥ 0.6) also provide `select_time()`. It keeps several, possibly irregular, time slices in one cube, so a single `write_tif()` call writes only those days:

```r
cube |>
  select_time(acq_dates) |>
  write_tif(dir = out_dir, prefix = "HLS_S30_", pack = pack, COG = TRUE)
```

Check `packageVersion("gdalcubes")` if the function isn't found, and update the package if necessary.

## Comparison

| Approach | What happens |
|---|---|
| New `cube_view()` per date | Works in any version, but the view is redefined every iteration |
| `slice_time()` in a loop | One cube definition, one file per call |
| `select_time()` | One cube definition, one `write_tif()` call (gdalcubes ≥ 0.6) |

All three produce the same output files. The choice is mainly about code style.

## Notes

- **Partially empty days:** A tile footprint can intersect your bbox while your AOI lies outside the actual data swath. The resulting file is then all NA. The `terra`-based cleanup step from before still applies.
- **Expired tokens:** For long runs, re-sign the items (`items_sign()`) and recreate `col` and `cube` if you get HTTP 403 errors.


**User**
Cool. I like the select_time solution. But when I do that, the output files have no crs information. Why is that?


**Claude**
<details>
<summary>Thinking</summary>

I'm trying to recall whether gdalcubes has a known problem where select_time gives an irregular time dimension and then write_tif loses the CRS. I'm not certain about that, so I'm going back to the select_time_cube implementation to see how it builds the new labeled-time cube.

I'm weighing whether the CRS loss comes from how select_time builds its labeled-time reference, specifically whether the srs gets copied over from the regular one. I can't confirm a known issue, so the practical options are updating the package, setting the CRS on the output with terra after writing, or using a slice_time loop instead.

I'm sketching a terra-based fix: read the file, set the CRS to EPSG:32632, and write it to a temp file, since I can't overwrite the source in place. I'll be upfront that I'm not sure of the exact cause, so this is a workaround.
</details>


**Claude**
# Why `select_time()` Loses the CRS

This is most likely a **limitation or bug in gdalcubes**, not something in your code.

`select_time()` turns the cube's regular time axis into an **irregular (labeled) time axis**. Internally, gdalcubes then uses a different type of spatiotemporal reference object for the cube. In some versions, the spatial reference (`srs`) is apparently not carried over to this new object correctly. `write_tif()` takes the CRS from that reference, so the GeoTIFFs end up with correct pixel geometry (extent and resolution) but no CRS.

`slice_time()` keeps a regular time axis, so it is not affected. That is why the loop version writes the CRS correctly.

I can't say for certain in which versions this occurs. You can check it directly.

## 1. Diagnose

```r
packageVersion("gdalcubes")

cube_sel <- select_time(cube, acq_dates)

srs(cube)       # should print EPSG:32632
srs(cube_sel)   # empty or different? Then the CRS is lost at this step
```

You can also check the written files:

```r
library(terra)
f <- list.files(out_dir, pattern = "\\.tif$", full.names = TRUE)
crs(rast(f[1]), describe = TRUE)
```

## 2. Solutions

### Option A: Update gdalcubes

If you're not on the latest version, update first. The issue may already be fixed.

```r
install.packages("gdalcubes")
# or the development version:
# remotes::install_github("appelmar/gdalcubes")
```

If it persists in the newest version, consider reporting it at <https://github.com/appelmar/gdalcubes/issues>, together with the output of `srs(cube_sel)`.

### Option B: Assign the CRS after writing

The geometry in the files is correct, so you only need to attach the CRS. This uses `terra`:

```r
library(terra)

files <- list.files(out_dir, pattern = "\\.tif$", full.names = TRUE)

for (f in files) {
  r <- rast(f)
  if (crs(r) == "") {
    crs(r) <- "EPSG:32632"
    tmp <- tempfile(fileext = ".tif")
    writeRaster(r, tmp, datatype = "INT2S", NAflag = -9999,
                filetype = "COG", overwrite = TRUE)
    file.copy(tmp, f, overwrite = TRUE)
    file.remove(tmp)
  }
}

crs(rast(files[1]), describe = TRUE)   # now EPSG:32632
```

The result is written to a temporary file first because terra cannot overwrite the file it is currently reading from. `INT2S` with `NAflag = -9999` preserves the original HLS data type and nodata value.

### Option C: Use the `slice_time()` loop

The loop from the previous answer produces the same files with the CRS set correctly. It's a few lines more but avoids the post-processing step:

```r
for (d in acq_dates) {
  cube |>
    slice_time(datetime = d) |>
    write_tif(dir = out_dir, prefix = "HLS_S30_", pack = pack, COG = TRUE)
}
```

## Recommendation

If updating doesn't fix it, **Option C** is the cleanest choice because it produces correct files directly. If you prefer the one-call `select_time()` syntax, combine it with **Option B**.


