# CAD TaxMap — Get Property Lines Into CAD, Fast

## What this does

Give Claude a link to any county GIS portal, tell it which parcels you want, and get a CAD-ready DXF file with closed polylines on a named layer, in the correct state plane coordinates. No manual exporting, no format conversion, no coordinate headaches.

Tested on: Orange County NY, Suffolk County NY, Cortland County NY, Lehigh County PA. Works with any ArcGIS REST endpoint nationwide.

---

## Setup (one time, ~2 minutes)

### 1. Download the skill file

Download **cad-taxmap.skill** from the link in the post.

### 2. Install it in Claude

- Open **claude.ai**
- Go to **Customize** (left sidebar)
- Click **Skills**
- Click the **+** button
- Select **Upload a skill** (or "Create skill")
- Upload the **cad-taxmap.skill** file
- Make sure the skill toggle is **on**

Note: Skills require a Pro, Max, Team, or Enterprise plan.

### 3. Enable Code Execution and Web Search

The skill needs Claude to run Python code and access URLs:

- Go to **Settings** → **Capabilities**
- Make sure **Code execution and file creation** is enabled
- In any chat, make sure **Web Search** is enabled (Claude needs this to find REST endpoints from viewer URLs)

That's it. You're set up.

---

## Using it

### Step 1: Find your county's GIS portal

Google `[county name] [state] GIS parcels` or `[county name] GIS viewer`. You're looking for the county's online parcel/tax map viewer. Almost every county in the US has one.

Examples of URLs that work:
- `https://gisapps.suffolkcountyny.gov/gisviewer/`
- `https://gis.orangecountygov.com/arcgis/rest/services/...`
- `https://cortland-county-gis-portal-cortlandplanning.hub.arcgis.com/`
- `https://www.arcgis.com/apps/webappviewer/index.html?id=...`

Any of these formats work. You do NOT need to find the technical REST endpoint — Claude will figure that out.

### Step 2: Tell Claude what you want

Open a new chat in Claude and type something like:

> Get me the parcels for the Village of Patchogue from https://gisapps.suffolkcountyny.gov/gisviewer/

Or:

> Download Allentown property lines from https://www.arcgis.com/apps/webappviewer/index.html?id=42536953612d4526b62805bb7e2782a4

That's it. One message. Claude will:
- Find the ArcGIS REST endpoint behind the viewer
- Identify the parcel polygon layer
- Detect the correct coordinate system
- Download all features with pagination
- Run validation checks (feature count, coordinate sanity, datum accuracy)
- Export as closed LWPOLYLINEs in a DXF
- Give you the file to download

### Step 3: Open in Civil 3D

1. Download the DXF file Claude gives you
2. Open it in Civil 3D (File > Open)
3. Set your drawing coordinate system:
   - **Settings** tab in Toolspace
   - Right-click the drawing name → **Edit Drawing Settings**
   - **Units and Zone** tab
   - Set the zone Claude tells you (e.g., NY83-EF, PA83-SF)
4. Save as DWG
5. XREF or INSERT into your working drawing

---

## What Claude checks automatically

Every run produces a validation summary:

```
=== VALIDATION SUMMARY ===
Endpoint:       ...GISViewer/MapServer/57/query
Filter:         MUNICIPALITY='Patchogue'
Native CRS:     WKID 6539
Target CRS:     WKID 6539 (match)
Count check:    7,045 features — matches expected ✓
Pagination:     OK — no duplicate OIDs ✓
Coord check:    (1253994.41, 222729.14) — plausible state plane ✓
Datum check:    3.06 ft offset confirmed — native CRS avoids this ✓
DXF output:     7,047 polylines on layer PARCELS ✓
==========================
```

- **Count check** — confirms all parcels were downloaded (catches pagination failures)
- **Pagination check** — detects servers that silently ignore page offsets
- **Coordinate check** — catches lat/lon or Web Mercator coordinates that would be wrong in CAD
- **Datum check** — measures the ~3 ft error that occurs when coordinates roundtrip through WGS84, confirms the native CRS approach avoided it
- **Auth detection** — if the server needs credentials, tells you instead of failing silently

---

## Tips

- **You don't need the REST URL.** A link to the county's GIS viewer or map app is enough. Claude will find the service behind it.
- **You don't need to know the CRS.** Claude auto-detects it from the server metadata. If the service is in Web Mercator (common for ArcGIS Online), it auto-resolves to the correct state plane zone.
- **XREF is better than INSERT** for large parcel sets. 50K+ polylines will bloat your drawing if inserted directly.
- **DXF has no projection metadata.** You must set the drawing coordinate system in Civil 3D manually. Claude will tell you which zone to use.
- **Need attributes?** Ask Claude for a shapefile too — it'll generate a zipped .shp with owner names, addresses, SBL, etc. Import via MAPIMPORT in Civil 3D.

---

## What it won't work with

- GIS portals that aren't ArcGIS (WFS/GeoServer — Claude will detect this and tell you)
- Servers that require authentication/login (Claude will tell you)
- Servers that don't support pagination (Claude will tell you how many parcels it got vs. expected)

---

Built with Claude · aedonyx.io
