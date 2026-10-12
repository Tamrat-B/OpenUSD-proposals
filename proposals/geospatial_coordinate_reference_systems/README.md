# Geospatial Coordinate Reference Systems for OpenUSD

**Requirements baseline for review.** This revision establishes terms,
functional requirements, initial scope and roadmap. It records agreed direction
without approving a particular field encoding or implementation architecture.
The detailed authored model and normative runtime contract belong to the next
alignment round. These requirements are their acceptance criteria; this
requirements-only revision is not a complete implementation specification.

## Contributors

- **Esri** (Tamrat Belayneh, Simon Haegler)
- **Nvidia** (Aaron Luk)
- The case study, the WKT encoding section and the runtime coordinate
  transformation section are Sébastien Vielliard's work.
- Also draws on work by David de Koning, Sébastien Vielliard, Simon Haegler
  and Tamrat Belayneh in the AOUSD AECO Interest Group.

## Introduction

Geospatial information lets USD content state where it belongs on Earth or
another celestial body, while preserving the coordinates and CRS in which it
was authored. A shared, self-contained WKT definition supplies coordinate
meaning. Resolution supports both visual scenes and non-visual measurements,
with the same placement available to rendering, queries and analysis.

This baseline specifies the required outcomes. Geographic source positions,
input-only geospatial placement and ordinary model-local offsets have distinct
roles. The follow-up model and runtime contract will specify their exact USD
representations and evaluation, measured against the requirements here.

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

### Families of CRS

ISO 19111:2019 (*Referencing by coordinates*) is the model behind every
encoding used here: a CRS is a **coordinate system** — axes with directions
and units — attached to a **datum** that fixes those axes to the Earth.
OGC WKT 2 is its text serialization, and EPSG codes identify its well-known
instances.

| Family | Coordinate system | Origin | Typical use | Example |
|--------|-------------------|--------|-------------|---------|
| **Geographic** | `CS[ellipsoidal]`, angular + height | Ellipsoid | GNSS output, web mapping, global exchange | WGS 84 (EPSG:4979) |
| **Geocentric** | `CS[Cartesian, 3]`, linear | Earth's centre of mass | GNSS processing, datum transformation | ITRF2020 (EPSG:9988) |
| **Projected** | `CS[Cartesian, 2]`, linear | Projection false origin | National mapping, GIS, civil engineering | UTM zone 11N (EPSG:32611) |
| **Engineering** | `CS[Cartesian, 2\|3]`, linear | Arbitrary, stated by the datum | BIM and CAD authoring, plant layouts | Construction site grid |

A projected CRS always contains a base geographic CRS, a projection method
(e.g., Transverse Mercator, Lambert Conformal Conic) and its parameters.
An engineering CRS has no geodetic relationship of its own — architects
author in one from the first day of a project — and acquires one only when
something anchors it.

A **derived CRS** is any of these obtained from another by a named
conversion: a site calibration over a projected CRS, a topocentric plane
over a geographic one. It is a complete CRS, not a transformation layered
on one.

#### Grid and ground coordinates

A projection cannot flatten a curved surface without distorting it.

**Distance.** The **grid scale factor** of the projection times the
**elevation factor** — the survey is measured at the site's height, the
projection computed on the ellipsoid — gives the **combined scale factor**.
A **grid** coordinate carries that distortion; a **ground** coordinate
matches a tape measure on site. Near Paris on Lambert-93 at 77.5 m the
factor is 0.999881 (−118.7 ppm): a kilometre on the ground is 999.881 m on
the grid.

**Direction.** **Grid convergence** is the angle between grid north — the
northing axis of the projection — and true north. At the same Lambert-93
site it is about 31 arcmin, corresponding to some 9 m over a kilometre.
A bearing read off a national grid is a grid bearing. This describes the
projected CRS's axes; it does not assign north to a particular USD stage axis
or redefine the model's geodetic attitude. Their exact representation belongs to the follow-up model and runtime contract.

Survey and construction work in ground coordinates, regional GIS in grid
coordinates. A CRS states which of the two its numbers are and where its
axes point, and both are properties of the site, not of any object placed
in it.

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

A CRS used for 3D model placement must give three coordinate components and a
computable link to a geodetic datum, so that every position it records can be
converted to a geodetic one. How the third component is defined depends on the
type: an axis of the geocentric frame, an ellipsoidal height, or a height in a
declared vertical CRS. A 2D CRS does not qualify on its own, nor does a local
grid tied to the Earth only by text. Exact component and placement-basis
conventions are a follow-up design obligation.

The types below run from global to local; each is defined from the one above it.

| Type | WKT 2 | Components | Third component | Link to the geodetic datum | One scene unit is |
|------|-------|------------|-----------------|----------------------------|-------------------|
| **Geocentric (ECEF)** | `GEODCRS`, `CS[Cartesian, 3]` | X, Y, Z | Cartesian axis; no height | Direct | GLOBE |
| **Geographic 3D** | `GEOGCRS`, `CS[ellipsoidal, 3]` | longitude, latitude, height | Ellipsoidal height | Direct | — (angular) |
| **Geographic compound** | `COMPOUNDCRS` (`GEOGCRS` 2D + `VERTCRS`) | longitude, latitude, height | Gravity-related height | Direct; ellipsoidal height through the vertical datum's geoid model | — (angular) |
| **Projected compound** | `COMPOUNDCRS` (`PROJCRS` + `VERTCRS`) | easting, northing, height | Gravity-related height | Map projection of the base geographic CRS | GRID |
| **Topocentric** | Derived from a `BASEGEOGCRS`, EPSG 9837 | east, north, up | Up along the ellipsoid normal at the origin | Tangent plane at a stated latitude, longitude and height | GROUND |
| **Local site, calibrated** | `DERIVEDPROJCRS`, EPSG 9624 (+ EPSG 1046 vertical) | site X, Y, Z | Calibrated height | Affine fit over a projected compound CRS | GROUND |
| **Local site, engineering only** | `ENGCRS` with `EDATUM[ANCHOR[...]]` | site X, Y, Z | Arbitrary | Free text only: **not computable** | GROUND |

**GLOBE** means one scene unit is a length in a single Cartesian frame for the
whole Earth, **GRID** one unit of a map projection, **GROUND** one unit measured
on site (see [grid and ground coordinates](#grid-and-ground-coordinates)).
The last row is the native state of a BIM export; it can be placed once a site
calibration derives it from a projected CRS, which is how ISO/TS 15143-4 models
a worksite localization.

What a CRS may describe depends on the property, not on its type. A source
position may be recorded in any of the computable types above, in the units the
WKT declares — degrees and metres for a geographic CRS. Geometry and ordinary
xformOps are always lengths in scene units, and no reading of the scene takes an
angular coordinate as a scene distance
([requirement 12](#functional-requirements)). A resolution that cannot be
completed — including an out-of-domain or non-finite result on a geographic
path — is a detectable failure, not a placement
([requirement 21](#functional-requirements)).

The 2D UTM example in [Appendix A](#wgs-84--utm-zone-11n-epsg32611) illustrates a horizontal CRS definition,
which can be components of a complete 3D definition. They are not complete
3D model-placement bindings: adding a numeric third component does not supply
a vertical CRS, and a reader must not infer zero height or an ellipsoidal height
reference. This clarification does not define a separate 2D placement mode or
settle the measurement-coordinate association in question 11.

For dynamic datums, a CRS definition can carry a frame reference epoch in
`DYNAMIC[FRAMEEPOCH[...]]`. A coordinate epoch is separate: the
`COORDINATEMETADATA` wrapper with `EPOCH[...]` illustrates roadmap question 15,
outside the initial CRS-only scope.

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

## Encoding the CRS: OGC WKT 2 (ISO 19162:2019)

We propose updating the OpenUSD schema to support describing the Coordinate Reference System (CRS) as a self-contained [WKT v2.1.11](https://docs.ogc.org/is/18-010r11/18-010r11.pdf) string ("Well-known text representation of coordinate reference systems"). This is equivalent to [ISO 19162:2019](https://www.iso.org/standard/76496.html).

This section covers the encoding only. The functional outcomes are in
[Design overview](#design-overview). Exact schema and binding definitions
belong to the model/runtime follow-up.

Key benefits:

- Standardization: Uses a mature, ISO-compliant format widely adopted in GIS and engineering.
- Self-Contained: Encodes the CRS definition (ellipsoid, datum, projection, and units) in a single string, without requiring a registry lookup to read that definition; computing a transformation may still need the engine's operation database or grids.
- Precision: Supports accurate definition of any Coordinate Reference Systems.

## Terms

The background above covers the geodesy.
This section fixes the words this proposal uses for its own constructs,
and separates several that mean different things
to the industries this proposal serves.

### Terms this proposal defines

**CRS binding.**
An association between a named CRS and the content of a prim and its subtree
down to the next direct binding. It gives explicitly identified CRS positions
their coordinate meaning; it does not turn ordinary model offsets or every
vector-valued property into absolute CRS coordinates.

**Native CRS.**
The CRS in which an asset's currently authored coordinates are expressed.
It gives those coordinate values their meaning;
it is not a record of earlier CRSs or of the asset's conversion history.

**Anchor.**
The model-placement prim at which CRS coordinates enter a scene:
its own location is a position in the CRS bound to it.
Model geometry beneath an anchor, up to but excluding any nested anchor,
is positioned relative to it by offsets in ordinary scene units,
with no CRS coordinates of its own.
Measurement-coordinate properties have a separate role under
question 11 in the functional question table.
What constrains the CRS an anchor may be bound to is a design question,
not part of the term.

**Position and offset.**
A *position* is a coordinate in a CRS; an *offset* is a distance
from another prim, in scene units.
An anchor's location is a position. Model geometry beneath it uses offsets,
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
   The composed scene must carry the CRS information needed to interpret its coordinates
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
   can supply coordinate operations. The proposal specifies data and observable
   runtime behavior, including validation and resolved queries; it does not
   prescribe callable interfaces or a particular implementation architecture.

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

Existing identifiers remain unchanged through merge and the follow-up review.
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
an implementation architecture. Design sketches and prototype code do not settle the open
questions or establish that the model is complete. An applied API schema is a USD data-model category,
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
coordinate epochs. Epoch representations remain roadmap candidates, not initial
conformance obligations. Exact placement, WKT normalization, coordinate-role
and dependency-carrier definitions belong to the follow-up review; proposing a
definition and agreeing to it are distinct steps.

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
    CRS definitions use the proposal's prescribed WKT string normal form, whose
    specified lexical variants normalize to identical text without loss of
    represented information or changed coordinate interpretation, while
    comparisons distinguish serialized-definition identity from CRS equivalence
    and do not infer different CRS meaning or a necessary transformation solely
    from different normalized text.

    *One writer produces compact WKT and another adds indentation and line breaks;
    OGC WKT also permits keyword case and delimiter variants under its syntax rules.
    These differences need not imply different coordinate definitions.
    A prescribed normal form makes identical serialized definitions comparable,
    while requiring valid WKT from other tools to be normalized before conforming
    authoring. That is an additional interchange constraint, not a restriction OGC
    already imposes. The proposed profile is under review in question 10.
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
    prohibit intentional project placement under requirement 8; the
    coordinate context of the anchor's project adjustment is defined under
    question 3; its ordinary stack applies after CRS placement and resolution.*

12. **No angle read as a length.**
    Every coordinate's unit, and the surface its height is measured from,
    are unambiguous, and no reading of the scene takes
    an angular coordinate as a scene distance.

    *A latitude of 48.8584 read as 48 metres passes a numeric plausibility
    check, and so does a height whose reference surface was never stated:
    ellipsoidal and gravity-related heights differ by tens of metres
    over most of the Earth. Geographic source positions are permitted under
    the decision on question 1, but their angular components are not ordinary
    length-valued translates. The distinct placement properties proposed under
    question 2 preserve this distinction, including for consumers that ignore
    geospatial information.*

13. **One axis mapping.**
    The semantic component order is specified once for each supported CRS
    coordinate system independently of its declared storage order, with
    easting/northing/up in that order where those components apply, and its scene
    representation follows the common convention under requirement 14.

    *EPSG:3006 declares northing before easting. Reading its storage order as
    scene X/Y transposes plausible coordinates. Geographic and geocentric
    positions likewise need explicit semantic tuples; their components are not
    automatically easting/northing/up. Coordinate tuples and their representation
    in stage axes remain distinct.*

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
    requirement 8; the adjustment's coordinate context is proposed under question 3.*

17. **The same answer for every consumer.**
    World positions, bounds, instances, physics bodies and rendered images use
    the same resolution without requiring a renderer, and a derived ordinary
    USD copy that preserves resolved placement over stated spatial and time
    coverage must be available for viewers performing no geospatial computation.

    *A building that renders in the right place while a spatial query uses its
    unconverted coordinates is two scenes, not one. The same applies to image
    and grid samples. The source remains an input-only scene; library-free
    viewers use an explicit bake into ordinary geometry and transforms instead
    of computed properties stored beside the source inputs. Such a bake must
    retain local detail at geospatial magnitudes under requirement 23 and state
    its approximation coverage under requirement 24; an affine matrix alone is
    insufficient when the conversion is nonlinear over that coverage.*

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
    positions, whose placement encoding is proposed under question 2.
    A climate grid can instead have fixed positions and changing measurements:
    a new measurement time does not by itself move a sample, and this
    requirement does not define interpolation or resampling of its values.*

21. **Never placed by a guess.**
    A requested transformation outside the supported scope or one that
    cannot be computed — no engine, a definition
    that cannot be read or is unsupported, a missing grid,
    a point outside the transformation's domain of validity,
    invalid placement data or a non-finite result, including on a geographic
    path, a requested coordinate-epoch operation outside scope —
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
    A malformed definition, an invalidly authored placement field or a
    binding to nothing is visible in the authored scene. Such a field cannot be
    silently ignored to produce a successful placement. Grid coverage and engine
    availability are resolution checks; a non-finite engine result is a failure,
    whether or not the engine raises an exception.*

<!-- Start a separate list so the new identifier renders as 30. -->

30. **Measurements remain usable as data.**
    Georeferenced measurements remain accessible to consumers together with
    their associated positions and times, without requiring renderable geometry
    or a visualization, and coordinate resolution preserves measurement values
    and their associations using explicitly declared coordinate domains and
    dimension ordering rather than incidental array storage order.

    *An image, terrain model or climate grid carries values that a consumer
    can analyze to identify features or trends, not just colors to display.
    Adding or removing a visualization leaves those values and their
    associations intact.
    Derived products can be returned to a GIS with their coordinates,
    values and times still matched.
    Coordinate arrays can declare a different dimension order from the
    measurement variable; matching their flattened indices independently can
    attach a plausible location to the wrong measurement. Dimension declarations
    in the native format can supply that order: this does not require duplicated
    USD metadata. This requires access and preservation, not an analysis
    algorithm, storage format or interpolation of measurement values.*

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
    It reads ordinary local geometry and xformOps with their existing meaning;
    it does not apply the separate CRS placement attributes.
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
    an invalid placement field, or a dependency declaration missing or
    left behind by a written-out result —
    can be detected in the authored scene without computing CRS transformations.

    *A description that nothing validates against is violated at render time.
    Because resolution writes nothing into the scene,
    the CRS intent is still present as data, and can be checked.
    Detecting nested positions does not by itself make nesting invalid:
    question 5 permits direct bindings that establish independent anchors.
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
    Operation-attributed accuracy, placement-approximation distance under
    requirement 24 and implementation agreement under requirement 29 are
    distinct quantities.
    A stated agreement between two implementations means nothing
    until an adopter can tell an operation choice from numerical drift,
    and a lower-accuracy operation quietly substituted for an unavailable one,
    reporting success, is a failure hidden inside a result.*

29. **Implementable from the text alone.**
    Two implementations built from this proposal without consulting its authors
    give the same coordinate interpretation and, using equivalent coordinate
    operations, place the same scene within a stated agreement distance using
    a specified distance measure and units at stated output coordinate magnitudes.

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
rather than reopening those answers. Questions 2, 3, 11, 13 and 14 separate existing
requirements or agreed behavior from the specific choices still to be made.
The proposal remains subject to author review.

| # | Question | Requirements and status |
|--:|---|---|
| 1 | Are geographic source positions permitted? | 5, 8, 12, 17–20, 22, 30; Decided: geographic source positions, separate from ordinary length-valued geometry and translates. |
| 2 | How are source position and physical attitude recorded? | 9–12, 20; Input-only meaning agreed; no `crs:scale`; quaternion versus one heading/pitch/roll tuple remains a follow-up choice. |
| 3 | How do project adjustments retain meaning when the output CRS changes? | 5, 6, 8, 9, 11, 15, 16, 19; Anchor project adjustments and descendant model-local meaning agreed; exact chart/reset/instance rules require follow-up review. |
| 4 | Who authors unit and up-axis corrections? | 14, 15; Decided: writer or assembler. |
| 5 | Where do absolute positions give way to relative model offsets? | 11, 19, 27; Decided direction: a direct model binding establishes the anchor, independently of CRS equality. |
| 6 | What are the component and scene-frame conventions? | 12–14; One explicit convention required; supported component profile and mappings belong to the follow-up; south-oriented axes are a known extension. |
| 7 | Does site calibration need its own USD mechanism? | 2, 4; Decided: shared CRS definition and binding for the initial scope. |
| 8 | Who chooses the output CRS? | 16, 17, 21, 23; Decided: consumer, with a scene-provided default. |
| 9 | How do geographic coordinate results relate to Cartesian scene geometry, frames and bounds? | 12, 16–18, 22; Exact scene-chart representation requires follow-up alignment. |
| 10 | Which WKT string normal form and comparison rules apply? | 1–3, 21, 27–29, 31; A lossless normal form and distinct identity/equivalence comparisons are required; the detailed profile belongs to the follow-up. |
| 11 | How are measurements associated with their native coordinates? | 2, 5–8, 12, 18, 19, 21, 27, 30; Explicit domains and dimension ordering required; exact carrier and format profiles require follow-up review. |
| 12 | Which authored carrier exposes dependency without traversal? | 7, 19, 25–27; Complete conservative coverage required; Profiles carrier and maintenance details belong to the follow-up. |
| 13 | How are extent error and cross-implementation agreement established? | 17, 18, 21–24, 28, 29; Engine operation/resource responsibility decided; detailed coverage and distance conventions require follow-up alignment. |
| 14 | How does a bake record coordinate context, origin and sampling? | 17, 19, 20, 23, 24, 27, 30; Explicit, precision-preserving bake required; detailed representation and sampling guarantees belong to the follow-up. |
| 15 | How should coordinate epochs be added later? | 2, 3, 5, 7, 19, 21, 27, 28, 30; Roadmap, outside initial scope and not foreclosed by initial choices. |

The table records direction and review boundaries. Approving these requirements
does not approve an exact carrier, stored orientation type or complete runtime
algorithm. The detailed follow-up must meet them without inventing scene facts.

## Recorded direction

Geographic source positions, including time samples, are permitted. Their
angular components remain distinct from ordinary USD distances. Consumers
that ignore geospatial information retain the ordinary interpretation of
geometry and transforms; resolution leaves authored inputs unchanged.

Source placement records physical meaning, while output orientation,
projection convergence and distance scale are computed results. The model
must not add source properties for those results. The anchor's ordinary
transforms express project adjustments; descendants retain their ordinary
model-local meaning. The stored representation and complete evaluation remain
the follow-up contract, rather than implicit rules supplied by a prototype.

#### Decision on question 4: authored unit and up-axis conformance

The writer or scene assembler authors the correctives needed to bring model
content into the destination stage's units and up axis, following UsdGeom.
Correctives may be authored at the assembly boundary without rewriting the
source asset. Readers honor those authored transforms; geospatial resolution
does not automatically repair asset unit or up-axis mismatches. This conformance
is separate from instance placement under requirement 15 and from interpreting
the units declared by a CRS or performing a requested coordinate conversion.

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
its adjustment coordinate context is proposed under question 3. The October 2 discussion
confirmed this as the intended behavior rather than a new output mode to design.
Question 9 retains the scene-chart representation for geographic outputs as follow-up work.

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
Coordinate-epoch support for these workflows is roadmap question 15;
CRS frame reference epochs remain in the initial scope.

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

## Next alignment

Merge the terms and functional requirements as the baseline. Review the
authored model and normative runtime contract against it in the follow-up,
including orientation storage and the remaining detailed conventions. Selected
build-loop demonstrations and explicit counterexamples inform that review;
prototype successes and failures remain separate from agreement.

Schema registration, consumer integrations, interoperability mappings and
additional geometry precision support are implementation or subsequent work,
not a prescribed architecture or prerequisites added to the initial scope.

## References

| Resource | Link |
|----------|------|
| AOUSD Geospatial CRS Working Document | [Google Doc](https://docs.google.com/document/d/1v9A5SCSz_yvoExFgb9kJ5qPo4CAAGZtXnvS2ZptfvXc) |
| OGC WKT-CRS Standard (ISO 19162:2019) | [OGC 18-010r11](https://docs.ogc.org/is/18-010r11/18-010r11.pdf) |
| OGC Abstract Spec: CRS (ISO 19111) | [OGC 18-058](https://docs.ogc.org/is/18-058/18-058.html) |
| EPSG Geodetic Parameter Registry | [epsg.org](https://epsg.org/) |
| Esri: Coordinate Systems — What's the Difference? | [ArcGIS Blog](https://www.esri.com/arcgis-blog/products/arcgis-pro/mapping/coordinate-systems-difference) |
| PROJ Library | [proj.org](https://proj.org/) |
| usdGeospatial Prototype (C++) | [GitHub](https://github.com/mistafunk/USD/tree/geospatial-prototype/pxr/usd/usdGeospatial) |
| Geospatial POC Implementations | [GitHub](https://github.com/mistafunk/aousd-geospatial-pocs) |
| OpenUSD | [GitHub](https://github.com/PixarAnimationStudios/OpenUSD) |
| AOUSD Geospatial Presentation | [Google Slides](https://docs.google.com/presentation/d/13hVKSXQjJ1IAAj2ZLQL8klVAHC22GBcqNvV2WYqRWRY) |

## Appendix A: WKT examples

These are expanded illustrations of common CRS types, not pre-normalized
authored tokens. The exact normalization profile is part of the follow-up. The 2D UTM definition illustrates a horizontal
component, not a complete 3D model-placement CRS. The case-study encodings and exact supported component profile belong
to the model/runtime follow-up.

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
