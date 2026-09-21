# Geometry and Geography Data Types

Champion: TBD

Authors: 
* [Sander Bylemans](https://github.com/SBylemans)

Slack: TBD.

GitHub issue: [244](https://github.com/bitol-io/open-data-contract-standard/issues/244)

Applies to:
* [x] ODCS - Open Data Contract Standard
* [ ] ODPS - Open Data Product Standard
* [ ] OORS - Open Observability Results Standard
* [ ] OOCS - Open Orchestration and Control Standard
* [ ] OMMS - Open Maturity Model Standard
* [ ] OMDS - Open Metadata Difference Standard

## Summary

This RFC introduces two new values for `logicalType` in ODCS — `geometry` and `geography` — together with a dedicated set of `logicalTypeOptions` (`subType`, `crs`, `dimensions`, `algorithm`, `encoding`, `bbox`, `orientation`, and `epoch`). `geometry` represents shapes in a flat-earth (planar/Euclidean) coordinate system; `geography` represents coordinates on a round-earth (spherical/ellipsoidal) model. Both align with ISO 19125-1 (Simple Features for SQL), Apache Iceberg v3, GeoArrow, GeoParquet 2.0, and the Apache Parquet native `GEOMETRY` / `GEOGRAPHY` logical types. The physical encoding format (WKT, WKB, GeoJSON, etc.) is captured by the `encoding` option in `logicalTypeOptions`, while `physicalType` carries the target system's native column type.

## Motivation

### Why are we doing this?

Geospatial data is a first-class data shape in modern analytics, logistics, real estate, infrastructure, and scientific datasets. Virtually every major database and data lakehouse ships a native geometric or geographic type — PostGIS, BigQuery `GEOGRAPHY`, Snowflake `GEOGRAPHY`, Databricks (Delta Lake + Iceberg v3), DuckDB, Apache Sedona, Hive, Presto/Trino, and others. Formats like GeoParquet 2.0 and GeoArrow have standardized the columnar representation of geospatial data, and Apache Parquet now ships native `GEOMETRY` and `GEOGRAPHY` logical types.

ODCS today has no standard way to describe a geospatial column. Authors are forced to use `logicalType: string` (for WKT) or leave the column type opaque, which:

- masks whether the data is a point, polygon, or line,
- gives no hint about the coordinate reference system (CRS) a consumer needs to interpret the values, and
- defeats the interoperability goal of the standard.

Adding `geometry` and `geography` as first-class logical types matches the precedent set by Apache Iceberg v3, which added exactly these two types, and makes ODCS contracts machine-readable for geospatial tooling.

### Use cases

1. **Parcel and land-registry data**: A data contract declares a `parcel_boundary` column as `logicalType: geometry` with `subType: Polygon` and `crs: EPSG:28992`, letting GIS tools load the correct projection without manual configuration.
2. **Ride-sharing and logistics**: A `pickup_location` column is declared as `logicalType: geography` (round-earth), ensuring that distance calculations account for Earth's curvature.
3. **Sensor and IoT data**: A `gps_track` column is typed `logicalType: geography`, `subType: LineString`, `dimensions: 3` (XYZ with altitude), enabling spatial analytics over device trajectories.
4. **Data lakehouse migration**: Teams migrating geospatial tables from PostGIS to Snowflake or from Hive to Iceberg v3 use the contract to generate correct DDL without hand-editing.
5. **Multi-system interoperability**: GeoParquet and GeoArrow consumers discover from the contract which CRS and geometry type to expect, removing the need to inspect file-level metadata separately.

### Alignment with guiding values

- **Small standard over large**: This RFC adds two `logicalType` values and six `logicalTypeOptions`. No new top-level structures.
- **Interoperability over readability**: The shape maps onto PostGIS, BigQuery, Snowflake, Databricks, DuckDB, Apache Sedona, GeoParquet, GeoArrow, and Apache Iceberg v3.
- **Non-breaking**: `geometry` and `geography` are new optional `logicalType` values. Existing contracts are unaffected.

## Design and examples

### Core concept

Geospatial data comes in two flavours:

- **`geometry`** — uses a flat-earth (Euclidean/planar) coordinate system. Coordinates are interpreted as points on a plane using straight-line arithmetic. Fast and suitable for local or projected coordinate systems (e.g., engineering drawings, city-scale maps).
- **`geography`** — uses a round-earth (geodetic/spherical or ellipsoidal) model. Coordinates are longitude/latitude on Earth's surface. Distance and area calculations account for the Earth's curvature. Preferred for global or cross-regional data.

The physical encoding (how the bytes are laid out on disk) is separate from both the logical type and the target system's column type. It is captured by the `encoding` option in `logicalTypeOptions`:

| `encoding` value | Description                                                                                                    |
| ---------------- | -------------------------------------------------------------------------------------------------------------- |
| `wkb`            | Well-Known Binary — compact binary encoding defined by OGC. **Canonical** for GeoParquet 2.0 and Parquet native `GEOMETRY` / `GEOGRAPHY` (the only encoding they accept). |
| `wkt`            | Well-Known Text — human-readable string, e.g. `POINT (4.9 52.4)`                                             |
| `geojson`        | GeoJSON encoding (JSON object with `type` and `coordinates`)                                                   |
| `ewkt`           | Extended WKT — PostGIS extension that embeds the SRID in the string                                            |
| `ewkb`           | Extended WKB — PostGIS extension that embeds the SRID in binary form                                           |

`physicalType` carries the target system's native column type (e.g. `GEOMETRY` in PostGIS, `GEOGRAPHY` in BigQuery, `STRING` in Databricks). See the [Physical type mapping](#physical-type-mapping) section below.

### New `logicalType` values

| Value        | Description                                                                                          |
| ------------ | ---------------------------------------------------------------------------------------------------- |
| `geometry`   | A geometric shape in a flat-earth (planar/Euclidean) coordinate system, per ISO 19125-1.            |
| `geography`  | A geographic shape on a round-earth (geodetic) model, using longitude/latitude coordinates.          |

### `logicalTypeOptions` for `geometry` and `geography`

| Option       | Applies to            | Required | Type    | Description                                                                                                                                                                                                                                        |
| ------------ | --------------------- | -------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `subType`    | geometry, geography   | No       | string  | The geometry subtype per ISO 19125-1. One of `Point`, `LineString`, `Polygon`, `MultiPoint`, `MultiLineString`, `MultiPolygon`, `GeometryCollection`. When omitted, any subtype is accepted.                                                       |
| `crs`         | geometry, geography   | Yes (geometry) / No (geography) | string  | The Coordinate Reference System. Accepted forms: authority codes (e.g. `EPSG:4326`, `OGC:CRS84`) — **recommended**; OGC URN identifiers (e.g. `urn:ogc:def:crs:EPSG::4326`); an inline PROJJSON document (as a JSON string); a `srid:<n>` SRID reference (e.g. `srid:0` for unspecified); or a `projjson:<key>` reference to a PROJJSON blob stored elsewhere in metadata. Required for `geometry` — no universal default exists for planar coordinate systems. When omitted for `geography`, `EPSG:4326` (WGS 84) is assumed; note that GeoParquet 2.0 and Parquet native geospatial use the equivalent `OGC:CRS84` to make the (longitude, latitude) axis order explicit. |
| `dimensions` | geometry, geography   | No       | integer | Number of coordinate dimensions: `2` (XY, default), `3` (XYZ or XYM), `4` (XYZM).                                                                                                                                                                |
| `algorithm`   | geography only        | No       | string  | Interpretation of edges between vertices. One of `spherical` (great-circle arcs on a sphere, default), `vincenty` (Vincenty's formulae on an ellipsoid), `thomas`, `andoyer`, or `karney` (GeographicLib). Values align with Apache Parquet native geospatial and GeoParquet 2.0. Ignored for `geometry` (edges are always planar). |
| `encoding`    | geometry, geography   | No       | string  | The physical serialisation format of the geometry value. One of `wkb` (canonical for GeoParquet 2.0 and Parquet native geospatial), `wkt`, `geojson`, `ewkt`, `ewkb`. When omitted, the encoding is system-defined or unspecified.                |
| `bbox`        | geometry, geography   | No       | array   | Bounding box of the column's spatial data, expressed in the column's own `crs` (matching GeoParquet 2.0). Format: `[xmin, ymin, xmax, ymax]` for 2D data; `[xmin, ymin, zmin, xmax, ymax, zmax]` when a Z dimension is present; `[xmin, ymin, zmin, mmin, xmax, ymax, zmax, mmax]` when both Z and M are present. Used as a spatial extent validation hint. |
| `orientation` | geometry, geography   | No       | string  | Winding order for polygon rings. Currently only `counterclockwise` is defined (exterior rings counterclockwise, interior rings clockwise), matching GeoParquet 2.0. Recommended for `geography` with non-planar edges to avoid ambiguity around which side of a ring is "inside". |
| `epoch`       | geometry, geography   | No       | number  | Decimal year (e.g. `2021.47`) indicating the coordinate epoch for dynamic CRSs whose reference frames evolve over time. Optional; only meaningful when the `crs` is dynamic. |

### Example 1: Minimal — a GPS coordinate column

```yaml
apiVersion: v3.2.0
kind: DataContract
id: vehicle-locations
name: Vehicle Locations

schema:
  - name: locations
    physicalName: vehicle_locations
    properties:
      - name: vehicle_id
        logicalType: string
        primaryKey: true
        required: true
      - name: location
        logicalType: geography
        required: true
        logicalTypeOptions:
          encoding: wkt
```

### Example 2: Structured — a cadastral parcel table with polygons

```yaml
apiVersion: v3.2.0
kind: DataContract
id: cadastral-parcels
name: Cadastral Parcels

schema:
  - name: parcels
    physicalName: cadastral_parcels
    description: "Land parcel boundaries in the Dutch national projection."
    properties:
      - name: parcel_id
        logicalType: string
        primaryKey: true
        required: true
      - name: municipality
        logicalType: string
        required: true
      - name: boundary
        logicalType: geometry
        required: true
        description: "Parcel boundary polygon in the Dutch RD New projection."
        logicalTypeOptions:
          subType: Polygon
          crs: EPSG:28992
          dimensions: 2
          encoding: wkb
      - name: centroid
        logicalType: geometry
        logicalTypeOptions:
          subType: Point
          crs: EPSG:28992
          dimensions: 2
          encoding: wkt
```

### Example 3: 3D trajectory with geography

```yaml
schema:
  - name: flights
    physicalName: flight_trajectories
    properties:
      - name: flight_id
        logicalType: string
        primaryKey: true
        required: true
      - name: trajectory
        logicalType: geography
        required: true
        description: "Full 3D flight path (longitude, latitude, altitude in metres)."
        logicalTypeOptions:
          subType: LineString
          crs: EPSG:4326
          dimensions: 3
          algorithm: vincenty
          encoding: wkb
```

### Physical type mapping

The `physicalType` field carries the target system's native column type, while `logicalType` declares the shape and semantics.

| Target system             | Typical `physicalType` value       |
| ------------------------- | ---------------------------------- |
| PostGIS (PostgreSQL)      | `geometry` / `geography`           |
| BigQuery                  | `GEOGRAPHY`                        |
| Snowflake                 | `GEOGRAPHY` / `GEOMETRY`           |
| Databricks / Delta Lake   | `STRING` (WKT) or `BINARY` (WKB)  |
| Apache Iceberg v3         | `geometry` / `geography`           |
| DuckDB (spatial ext.)     | `GEOMETRY`                         |
| Apache Sedona             | `geometry`                         |
| GeoParquet 2.0            | `GEOMETRY` / `GEOGRAPHY` (native Parquet logical types over `BYTE_ARRAY`, WKB) |
| Oracle Spatial            | `SDO_GEOMETRY`                     |
| SQL Server                | `geometry` / `geography`           |

### Coordinate Reference Systems

The `crs` option accepts several forms:

- **Authority codes** — e.g. `EPSG:4326`, `OGC:CRS84`. **Recommended.**
- **OGC URN identifiers** — e.g. `urn:ogc:def:crs:EPSG::4326`.
- **Inline PROJJSON** — a full PROJJSON document as a JSON string, for CRSs that cannot be referenced by an authority code.
- **`srid:<n>`** — an SRID reference (e.g. `srid:0` for unspecified), aligning with Apache Parquet native geospatial.
- **`projjson:<key>`** — a reference to a PROJJSON blob stored elsewhere in metadata.

Common values:

| CRS name                                                 | Authority code (recommended) | OGC URN                        |
| -------------------------------------------------------- | ---------------------------- | ------------------------------ |
| WGS 84 (longitude/latitude, explicit lon/lat axis order) | `OGC:CRS84`                  | `urn:ogc:def:crs:OGC::CRS84`   |
| WGS 84 (longitude/latitude)                              | `EPSG:4326`                  | `urn:ogc:def:crs:EPSG::4326`   |
| WGS 84 / Pseudo-Mercator                                 | `EPSG:3857`                  | `urn:ogc:def:crs:EPSG::3857`   |
| Dutch RD New                                             | `EPSG:28992`                 | `urn:ogc:def:crs:EPSG::28992`  |
| UTM Zone 32N                                             | `EPSG:32632`                 | `urn:ogc:def:crs:EPSG::32632`  |

`EPSG:4326` and `OGC:CRS84` refer to the same datum (WGS 84); they differ only in the conventional axis order (`EPSG:4326` is defined as latitude/longitude, `OGC:CRS84` as longitude/latitude). GeoParquet 2.0 and Parquet native geospatial use `OGC:CRS84` to make the axis order unambiguous.

When `logicalType` is `geometry`, `crs` is **required**: there is no universal default for planar coordinate systems, and assuming one leads to data quality issues.

When `logicalType` is `geography` and `crs` is omitted, `EPSG:4326` (WGS 84 longitude/latitude) is assumed, matching the GeoJSON convention and the historical ODCS default. For GeoParquet 2.0 interoperability, prefer `OGC:CRS84` explicitly.

### Geometry subtypes (ISO 19125-1)

The `subType` option maps directly to the ISO 19125-1 Simple Features geometry hierarchy:

| `subType` value      | Description                                         |
| -------------------- | --------------------------------------------------- |
| `Point`              | A single coordinate pair (or triplet for 3D)        |
| `LineString`         | An ordered sequence of points connected by lines    |
| `Polygon`            | A closed ring with optional interior rings (holes)  |
| `MultiPoint`         | A collection of points                              |
| `MultiLineString`    | A collection of line strings                        |
| `MultiPolygon`       | A collection of polygons                            |
| `GeometryCollection` | A heterogeneous collection of any geometry subtypes |

#### Mapping to GeoParquet 2.0 `geometry_types`

GeoParquet 2.0 expresses the set of subtypes present in a column as an array of strings under `geometry_types`, with dimension suffixes baked into each string (` Z`, ` M`, ` ZM`). ODCS keeps `subType` (a single string, the union of allowed subtypes) and `dimensions` (an integer) as separate options, in line with the rest of `logicalTypeOptions`. The mapping is straightforward:

| ODCS `subType` + `dimensions`       | GeoParquet 2.0 `geometry_types` entry |
| ----------------------------------- | ------------------------------------- |
| `Point`, `dimensions: 2`            | `Point`                               |
| `Point`, `dimensions: 3` (XYZ)      | `Point Z`                             |
| `LineString`, `dimensions: 3` (XYM) | `LineString M`                        |
| `Polygon`, `dimensions: 4` (XYZM)   | `Polygon ZM`                          |

When `subType` is omitted (any subtype accepted), the corresponding GeoParquet 2.0 form is an empty `geometry_types` array (unknown/mixed types).

### Spatial extent (bounding box)

A bounding box can be declared at two levels to document and validate the spatial extent of geospatial data. Bounding boxes are always expressed in the CRS of the geometry they describe, matching GeoParquet 2.0 (whose bbox is stated in the column's own `crs`, not forced to WGS 84).

#### Column-level `bbox`

The `bbox` option in `logicalTypeOptions` records the expected spatial extent of an individual geometry or geography column, expressed in the column's own `crs`. Format: `[xmin, ymin, xmax, ymax]` for 2D data; `[xmin, ymin, zmin, xmax, ymax, zmax]` when a Z dimension is present; `[xmin, ymin, zmin, mmin, xmax, ymax, zmax, mmax]` when both Z and M are present. It serves as a validation hint: values falling outside the declared bounding box indicate data quality issues.

```yaml
logicalTypeOptions:
  subType: Polygon
  crs: EPSG:28992
  bbox: [12621, 306846, 278026, 619256]   # Netherlands in RD New (EPSG:28992)
```

When a dataset contains multiple spatial columns, each column carries its own `bbox`, which makes the per-column extent precise and unambiguous.

#### Table-level `spatialExtent`

A `spatialExtent` block at the schema (table) level captures the combined geographic footprint of the entire dataset. Because a table may combine columns with different CRSs, the table-level `spatialExtent.bbox` MUST carry its own `crs` alongside the array. When only one spatial column exists (or all spatial columns share a CRS), reusing that column's CRS is the natural choice.

```yaml
schema:
  - name: parcels
    physicalName: cadastral_parcels
    spatialExtent:
      crs: EPSG:28992
      bbox: [12621, 306846, 278026, 619256]
    properties:
      - name: boundary
        logicalType: geometry
        logicalTypeOptions:
          subType: Polygon
          crs: EPSG:28992
          bbox: [12621, 306846, 278026, 619256]
```

### Relationship to existing types

`geometry` and `geography` are not modelled as `string` or `binary` because:

- A string column gives no hint that the value encodes spatial data, its subtype, or its CRS.
- The binary / WKB encoding carries no standard metadata channel for CRS or subtype.
- Native geospatial types in target systems (PostGIS, BigQuery, Snowflake, Iceberg) are distinct from generic strings and binaries; the logical type should reflect that.

`physicalType` continues to carry the target-system column syntax, keeping the logical/physical separation consistent with the rest of ODCS. The serialisation format (WKT, WKB, etc.) is expressed via `logicalTypeOptions.encoding` because it describes how the geometry value is encoded, not which storage system holds it.

## Alternatives

### Alternative A: `logicalType: binary` for WKB columns (rejected)

The original issue request was to add `logicalType: binary` so that WKB-encoded geospatial data could be distinguished from plain strings.

**Rejected because**: a bare binary type does not communicate that the data is geospatial, what geometry subtype it contains, or which CRS applies. Consumers would still need out-of-band documentation to interpret the column. A dedicated `geometry` / `geography` pair is more expressive and directly interoperable with target system types.

### Alternative B: Single `geometry` type with a `model` option (rejected)

Collapse `geometry` and `geography` into one type, using a `model: flat | spherical` option to distinguish them.

**Rejected because**: `geometry` and `geography` are separate, named types in every major system surveyed (PostGIS, BigQuery, Snowflake, Iceberg v3). Merging them into one logical type would require contract readers to inspect `logicalTypeOptions` to understand a fundamental semantic difference. Keeping them separate mirrors the ecosystem and avoids ambiguity.

### Alternative C: Rely on `customProperties` (rejected)

Keep ODCS silent about geospatial types and let tools namespace CRS and subtype information under `customProperties`.

**Rejected because**: geospatial columns are a common and stable feature of the data landscape. Leaving them outside the standard forces every tool to invent its own conventions, which defeats the interoperability goal ODCS exists to serve.

### Alternative D: Proliferate per-subtype logical types (rejected)

Add dedicated logical types for each geometry variant: `logicalType: point`, `logicalType: polygon`, etc.

**Rejected because**: Apache Iceberg v3, GeoArrow, and GeoParquet all settled on two top-level types (`geometry` / `geography`) with the subtype as a secondary property. Following this precedent avoids type proliferation and keeps the standard small.

## Decision

> The decision made by the TSC.

TBD.

## Consequences

- **Non-breaking**: `geometry` and `geography` are new optional `logicalType` values; no existing contract is affected.
- **Interoperable**: Maps cleanly onto PostGIS, BigQuery, Snowflake, Databricks/Iceberg v3, DuckDB, GeoParquet, and GeoArrow.
- **Composable with other RFCs**: Works with [RFC-0034](0034-measures-and-dimensions.md) (geospatial columns can coexist with measures and dimensions) and [RFC-0041](0041-synonyms.md) (synonyms on geospatial columns aid discovery).
- **Validator impact**: Contract validators MUST warn (or error) when `logicalType` is `geometry` and `logicalTypeOptions.crs` is absent — no default exists for planar coordinate systems. Validators SHOULD warn when `logicalTypeOptions.encoding` is absent for either type, as omitting it leaves the serialisation format ambiguous for consumers.
- **Binary support**: The original issue request for a binary logical type is addressed by combining `logicalType: geometry` (or `geography`) with `logicalTypeOptions.encoding: wkb`.

## References

- [ISO 19125-1: Geographic information — Simple feature access — Part 1: Common architecture](https://www.iso.org/standard/40114.html)
- [OGC 07-092r3: Definition identifier URNs in OGC namespace](https://docs.ogc.org/is/07-092r3/07-092r3.html) — specifies the `urn:ogc:def:crs:EPSG::*` URN format
- [Apache Iceberg v3 spec — geometry and geography types](https://iceberg.apache.org/spec/#primitive-types)
- [GeoParquet 2.0 specification (v2.0.0-rc.1)](https://geoparquet.org/releases/v2.0.0-rc.1/)
- [Apache Parquet native geospatial logical types (`GEOMETRY`, `GEOGRAPHY`)](https://github.com/apache/parquet-format/blob/master/Geospatial.md)
- [GeoArrow specification](https://geoarrow.org/)
- [PostGIS geometry/geography reference](https://postgis.net/docs/manual-3.5/using_postgis_dbmanagement.html#PostGIS_GeographyVSGeometry)
- [Snowflake GEOGRAPHY data type](https://docs.snowflake.com/en/sql-reference/data-types-geospatial)
- [BigQuery GEOGRAPHY type](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#geography_type)
- [RFC-0002 Data Types](approved/odcs-v3.0.0/0002-types.md) — ODCS logical/physical types foundation
- [RFC-0017 New Date Types](approved/odcs-v3.1.0/0017-new-date-types.md) — precedent for adding new logical types with `logicalTypeOptions`
- [RFC-0042 Vector Type](0042-vector-type.md) — precedent for a rich `logicalTypeOptions` structure alongside a new `logicalType`
