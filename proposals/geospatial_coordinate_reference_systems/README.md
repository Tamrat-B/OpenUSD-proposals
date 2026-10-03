# Geospatial Coordinate Reference Systems for OpenUSD

## Contributors

- **Esri** (Tamrat Belayneh, Simon Haegler)
- **Nvidia** (Aaron Luk)
- The case study, the WKT encoding section and the runtime coordinate
  transformation section are Sébastien Vielliard's work.
- Also draws on work by David de Koning, Sébastien Vielliard, Simon Haegler
  and Tamrat Belayneh in the AOUSD AECO Interest Group.

## Introduction

This proposal introduces first-class support for
**Geospatial Coordinate Reference Systems (CRS)** in OpenUSD.
It defines two new schemas that allow USD scenes to declare
*where on a celestial body* their 3D content is located,
enabling interoperability with GIS, AECO, digital twin,
and simulation workflows that require real-world positioning.

The core idea is simple:
a CRS definition is stored as an OGC WKT string on a typed prim,
and an applied API schema binds that CRS to any `UsdGeomXformable` prim
in the scene hierarchy.
Child prims inherit their parent's CRS binding,
and a runtime reprojection pipeline transforms geometry
between different CRS zones within a single composed stage.

This proposal is the result of collaborative work
within the AOUSD AECO Interest Group,
with contributions from Esri, Pixar, NVIDIA, Bentley, and Trimble.

## Motivation

### The geospatial gap in OpenUSD

OpenUSD provides a powerful scene description framework
with rich support for geometry, materials, lighting, and physics.
However, it currently has no mechanism to express
*where a scene is located in or on a celestial body*.

A building model authored in USD can describe its shape,
appearance, and internal structure in exquisite detail —
but there is no standard way to say
"this building sits at 34.0561°N, 117.1956°W"
or "these coordinates are in UTM zone 11N."

This is not a niche requirement.
Every AECO project, every digital twin,
every GIS visualization, and every urban simulation
needs to place 3D content at real-world coordinates.
Without a standard mechanism, each tool and pipeline
invents its own custom metadata,
leading to data loss at interchange boundaries
and preventing true interoperability.

### The expanding scope of USD

USD was originally designed for film production workflows,
where scenes exist in an abstract coordinate space
and absolute position on the Earth is irrelevant.

As USD adoption expands into AECO, GIS, defense, simulation,
and digital twin applications through the AOUSD alliance,
the need for geospatial positioning has become critical.
These industries routinely work with coordinate reference systems,
and their tools (ArcGIS, QGIS, FME, Bentley, Trimble, etc.)
all expect CRS metadata on imported geometry.

The glTF format has already recognized this need
with its own geospatial extension proposal.
IFC 5 is evaluating USD as a potential foundation.
OpenUSD must provide a standard answer
to the question "where is this scene?"

## Problem statement

### Placing 3D content on a celestial body

To place a 3D model at a real-world location, three things are needed:

1. **A Coordinate Reference System (CRS)** that defines
   the mathematical relationship between coordinates and positions
   on the Earth's surface (e.g., "UTM zone 11N" or "WGS 84 geographic").

2. **Coordinates** in that CRS
   (e.g., Easting = 481,948.63 m, Northing = 3,768,393.52 m).

3. **A binding mechanism** that associates the CRS
   with the geometry in the scene graph.

USD currently provides none of these as first-class features.
Users must fall back to custom metadata, primvars,
or out-of-band sidecar files to convey this information —
all of which are opaque to USD's composition engine,
rendering pipeline, and standard tooling.

### Why this matters now

1. **Data loss at interchange boundaries.**
   CRS metadata stored in `customData` or proprietary attributes
   is routinely stripped during USD export/import cycles
   across different tools.

2. **No standard for CRS inheritance.**
   Without a defined inheritance model,
   every prim in a large scene must redundantly carry CRS metadata,
   or tools must implement ad-hoc resolution logic.

3. **Precision hazards.**
   A UTM easting of 481,948 m exceeds the useful range of `float32`,
   and implementations store it in a `point3f` array anyway,
   because nothing tells them not to.
   The result is visible jitter and drift
   in scenes that look correct on paper.

4. **Multi-CRS composition is undefined.**
   A cross-state pipeline spans UTM zones 11 and 12,
   and a regional twin draws on imagery, terrain and vectors
   that each arrive in their own CRS.
   Today the only way to put them on one stage
   is to convert them all first,
   which is a cost the largest projects cannot pay.

5. **Industry adoption is blocked.**
   GIS vendors, AECO tool makers, and digital twin platforms
   cannot fully adopt USD without a standard way to express CRS,
   because it is a foundational requirement for their workflows.

## Background: Coordinate Reference Systems

This section provides context for readers unfamiliar with geospatial concepts.

### Geographic vs. Projected CRS

A **Geographic CRS** uses angular coordinates
(latitude and longitude in degrees)
on a mathematical model of the Earth's shape (an ellipsoid).
Example: WGS 84 (EPSG:4326) — the CRS used by GPS.

A **Projected CRS** mathematically projects
the curved Earth surface onto a flat 2D plane.
Coordinates are linear (metres or feet).
Example: UTM zone 11N (EPSG:32611) — used for Southern California.

Every projected CRS contains a base geographic CRS,
a projection method (e.g., Transverse Mercator, Lambert Conformal Conic),
and projection parameters (central meridian, scale factor, false easting, etc.).

### CRS encodings: OGC WKT, EPSG, and WKID

There are several ways to identify a CRS:

| Encoding | Description | Example |
|----------|-------------|---------|
| **EPSG code** | Integer ID from the IOGP geodesy registry | `32611` |
| **WKID** | Well-Known ID — same concept, used by Esri (includes Esri-specific codes beyond EPSG) | `32611` |
| **OGC WKT** | Self-contained text string defining the full CRS (ISO 19162:2019) | `PROJCRS["WGS 84 / UTM zone 11N", ...]` |
| **PROJ string** | Compact string for the PROJ library | `+proj=utm +zone=11 +datum=WGS84` |

This proposal uses **OGC WKT 2** (ISO 19162:2019)
as the canonical CRS encoding,
with EPSG authority identifiers embedded within the WKT via `ID["EPSG", code]`.

### 3D CRS types

All CRS definitions in this proposal must be 3D.
The supported types are:

| Type | WKT Keyword | Axes | Example |
|------|-------------|------|---------|
| 3D Projected | `COMPOUNDCRS` (PROJCRS + VERTCRS) | Easting, Northing, Up | NAD83 / UTM 11N + NAVD88 height |
| 3D Geographic | `GEOGCRS` with 3 axes | Lat, Lon, Height | WGS 84 (EPSG:4979) |
| 3D Geocentric (ECEF) | `GEODCRS` | X, Y, Z | ITRF2020 |
| 3D Engineering | `DERIVEDPROJCRS` | Site X, Y, Z | Construction project grid |

For dynamic datums (time-dependent reference frames),
WKT 2 supports the `COORDINATEMETADATA` wrapper
with `FRAMEEPOCH` and `EPOCH` clauses
for high-precision applications.
Coordinate-epoch use is roadmap question 15, outside the initial CRS-only scope.

## Case study for the WGS 84 approach

WGS 84 (EPSG:4326) became dominant mainly because GPS uses it natively — every GPS receiver outputs coordinates in this datum. It is a global geodetic reference, so unlike national datums it works anywhere on Earth with one definition. The web mapping ecosystem reinforced this: GeoJSON mandates it, and most APIs and OGC standards default to it. Being a simple longitude/latitude geographic CRS makes it human-readable and easy to re-project from.

### Eiffel Tower example

Currently, OpenUSD georeferencing relies on stage-level metadata to define the relationship between virtual and physical space. Consider a high-fidelity model of the Eiffel Tower:

- Metrics: Authored at a scale of `metersPerUnit = 1.0`.
- Orientation: Configured as Y-Up, a common legacy default in many DCC tools.
- Coordinates: Vertex positions use 32-bit single-precision floats (`float3`).
- Local Origin: To avoid "floating-point jitter" caused by large coordinate values, the center of the square base is placed at `(0,0,0)`.

![Eiffel Tower](./eiffel_tower.png)

### WGS 84 georeferencing approach

The [omniGeospatial](https://docs.omniverse.nvidia.com/kit/docs/omni.usd.schema.geospatial/0.0.1/USD_SCHEMAS.html) schema (and [Hydra plugin](https://github.com/NVIDIA-Omniverse/OpenUSD-plugin-samples/tree/main/src/hydra-plugins#creating-a-custom-hydra-20-scene-index-for-geospatially-aware-transforms) as introduced by Nvidia Omniverse) georeferences such a model by applying a `WGS84ReferencePositionAPI` to a parent `Xform` prim.

This establishes a global anchor using:

- Latitude: 48.8584 N
- Longitude: 2.2945 E
- Altitude: 33.0 m (Height above the WGS84 ellipsoid).

The runtime engine then performs an Earth-Centered, Earth-Fixed (ECEF) conversion and a rotation into a Local Tangent Plane, such as East-North-Up (ENU).

### Limitations of only using a WGS 84 anchor point

While likely sufficient for visual effects work, the above method is inadequate for high-precision Survey and Construction for two primary reasons:

1. Datum Ambiguity: The term "WGS84" technically denotes a Datum Ensemble with an inherent low accuracy of approximately 2 meters. WGS84 coordinates are dynamic, changing over time due to tectonic plate motions, and up to 10 cm per year. For WGS84 coordinates to be accurate, they must be provided with the corresponding realization and measurement epoch (e.g., "WGS 84 (G2296) at epoch 2026.25"). This necessary detail is currently unsupported in standard USD schemas.

2. Axis Orientation: The current definition of "USD North" lacks the precision needed to align with the "True North" of a national geodetic Coordinate Reference System (CRS). Furthermore, AECO projects often require heights to be referenced to the geoid (orthometric height) for gravity-dependent systems, such as drainage. Current USD methods cannot precisely align with local vertical systems (like IGN 69) or map projections (like Lambert 93), which for example are the official systems used for AECO projects in France.

## Encoding the CRS: OGC WKT 2.1.11

We propose updating the OpenUSD schema to support describing the Coordinate Reference System (CRS) as a self-contained [WKT v2.1.11](https://docs.ogc.org/is/18-010r11/18-010r11.pdf) string ("Well-known text representation of coordinate reference systems"). This is equivalent to [ISO 19162:2019](https://www.iso.org/standard/76496.html).

This section covers the encoding only. The schema that carries it, and how a
CRS binds to the scene graph, are in [Design overview](#design-overview) and
[Detailed design](#detailed-design) below.

Key benefits:

- Standardization: Uses a mature, ISO-compliant format widely adopted in GIS and engineering.
- Self-Contained: Encodes the CRS definition (ellipsoid, datum, projection, and units) in a single string, without requiring a registry lookup to read that definition; computing a transformation may still need the engine's operation database or grids.
- Precision: Supports accurate definition of any Coordinate Reference Systems.

### WKT examples

The two examples here are worked against the case study. Reference encodings
of common CRS types are in [Appendix A](#appendix-a-wkt-examples).
Formatting these examples for readability does not select the authored WKT
normal form; that remains question 10.

#### WGS 84 ENU (east-north-up) local tangent plane

The following WKT string encodes, as a CRS, the same georeference that the `WGS84ReferencePositionAPI` [case study](#case-study-for-the-wgs-84-approach) above carries as a scene-level position. It is here to show what WKT 2 can express, and not to offer a second way of placing content: a CRS names the system a prim's coordinates are expressed in, and the prim's own transform gives its position within that system. The two compose, and neither replaces the other. It defines a 3D local coordinate system centered at the Eiffel Tower with a Y-Up orientation. The CRS retains its declared coordinate meaning. Authored unit and up-axis conformance is the decision on question 4; remaining component mappings are open question 6. See [Stage metadata](#stage-metadata-metersperunit-and-upaxis).

```lisp
GEODCRS["Y-Up Local Tangent Plane at Eiffel Tower",
    BASEGEOGCRS["WGS 84",
        DATUM["World Geodetic System 1984",
            ELLIPSOID["WGS 84", 6378137, 298.257223563, LENGTHUNIT["metre", 1]],
            ID["EPSG", 6326]],
        ID["EPSG", 4979]],
    DERIVINGCONVERSION["Topocentric at Eiffel Tower",
        METHOD["Geographic/topocentric conversions", ID["EPSG", 9837]],
        PARAMETER["Latitude of topocentric origin", 48.8584,
            ANGLEUNIT["degree", 0.0174532925199433], ID["EPSG", 8834]],
        PARAMETER["Longitude of topocentric origin", 2.2945,
            ANGLEUNIT["degree", 0.0174532925199433], ID["EPSG", 8835]],
        PARAMETER["Ellipsoidal height of topocentric origin", 33.0,
            LENGTHUNIT["metre", 1.0], ID["EPSG", 8836]]],
    CS[Cartesian, 3],
    AXIS["X", east], AXIS["Y", up], AXIS["Z", south],
    LENGTHUNIT["metre", 1]]
```

#### Derived CRS with affine site calibration

For projects requiring high accuracy, we can compute the transformation between the National CRS (e.g., Lambert-93 + IGN69) and the USD local CRS using least squares. This transformation—which can include affine transformations (EPSG 9624) and vertical adjustment planes to account for localized vertical deviations—can be encoded directly into the WKT string.

```lisp
COMPOUNDCRS["Site Local Coordinate System (X East, Y Up, Z South)",
    DERIVEDPROJCRS["Site Local Horizontal (Derived from ETRS89-FRA/Lambert-93)",
        BASEPROJCRS["ETRS89-FRA [RGF93 v2b] / Lambert-93",
            BASEGEOGCRS["ETRS89-FRA [RGF93 v2b]",
                DATUM["ETRS89-FRA [RGF93 v2b]",
                    ELLIPSOID["GRS 1980", 6378137, 298.257222101,
                        LENGTHUNIT["metre", 1]]],
                ID["EPSG", 9782]],
            CONVERSION["Lambert-93",
                METHOD["Lambert Conic Conformal (2SP)", ID["EPSG", 9802]],
                PARAMETER["Latitude of false origin", 46.5,
                    ANGLEUNIT["degree", 0.0174532925199433], ID["EPSG", 8821]],
                PARAMETER["Longitude of false origin", 3,
                    ANGLEUNIT["degree", 0.0174532925199433], ID["EPSG", 8822]],
                PARAMETER["Latitude of 1st standard parallel", 44,
                    ANGLEUNIT["degree", 0.0174532925199433], ID["EPSG", 8823]],
                PARAMETER["Latitude of 2nd standard parallel", 49,
                    ANGLEUNIT["degree", 0.0174532925199433], ID["EPSG", 8824]],
                PARAMETER["Easting at false origin", 700000,
                    LENGTHUNIT["metre", 1.0], ID["EPSG", 8826]],
                PARAMETER["Northing at false origin", 6600000,
                    LENGTHUNIT["metre", 1.0], ID["EPSG", 8827]]],
            ID["EPSG", 9794]],
        DERIVINGCONVERSION["Site Calibration Horizontal Affine",
            METHOD["Affine parametric transformation", ID["EPSG", 9624]],
            PARAMETER["A0", 10.0, LENGTHUNIT["metre", 1.0], ID["EPSG", 8623]],
            PARAMETER["A1", 1.00005, SCALEUNIT["unity", 1.0], ID["EPSG", 8624]],
            PARAMETER["A2", 0.00001, SCALEUNIT["unity", 1.0], ID["EPSG", 8625]],
            PARAMETER["B0", 5.0, LENGTHUNIT["metre", 1.0], ID["EPSG", 8639]],
            PARAMETER["B1", -0.00001, SCALEUNIT["unity", 1.0], ID["EPSG", 8640]],
            PARAMETER["B2", 1.00005, SCALEUNIT["unity", 1.0], ID["EPSG", 8641]]],
        CS[Cartesian, 2],
        AXIS["X", east, ORDER[1]],
        AXIS["Z", south, ORDER[3]],
        LENGTHUNIT["metre", 1.0]],
    VERTCRS["Site Local Vertical (Inclined Plane)",
        BASEVERTCRS["NGF-IGN69 height",
            VDATUM["Nivellement General de la France - IGN69",
                ID["EPSG", 5119]],
            ID["EPSG", 5720]],
        DERIVINGCONVERSION["Inclined Plane Vertical Adjustment",
            METHOD["Vertical Offset and Slope", ID["EPSG", 1046]],
            PARAMETER["Vertical Offset", 1.5,
                LENGTHUNIT["metre", 1.0], ID["EPSG", 8603]],
            PARAMETER["Inclination in latitude", 0.00001,
                ANGLEUNIT["radian", 1.0], ID["EPSG", 8730]],
            PARAMETER["Inclination in longitude", 0.00001,
                ANGLEUNIT["radian", 1.0], ID["EPSG", 8731]]],
        CS[vertical, 1],
        AXIS["Y", up, ORDER[2]],
        LENGTHUNIT["metre", 1.0]]
]
```

## Terms

The background above covers the geodesy.
This section fixes the words this proposal uses for its own constructs,
and separates several that mean different things
to the industries this proposal serves.

### Terms this proposal defines

**CRS binding.**
A statement that the coordinates of a prim,
and of its subtree down to the next binding,
are expressed in a named CRS.

**Native CRS.**
The CRS in which an asset's currently authored coordinates are expressed.
It gives those coordinate values their meaning;
it is not a record of earlier CRSs or of the asset's conversion history.

**Anchor.**
The prim at which CRS coordinates enter a scene:
its own location is a position in the CRS bound to it.
Content beneath an anchor, up to but excluding any nested anchor,
is positioned relative to it by offsets in ordinary scene units,
with no CRS coordinates of its own.
What constrains the CRS an anchor may be bound to is a design question,
not part of the term.

**Position and offset.**
A *position* is a coordinate in a CRS; an *offset* is a distance
from another prim, in scene units.
An anchor's location is a position. Content beneath it is offsets,
up to but excluding any nested anchor.

**Resolution.**
The computation that takes a composed stage and produces
where each prim is in one output CRS.
Resolution is work a runtime does.

**Target CRS.**
The CRS a resolution produces its output in.
How it is chosen is not part of the term.

**Placement.**
Where an instance sits, how it is oriented and any instance-specific scale
within a CRS.
Placement records a real-world decision about an instance:
it may come from a survey or from adjustments to align the instance
with better-known features, and can differ from instance to instance.

**Conformance.**
A correction for the authoring conventions of a source asset —
its up axis, its units, the orientation it was modelled in.
Conformance is not survey data.
It is identical for every instance of that asset,
and separating it from placement is what keeps the asset reusable.

### Terms that collide

**Base.**
Three different things are called *base* by the industries this proposal serves,
and this proposal uses none of them unqualified.

- A **project's base CRS**, in GIS practice,
  is the CRS a project works in and brings its data into.
  It is the **Target CRS** when resolving into that project's working coordinates;
  a consumer selecting another output CRS follows the
  [decision on question 8](#decision-on-question-8-consumer-selected-output-context).
- **`BASEGEOGCRS`**, in WKT 2,
  is the geographic CRS a projected or derived CRS is built *from*.
  It sits underneath a CRS definition, not above a project.
- A **project base point**, in surveying practice,
  is a point on the ground, not a CRS.

**Local.**
Two established and incompatible uses.

- In AECO, a **local CRS** is a project or construction grid:
  a flat Cartesian grid agreed for a site,
  with its own origin and its axes along the construction drawings.
  Its relationship to ground distances can include scale distortion.
  This proposal says **project grid**.
- In GIS, **local** often means a topocentric CRS such as
  east-north-up (ENU), whose horizontal axes define a tangent plane.
  Its vertical component retains departure from that plane;
  flattening the surface onto the plane discards that departure.
  This proposal says **topocentric CRS**.

A topocentric CRS and a surface flattened onto its tangent plane
are different representations.

**Site calibration (localization).**
In ISO/TS 15143-4 and in construction machine control,
the operation of relating a site's working coordinates to a geodetic CRS.
Its representation by a CRS definition and binding is the
[decision on question 7](#decision-on-question-7-site-calibration).

**Reference epoch and coordinate epoch.**
A dynamic datum's reference epoch is the date to which its defining
parameters refer; a coordinate epoch is the date at which a coordinate
set's positions apply.
Neither is the time sample used to animate content in a scene.
A measurement's observation or forecast time likewise supplies neither epoch
unless that relationship is explicitly recorded.

## Design overview

### Principles

1. **Industry agnosticism.**
   The CRS mechanism must serve GIS, AECO, M&E, simulation,
   and defense equally — no single industry's conventions are privileged.

2. **Self-contained CRS definitions.**
   A USD file must carry all information needed to interpret its coordinates
   without relying on external registry lookups at runtime.

3. **Composition-friendly.**
   CRS definitions and bindings must compose correctly
   through all USD composition arcs (references, sublayers, inherits, etc.).

4. **Inheritance.**
   A CRS applies to a subtree.
   Any prim that needs a different one says so
   and its own subtree follows it.

5. **Precision-aware.**
   Geospatial magnitude is never carried
   in storage that cannot hold it.
   Where a format constrains precision,
   the design keeps the large values out of it
   rather than asking authors to accept the loss.

6. **Minimal disruption.**
   No changes to existing USD schemas or core APIs.
   The geospatial schemas are additive and optional.

7. **Extensible.**
   Third-party CRS libraries (PROJ, GDAL, Esri, Trimble)
   are abstracted behind an internal API.
   OpenUSD itself does not validate or interpret WKT strings.

8. **Interoperable.**
   The design should facilitate round-trip exchange
   with glTF (geospatial extension), IFC, CityGML, and OGC 3D Tiles.

### Functional requirements

<!--
Editing note for this section, for people and for agents alike.

Requirement numbers are identifiers, frozen at the review baseline of the pull
request that introduced this section. Other documents, comments and test
fixtures cite them.

- Do not renumber, and do not insert a requirement between two existing numbers.
- A new requirement takes the next unused number and is placed under
  the group heading it belongs to, even where that breaks the numeric sequence
  within that group.
- A withdrawn requirement keeps its number and its title; its sentence is
  replaced by "Withdrawn." and one line saying why.
- When citing a requirement anywhere outside this file, give number and title
  together: "requirement 21, Never placed by a guess".
- Each requirement is one sentence. The italic text after it is a case from
  practice and carries no requirement of its own. Requirements state observable
  outcomes; any required encoding constraint must be explicit and reviewed.
- Do not decide an open question in Terms or in a requirement. The table at the
  end of this section names the requirements each open question is decided
  against; the decision is made there, not here.

Renumbering happens once, at merge, with a published old-to-new mapping.
-->

What a solution has to do, including the explicitly proposed WKT encoding
constraints in requirements 2 and 31.
These are what an implementation is checked against,
and the terms on which a design change is argued:
it either serves one of these or it does not.
The questions the discussion has left open are decided the same way;
the table at the end of this section names the requirements each one is decided against.

Each requirement is one sentence.
The italic text that follows it is rationale or a case from practice,
and carries no requirement of its own.

These requirements constrain the geospatial data model and normative runtime
behavior, including resolved queries; they do not prescribe callable APIs or
an implementation architecture. Retained design sketches and prototype code
later in this document do not settle the open questions or establish that
the model is complete. An applied API schema is a USD data-model category,
distinct from a callable programming interface.

#### Initial scope and roadmap

The initial scope retains georeferencing, multi-CRS assembly, CRS-aware model
placement and non-visual measurement workflows, using CRS-only WKT as the
authored authority for each CRS definition. The Colorado
site-grid/State Plane + NAVD88 and France CC49/Lambert-93 examples illustrate
site calibration and assembly. They impose no limit on geographic extent or
dataset size; the city and global analytical cases under requirement 30 remain
part of the functional requirements.

The scene supplies the source CRS definitions and coordinate/placement data;
the consumer selects the output CRS under requirement 16. The transformation
engine chooses applicable geodetic operations and manages their resources,
including datum or height grids, while respecting the authored CRS definitions.
A non-epoch transformation is not excluded merely because it needs a grid or
an operation database. An engine unable to perform the requested transformation
reports failure under requirement 21; its operation choice and accuracy are
observable under requirement 28. This does not require every engine to support
every CRS or prescribe its algorithms, catalog or resource distribution.

Coordinate-epoch representation, interpretation and epoch-dependent resolution
are deferred to roadmap question 15. CRS frame-reference-epoch information,
measurement/observation times and USD time-sampled content retain their distinct
meanings; none supplies an unrecorded coordinate epoch. Epoch-dependent requests
are unsupported in the initial scope, even if an engine could compute them;
silently dropping epoch information or selecting a default epoch does not meet
requirement 21.

Scene-authored selection or pinning of a coordinate operation, model or resource
is separate roadmap work; the initial scope does not introduce USD properties
for it. Engine-selected resources and authored CRS information are not the same
authority. Initial model choices must preserve a path to these controls and to
coordinate epochs. Retained epoch sketches describe roadmap candidates, not
initial conformance obligations. The placement representation, WKT normalization
profile and result guarantees identified below still need completion.

**The CRS itself**

1. **Self-contained definitions.**
   A CRS carried in a scene can be read from the scene alone,
   without a lookup against an external registry.

   *A 2D State Plane zone has a registry code; the compound of that zone,
   its current realization and its vertical datum often has none,
   so an identifier alone cannot name the system the data is actually in,
   and a code that does exist can be missing or differently versioned
   when the scene is next opened.
   This is about the definition. The operation that transforms between
   two CRSs may need resources no scene carries, such as a datum grid;
   requirement 21 says what happens then.*

2. **Defined once, and describing no object.**
   A complete CRS definition used in many places
   is authored once and referred to, describes no object-specific placement,
   orientation or scale, and has its authored WKT as the sole authored authority
   for the information that WKT determines.

   *The unit of sharing is a complete WKT value within a layer stack.
   Its embedded components do not compose independently: a WKT override replaces
   the complete value and can mask a later correction in a weaker layer.
   Extracting information for a query or cache does not create another authored
   authority, and normalization cannot infer which separate definitions were
   intended to change together.
   A site calibration and an asset placement can involve the same arithmetic —
   an origin, a rotation, a scale — so their roles must remain distinguishable.
   A calibration defines the shared coordinate system; a placement puts a
   particular object in it.*

3. **Datum, realization and epoch.**
   The scene can identify the datum realization and any frame reference epoch
   defined by its CRS, without inferring a coordinate epoch for the coordinates.

   *A dynamic reference frame's reference epoch and the coordinate epoch
   of a set expressed in that frame are different quantities.
   An ITRF2020 definition carries a frame reference epoch of 2015; that describes
   the frame, not the epoch of every coordinate set expressed in it.
   An observation timestamp or a USD time code does not by itself establish
   the coordinate epoch either. Representation and interpretation of coordinate
   epochs are roadmap work under question 15, outside the initial scope.*

4. **A site's own grid is a CRS like any other.**
   A project grid — its origin, orientation and scale relative to a geodetic or
   projected CRS, and, where applicable, its heights relative to a vertical CRS,
   agreed once for a site — can be the CRS its content is expressed in,
   and content in it needs nothing that content in a national grid does not.

   *Construction works in a ground system: an origin on the site,
   axes along the construction drawings rather than grid north,
   and a scale of one, so a metre in the field is a metre in the system.
   Such a grid typically reads (1000, 1000) at its origin
   so that no coordinate on site is negative.
   Those are properties of the site, shared by every discipline on it.
   Placing that system in the world takes an affine adjustment
   horizontally and an inclined plane vertically; carrying that relation
   with the grid's definition is what lets a reader tell
   grid distances from ground distances.
   This is site calibration, sometimes called localization: a derived CRS
   carries its base CRS and the conversion defining the site's grid.
   An engineering CRS with no known relation to an Earth-referenced system
   does not establish an Earth location. Site calibration defines a shared
   coordinate system, not the placement of an individual model.*

<!-- Start a separate list so the new identifier renders as 31. -->

31. **Same definition, same meaning.**
    CRS definitions are authored in the proposal's
    prescribed WKT normal form, so OGC-permitted syntactic variants of the same
    definition normalize to identical text without losing represented information
    or changing coordinate interpretation or resolved coordinate results.

    *One writer produces compact WKT and another adds indentation and line breaks;
    OGC WKT also permits keyword case and delimiter variants under its syntax rules.
    These differences need not imply different coordinate definitions.
    A prescribed normal form makes identical serialized definitions comparable,
    while requiring valid WKT from other tools to be normalized before conforming
    authoring. That is an additional interchange constraint, not a restriction OGC
    already imposes. Its profile is open question 10.
    Changing an axis order, unit or datum can change the meaning;
    normalization cannot erase those differences, or discard names and remarks
    merely to make text equal. Identical text does not establish that a requested
    coordinate operation can be omitted.*

**Attaching it to content**

5. **A discoverable CRS.**
   The CRS in which any authored position is expressed
   can be determined from the scene alone.

   *An easting of 481,948 with a northing of 3,767,521
   in metres is valid in all sixty WGS 84 northern UTM zones
   and names a different place on the Earth in each.
   Coordinates whose CRS must be supplied out of band
   are not approximately located. They are not located at all.*

6. **Declared for a subtree, not a prim.**
   A CRS declared once applies to the content beneath it,
   and part of that content can declare a different one.

   *Assembling georeferenced data nests it: an asset in its own CRS,
   inside a dataset in another, inside a scene in a third.
   Each keeps its own native CRS, shared by its descendants
   and different from what is around it.
   Repeating the declaration for every descendant risks a missed update
   among coordinates intended to share the same CRS.*

7. **Composition agnostic**
   CRS declarations and their scope are interpreted on the composed stage,
   following the composition and value-resolution rules of the
   [AOUSD USD Core Specification v1.0.1](https://github.com/aousd/specifications-public/blob/main/core/1.0.1/core_spec.md).

   *Equivalent composed stage data has the same geospatial interpretation,
   regardless of the layers or composition arcs used to produce it.
   Core defines how opinions compose and resolve; this proposal defines
   the CRS interpretation and scope applied to that result.*

8. **Brought-in data keeps its coordinates and its CRS.**
   Data authored in one CRS can be brought into a project working in another,
   and given project-specific placement, with its authored coordinate values
   and CRS unchanged and that placement preserved in any supported requested
   output CRS.

   *This is the hierarchy a GIS runs on: each asset's own CRS,
   the project's CRS, and the placement of the asset in it.
   A reprojected copy made on import is a second dataset to maintain;
   replacing the original with it loses the native representation.
   In a GIS workflow, native coordinates are converted on demand into the
   project's CRS, then project-specific rotation, scale and translation
   adjust their placement in that CRS.
   The native CRS interprets the coordinates still authored on the asset;
   preserving it is not a requirement to track earlier CRSs or conversions.
   CRS conversion establishes a placement with known coordinate meaning;
   an intentional project adjustment then acts on that placement, separately
   from the rotation or scale required by the conversion itself.
   These instance-specific adjustments are separate from corrections
   to the source asset's conventions, as required by requirement 15.
   If an imagery dataset is shifted to align with survey control in a project,
   asking for its locations in another supported CRS retains that adjusted
   placement; it does not return the unadjusted locations.*

**Saying where content is**

9. **Positions, and offsets from them.**
   Content is placed by a position, orientation and scale with defined meaning
   in a named CRS, and the content beneath that placement is authored and moved
   as ordinary scene offsets in scene distances by someone who need know no geodesy.

   *A simulation team georeferences a road scene once, at its origin,
   and dresses it by moving props with the ordinary widget in ordinary
   units; no prop carries a coordinate, and nobody dressing the scene
   touches geodesy. A door is offset from its building's origin the same way.
   The position is where those distances meet the Earth, and what is up
   there is up for all of them: converting the position alone and leaving
   the orientation as authored lays a building on its side at mid-latitudes.
   A scan records ground distances, while a projected grid may have a
   different distance scale and grid north; converting the model's origin
   alone does not align its geometry with survey control.
   Place a tower on a site and not one byte of the tower changes;
   move the position and everything beneath it moves with it.*

10. **Offsets along the axes of their position.**
    Offsets beneath a CRS placement have an orientation and scale discoverable
    from its authored placement and CRS without resolving into another CRS,
    and resolution accounts for changes in local axes and scale as well as position.

    *A grid's axes differ from true east and north by the grid's convergence
    and scale at that point. Reading grid offsets as east and north
    misplaces content by an amount that grows with the distance
    from the position to the geometry.
    An editing tool asked to move something one metre east
    needs those axes without resolving the whole scene.
    The source placement's meaning is discoverable from the authored scene;
    orientation and scale in another output CRS are derived by resolution.
    Bringing independently CRS-bound survey content into the calibrated site's
    output context must account for the local rotation and scale between those
    CRSs as well as converting its position. This does not establish that one
    local affine approximation suffices over an arbitrary extent; the result
    and approximation guarantees remain question 13.*

11. **Position or offset, and the scene says which.**
    Whether an authored location is a position in a CRS or an offset from its
    parent can be read from the scene, and a CRS position does not accumulate
    ancestor xformOps.

    *Two positions in one chain are two absolute statements,
    not a base and an offset.
    Authoring a building corner as an independent position,
    where an offset from the building was meant,
    misplaces it by the whole distance between the two positions.
    That is the most common way to misplace a georeferenced scene,
    and it is only detectable if the scene distinguishes the two.
    Excluding ancestor xformOps does not discard enclosing CRS context or
    prohibit intentional project placement under requirement 8; the carrier
    and composed ordering of that placement remain question 3.*

12. **No angle read as a length.**
    Every coordinate's unit, and the surface its height is measured from,
    are unambiguous, and no reading of the scene takes
    an angular coordinate as a scene distance.

    *A latitude of 48.8584 read as 48 metres passes a numeric plausibility
    check, and so does a height whose reference surface was never stated:
    ellipsoidal and gravity-related heights differ by tens of metres
    over most of the Earth. Geographic source positions are permitted under
    the decision on question 1, but their angular components are not ordinary
    length-valued translates. Question 2 must specify a placement representation
    that preserves this distinction, including for consumers that ignore
    geospatial information.*

13. **One axis mapping.**
    Which scene axis carries which CRS component is fixed by this proposal,
    the same for every CRS and every implementation,
    and never taken from the axis order a CRS declares:
    where a CRS has easting, northing and up,
    X carries easting, Y northing, Z up, right-handed.

    *EPSG:3006, a horizontal CRS, declares northing before easting.
    Following that order as X and Y would transpose the scene,
    and a transposed pair is usually still a valid coordinate,
    so inspection does not catch it.
    Left to implementations, each would pick its own.*

14. **Scene conventions stay the scene's.**
    A CRS binding changes neither the scene's units nor its up axis,
    and where a CRS's units or axes differ from the scene's,
    the relation between the two is defined by this proposal once,
    not by each implementation.

    *A State Plane CRS is in US survey feet under a scene declared in metres;
    a Y-up asset from a graphics pipeline sits in a Z-up survey.
    Each is a fixed relation, and an implementation that guessed it
    would place content at a scale or on its side.
    The decision on question 4 assigns authoring of unit and up-axis
    correctives to the writer or assembler, following UsdGeom.*

15. **Placement separate from conformance.**
    Where an instance sits is recorded separately
    from the corrections that adapt its source asset's conventions.

    *Place the same tower fifty times across a site
    and there are fifty survey records, all different,
    and one rotation correcting the asset's up axis, the same every time.
    Recording them together copies that rotation into fifty survey records,
    where it is indistinguishable from something a surveyor measured.*

**Resolving a scene**

16. **One CRS out.**
    Content expressed in any number of CRSs resolves into one output CRS
    selected by the consumer, with the scene able to name a default for
    consumers that make no selection.

    *Data aggregated from several CRSs is useful to a runtime only once
    it is normalized into one; which one can differ from one resolve
    to the next, but there is one.
    A pipeline crosses UTM zones 11N and 12N.
    Read in zone 11N without conversion, the 12N half lands
    away from the endpoints it shares on the ground.
    A GIS host has a project CRS of its own and wants a scene
    authored elsewhere in it, without editing the scene;
    a viewer with no opinion needs the scene to say what it expects.
    The consumer's choice determines the output under the decision on question
    8; it is not a conversion of a mandatory project-CRS result.
    This does not prescribe the engine's internal computational path.
    Authored project-specific placement must still be preserved under
    requirement 8; its carrier and ordering remain question 3.*

17. **The same answer for every consumer.**
    A world position, a bound, an instance, a physics body and a rendered
    image all come from the same resolution, and none of them needs a renderer.

    *The projection is just the projection: whether its result feeds
    a renderer or an analytics engine does not change it.
    "Does this work without a renderer" is the first question
    a GIS or AECO pipeline asks.
    A building that renders in the right place while a spatial query
    still answers from its unconverted coordinates is two scenes, not one.
    The same is true of image or grid samples: an analytical query and
    an optional visualization refer to the same resolved sample locations.*

18. **Coordinates back out.**
    Any resolved position can be reported as coordinates in any CRS
    the scene or the consumer names, and the placement of one prim
    relative to another can be asked for in the output CRS.

    *A GIS wants a surveyed corner back as latitude, longitude and height
    whether or not anything in the scene is expressed that way.
    An analytical product similarly needs its sample locations in the
    receiving GIS's CRS, with the measurements and times associated
    with the same samples.
    A picking tool asking where two independently placed objects sit
    relative to each other gets the wrong answer by walking the authored
    hierarchy across a position: with one prim at 100 and an independently
    placed child at 20, the hierarchy says 120 and the resolved scene says 20.*

19. **Resolution leaves the scene as authored.**
    Resolving a scene writes nothing into it —
    authored coordinates, placement values, CRS definitions and bindings are unchanged —
    and writing a resolved result out is a separate, explicit act
    that records the CRS it was written in and,
    for content that varies over time, how it was sampled.

    *Whatever the runtime does — the conversion, or disregarding the
    transforms above a position — requires writing nothing into the scene.
    Choosing another output CRS recomputes resolved position, orientation and
    scale; it does not replace the source placement with those derived values.
    Once a placement has been baked into a matrix,
    the intent behind it has collapsed and nothing is left to check against.
    A written-out result that records its CRS resolves again
    to the same place, and a re-resolve does not transform it twice.
    Matrices baked at sampled times and interpolated afterwards
    are not the operation requirement 20 describes,
    and nothing downstream can tell the two apart from the matrices alone.*

20. **Positions between recorded moments.**
    A position recorded as samples over time is interpolated on the recorded
    values, in the CRS they were recorded in,
    and converting the result to another CRS does not change the path.

    *Take samples on the WGS 84 equator, a degree of longitude
    either side of the prime meridian, both at zero ellipsoidal height.
    Interpolated in the recording CRS, the midpoint is on the surface.
    Converted to Earth-centered, Earth-fixed coordinates first
    and interpolated there, the midpoint is about 971 m inside the ellipsoid.
    Telemetry from GPS, AIS or ADS-B arrives as latitude, longitude and
    altitude over time; the decision on question 1 permits geographic source
    positions, while question 2 must define their placement encoding.
    A climate grid can instead have fixed positions and changing measurements:
    a new measurement time does not by itself move a sample, and this
    requirement does not define interpolation or resampling of its values.*

21. **Never placed by a guess.**
    A requested transformation outside the supported scope or one that
    cannot be computed — no engine, a definition
    that cannot be read or is unsupported, a missing grid,
    a point outside the transformation's domain of validity,
    a requested change of coordinate epoch that the operation does not model —
    never places content by a substitute, a result that could only be
    partly computed is a failure and not a partial placement,
    and the failure surfaces where it can be known: in validation
    for what the authored scene reveals, from the engine for what
    only resolution can discover.

    *Coordinate-epoch support is outside the initial scope even when an
    installed engine has a motion model. Grid use for a supported non-epoch
    operation is an engine responsibility, not by itself an excluded capability.
    A substituted matrix is indistinguishable from a computed one.
    A quiet fallback turns a missing grid file into content
    confidently in the wrong place by hundreds of metres.
    For a conversion that shifts by 1,000 m, a two-point batch
    whose second point falls outside the operation's domain and is left in place
    returns a plausible number 1,000 m from the intended one,
    in an array the caller has been told succeeded.
    An operation that ignores a requested coordinate-epoch change can likewise
    return plausible coordinates while failing to perform the requested operation.
    The definitions and bindings survive the failure,
    so the scene is recoverable in a tool that has what was missing.
    A malformed definition or a binding to nothing is visible when the
    scene is authored; whether a grid covers the point, or an engine
    is present at all, is not, and no authoring API can promise to say so.*

<!-- Start a separate list so the new identifier renders as 30. -->

30. **Measurements remain usable as data.**
    Georeferenced measurements remain accessible to consumers together with
    their associated positions and times, without requiring renderable geometry
    or a visualization, and coordinate resolution preserves those associations
    and measurement values.

    *An image, terrain model or climate grid carries values that a consumer
    can analyze to identify features or trends, not just colors to display.
    Adding or removing a visualization leaves those values and their
    associations intact.
    Derived products can be returned to a GIS with their coordinates,
    values and times still matched.
    This requires access and preservation; it does not prescribe an analysis
    algorithm, a storage format or interpolation of measurement values.*

**Staying usable at real sizes**

22. **A CRS suited to the project's size.**
    The scheme supports site, regional and global projects
    without requiring their geometry to be approximated
    by a single tangent plane.

    *Flattening a curved surface onto a tangent plane discards
    its vertical departure: using a spherical radius of 6,371 km,
    the small-distance approximation gives about 8 cm at 1 km
    and about 785 m at 100 km.
    A full 3D topocentric CRS retains that vertical component.
    A construction grid can also have ground-to-grid distortion;
    choosing it does not guarantee undistorted ground distances.*

23. **Detail that does not depend on location.**
    Changing only an asset's geospatial placement does not reduce
    the precision of its asset-relative geometry as authored.

    *At a UTM easting of 481,948 m, adjacent float32 values
    are 3.125 cm apart, so millimetre detail cannot be reliably
    distinguished in an absolute float32 coordinate at that magnitude.
    The same detail can be represented as an asset-relative offset.*

24. **Extent under one position is bounded and stated.**
    Content beneath one position is placed to within a stated distance
    of where placing each of its points would put it,
    and the extent over which that holds is stated.

    *A map is curved and a placement about one point is flat,
    and the difference grows with the square of the distance from the position:
    for a UTM position resolved into geocentric coordinates,
    a point 1 km along the grid lands about 8 cm from where
    converting it directly would put it.
    A stated tolerance implies a maximum extent under one position,
    and content larger than that is split across several.*

**Living alongside everything else**

25. **Additive for consumers that ignore it.**
    A consumer that does not interpret the geospatial information
    reads the same scene it would have read without it.

    *Such a consumer does not get a correctly placed scene.
    It reads the authored coordinates as they stand,
    which for a projected CRS puts content
    hundreds of kilometres from the origin.
    What this requires is only that adding the CRS information
    changed nothing for it.*

26. **Declares its dependency.**
    A scene whose correct placement depends on resolving CRSs says so,
    in a way a consumer can read without traversing the scene,
    and the declaration covers every piece of placed content in it.

    *To a consumer that ignores it, a scene of bare offsets
    near the centre of the planet is indistinguishable from a correct one.
    What the consumer does with the declaration — refuse, defer, warn —
    is its own call. The dependency is a property of the data,
    so an author who omits the declaration has a defect a tool can find.*

27. **Checkable before use.**
    What the scene itself establishes — a position nested beneath another
    position, a binding to no definition, content outside any CRS,
    a dependency declaration missing or left behind by a written-out result —
    can be detected in the authored scene without resolving it.

    *A description that nothing validates against is violated at render time.
    Because resolution writes nothing into the scene,
    the CRS intent is still present as data, and can be checked.
    What the scene does not record cannot be checked from it:
    plausible offsets authored along the wrong axes
    are caught by comparing against survey control, not by inspection.*

28. **A result says what produced it.**
    A resolved result can name the coordinate operation that produced it
    and the accuracy attributed to it.

    *Two datum operations between the same pair of CRSs
    can legitimately place the same point metres apart.
    Choosing an operation and managing its grids belongs to the engine;
    identifying that choice and its attributed accuracy belongs to the
    resolved result. Neither requires duplicating the CRS definition or
    authoring the engine's catalog and grid-management policy in USD.
    A stated agreement between two implementations means nothing
    until an adopter can tell an operation choice from numerical drift,
    and a lower-accuracy operation quietly substituted for an unavailable one,
    reporting success, is a failure hidden inside a result.*

29. **Implementable from the text alone.**
    Two implementations built from this proposal without consulting its authors
    give the same coordinate interpretation and, using equivalent coordinate
    operations, place the same scene within a stated agreement distance in the
    output CRS's units at a stated coordinate magnitude.

    *Exact agreement is not achievable: engines differ in grid handling
    and in floating-point operation order, and two engines can differ
    by metres because they have different datum operations available,
    neither in error. Different operation choices are reported under requirement
    28, rather than hidden inside a promise of identical engine results.
    What makes operations comparable and how agreement is measured remain
    question 13. A count of matching digits is not comparable
    across CRS families; a distance at a magnitude is.
    The survey control this data derives from is generally good to
    centimetres, so agreement at the millimetre scale sits below the source.*

**Illustrative geographic data workflows**

Geographic datasets can supply measurements for analysis as well as optional
visualizations, with derived products returned to a GIS:

- A city-scale satellite image has samples of longitude, latitude, height
  and measurement. A consumer identifies features from the measurement
  values and obtains their locations in a suitable project CRS;
  topocentric/ENU coordinates are one candidate.
- A global climate grid has samples of longitude, latitude, height,
  measurement and time. A consumer identifies trends while retaining the
  association between the measurements, sample locations and recorded times;
  ECEF is one candidate output for the global extent. Positions can stay
  fixed while measurements change.

Both cases need explicit coordinate units and height references, access to
the measurements independently of a visualization, and coordinates back
out in a named CRS. They illustrate requirements 8, 12, 17, 18, 19, 22 and 30
and support the geographic-source intent recorded in the decision on question 1;
the candidate outputs do not decide how native geographic positions are recorded
or whether the Target CRS may be geographic.

**Functional decisions and remaining questions**

<!--
Editing note for this table, for people and for agents alike.

Open question numbers are identifiers, frozen at the review baseline of the
pull request that introduced this section, and "open question N" anywhere
outside this file means this table, not the older list under Design
considerations.

- Do not renumber or reorder. A new open question takes the next unused number
  at the bottom of the table.
- A decided question keeps its number and its line; the "Decided against"
  column gains "Decided:" and a pointer to where the decision paragraph lives.
  Do not delete it.
- A decision is recorded in that paragraph and in the design or runtime text it
  changes. It is not recorded by rewording a requirement or a Term.
- When citing an open question anywhere outside this file, give number and a
  short name together: "open question 3, the asset's native CRS".
-->

An answer to any of these is argued as whether it meets the requirements named.
Decisions on questions 1 and 5 make existing design intent explicit; geographic
source placement was also illustrated in the October 2 discussion. Decisions on
questions 7 and 8 record that discussion. Their rows remain for traceability
rather than reopening those answers. Questions 3, 11, 13 and 14 separate existing
requirements or agreed behavior from the specific choices still to be made.
The proposal remains subject to author review.

| # | Question | Decided against |
|--:|---|---|
| 1 | May a model-placement position be recorded in a geographic CRS, or only in one with length axes? | 5, 8, 12, 17, 18, 19, 20, 22, 30; Decided: [geographic source positions](#decision-on-question-1-geographic-source-positions); encoding remains question 2 |
| 2 | How does the scene identify a CRS placement's position, orientation and scale: through a binding, marked xformOps or distinct attributes? | 9, 10, 11, 12, 20 |
| 3 | In what coordinate context is project-specific placement recorded, and how does it compose with CRS placement and ordinary xformOps when the requested output CRS changes? | 5, 6, 8, 9, 11, 15, 16, 19; Decided: [working context and semantic order](#decision-on-question-3-working-context-and-placement-order); carrier and composed ordering remain open |
| 4 | Whose job is the up-axis and unit correction, the writer's or the reader's? | 14, 15; Decided: [writer or assembler](#decision-on-question-4-authored-unit-and-up-axis-conformance) |
| 5 | Does the scene record where CRS coordinates give way to scene offsets, or does the binding determine it? | 11, 19, 27; Decided: [direct binding establishes the anchor](#decision-on-question-5-the-position-and-offset-boundary) |
| 6 | What scene-axis mapping applies to CRS component sets other than easting, northing and up? | 13 |
| 7 | Does site calibration (localization) need a construct of its own? | 2, 4; Decided: [CRS definition and binding](#decision-on-question-7-site-calibration) |
| 8 | Is the consumer's chosen CRS the one the scene resolves into, or a conversion of a result resolved into the CRS the scene names? | 16, 17, 21, 23; Decided: [consumer-selected output context](#decision-on-question-8-consumer-selected-output-context) |
| 9 | Can a scene resolve into a geographic CRS? | 16, 18, 22 |
| 10 | What WKT string normalization profile preserves the represented information, and what comparisons can its normal form establish? | 1, 2, 3, 21, 27, 28, 29, 31 |
| 11 | How are a measurement dataset's source coordinate properties associated with its CRS, separately from model-placement properties and ordinary geometry offsets? | 2, 5, 6, 7, 12, 18, 19, 21, 27, 30; Decided: [composed subtree scope](#decision-on-question-11-coordinate-domains-and-property-roles); measurement-coordinate association remains open, model-placement encoding is question 2; coordinate epochs are roadmap question 15 |
| 12 | What authored declaration exposes the composed scene's CRS dependency without traversal, including referenced or unloaded content and explicit export? | 7, 19, 25, 26, 27; Coverage is already all placed content under requirement 26; carrier and composition remain open |
| 13 | How is placement-approximation error established for resolved frames, geometry and bounds, and how is cross-engine agreement measured for comparable operations? | 17, 18, 21, 22, 23, 24, 28, 29; Decided: [engine responsibility and required result behavior](#decision-on-question-13-operation-and-resource-responsibility); approximation and comparison rules remain open |
| 14 | How does a time-sampled export record its sampling so a reader can distinguish it from resolution of the original authored samples? | 18, 19, 20, 27, 30; Decided: [export context and preservation](#decision-on-question-14-explicit-export); sampling-record representation remains open |
| 15 | How should a future extension represent and associate coordinate epochs, and what must it specify for epoch-dependent resolution? | 2, 3, 5, 7, 19, 21, 27, 28, 30; Roadmap: [coordinate epochs](#roadmap-question-15-coordinate-epochs), outside the initial scope |

Requirement 18 already permits geographic output coordinates. Question 9 concerns
the distinct use of a geographic CRS as the resolved scene context for geometry,
frames and bounds; the generic source/target transformation sketch does not
specify that behavior.

Open questions 2 and 3 must account for complete authored xform stacks, including
multiple translations, rotations, scales and pivots, rather than relying on an
unidentified translate. A runtime boundary that excludes ancestor xformOps and
an authoring helper that inserts a reset into the source are different behaviors.
Requirement 19 requires resolution to leave authored data unchanged; requirement
25 constrains the effect of adding geospatial information on unaware consumers.
The working-context role of ancestor CRS bindings is separate from transform
accumulation; it does not settle how project-specific placement is carried.

Open question 10 covers the additional authored-text constraint in requirement
31 and its tradeoffs: normalizing the serialized WKT string, without changing
the represented CRS definition. OGC's permitted syntax does not itself prescribe a unique
serialization. Complete-WKT identity, equivalence of an embedded CRS and the
identity of a requested coordinate operation are distinct comparisons.

For open question 6, requirement 13 already fixes X as easting, Y as northing
and Z as up. Northing-first source storage does not change that mapping.
The background lists other component sets, including geocentric X/Y/Z, but
does not fix their mapping to the authored placement representation. Geographic
source component mapping must likewise be specified; neither mapping decides
question 9's geographic scene output.

Open question 12 concerns the authored carrier and coverage required by
requirement 26, which already requires coverage of every piece of placed content.
Discovering bindings by traversing the scene is not the promised declaration,
and unloaded content is not evidence of no dependency. The authored declaration
and its composition remain unspecified; the CRS binding schema is not itself a
declaration readable without traversal.

#### Decision on question 1: geographic source positions

A model-placement position may be expressed in a geographic CRS, including as
samples over time. This makes explicit the geographic CRS support listed in
the background and the latitude/longitude/height anchor illustrated on October
2; it does not select new attributes. Angular position components are distinct
from the model's ordinary length-valued transforms and offsets. The placement
encoding remains question 2, its component mapping question 6, and source-space
interpolation follows requirement 20. Allowing a geographic source position does
not decide whether geographic coordinates provide the scene context for resolved
geometry, oriented frames or bounds under question 9.

#### Decision on question 5: the position and offset boundary

A direct CRS binding on model content establishes an anchor on the composed prim.
Its placement is expressed in that CRS; descendants inherit the coordinate
context while using ordinary scene offsets, until another direct binding
establishes a new anchor. An inherited CRS does not make each child translation
a new absolute CRS position. This is the boundary already defined by the Anchor and Position
and offset terms, binding inheritance and the complete scene example; no second
boundary declaration is required. Choosing which properties carry the anchor's
position, orientation and scale remains question 2. Question 3 specifies the
separate role of project adjustments; this boundary does not settle their carrier.

#### Decision on question 11: coordinate domains and property roles

CRS scope follows the composed subtree and the nearest direct binding under
requirements 6 and 7. Independently bound descendants keep their own source CRS;
several domains in one external container can be represented by separately bound
prims. Sharing that container does not require one CRS for all of them. Ordinary
model vertices remain local offsets under the UsdGeom design, not separate
absolute CRS positions merely because they lie in a bound subtree. Question 2
identifies model-placement properties and question 5 fixes their scope boundary.
Question 11 remains for identifying the source coordinates of non-visual
measurements with their CRS, including coordinates supplied by an external data
asset. The proposal has not specified that association just by requiring that
measurements, locations and times remain matched, and a binding does not make
every vector-valued property an absolute CRS position. Requirement 30 preserves
these associations without prescribing a storage format.

#### Decision on question 14: explicit export

Requirements 19 and 30 already specify the export obligations: an explicit
resolved export records its output CRS and, for time-varying content, how it was
sampled, while preserving measurement values and their position/time associations.
The output is authored data in that recorded CRS; rereading it must not repeat
the original source-to-output placement conversion or depend on a private
"already resolved" flag. Ordinary composition flattening alone does not perform
this export. The placement and measurement-coordinate encodings follow questions
2 and 11, and dependency-declaration maintenance follows question 12. The remaining
export-specific choice is how to record sampling so interpolation of exported
samples is not mistaken for resolving the original samples under requirement 20.

#### Decision on question 4: authored unit and up-axis conformance

The writer or scene assembler authors the correctives needed to bring model
content into the destination stage's units and up axis, following UsdGeom.
Correctives may be authored at the assembly boundary without rewriting the
source asset. Readers honor those authored transforms; geospatial resolution
does not automatically repair asset unit or up-axis mismatches. This conformance
is separate from instance placement under requirement 15 and from interpreting
the units declared by a CRS or performing a requested coordinate conversion.

#### Decision on question 3: working context and placement order

An enclosing project's binding supplies working context without replacing an
independently bound dataset's authored CRS or coordinates. In the Colorado
illustration, design uses the calibrated site grid and survey control retains
State Plane + NAVD88 coordinates. With an unchanged supported output CRS and no
separate project adjustment of the control, correcting site calibration changes
the design's resolved placement, not the control's.

CRS-aware placement includes position, orientation and scale. Resolving that
placement accounts for the change in local axes and scale between source and
output CRSs; moving the origin alone is insufficient. An intentional project
adjustment acts on the CRS-correct placement, and is distinct from both this
conversion and source unit/up-axis conformance. This is the semantic order
confirmed on October 2, not a selected property layout or matrix multiplication
order. The context in which an adjustment is authored, its USD carrier and its
composition with ordinary xformOps still need specification. In particular,
"after CRS conversion" does not mean reusing the same numeric adjustment in every
requested output CRS; requirement 8 preserves the adjusted placement itself.

#### Decision on question 7: site calibration

Site calibration, also called localization, relates a site's local grid to a
known Earth-referenced base CRS. A derived CRS records the base and the supported
conversion defining that grid, including horizontal and vertical calibration
where applicable. The shared CRS definition and its binding carry this meaning
under requirements 2 and 4; no separate USD localization construct is needed.
An unlocated engineering CRS alone does not supply this relationship. Individual
asset placement remains separate under requirements 8 and 15. This answers the
functional question discussed on October 2; it does not require an engine to
support every calibration method or remove requirement 21's resource checks.

#### Decision on question 8: consumer-selected output context

The consumer's requested CRS is the resolved output context; an
enclosing project binding does not require first producing a result in its CRS.
This defines the output's meaning, not an engine's internal computational path.
Authored project-specific placement must still be preserved under requirement 8;
its carrier and composed ordering remain question 3. The October 2 discussion
confirmed this as the intended behavior rather than a new output mode to design.
Whether the scene's output CRS may itself be geographic remains question 9.

#### Decision on question 13: operation and resource responsibility

The transformation engine selects applicable geodetic operations between the
authored source and selected output CRSs and manages the required grids or other
resources. The initial USD model need not mirror its operation catalog, download
policy or grid interpolation algorithms. CRS definitions remain self-contained
under requirement 1 and authoritative under requirement 2; that does not make
computing every transformation independent of external resources.

This records the responsibility agreed on October 2, not a guarantee that every
engine can perform every request or produces identical results from different
valid operations. Requirements 17, 21 and 28 already require shared placement for
visual and non-visual consumers, detectable failure without a substituted or
partial placement, and operation/accuracy reporting. Those behaviors are not
additional open decisions. Question 13 retains the method for establishing the
approximation distance and valid extent under requirement 24, and the agreement
measure for comparable operations under requirement 29; operation accuracy alone
does not measure placement approximation or supply a requested tolerance.
Comparison must distinguish different valid operations from numerical drift and
specify a distance measure for geographic coordinate outputs. Scene-authored
operation/resource controls and coordinate epochs remain roadmap work.

#### Roadmap question 15: coordinate epochs

Coordinate-epoch representation, interpretation and epoch-dependent resolution
are outside the initial scope, not foreclosed by it. A future extension
must answer two distinct parts:

- What is the authoritative epoch representation, which coordinate values does
  it describe, and how does the association behave through composition, reuse
  and export?
- What operation/motion-model information, resources and applicability are
  needed for a requested epoch-dependent result, and what must failure and
  result reporting establish?

COORDINATEMETADATA is an existing OGC representation to evaluate in that work,
not a requirement to introduce a separate USD epoch property. If distinct
metadata values embed the same CRS, complete-WKT sharing can repeat that CRS and
an epoch-only override can mask a later CRS correction in a weaker layer. The
future model must expose that tradeoff and respect the authored-authority rule
in requirement 2. It must keep coordinate epochs distinct from frame reference
epochs, observation times and USD time codes. The initial model must preserve a
path to adding this support; no future carrier or model is chosen here.

### Schema design

Two new schemas are introduced:

| Schema | Type | Purpose |
|--------|------|---------|
| `GeospatialCRS` | Typed (IsA) | Defines a CRS as a first-class USD prim |
| `GeospatialCRSBindingAPI` | Applied (HasA) | Binds a CRS prim to a `UsdGeomXformable` |

This two-schema approach separates the CRS *definition*
from the CRS *usage*,
allowing a single CRS definition to be shared
across many prims and scenes via USD references.

### CRS library pattern

CRS definitions are intended to live in shared **library layers**
(e.g., `crs_library.usda`) that can be referenced by any scene:

```
crs_library.usda
├── /CRS/WGS84_UTM11N       (GeospatialCRS)
├── /CRS/NAD83_UTM11N        (GeospatialCRS)
├── /CRS/NAD83_CA_Zone5      (GeospatialCRS)
└── /CRS/WGS84_Geographic3D  (GeospatialCRS)
```

This pattern is analogous to shared material libraries in M&E workflows.
Authoring tools can ship standard CRS libraries,
and users can create custom ones for local/site-specific CRS definitions.

### CRS binding and inheritance

The `GeospatialCRSBindingAPI` is applied to a `UsdGeomXformable` prim
and references a `GeospatialCRS` prim (from the same stage or an external layer).

CRS bindings inherit down the prim hierarchy.
A prim without a direct CRS binding
resolves its CRS by walking up to the nearest ancestor
that has one — analogous to `UsdShadeMaterialBindingAPI` resolution.

A child prim may override its parent's CRS
by applying its own `GeospatialCRSBindingAPI` with a different CRS reference.
This is how multi-CRS scenes are composed
(e.g., one subtree in UTM zone 11N, another in UTM zone 18N).

For model placement, a direct binding establishes an anchor, and an inherited
binding supplies coordinate context for ordinary offsets below it until the
next direct binding, as recorded in the decision on question 5. The position,
orientation and scale encoding at that anchor remains question 2. The following
translate-based examples illustrate the boundary; they do not settle that
encoding for arbitrary xform stacks or geographic positions.

### Precision handling

USD mesh geometry uses `point3f[]` (float32),
which provides ~7 decimal digits of precision.
A UTM easting of 481,948.63 requires 8+ significant digits,
so storing geospatial coordinates directly in `point3f` causes visible artifacts.

The solution is a two-tier approach:

| Tier | Data | Type | Precision |
|------|------|------|-----------|
| **Anchor** | Geospatial position | `double3 xformOp:translate` | float64 (~15 digits) |
| **Detail** | Mesh vertices | `point3f[] points` | float32 (~7 digits) |

The large CRS coordinates are carried in the double-precision
`xformOp:translate` on the CRS-bound `Xform` prim.
Mesh vertex positions are small offsets (typically < 1 km)
relative to that anchor, staying well within float32 range.

## Detailed design

### GeospatialCRS typed schema

A concrete typed schema that defines a CRS as a first-class USD prim.

**Prim type name:** `CoordinateReferenceSystem`

**Attributes:**

| Attribute | API Name | Type | Variability | Description |
|-----------|----------|------|-------------|-------------|
| `crs:wkt` | `wellKnownText` | `uniform token` | Uniform | OGC WKT 2 string (ISO 19162:2019) defining the CRS |

The `crs:wkt` attribute is `uniform` because a CRS definition
does not vary over time or across a mesh.
It is a `token` (not `string`) to enable efficient caching and comparison.

**USDA syntax:**

```usda
def CoordinateReferenceSystem "WGS84_UTM11N" (
    doc = "WGS 84 / UTM zone 11N (EPSG:32611)"
)
{
    uniform token crs:wkt = """PROJCRS["WGS 84 / UTM zone 11N",
        BASEGEOGCRS["WGS 84",
            DATUM["World Geodetic System 1984",
                ELLIPSOID["WGS 84",6378137,298.257223563,
                    LENGTHUNIT["metre",1.0]]],
            PRIMEMERIDIAN["Greenwich",0,
                ANGLEUNIT["degree",0.0174532925199433]],
            ID["EPSG",4326]],
        CONVERSION["UTM zone 11N",
            METHOD["Transverse Mercator",
                ID["EPSG",9807]],
            PARAMETER["Latitude of natural origin",0,
                ANGLEUNIT["degree",0.0174532925199433],
                ID["EPSG",8801]],
            PARAMETER["Longitude of natural origin",-117,
                ANGLEUNIT["degree",0.0174532925199433],
                ID["EPSG",8802]],
            PARAMETER["Scale factor at natural origin",0.9996,
                SCALEUNIT["unity",1.0],
                ID["EPSG",8805]],
            PARAMETER["False easting",500000,
                LENGTHUNIT["metre",1.0],
                ID["EPSG",8806]],
            PARAMETER["False northing",0,
                LENGTHUNIT["metre",1.0],
                ID["EPSG",8807]]],
        CS[Cartesian,2],
            AXIS["(E)",east,ORDER[1],
                LENGTHUNIT["metre",1.0]],
            AXIS["(N)",north,ORDER[2],
                LENGTHUNIT["metre",1.0]],
        ID["EPSG",32611]]"""
}
```

**C++ API:**

```cpp
class UsdGeospatialCoordinateReferenceSystem : public UsdTyped {
public:
    static UsdGeospatialCoordinateReferenceSystem
    Define(const UsdStagePtr &stage, const SdfPath &path);

    static UsdGeospatialCoordinateReferenceSystem
    Get(const UsdStagePtr &stage, const SdfPath &path);

    UsdAttribute GetWellKnownTextAttr() const;
    UsdAttribute CreateWellKnownTextAttr(
        VtValue const &defaultValue = VtValue(),
        bool writeSparsely = false) const;

    /// Walk up the prim hierarchy from `prim` and return the WKT token
    /// of the first ancestor (or self) with a valid crs:wkt attribute.
    static TfToken ComputeCoordinateReferenceSystem(UsdPrim const &prim);
};
```

### GeospatialCRSBindingAPI applied schema

A single-apply API schema that binds a CRS to a `UsdGeomXformable` prim.

**Schema type name:** `BindingAPI`
**Can only apply to:** `Xformable`

**Behavior:**

When `Bind()` is called, the schema:
1. Adds a USD **reference** to the specified `GeospatialCRS` prim
   (internal or external).
2. Calls `SetResetXformStack()` on the prim,
   ensuring `!resetXformStack!` appears in the `xformOpOrder`,
   so the prim's `xformOp:translate` is interpreted
   as an absolute position in the bound CRS.

**C++ API:**

```cpp
class UsdGeospatialBindingAPI : public UsdAPISchemaBase {
public:
    static UsdGeospatialBindingAPI Apply(const UsdPrim &prim);
    static bool CanApply(const UsdPrim &prim,
                         std::string *whyNot = nullptr);

    /// Bind via internal reference to a CRS prim on the same stage.
    bool Bind(UsdGeospatialCoordinateReferenceSystem const &crs) const;

    /// Bind via external reference to a CRS prim in another layer.
    bool Bind(std::string const &layerPath,
              SdfPath const &crsPrimPath) const;
};
```

### Complete scene example

The following example places a simplified building
at the Esri headquarters in Redlands, California (34.0561°N, 117.1956°W),
using WGS 84 / UTM zone 11N (EPSG:32611).

```usda
#usda 1.0
(
    defaultPrim = "World"
    metersPerUnit = 1
    upAxis = "Z"
)

# ── Stage root: CRS-bound to UTM 11N ────────────────────────────────
def Xform "World" (
    apiSchemas = ["GeospatialCRSBindingAPI"]
    references = [@./crs_library.usda@</CRS/WGS84_UTM11N>]
)
{
    uniform token[] xformOpOrder = ["!resetXformStack!", "xformOp:translate"]

    # Esri HQ — UTM 11N coordinates (metres)
    double3 xformOp:translate = (481948.63, 3768393.52, 400)

    # ── Building inherits CRS from /World ────────────────────────────
    def Xform "EsriCampus" (
        references = [@./esri_hq_building.usda@</EsriHQ>]
    )
    {
        # Local offset (metres) — stays within float32 range
        double3 xformOp:translate = (0, 0, 0)
        uniform token[] xformOpOrder = ["xformOp:translate"]
    }

    # ── Different CRS zone in the same scene ─────────────────────────
    def Xform "NewYorkCity" (
        apiSchemas = ["GeospatialCRSBindingAPI"]
        references = [@./crs_library.usda@</CRS/WGS84_UTM18N>]
    )
    {
        uniform token[] xformOpOrder = ["!resetXformStack!", "xformOp:translate"]
        # Times Square — UTM 18N coordinates (metres)
        double3 xformOp:translate = (583960, 4507523, 10)

        def Xform "TimesSquareBuilding"
        {
            double3 xformOp:translate = (0, 0, 0)
            uniform token[] xformOpOrder = ["xformOp:translate"]

            def Mesh "Walls"
            {
                int[] faceVertexCounts = [4, 4, 4, 4, 4, 4]
                int[] faceVertexIndices = [
                    0,1,2,3, 4,7,6,5, 0,4,5,1,
                    1,5,6,2, 2,6,7,3, 4,0,3,7
                ]
                point3f[] points = [
                    (-25,-15,0),(25,-15,0),(25,15,0),(-25,15,0),
                    (-25,-15,60),(25,-15,60),(25,15,60),(-25,15,60)
                ]
            }
        }
    }
}
```

Key observations:
- `/World` is bound to UTM 11N (EPSG:32611) — the scene's default **Target CRS**
  when the consumer does not select another output context.
- `/World/EsriCampus` has no CRS binding — it **inherits** UTM 11N from `/World`.
- `/World/NewYorkCity` overrides with UTM 18N (EPSG:32618).
  With that default output, its geometry would be **reprojected** to UTM 11N.
- Mesh vertices (`point3f[]`) are small local offsets.
  Large geospatial positions are in `double3 xformOp:translate`.

### CRS library example

```usda
#usda 1.0

def "CRS" {
    def CoordinateReferenceSystem "WGS84_UTM11N"
    {
        uniform token crs:wkt = """PROJCRS["WGS 84 / UTM zone 11N",
            BASEGEOGCRS["WGS 84", ...],
            CONVERSION["UTM zone 11N", ...],
            ID["EPSG",32611]]"""
    }

    def CoordinateReferenceSystem "WGS84_UTM18N"
    {
        uniform token crs:wkt = """PROJCRS["WGS 84 / UTM zone 18N",
            BASEGEOGCRS["WGS 84", ...],
            CONVERSION["UTM zone 18N", ...],
            ID["EPSG",32618]]"""
    }

    def CoordinateReferenceSystem "NAD83_CA_Zone5_ftUS"
    {
        uniform token crs:wkt = """PROJCRS["NAD83 / California zone 5 (ftUS)",
            BASEGEOGCRS["NAD83", ...],
            CONVERSION["SPCS83 California zone 5", ...],
            ID["EPSG",2229]]"""
    }
}
```

### Programmatic authoring (C++)

```cpp
#include "pxr/usd/usdGeospatial/coordinateReferenceSystem.h"
#include "pxr/usd/usdGeospatial/bindingAPI.h"
#include "pxr/usd/usdGeom/xform.h"

// Define a CRS prim
auto crs = UsdGeospatialCoordinateReferenceSystem::Define(
    stage, SdfPath("/CRS/WGS84_UTM11N"));
crs.CreateWellKnownTextAttr().Set(VtValue(TfToken(wktString)));

// Create a geolocated Xform
auto xform = UsdGeomXform::Define(stage, SdfPath("/World"));
auto api = UsdGeospatialBindingAPI::Apply(xform.GetPrim());
api.Bind(crs);

// Set geospatial coordinates (UTM 11N, metres)
UsdGeomXformOp translateOp = xform.AddTranslateOp(
    UsdGeomXformOp::PrecisionDouble);
translateOp.Set(GfVec3d(481948.63, 3768393.52, 400.0));

// Resolve CRS for any prim (walks up hierarchy)
TfToken wkt = UsdGeospatialCoordinateReferenceSystem
    ::ComputeCoordinateReferenceSystem(somePrim);
```

### Programmatic authoring (Python)

```python
from pxr import Usd, UsdGeom, UsdGeospatial, Gf

stage = Usd.Stage.CreateNew("scene.usda")
stage.SetMetadata("metersPerUnit", 1.0)
UsdGeom.SetStageUpAxis(stage, UsdGeom.Tokens.z)

# Define CRS
crs = UsdGeospatial.CoordinateReferenceSystem.Define(
    stage, "/CRS/WGS84_UTM11N")
crs.CreateWellKnownTextAttr().Set(wkt_string)

# Bind to Xform
world = UsdGeom.Xform.Define(stage, "/World")
stage.SetDefaultPrim(world.GetPrim())
api = UsdGeospatial.BindingAPI.Apply(world.GetPrim())
api.Bind(crs)

# Set geospatial position
world.AddTranslateOp(UsdGeomXformOp.PrecisionDouble).Set(
    Gf.Vec3d(481948.63, 3768393.52, 400.0))
```

## Runtime coordinate transformation

### When coordinates must be harmonized

There are several runtime situations where coordinates defined in different CRS need to be harmonized:

- rendering
- stage queries (e.g. bounding boxes)
- indirect transformation requests (e.g. upon stage flattening)
- explicit transformation requests using the new API calls

Requirements 17 and 18 already require rendering and non-visual queries to use
the same resolution and permit coordinate results in a named geographic CRS.
That query support does not define geographic scene frames or bounds (question
9). Resolution itself writes nothing into the scene. An explicit resolved export
instead authors output-CRS data and records its sampling and measurement/time
associations under requirements 19 and 30; ordinary composition flattening is
not coordinate conversion. Question 14 concerns the sampling record, not whether
these context and preservation obligations apply.

### The transformation abstraction

We propose to introduce an internal abstraction to compute coordinate transformations from one CRS to another:

```text
transform(inout ArrayVector3d coordinates, in WKT crsIn, in WKT crsOut)
```

The OpenUSD plugin system is employed to register implementations.

### Target CRS and runtime reprojection

The **Target CRS** is the consumer-selected resolved output context under
requirement 16 and the decision on question 8. The existing scene sketch uses
the CRS bound to the composed `defaultPrim` as the default when a consumer
makes no selection; it does not override a consumer's choice or require an
intermediate project-CRS result.

Resolution derives coordinates and model placement in that output CRS,
accounting for local orientation and scale as well as position under
requirements 9 and 10. The source CRS bindings and authored placement remain
unchanged under requirement 19. Intentional project adjustments retain their
meaning under requirement 8; their carrier and composed ordering remain
question 3. Model position/orientation/scale property roles remain question 2;
source measurement-coordinate association remains question 11. A directly bound
model prim establishes the anchor under question 5's decision; inherited model
transforms remain ordinary offsets rather than new absolute CRS positions.

This reprojection is implemented as a
**Hydra 2.0 Scene Index Filter** (for rendering)
and as API methods (for computation),
using an abstracted interface to third-party CRS libraries
(PROJ, GDAL, Esri projection engine, etc.).

### Default Implementation using the "PROJ" library

The OpenSource [PROJ](https://proj.org/) library can be used to create a bidirectional "transformer" object by parsing the USD WKT2 string and defining the target Coordinate Reference System (CRS) by its EPSG code.

The following sample code, using the `pyproj` library (a common Python binding for PROJ), demonstrates how to create a transformer between the Derived CRS with Affine Site Calibration WKT string provided in the document and the target EPSG:10499.

```python
import pyproj

# 1. Define the Coordinate Reference Systems (CRS)

# 1a. Create a CRS object from the USD WKT string.
crs_from = pyproj.CRS.from_wkt(USD_WKT_STRING)

# 1b. Create a CRS object from the target EPSG code.
crs_to = pyproj.CRS.from_string("EPSG:10499")

# 2. Create the Transformer
transformer = pyproj.Transformer.from_crs(crs_from, crs_to)

# Example usage
target_x, target_y, target_z = transformer.transform(usd_x, usd_z, usd_y)
```

## Interaction with existing USD features

### Stage metadata: metersPerUnit and upAxis

Under requirement 14, a CRS binding changes neither `metersPerUnit` nor `upAxis`.
The decision on question 4 assigns authored unit and up-axis correctives to the
writer or assembler; readers honor those transforms. This is separate from
interpreting coordinates in their declared CRS units or performing a requested
coordinate conversion. Scene conventions do not relabel source CRS coordinates.

Requirement 13 fixes the logical mapping for easting, northing and up,
independently of source storage order. Functional open question 6 concerns
mappings for other CRS component sets.

### Transform stack and resetXformStack

The `GeospatialCRSBindingAPI` emits `!resetXformStack!`
at the beginning of the `xformOpOrder` for CRS-bound prims.
This ensures that the `xformOp:translate` value
is interpreted as an **absolute position** in the bound CRS,
not as a relative offset from a parent transform.

Without this reset, USD's standard transform concatenation
would add the geospatial coordinates of parent and child,
producing nonsensical world-space positions.

### Composition arcs

CRS bindings compose through standard USD composition arcs:

- **References:** CRS library prims are imported via `references`.
- **Sublayers:** CRS libraries can be included as sublayers.
- **Inherits:** CRS class prims can be inherited
  (as demonstrated in the POC implementations).
- **Payloads:** CRS bindings survive payload loading/unloading.

The standard USD composition order (LIVRPS) applies;
a stronger arc can override a weaker arc's CRS binding.

### Relationship to UsdGeom

The `GeospatialCRSBindingAPI` is restricted to `UsdGeomXformable` prims.
It does not modify `UsdGeomMesh`, `UsdGeomPoints`, or other geometry schemas.
Geometry prims remain unchanged —
their vertex positions are always local offsets
relative to the nearest ancestor `Xform`.

## Industry use cases

### AECO

Building Information Modeling (BIM) workflows require
placing architectural models at surveyed site coordinates.
A hospital designed in Revit or ArchiCAD
must be positioned at its planned construction site
for clash detection, permitting, and construction coordination.

With this proposal, a BIM model exported to USD
retains its real-world position via CRS metadata,
enabling seamless integration with GIS site maps
and other geolocated assets.

### GIS and digital twins

Digital twin platforms aggregate data from dozens of sources
(LiDAR, photogrammetry, BIM, IoT sensors)
into a unified 3D view of a city, campus, or infrastructure network.
All this data arrives in various CRS —
the platform must reproject everything into a common frame.

This proposal provides the standard mechanism
for each USD layer to declare its CRS,
enabling the digital twin platform to compose and reproject
automatically rather than relying on manual coordinate transformations.

### Infrastructure and utilities

Pipeline, rail, and utility networks span hundreds of kilometres,
often crossing multiple UTM zones or State Plane regions.
A single USD stage must compose assets from different CRS zones
and display them correctly in a unified view.

The multi-CRS composition and runtime reprojection
described in this proposal directly address this use case.

### Defense and simulation

Military simulation and training environments
require precise geolocation of terrain, buildings, and vehicles
in CRS tied to national geodetic reference frames.
Dynamic datums and coordinate epochs
(supported via WKT 2's `COORDINATEMETADATA`)
are essential for high-precision positioning.

## Interoperability

### glTF geospatial extension

The Khronos Group is developing a geospatial extension for glTF
that encodes CRS metadata on nodes.
The approach is conceptually similar to this proposal
(CRS definition + binding to scene graph nodes).
Alignment between the USD and glTF approaches
would facilitate round-trip exchange between the two formats.

### IFC and BIM workflows

IFC (Industry Foundation Classes) is the open standard for BIM data.
IFC 5 is evaluating USD as a potential geometry backbone.
IFC's `IfcMapConversion` and `IfcProjectedCRS` entities
map directly to this proposal's `GeospatialCRS` and `GeospatialCRSBindingAPI`.
A standard USD CRS mechanism would simplify IFC-to-USD conversion.

### CityGML and OGC 3D Tiles

CityGML and OGC 3D Tiles both carry CRS metadata.
Converting these formats to USD currently requires
discarding or side-channeling CRS information.
This proposal preserves it as first-class scene data.

## Design considerations

### Why WKT and not bare EPSG codes

The proposal uses full OGC WKT 2 strings
rather than simple EPSG integer codes for several reasons:

1. **Self-contained.**
   A WKT string carries the complete CRS definition.
   No external registry lookup is needed at runtime.

2. **Supports custom CRS.**
   Site calibration grids, local engineering CRS,
   and derived projected CRS have no EPSG code.
   WKT can represent any CRS.

3. **Authority IDs are embedded.**
   The WKT `ID["EPSG", 32611]` clause provides
   the familiar integer code for tools that prefer it,
   so nothing is lost.

4. **Dynamic datums.**
   WKT 2 supports `DYNAMIC[FRAMEEPOCH[...]]` and `EPOCH[...]`
   for time-dependent reference frames — critical for
   high-precision surveying and tectonic-plate-aware applications.

5. **ISO standard.**
   OGC WKT 2 is formally standardized as ISO 19162:2019,
   ensuring long-term stability and broad industry support.

### Why a typed prim and not stage metadata

CRS could theoretically be stored as stage-level metadata
(like `metersPerUnit`).
This was rejected because:

1. **A stage often contains multiple CRS zones.**
   Stage metadata is a single value;
   a prim-based approach supports different CRS
   at different points in the hierarchy.

2. **Composition.**
   Prim-level CRS composes through references and sublayers.
   Stage metadata has limited composition semantics.

3. **Reuse.**
   A CRS prim can be referenced by many scenes.
   Stage metadata must be duplicated.

### Alternate approaches considered

| Approach | Mechanism | Pros | Cons |
|----------|-----------|------|------|
| **A: Primvar** | `asset primvars:geolocation:crs` | Auto-inheritance via primvar system | Requires custom asset-path resolution; non-standard prim types |
| **B: String primvar + inherits** | `string primvars:geolocation:crs:wkt` + `inherits` | Simplest implementation; leverages both inherits and primvar inheritance | WKT duplicated on every inheriting prim in flattened stage |
| **C: Abstract class + plain attribute** | `class` prims with `crs:wkt` attribute | Idiomatic USD class usage | No automatic inheritance for plain attributes; requires manual parent-walking |
| **D: Typed prim + applied API (this proposal)** | `GeospatialCRS` prim + `GeospatialCRSBindingAPI` | Clean separation of definition and usage; reference-based binding; API-driven inheritance | Requires new schema registration |

Approach D was selected because it provides the cleanest separation
of concerns, composes naturally through USD references,
and follows established patterns
(cf. `UsdShadeMaterialBindingAPI`).

The other approaches (A, B, C) were explored in
[proof-of-concept implementations](https://github.com/mistafunk/aousd-geospatial-pocs)
and informed the final design.

### Status of earlier design questions

These entries reconcile the earlier design list with the functional decisions
above. Their numbers are local to this list; they are not functional question
identifiers. Implementation choices and additional interoperability work do not
reopen settled coordinate behavior.

1. **Unit and up-axis conformance.**
   The decision on functional question 4 assigns authored correctives to the
   writer or assembler. It does not change the CRS's declared coordinate meaning.

2. **Axis mapping.**
   Requirement 13 fixes easting/northing/up independently of source storage or
   declared CRS axis order. Functional open question 6 concerns mappings for
   other CRS component sets.

3. **Third-party library abstraction.**
   Callable signatures and plugin registration belong to implementations.
   The runtime sketch does not define a standard programming interface.
   Requirement 21 already requires detectable failure, with no substitute or
   partial placement, and requirement 28 requires operation/accuracy reporting.
   Coordinate-epoch representation is deferred under functional question 15;
   the wrapper example does not settle it for the initial model.

4. **WKT validation.**
   Requirements 21 and 27 distinguish checks possible on authored data from
   failures discoverable only during resolution; requirement 31 also requires
   checking the prescribed normal form. These are observable validation
   obligations, not a requirement that USD's core parser interpret WKT on every
   stage read. The normalization profile remains functional question 10.

5. **Single-precision geometry.**
   The anchor/detail design and requirement 23 already preserve authored local
   detail at geospatial magnitudes; requirement 24 bounds placement error over
   an extent. Double-precision geometry support is additional work, not a
   prerequisite imposed by this proposal on every georeferenced asset.

6. **Reprojection performance.**
   Caching, LOD and batching are implementation choices. They must preserve
   shared placement, supported failure behavior and stated approximation limits
   under requirements 17, 21 and 24; no standard cache or Hydra strategy is
   selected here.

7. **glTF interop.**
   Alignment remains interoperability work. This proposal does not define a
   formal USD/glTF mapping; such work must preserve the coordinate and placement
   meaning established here rather than select a second geospatial model.

8. **IFC 5 requirements.**
   Additional IFC use cases can inform future requirements and mapping work.
   The mention of IFC does not supply missing model-placement, axis-mapping or
   sampling rules for the functional questions above.

### Risks

1. **WKT complexity.**
   WKT strings are verbose and easy to author incorrectly.
   Mitigation: provide standard CRS library files
   and authoring-tool validation.

2. **Third-party dependency.**
   Correct reprojection requires PROJ or an equivalent library.
   If no CRS library is available, the runtime cannot reproject.
   Mitigation: graceful degradation — CRS metadata is preserved
   even without a reprojection engine.

3. **Performance.**
   Per-prim reprojection at render time
   could be expensive for large scenes.
   Mitigation: implement caching, pre-transform at export,
   and batch reprojection in the Scene Index Filter.

4. **Adoption resistance.**
   M&E users who do not need geospatial features
   may perceive this as unnecessary complexity.
   Mitigation: the schemas are optional and additive —
   they do not affect scenes that do not use them.

## Relationship to other proposals

- **[OpenExec](../openexec/README.md):**
  Geospatial reprojection could leverage the OpenExec framework
  for deferred evaluation of CRS transformations.

- **[Semantic Schema](../semantic_schema/README.md):**
  Geospatial prims could carry semantic labels
  (e.g., "building", "road", "terrain") for GIS classification.

- **[Identifier Separation of Concerns](../identifier_separation_of_concerns/README.md):**
  BIM/GIS assets often carry source identifiers
  (IFC GlobalId, GIS feature ID) that should survive
  round-trip through USD.

- **[Revise Use of Layer Metadata](../revise_use_of_layer_metadata/README.md):**
  Relevant to the discussion of whether CRS belongs at
  the stage level vs. prim level.

## Prototype implementations

Working prototype implementations exist:

| Implementation | Approach | Repository |
|----------------|----------|------------|
| **usdGeospatial schema** | C++ typed schema + applied API, PROJ integration | [mistafunk/USD (geospatial-prototype branch)](https://github.com/mistafunk/USD/tree/geospatial-prototype/pxr/usd/usdGeospatial) |
| **POC: Asset primvar** | Python + primvar with asset path | [mistafunk/aousd-geospatial-pocs (David de Koning)](https://github.com/mistafunk/aousd-geospatial-pocs) |
| **POC: String primvar + inherits** | Python + WKT in primvar + class inherits | [mistafunk/aousd-geospatial-pocs (David de Koning)](https://github.com/mistafunk/aousd-geospatial-pocs) |
| **POC: Class inheritance** | Python + abstract class with manual parent-walk | [mistafunk/aousd-geospatial-pocs (Simon Haegler)](https://github.com/mistafunk/aousd-geospatial-pocs) |
| **Esri HQ placement demo** | Python + usd-core + pyproj, USDA scene files | This proposal's accompanying files |

## Next steps

1. **Gather community feedback** on this proposal
   through the OpenUSD-proposals review process.

2. **Formalize the schema definition** (`schema.usda`)
   and register the `usdGeospatial` library
   in the OpenUSD build system.

3. **Settle what the transformation abstraction must carry**
   beyond coordinates — epoch, orientation, and failure reporting.

4. **Collaborate with the Geometry Working Group**
   on double-precision geometry support.

5. **Implement the Hydra 2.0 Scene Index Filter**
   for runtime CRS reprojection.

6. **Produce interoperability guidelines**
   for glTF, IFC, CityGML, and 3D Tiles exchange.

7. **Ship standard CRS library files**
   with common EPSG definitions.

8. **Make the sample WKT strings consistent**
   in their use of WKT v2 elements,
   so the same CRS is described the same way throughout.

9. **Demonstrate the accuracy claim**
   by extending the PROJ sample above
   into a worked conversion from USD coordinates
   to a national CRS, with the expected accuracy stated
   and checked.

## References

| Resource | Link |
|----------|------|
| AOUSD Geospatial CRS Working Document | [Google Doc](https://docs.google.com/document/d/1v9A5SCSz_yvoExFgb9kJ5qPo4CAAGZtXnvS2ZptfvXc) |
| OGC WKT-CRS Standard (ISO 19162:2019) | [OGC 18-010r7](https://docs.ogc.org/is/18-010r7/18-010r7.html) |
| OGC Abstract Spec: CRS (ISO 19111) | [OGC 18-058](https://docs.ogc.org/is/18-058/18-058.html) |
| EPSG Geodetic Parameter Registry | [epsg.org](https://epsg.org/) |
| Esri: Coordinate Systems — What's the Difference? | [ArcGIS Blog](https://www.esri.com/arcgis-blog/products/arcgis-pro/mapping/coordinate-systems-difference) |
| PROJ Library | [proj.org](https://proj.org/) |
| usdGeospatial Prototype (C++) | [GitHub](https://github.com/mistafunk/USD/tree/geospatial-prototype/pxr/usd/usdGeospatial) |
| Geospatial POC Implementations | [GitHub](https://github.com/mistafunk/aousd-geospatial-pocs) |
| OpenUSD | [GitHub](https://github.com/PixarAnimationStudios/OpenUSD) |
| AOUSD Geospatial Presentation | [Google Slides](https://docs.google.com/presentation/d/13hVKSXQjJ1IAAj2ZLQL8klVAHC22GBcqNvV2WYqRWRY) |

## Appendix A: WKT examples

These are reference encodings of common CRS types. The two examples worked
against the case study are in [WKT examples](#wkt-examples) above.

### WGS 84 / UTM zone 11N (EPSG:32611)

```lisp
PROJCRS["WGS 84 / UTM zone 11N",
    BASEGEOGCRS["WGS 84",
        DATUM["World Geodetic System 1984",
            ELLIPSOID["WGS 84",6378137,298.257223563,
                LENGTHUNIT["metre",1.0]]],
        PRIMEMERIDIAN["Greenwich",0,
            ANGLEUNIT["degree",0.0174532925199433]],
        ID["EPSG",4326]],
    CONVERSION["UTM zone 11N",
        METHOD["Transverse Mercator",
            ID["EPSG",9807]],
        PARAMETER["Latitude of natural origin",0,
            ANGLEUNIT["degree",0.0174532925199433],
            ID["EPSG",8801]],
        PARAMETER["Longitude of natural origin",-117,
            ANGLEUNIT["degree",0.0174532925199433],
            ID["EPSG",8802]],
        PARAMETER["Scale factor at natural origin",0.9996,
            SCALEUNIT["unity",1.0],
            ID["EPSG",8805]],
        PARAMETER["False easting",500000,
            LENGTHUNIT["metre",1.0],
            ID["EPSG",8806]],
        PARAMETER["False northing",0,
            LENGTHUNIT["metre",1.0],
            ID["EPSG",8807]]],
    CS[Cartesian,2],
        AXIS["(E)",east,ORDER[1],
            LENGTHUNIT["metre",1.0]],
        AXIS["(N)",north,ORDER[2],
            LENGTHUNIT["metre",1.0]],
    ID["EPSG",32611]]
```

### Compound CRS: NAD83 / California zone 5 (ftUS) + NAVD88 height

```lisp
COMPOUNDCRS["NAD83 / California zone 5 (ftUS) + NAVD88 height (ftUS)",
    PROJCRS["NAD83 / California zone 5 (ftUS)",
        BASEGEOGCRS["NAD83",
            DATUM["North American Datum 1983",
                ELLIPSOID["GRS 1980",6378137,298.257222101,
                    LENGTHUNIT["metre",1.0]]],
            ID["EPSG",4269]],
        CONVERSION["SPCS83 California zone 5 (US Survey feet)",
            METHOD["Lambert Conic Conformal (2SP)",
                ID["EPSG",9802]],
            PARAMETER["Latitude of false origin",33.5,
                ANGLEUNIT["degree",0.0174532925199433]],
            PARAMETER["Longitude of false origin",-118,
                ANGLEUNIT["degree",0.0174532925199433]],
            PARAMETER["Latitude of 1st standard parallel",35.4666666666667,
                ANGLEUNIT["degree",0.0174532925199433]],
            PARAMETER["Latitude of 2nd standard parallel",34.0333333333333,
                ANGLEUNIT["degree",0.0174532925199433]],
            PARAMETER["Easting at false origin",6561666.667,
                LENGTHUNIT["US survey foot",0.304800609601219]],
            PARAMETER["Northing at false origin",1640416.667,
                LENGTHUNIT["US survey foot",0.304800609601219]]],
        CS[Cartesian,2],
            AXIS["(E)",east,LENGTHUNIT["US survey foot",0.304800609601219]],
            AXIS["(N)",north,LENGTHUNIT["US survey foot",0.304800609601219]],
        ID["EPSG",2229]],
    VERTCRS["NAVD88 height (ftUS)",
        VDATUM["North American Vertical Datum 1988"],
        CS[vertical,1],
            AXIS["gravity-related height (H)",up,
                LENGTHUNIT["US survey foot",0.304800609601219]],
        ID["EPSG",6360]]]
```

### 3D Geographic with dynamic datum and epoch

This `COORDINATEMETADATA` example illustrates roadmap question 15, not an
initial-scope coordinate-epoch encoding or a conforming CRS-only `crs:wkt` value.

```lisp
COORDINATEMETADATA[
    GEOGCRS["WGS 84 (G2296)",
        DYNAMIC[FRAMEEPOCH[2024]],
        DATUM["World Geodetic System 1984 (G2296)",
            ELLIPSOID["WGS 84",6378137,298.257223563,
                LENGTHUNIT["metre",1.0]]],
        PRIMEMERIDIAN["Greenwich",0,
            ANGLEUNIT["degree",0.0174532925199433]],
        CS[ellipsoidal,3],
            AXIS["geodetic latitude (Lat)",north,ORDER[1],
                ANGLEUNIT["degree",0.0174532925199433]],
            AXIS["geodetic longitude (Lon)",east,ORDER[2],
                ANGLEUNIT["degree",0.0174532925199433]],
            AXIS["ellipsoidal height (h)",up,ORDER[3],
                LENGTHUNIT["metre",1.0]],
        ID["EPSG",10605]],
    EPOCH[2026.0]]
```

### 3D Geocentric (ECEF) with dynamic datum

```lisp
GEODCRS["ITRF2020",
    DYNAMIC[FRAMEEPOCH[2015]],
    DATUM["International Terrestrial Reference Frame 2020",
        ELLIPSOID["GRS 1980",6378137,298.257222101,
            LENGTHUNIT["metre",1.0]]],
    PRIMEMERIDIAN["Greenwich",0,
        ANGLEUNIT["degree",0.0174532925199433]],
    CS[Cartesian,3],
        AXIS["(X)",geocentricX,ORDER[1],
            LENGTHUNIT["metre",1.0]],
        AXIS["(Y)",geocentricY,ORDER[2],
            LENGTHUNIT["metre",1.0]],
        AXIS["(Z)",geocentricZ,ORDER[3],
            LENGTHUNIT["metre",1.0]],
    ID["EPSG",9990]]
```

## Appendix B: AI-assisted drafting

This proposal was drafted with the assistance of Claude (Anthropic).
The AI was provided with the AOUSD Geospatial CRS working document,
the existing POC implementations, the usdGeospatial prototype README,
OGC standards documentation, and the OpenUSD proposals format guidelines.
All technical content was reviewed, verified,
and refined by the human authors.
