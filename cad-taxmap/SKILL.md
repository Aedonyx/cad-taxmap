---
name: cad-taxmap
description: Download GIS parcel and tax map data from ArcGIS REST endpoints and convert to CAD-ready DXF files with proper coordinate projection. Use this skill whenever the user wants to get GIS data, tax map parcels, or property lines into AutoCAD or Civil 3D, download parcels or features from a county/municipal GIS portal, convert GeoJSON or shapefiles to DXF, or mentions ArcGIS REST services, parcel boundaries, property lines, tax maps, or zoning layers in a CAD context. Also trigger when the user references any ArcGIS MapServer or FeatureServer URL, asks about importing GIS data into a drawing, or mentions "CAD TaxMap".
---

# CAD TaxMap

This skill downloads vector features from ArcGIS REST endpoints via paginated queries and exports closed LWPOLYLINEs in DXF format for direct use in AutoCAD/Civil 3D. It includes endpoint discovery, runtime validation, and clear error reporting.

## Requirements

- Python 3 with `urllib`, `json`, `subprocess` (stdlib)
- `ezdxf` (pip install)
- `gdal-bin` / `ogr2ogr` (required for datum spot-check; also needed for shapefile output)

## Workflow

### Step 0: Endpoint discovery

The user will often provide a GIS viewer URL, not a REST endpoint. Your job is to find the underlying ArcGIS REST service. This is the most critical step — without the right endpoint, nothing else works.

#### 0a. If the user provides a direct REST endpoint

URLs containing `/arcgis/rest/services/` or `/server/rest/services/` with a `/MapServer/` or `/FeatureServer/` path are direct endpoints. Proceed to Step 1.

#### 0b. If the user provides a viewer/app URL

The user will give something like:
- `https://gisapps.countyname.gov/taxmapviewer/`
- `https://gis.countyname.gov/portal/apps/webappbuilder/...`
- `https://countyname.maps.arcgis.com/apps/...`

To find the REST endpoint:

1. **Web search first**: Search for `{county name} {state} ArcGIS REST services parcels MapServer` — this often finds the endpoint directly from search results without needing to scrape the viewer.

2. **Fetch the viewer page**: If search doesn't work, fetch the viewer URL and look for REST service URLs in the HTML/JS source. Search for patterns like:
   - `MapServer` or `FeatureServer` in URLs
   - `rest/services` in URLs
   - JSON config files referenced in the page (e.g., `config.json`, `appconfig.js`)

3. **Try common URL patterns**: Many counties use predictable structures:
   ```
   https://gis.{county}.gov/arcgis/rest/services/
   https://gis.{county}.gov/server/rest/services/
   https://gisapps.{county}.gov/arcgis/rest/services/
   ```
   Fetch the services root with `?f=json` to list available services.

4. **Find the parcel layer**: Once you find a MapServer, fetch `{MapServer_URL}?f=json` to list all layers. Look for layers named: "Tax Parcels", "Parcels", "Tax Map", "Property", "Lots", or similar. The layer with `geometryType: esriGeometryPolygon` and fields like PARCELID, SBL, PIN, MUNICIPALITY is the one you want.

#### 0c. If it's NOT an ArcGIS service

If you cannot find an ArcGIS REST endpoint, the site may use a different technology. Check for:

- **WFS (Web Feature Service)**: Look for `?service=WFS` or `geoserver/wfs` in URLs. If found, report to the user:
  ```
  This GIS portal uses WFS (Web Feature Service), not ArcGIS REST.
  WFS endpoints can still provide parcel data but require a different approach.
  The endpoint appears to be: {url}
  ```
  For WFS, you can attempt: fetch capabilities with `?service=WFS&request=GetCapabilities`, identify the parcel layer, then download with `?service=WFS&request=GetFeature&typeName={layer}&outputFormat=application/json`. Convert the resulting GeoJSON to DXF using the same ezdxf approach, but note the datum spot-check is especially important since WFS typically returns WGS84.

- **GeoServer**: Look for `/geoserver/` in URLs. GeoServer typically exposes WFS — use the WFS approach above.

- **Custom/proprietary API**: If no standard service is found, report to the user:
  ```
  This GIS portal does not appear to use ArcGIS REST or WFS.
  It may use a proprietary API that this skill cannot access.
  Alternative: Check https://gis.ny.gov/parcels for statewide parcel data
  available as downloads or via NYS GIS services.
  ```

- **NYS Statewide Parcels fallback**: For any NY county, statewide parcel data is available at:
  `https://gisservices.its.ny.gov/arcgis/rest/services/`
  This is an ArcGIS REST endpoint and works with this skill directly.

### Step 1: Identify the target CRS

Auto-detect from the service metadata's `latestWkid` or `wkid` field and use that directly. Only ask the user if the metadata doesn't include spatial reference info.

Common CRS values:
- NY State Plane East (US ft): EPSG:2260
- NY State Plane Long Island (US ft): EPSG:2263 (or WKID 6539 for NAD83(2011) variant — Suffolk County uses this)
- NY State Plane Central (US ft): EPSG:2261
- NY State Plane West (US ft): EPSG:2262
- CT State Plane (US ft): EPSG:2234
- NJ State Plane (US ft): EPSG:2263
- MA State Plane Mainland (US ft): EPSG:2249
- MA State Plane Island (US ft): EPSG:2250

Note: `outSR` accepts both EPSG codes and ArcGIS WKIDs. When the native CRS uses a WKID (e.g., 6539), use that same WKID as the `outSR` to avoid any server-side reprojection.

### Step 2: Validate the endpoint and CRS before downloading

Before downloading all features, run these pre-flight checks:

#### 2a. Fetch service metadata and confirm the layer exists

```python
import json, urllib.request

# Strip /query from the URL to get the layer root
layer_url = base_url.replace("/query", "")
meta_url = f"{layer_url}?f=json"
req = urllib.request.Request(meta_url, headers={"User-Agent": "Mozilla/5.0"})
try:
    with urllib.request.urlopen(req, timeout=30) as resp:
        meta = json.loads(resp.read().decode())
except urllib.error.HTTPError as e:
    if e.code == 403:
        print("ERROR: 403 Forbidden — this service requires authentication.")
        print("The server may need a token or login credentials.")
        print("Ask the user if they have access credentials for this GIS portal.")
        # STOP HERE — do not proceed without auth
    elif e.code == 401:
        print("ERROR: 401 Unauthorized — this service requires authentication.")
        print("Ask the user for their ArcGIS token or portal credentials.")
        # STOP HERE
    else:
        raise

# Check geometry type
geom_type = meta.get("geometryType", "unknown")
print(f"Geometry type: {geom_type}")

# Check native spatial reference
native_sr = meta.get("extent", {}).get("spatialReference", {})
native_wkid = native_sr.get("latestWkid") or native_sr.get("wkid")
print(f"Native CRS WKID: {native_wkid}")

# --- WEB MERCATOR DETECTION ---
# WKID 3857 and 102100 are Web Mercator (meters, distorted at high latitudes).
# ArcGIS Online and statewide services commonly use this. It is NEVER appropriate
# for CAD work. If detected, look up the correct state plane zone.
WEB_MERCATOR_WKIDS = {3857, 102100, 102113}

if native_wkid in WEB_MERCATOR_WKIDS:
    print(f"WARNING: Native CRS is Web Mercator (WKID {native_wkid}).")
    print(f"         Web Mercator is NOT suitable for CAD — coordinates are in")
    print(f"         meters with significant distance distortion at NY latitudes.")
    print(f"         Looking up correct state plane zone...")

    # NY State Plane zone lookup by county
    # Central zone counties (EPSG:2261):
    NY_CENTRAL = {"Broome","Cayuga","Chemung","Chenango","Cortland","Herkimer",
        "Jefferson","Lewis","Madison","Oneida","Onondaga","Oswego","Schuyler",
        "Seneca","Steuben","Tioga","Tompkins"}
    # West zone counties (EPSG:2262):
    NY_WEST = {"Allegany","Cattaraugus","Chautauqua","Erie","Genesee","Livingston",
        "Monroe","Niagara","Ontario","Orleans","Wayne","Wyoming","Yates"}
    # East zone counties (EPSG:2260) — everything else in NY
    # Long Island (EPSG:2263 or WKID 6539):
    NY_LONG_ISLAND = {"Suffolk","Nassau","Queens","Kings","Richmond","New York","Bronx"}

    # Try to determine county from the where clause or layer fields
    # If we have a COUNTY_NAME in the where clause, extract it
    import re
    county_match = re.search(r"COUNTY_NAME='([^']+)'", where_clause) if 'where_clause' in dir() else None
    if county_match:
        county = county_match.group(1)
        if county in NY_CENTRAL:
            target_epsg = 2261
            print(f"         {county} County → NY State Plane Central (EPSG:2261)")
        elif county in NY_WEST:
            target_epsg = 2262
            print(f"         {county} County → NY State Plane West (EPSG:2262)")
        elif county in NY_LONG_ISLAND:
            target_epsg = 2263
            print(f"         {county} County → NY State Plane Long Island (EPSG:2263)")
        else:
            target_epsg = 2260
            print(f"         {county} County → NY State Plane East (EPSG:2260)")
    else:
        # Can't auto-detect — ask the user
        print("         Could not auto-detect county from query filter.")
        print("         Ask the user which state plane zone to use:")
        print("           EPSG:2260 — NY East (Hudson Valley, Capital District)")
        print("           EPSG:2261 — NY Central (Syracuse, Cortland, Finger Lakes)")
        print("           EPSG:2262 — NY West (Buffalo, Rochester)")
        print("           EPSG:2263 — NY Long Island (Nassau, Suffolk, NYC)")
        # Default to East as a fallback, but flag it
        target_epsg = 2260
        print(f"         Defaulting to EPSG:2260 (East) — CONFIRM WITH USER")
else:
    # Native CRS is not Web Mercator — use it directly
    target_epsg = native_wkid
```

#### 2b. Get the expected total feature count

```python
count_url = base_url + "?" + urllib.parse.urlencode({
    "where": where_clause,  # e.g., "1=1" or "MUNICIPALITY='Patchogue'"
    "returnCountOnly": "true",
    "f": "json"
})
req = urllib.request.Request(count_url, headers={"User-Agent": "Mozilla/5.0"})
with urllib.request.urlopen(req, timeout=30) as resp:
    count_data = json.loads(resp.read().decode())
expected_count = count_data.get("count", 0)
print(f"Expected feature count: {expected_count}")
```

### Step 3: Paginate and download in native CRS

**CRITICAL: Use `f=json` with `outSR={EPSG}`, NOT `f=geojson`.**

GeoJSON (`f=geojson`) forces WGS84 output per spec, requiring reprojection back to state plane. This roundtrip introduces ~3 ft of datum error (WGS84/NAD83 transformation inconsistency between the ArcGIS server and ogr2ogr). Using `f=json` with `outSR` gets coordinates directly in the target CRS from the server — no reprojection needed, no datum ambiguity.

Default query parameters:
- `where=1=1` (all features, or a filter like `MUNICIPALITY='Patchogue'`)
- `outFields=OBJECTID` (minimize payload — DXF can't carry attributes anyway)
- `f=json`
- `returnGeometry=true`
- `outSR={target EPSG/WKID code}`
- `resultRecordCount=1000`
- `resultOffset=0` (increment by `resultRecordCount` each batch)

Use `resultRecordCount=1000` (not 2000) with `time.sleep(3)` between requests to avoid 503 errors from rate limiting.

Pagination loop with retry and **duplicate detection**:
```python
import urllib.parse, time

all_features = []
seen_oids = set()
offset = 0
batch = 1000
null_geom_count = 0
pagination_broken = False

while True:
    params = urllib.parse.urlencode({
        "where": where_clause,
        "outFields": "OBJECTID",
        "f": "json",
        "returnGeometry": "true",
        "outSR": target_epsg,
        "resultRecordCount": batch,
        "resultOffset": offset
    })
    url = f"{base_url}?{params}"
    data = None
    for attempt in range(3):
        try:
            req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0"})
            with urllib.request.urlopen(req, timeout=90) as resp:
                data = json.loads(resp.read().decode())
            break
        except urllib.error.HTTPError as e:
            if e.code in (401, 403):
                print(f"ERROR: {e.code} — authentication required.")
                print("This endpoint requires a token or credentials to access.")
                print("Ask the user if they have login credentials for this GIS portal.")
                break  # Don't retry auth errors
            elif e.code == 503:
                print(f"  Rate limited (503), waiting 10s...")
                time.sleep(10)
            else:
                print(f"  Retry {attempt+1}: HTTP {e.code}")
                time.sleep(5)
        except Exception as e:
            print(f"  Retry {attempt+1}: {e}")
            time.sleep(5)
    if data is None:
        print(f"ERROR: Failed to fetch offset {offset} after 3 retries")
        break

    features = data.get("features", [])
    if not features:
        break

    # --- PAGINATION DUPLICATE CHECK ---
    # Some servers silently ignore resultOffset and return the same
    # features every time. Detect this by checking OBJECTIDs.
    batch_oids = set()
    new_in_batch = 0
    for feat in features:
        oid = feat.get("attributes", {}).get("OBJECTID")
        if oid is not None:
            batch_oids.add(oid)
            if oid not in seen_oids:
                new_in_batch += 1
                seen_oids.add(oid)

    if offset > 0 and new_in_batch == 0:
        print(f"ERROR: Pagination broken — offset {offset} returned the same "
              f"features as a previous batch. This server does not support "
              f"resultOffset pagination.")
        print(f"       Got {len(all_features)} features before pagination failed.")
        print(f"       The server's maxRecordCount may be the hard limit.")
        pagination_broken = True
        break
    elif offset > 0 and new_in_batch < len(features) * 0.5:
        print(f"  WARNING: {len(features) - new_in_batch} duplicate OIDs in this batch")

    # Filter out null/empty geometries
    for feat in features:
        geom = feat.get("geometry")
        if geom and geom.get("rings"):  # polygon
            all_features.append(feat)
        elif geom and geom.get("paths"):  # polyline
            all_features.append(feat)
        elif geom and ("x" in geom and "y" in geom):  # point
            all_features.append(feat)
        else:
            null_geom_count += 1

    offset += batch
    if len(features) < batch:
        break
    time.sleep(3)

if null_geom_count > 0:
    print(f"WARNING: Skipped {null_geom_count} features with null/empty geometry")
```

### Step 4: Post-download validation

Run ALL of these checks before writing the DXF. None are optional.

#### 4a. Feature count verification

```python
actual_count = len(all_features) + null_geom_count
if pagination_broken:
    print(f"Count check:    INCOMPLETE — pagination failed at {actual_count} of "
          f"{expected_count} expected ⚠")
    print(f"                Server does not support resultOffset. Got {len(all_features)} "
          f"valid features (the server's maxRecordCount limit).")
elif actual_count == expected_count:
    print(f"Count check:    {actual_count:,} features — matches expected ✓")
elif actual_count >= expected_count * 0.95:
    print(f"Count check:    {actual_count:,} of {expected_count:,} expected — within 5% ✓")
else:
    print(f"Count check:    {actual_count:,} of {expected_count:,} expected — "
          f"MISSING {expected_count - actual_count} features ✗")
```

#### 4b. Coordinate range sanity check

Verify the first feature's coordinates are in a plausible range for state plane (not lat/lon). US state plane coordinates in feet are typically 5-7 digit numbers. Lat/lon would be small numbers (-180 to 180, -90 to 90).

```python
def get_first_coord(feat):
    """Extract first coordinate pair from any geometry type."""
    geom = feat.get("geometry", {})
    if geom.get("rings"):
        return geom["rings"][0][0]
    elif geom.get("paths"):
        return geom["paths"][0][0]
    elif "x" in geom:
        return [geom["x"], geom["y"]]
    return None

sample_pt = get_first_coord(all_features[0])
if sample_pt:
    x, y = sample_pt[0], sample_pt[1]
    # Check 1: Lat/lon (small values)
    if abs(x) < 1000 and abs(y) < 1000:
        print(f"Coord check:    ({x:.4f}, {y:.4f}) — ERROR: looks like lat/lon, "
              f"not state plane. Check outSR parameter ✗")
    # Check 2: Web Mercator (large negative X around -7M to -9M for US, Y around 4M-6M)
    elif x < -1_000_000 and abs(y) > 1_000_000:
        print(f"Coord check:    ({x:.0f}, {y:.0f}) — ERROR: looks like Web Mercator "
              f"(meters), not state plane (US ft). Did outSR get set to 3857? ✗")
    # Check 3: Unreasonably large
    elif abs(x) > 100_000_000 or abs(y) > 100_000_000:
        print(f"Coord check:    ({x:.1f}, {y:.1f}) — ERROR: unreasonably large. "
              f"Possible CRS mismatch ✗")
    # Check 4: Plausible state plane
    else:
        print(f"Coord check:    ({x:.2f}, {y:.2f}) — plausible state plane ✓")
```

#### 4c. Datum accuracy spot-check (MANDATORY — always run this)

This check is NOT optional. It must run on every single download. It catches the ~3 ft datum roundtrip error that occurs when GeoJSON (WGS84) coordinates are reprojected back to state plane.

The test: take one feature's native CRS coordinates, fetch the same feature in WGS84 from the server, reproject that WGS84 point back to the target CRS using ogr2ogr, and measure the delta. If > 0.1 ft, the native CRS approach is confirmed necessary.

```python
import subprocess, os

spot_oid = all_features[0].get("attributes", {}).get("OBJECTID")
native_pt = get_first_coord(all_features[0])  # from step 4b

# Clean up stale temp files FIRST — a leftover file from a previous run
# caused a 10M ft false delta during testing
for tmp in ["/tmp/spot_in.geojson", "/tmp/spot_out.geojson"]:
    if os.path.exists(tmp):
        os.remove(tmp)

if spot_oid and native_pt:
    # Fetch same feature in WGS84
    # Try f=json with outSR=4326 first (more reliable than f=geojson,
    # which some servers don't support for single-feature queries)
    wgs_url = base_url + "?" + urllib.parse.urlencode({
        "where": f"OBJECTID={spot_oid}",
        "outFields": "OBJECTID",
        "f": "json",
        "returnGeometry": "true",
        "outSR": 4326,
        "resultRecordCount": 1
    })
    try:
        req = urllib.request.Request(wgs_url, headers={"User-Agent": "Mozilla/5.0"})
        with urllib.request.urlopen(req, timeout=30) as resp:
            wgs_data = json.loads(resp.read().decode())
        wgs_feat = wgs_data["features"][0]
        geom = wgs_feat["geometry"]
        if geom.get("rings"):
            wgs_pt = geom["rings"][0][0]
        elif geom.get("paths"):
            wgs_pt = geom["paths"][0][0]
        elif "x" in geom:
            wgs_pt = [geom["x"], geom["y"]]
    except Exception:
        # Fallback: try f=geojson
        gj_url = base_url + "?" + urllib.parse.urlencode({
            "where": f"OBJECTID={spot_oid}",
            "outFields": "OBJECTID",
            "f": "geojson",
            "returnGeometry": "true"
        })
        req = urllib.request.Request(gj_url, headers={"User-Agent": "Mozilla/5.0"})
        with urllib.request.urlopen(req, timeout=30) as resp:
            gj_data = json.loads(resp.read().decode())
        gj_feat = gj_data["features"][0]
        coords = gj_feat["geometry"]["coordinates"]
        if gj_feat["geometry"]["type"] == "Polygon":
            wgs_pt = coords[0][0]
        elif gj_feat["geometry"]["type"] == "MultiPolygon":
            wgs_pt = coords[0][0][0]

    # Reproject WGS84 back to target CRS using ogr2ogr
    pt_gj = {
        "type": "FeatureCollection",
        "features": [{"type": "Feature",
                       "geometry": {"type": "Point", "coordinates": [wgs_pt[0], wgs_pt[1]]},
                       "properties": {}}]
    }
    with open("/tmp/spot_in.geojson", "w") as f:
        json.dump(pt_gj, f)

    subprocess.run([
        "ogr2ogr", "-f", "GeoJSON", "/tmp/spot_out.geojson", "/tmp/spot_in.geojson",
        "-s_srs", "EPSG:4326", "-t_srs", f"EPSG:{target_epsg}", "-overwrite"
    ], capture_output=True)

    with open("/tmp/spot_out.geojson") as f:
        rp = json.loads(f.read())
    rp_pt = rp["features"][0]["geometry"]["coordinates"]

    dx = native_pt[0] - rp_pt[0]
    dy = native_pt[1] - rp_pt[1]
    dist = (dx**2 + dy**2)**0.5

    print(f"    Native CRS:      ({native_pt[0]:.4f}, {native_pt[1]:.4f})")
    print(f"    Server WGS84:    ({wgs_pt[0]:.6f}, {wgs_pt[1]:.6f})")
    print(f"    Reprojected:     ({rp_pt[0]:.4f}, {rp_pt[1]:.4f})")
    print(f"    Delta: dX={dx:.4f} ft, dY={dy:.4f} ft, total={dist:.4f} ft")

    if dist < 0.1:
        print(f"Datum check:    No significant error ({dist:.2f} ft) ✓")
    elif dist < 1.0:
        print(f"Datum check:    Minor offset ({dist:.2f} ft) — within tolerance ✓")
    else:
        print(f"Datum check:    {dist:.2f} ft offset confirmed — native CRS avoids this ✓")
```

### Step 5: Export to DXF with ezdxf

**Do NOT use ogr2ogr for DXF export** — it writes polygons as HATCH entities. Use `ezdxf` to write closed LWPOLYLINEs:

```python
import ezdxf

doc = ezdxf.new("R2010")
msp = doc.modelspace()
layer_name = "PARCELS"  # adjust to match content
doc.layers.add(layer_name)

polyline_count = 0
for feat in all_features:
    geom = feat.get("geometry", {})

    # Handle polygons (rings)
    if geom.get("rings"):
        for ring in geom["rings"]:
            pts = [(p[0], p[1]) for p in ring]
            if len(pts) >= 2:
                msp.add_lwpolyline(pts, dxfattribs={"layer": layer_name}, close=True)
                polyline_count += 1

    # Handle polylines (paths)
    elif geom.get("paths"):
        for path in geom["paths"]:
            pts = [(p[0], p[1]) for p in path]
            if len(pts) >= 2:
                msp.add_lwpolyline(pts, dxfattribs={"layer": layer_name})
                polyline_count += 1

    # Handle points
    elif "x" in geom and "y" in geom:
        msp.add_point((geom["x"], geom["y"]), dxfattribs={"layer": layer_name})
        polyline_count += 1

doc.saveas("output.dxf")
print(f"Wrote {polyline_count} entities to DXF")
```

The layer name should reflect the content (e.g., "PARCELS", "ZONING", "WETLANDS", "ROADS").

### Step 6: Also provide a GeoJSON and shapefile (optional)

If the user also wants GIS formats or needs attribute data:
- Save a GeoJSON using `f=geojson` (separate download, for GIS use only — not for CAD)
- Generate a shapefile with ogr2ogr from the GeoJSON (preserves attributes, importable via Civil 3D `MAPIMPORT`)

## Deliverables

Always provide:
1. **DXF file** — closed LWPOLYLINEs on a named layer, coordinates in the target state plane (native from server)

Optionally provide:
2. **GeoJSON file** — for GIS reference (note: will be in WGS84)
3. **Shapefile (zipped)** — if attributes are needed in CAD

## Print validation summary

After all checks, print a summary block so the user can see the health of the data at a glance:

```
=== VALIDATION SUMMARY ===
Endpoint:       https://...MapServer/57/query
Filter:         MUNICIPALITY='Patchogue'
Native CRS:     WKID 6539
Target CRS:     WKID 6539 (match)
Expected count: 7,045
Actual count:   7,045 (7,045 valid + 0 null geometry) ✓
Pagination:     OK — no duplicate OIDs detected ✓
Coord check:    (1253994.41, 222729.14) — plausible state plane ✓
Datum check:    3.06 ft offset confirmed — native CRS avoids this ✓
DXF output:     7,047 polylines on layer PARCELS ✓
==========================
```

**The datum spot-check and pagination check MUST appear in this summary every time.** If either was skipped or errored, report that explicitly — never silently omit them.

If any check fails, the summary should make it obvious what went wrong:

```
=== VALIDATION SUMMARY ===
Endpoint:       https://...MapServer/12/query
Native CRS:     WKID 2260
Target CRS:     WKID 2260 (match)
Expected count: 15,000
Actual count:   2,000 (2,000 valid + 0 null geometry) ✗
Pagination:     BROKEN — server ignores resultOffset, returned same 2,000
                features repeatedly. maxRecordCount=2000 is the hard limit. ✗
Coord check:    (580123.45, 750234.56) — plausible state plane ✓
Datum check:    3.04 ft offset confirmed — native CRS avoids this ✓
DXF output:     2,003 polylines on layer PARCELS ⚠ (INCOMPLETE)
==========================
NOTE: This DXF contains only 2,000 of 15,000 parcels because the server
does not support pagination. The user may need to apply spatial or attribute
filters to download in smaller batches, or contact the county GIS department
for a bulk data export.
```

## CAD Import Notes

Tell the user:
- DXF has no projection metadata. Set the drawing coordinate system in Civil 3D: Settings > Edit Drawing Settings > Units and Zone. Match to the target CRS.
- For background reference, XREF is cleaner than INSERT (avoids bloating the working file with 10K+ polylines).
- Open the DXF directly with File > Open, or insert into existing drawing with INSERT at 0,0,0 scale 1.0.

## Edge Cases

- **Spatial filter**: Add `geometry` and `geometryType=esriGeometryEnvelope` with `spatialRel=esriSpatialRelIntersects` and `inSR=4326` to the query params, or filter by attribute (e.g., `where=MUNICIPALITY='WALLKILL'`).
- **Line/point layers**: The DXF export code handles all three geometry types (polygons, polylines, points). Check `geometryType` from the service metadata to confirm.
- **Large datasets (>50K features)**: Warn the user the DXF will be large. Consider suggesting spatial or attribute filtering.
- **Mixed geometry**: Some layers contain both polygons and multipolygons, or mixed types. The export code handles this by checking for `rings`, `paths`, and `x`/`y` independently.

## Why NOT GeoJSON for CAD

The GeoJSON spec mandates WGS84 (EPSG:4326) coordinates. When an ArcGIS server converts NAD83 state plane data to GeoJSON, it applies a datum transformation. When you then reproject back to state plane with ogr2ogr, the reverse transformation doesn't perfectly cancel out — resulting in ~3 ft of positional error in the NY/Northeast US area. Requesting `f=json` with `outSR={EPSG}` bypasses this entirely by getting the server's native coordinates directly.
