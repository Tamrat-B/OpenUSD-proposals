# AECO CRS samples: control points and WKT2 templates

Reference material for the [OpenUSD geospatial CRS proposal](../README.md)
([PixarAnimationStudios/OpenUSD-proposals #121](https://github.com/PixarAnimationStudios/OpenUSD-proposals/pull/121)).

This folder holds two real sites. For each site, the same control points are given in
several coordinate reference systems. Each CRS comes as a pair of files:

- `*_WKT2.txt`: the CRS as an OGC WKT2 string (ISO 19162:2019).
- `*_Coordinates.csv`: the control points expressed in that CRS.

Each WKT describes the coordinates of its CSV (and of the LandXML file for the Colorado
site calibration) exactly as written, including the axis units: US survey feet for
Colorado, metres for France. The CSV column headers repeat the unit.

Every WKT string was parsed by PROJ 9.8.1 (`proj_create()`, EPSG dataset v12.029) and
checked against the EPSG definition of its components. The coordinates were computed by
PROJ from the WKT strings, not from EPSG codes.

Licence: Apache License 2.0, copyright 2026 Trimble Inc. See [LICENSE](LICENSE) and
[NOTICE](NOTICE).

## Contents

```
README.md, LICENSE, NOTICE
Colorado/                                          Trimble Westminster, Colorado, USA (5 control points)
  01_NAD83-2011_Geographic3D_WKT2.txt / _Coordinates.csv
  02_NAD83-2011_ColoradoCentral_NAVD88_WKT2.txt / _Coordinates.csv
  03_Westminster_Site_Calibration_WKT2.txt / _Coordinates.csv
  Trimble Westminster Park and Ride LandXML.xml    terrain model, in the site calibration CRS (03)
  Colorado.kml                                     control points and terrain breaklines for Google Earth
France/                                            Trimble Paris, Joinville-le-Pont, France (59 control points)
  01_RGF93v2b_CC49_IGN69_WKT2.txt / _Coordinates.csv
  02_RGF93v2b_Lambert93_IGN69_WKT2.txt / _Coordinates.csv
  03_RGF93v2b_Geocentric_WKT2.txt / _Coordinates.csv
  04_RGF93v2b_Geographic3D_WKT2.txt / _Coordinates.csv
  05_ITRF2014_Geocentric_epoch2019.0_WKT2.txt / _Coordinates.csv
  06_ITRF2020_Geocentric_epoch2026.5_WKT2.txt / _Coordinates.csv
  France.kml                                       control points for Google Earth
  Trimble_Joinville_Batiment_A_Enveloppe.ifc       building envelope, placed with the CC49 CRS (01)
```

Lengths are in US survey feet for Colorado (1 ftUS = 1200/3937 m) and in metres for
France. Angles are in decimal degrees.

## Colorado: site calibration with an affine transformation and an inclined plane

All lengths are in US survey feet, the distance and height unit of the Trimble job
and of the LandXML design.

| File | CRS | WKT2 shape |
|---|---|---|
| 01 | NAD83(2011) geographic 3D, ellipsoidal height in ftUS (EPSG:6319 with the height in ftUS; no EPSG code) | `GEOGCRS` |
| 02 | NAD83(2011) / Colorado Central (ftUS) + NAVD88 height (ftUS) (EPSG:6428 + EPSG:6360, GEOID18) | `COMPOUNDCRS` |
| 03 | Westminster site grid + site height (ftUS), the Trimble site calibration | `COMPOUNDCRS` of a `DERIVEDPROJCRS` and a derived `VERTCRS` |

File 03 is the affine / inclined-plane template:

```
COMPOUNDCRS["Westminster site grid + site height",
    DERIVEDPROJCRS[...  BASEPROJCRS EPSG:6428, Affine parametric transformation (EPSG 9624)],
    VERTCRS[...        BASEVERTCRS EPSG:6360,  Vertical Offset and Slope (EPSG 1046)]]
```

- Horizontal: the Trimble horizontal adjustment (origin, rotation, scale, translation)
  is written as the six affine parameters A0 to B2. A0 and B0 are in US survey feet;
  A1, A2, B1, B2 are dimensionless.
- Vertical: Trimble defines the inclined plane on site-grid coordinates, whereas
  EPSG 1046 evaluates it at the geographic latitude and longitude. The slopes are
  therefore converted to inclinations in arc-seconds about the evaluation point.
  The vertical component depends on the horizontal position, so the compound CRS
  has to be resolved as a whole. The vertical offset is in US survey feet.
- Units in derived CRSs: in WKT2, a base CRS (`BASEPROJCRS`, `BASEVERTCRS`) has no `CS`
  clause, so the unit must be stated on the length parameters and on the derived axes.
  PROJ infers the unit of the base projected CRS from its false easting, but reads the
  base vertical CRS with a metre axis whatever its ID. The result is still correct
  because the offset and the site height axis state their unit.

Accuracy of the templates:

| Check | Max difference |
|---|---|
| 03 computed from the WKT vs the Trimble calibration formulas | 0.0000 mm horizontal, 0.0005 mm vertical |
| 02 computed from the WKT vs the surveyed state plane coordinates | 0.5 mm horizontal, 0.8 mm vertical |
| 03 computed from the WKT vs the surveyed site coordinates | 3.2 mm horizontal, 0.8 mm vertical |

The last row gives the residuals of the calibration itself (five control points, an
over-determined fit), not an error of the WKT encoding.

`Trimble Westminster Park and Ride LandXML.xml` is the Westminster terrain model
(surface "TWCO - Finished design", 709 breaklines) together with the five control points.
It is authored in the site calibration CRS of file 03, in the same unit (US survey feet).
LandXML writes points in northing, easting, elevation order, whereas the WKT axes and the
CSV are easting, northing, height. For example, `Control1` reads
`1206317.47 3108871.45 5484.68` in the LandXML and
`3108871.4501, 1206317.4694, 5484.6779` in `03_Westminster_Site_Calibration_Coordinates.csv`.

`Colorado.kml` shows the same control points and terrain breaklines in Google Earth
(WGS 84). Trimble Business Center exported it from the site calibration CRS, so it shows
where the calibration puts the terrain model on the ground.

## France: projections, geocentric frames and coordinate epochs

| File | CRS | WKT2 shape |
|---|---|---|
| 01 | RGF93 v2b / CC49 + NGF-IGN69 height (source of the control points) | `COMPOUNDCRS` |
| 02 | RGF93 v2b / Lambert-93 + NGF-IGN69 height (EPSG:10499) | `COMPOUNDCRS` |
| 03 | RGF93 v2b geocentric (EPSG:9780) | `GEODCRS`, `CS[Cartesian,3]` |
| 04 | RGF93 v2b geographic 3D (EPSG:9781) | `GEOGCRS` |
| 05 | ITRF2014 geocentric at epoch 2019.0 (EPSG:7789) | `COORDINATEMETADATA` |
| 06 | ITRF2020 geocentric at epoch 2026.5 (EPSG:9988) | `COORDINATEMETADATA` |

Heights use the RAF20 geoid grid. File 06 is the current-epoch template:

```
COORDINATEMETADATA[
    GEODCRS["ITRF2020", DYNAMIC[FRAMEEPOCH[2015]], ... CS[Cartesian,3] ... ID["EPSG",9988]],
    EPOCH[2026.5]]
```

- The CRS is Cartesian with length axes. The coordinate epoch lives in
  `COORDINATEMETADATA`, not in the CRS, and is repeated in the last column of the CSV.
- `pyproj.CRS` rejects `COORDINATEMETADATA` ("Input is not a CRS"). The PROJ C API
  accepts it in `proj_create()` and `proj_create_crs_to_crs_from_pj()` and applies the epoch.
- The ITRF coordinates assume RGF93 v2b = ETRF2000 at epoch 2019.0 and no motion of the
  points in the plate-fixed ETRF2000 frame. The Eurasian plate moved them 184.6 mm in
  ITRF2020 between 2019.0 and 2026.5.
- Asked for ITRF2020 at 2019.0 to ITRF2020 at 2026.5, PROJ 9.8.1 returns a null
  transformation (0 mm instead of 184.6 mm). The epoch must travel with the coordinates,
  and a change of epoch has to go through a plate-fixed frame.

`France.kml` shows the 59 control points in Google Earth (WGS 84), exported from
Trimble Business Center.

### Joinville building envelope (IFC)

`Trimble_Joinville_Batiment_A_Enveloppe.ifc` is the envelope of building A at the Trimble
Paris site (IFC4, metres, Z up). The IFC carries no georeferencing of its own: no
`IfcMapConversion`, and no reference latitude or longitude on `IfcSite`. Its model lengths
are physical. Placed with the proposal's model-placement inputs, it aligns with the CC49
control points of file 01:

```
crs:wkt         = 01_RGF93v2b_CC49_IGN69_WKT2.txt   (RGF93 v2b / CC49 + NGF-IGN69 height)
crs:position    = (1661117.993, 8180047.224, 37.270)
crs:orientation = (161.7777, 0, 0)
```

| Point | Model X (m) | Model Y (m) | Model Z (m) | CC49 E (m) | CC49 N (m) | NGF-IGN69 H (m) |
|---|---:|---:|---:|---:|---:|---:|
| P1 | 9.914 | 6.381 | 0.150 | 1661110.508 | 8180038.115 | 37.420 |
| P2 | -10.343 | -6.308 | 0.150 | 1661125.909 | 8180056.395 | 37.420 |

- The model +Y axis has a CC49 grid bearing of 162.1774°. The meridian convergence at the
  origin is -0.3996°, so the heading, clockwise from true north, is 161.7777°.
- Pitch and roll are 0: the two points have the same height and do not determine a tilt.
- Placing P1 and P2 in the east/north/up frame at the origin and projecting them to CC49
  reproduces the table within 0.8 mm. The difference is the CC49 combined scale factor
  (about -93 ppm) over 12 m, which a reader applies at runtime.
- The IFC declares `TrueNorth = (1, 0)` in its representation context. The placement above
  is derived from the control points and does not use that value.

## Notes

The KML files are for visual context only. The CSV files and WKT strings are the
reference data.
