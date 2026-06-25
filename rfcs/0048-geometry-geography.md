# Geometry and Geography Data Types

Champion: [Sander Bylemans](https://github.com/SBylemans)

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

This RFC introduces two new values for `logicalType` in ODCS — `geometry` and `geography` — together with a dedicated set of `logicalTypeOptions` (`subType`, `crs`, `dimensions`, and `algorithm`). `geometry` represents shapes in a flat-earth (planar/Euclidean) coordinate system; `geography` represents coordinates on a round-earth (spherical/ellipsoidal) model. Both align with ISO 19125-1 (Simple Features for SQL), Apache Iceberg v3, GeoArrow, and GeoParquet. The physical encoding is captured by the existing `physicalType` field using well-established formats such as WKT and WKB.

## Motivation

### Why are we doing this?

Geospatial data is a first-class data shape in modern analytics, logistics, real estate, infrastructure, and scientific datasets. Virtually every major database and data lakehouse ships a native geometric or geographic type — PostGIS, BigQuery `GEOGRAPHY`, Snowflake `GEOGRAPHY`, Databricks (Delta Lake + Iceberg v3), DuckDB, Apache Sedona, Hive, Presto/Trino, and others. Formats like GeoParquet and GeoArrow have standardized the columnar representation of geospatial data.

ODCS today has no standard way to describe a geospatial column. Authors are forced to use `logicalType: string` (for WKT) or leave the column type opaque, which:

- masks whether the data is a point, polygon, or line,
- gives no hint about the coordinate reference system (CRS) a consumer needs to interpret the values, and
- defeats the interoperability goal of the standard.

Adding `geometry` and `geography` as first-class logical types matches the precedent set by Apache Iceberg v3, which added exactly these two types, and makes ODCS contracts machine-readable for geospatial tooling.

### Use cases

1. **Parcel and land-registry data**: A data contract declares a `parcel_boundary` column as `logicalType: geometry` with `subType: Polygon` and `crs: urn:ogc:def:crs:EPSG::28992`, letting GIS tools load the correct projection without manual configuration.
2. **Ride-sharing and logistics**: A `pickup_location` column is declared as `logicalType: geography` (round-earth), ensuring that distance calculations account for Earth's curvature.
3. **Sensor and IoT data**: A `gps_track` column is typed `logicalType: geography`, `subType: LineString`, `dimensions: 3` (XYZ with altitude), enabling spatial analytics over device trajectories.
4. **Data lakehouse migration**: Teams migrating geospatial tables from PostGIS to Snowflake or from Hive to Iceberg v3 use the contract to generate correct DDL without hand-editing.
5. **Multi-system interoperability**: GeoParquet and GeoArrow consumers discover from the contract which CRS and geometry type to expect, removing the need to inspect file-level metadata separately.

### Alignment with guiding values

- **Small standard over large**: This RFC adds two `logicalType` values and four optional `logicalTypeOptions`. No new top-level structures.
- **Interoperability over readability**: The shape maps onto PostGIS, BigQuery, Snowflake, Databricks, DuckDB, Apache Sedona, GeoParquet, GeoArrow, and Apache Iceberg v3.
- **Non-breaking**: `geometry` and `geography` are new optional `logicalType` values. Existing contracts are unaffected.

## Design and examples

### Core concept

Geospatial data comes in two flavours:

- **`geometry`** — uses a flat-earth (Euclidean/planar) coordinate system. Coordinates are interpreted as points on a plane using straight-line arithmetic. Fast and suitable for local or projected coordinate systems (e.g., engineering drawings, city-scale maps).
- **`geography`** — uses a round-earth (geodetic/spherical or ellipsoidal) model. Coordinates are longitude/latitude on Earth's surface. Distance and area calculations account for the Earth's curvature. Preferred for global or cross-regional data.

The physical encoding (how the bytes are laid out on disk) is separate from the logical type and is captured by `physicalType`:

| `physicalType` value | Description                                                          |
| -------------------- | -------------------------------------------------------------------- |
| `wkt`                | Well-Known Text — human-readable string, e.g. `POINT (4.9 52.4)`   |
| `wkb`                | Well-Known Binary — compact binary encoding defined by OGC           |
| `geojson`            | GeoJSON encoding (JSON object with `type` and `coordinates`)         |
| `ewkt`               | Extended WKT — PostGIS extension that embeds the SRID in the string  |
| `ewkb`               | Extended WKB — PostGIS extension that embeds the SRID in binary form |

### New `logicalType` values

| Value        | Description                                                                                          |
| ------------ | ---------------------------------------------------------------------------------------------------- |
| `geometry`   | A geometric shape in a flat-earth (planar/Euclidean) coordinate system, per ISO 19125-1.            |
| `geography`  | A geographic shape on a round-earth (geodetic) model, using longitude/latitude coordinates.          |

### `logicalTypeOptions` for `geometry` and `geography`

| Option       | Applies to            | Required | Type    | Description                                                                                                                                                                                                                                        |
| ------------ | --------------------- | -------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `subType`    | geometry, geography   | No       | string  | The geometry subtype per ISO 19125-1. One of `Point`, `LineString`, `Polygon`, `MultiPoint`, `MultiLineString`, `MultiPolygon`, `GeometryCollection`. When omitted, any subtype is accepted.                                                       |
| `crs`        | geometry, geography   | No       | string  | The Coordinate Reference System in OGC URN format, e.g. `urn:ogc:def:crs:EPSG::4326`. Short-form EPSG codes (e.g. `EPSG:4326`) are also accepted. When omitted, the default is `urn:ogc:def:crs:EPSG::4326` (WGS 84 longitude/latitude).         |
| `dimensions` | geometry, geography   | No       | integer | Number of coordinate dimensions: `2` (XY, default), `3` (XYZ or XYM), `4` (XYZM).                                                                                                                                                                |
| `algorithm`  | geography only        | No       | string  | Interpretation of edges between vertices. One of `spherical` (great-circle arcs on the unit sphere, default) or `vincenty` (geodesic on a reference ellipsoid). Ignored for `geometry`.                                                           |

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
        physicalType: wkt
        required: true
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
        physicalType: wkb
        required: true
        description: "Parcel boundary polygon in the Dutch RD New projection."
        logicalTypeOptions:
          subType: Polygon
          crs: urn:ogc:def:crs:EPSG::28992
          dimensions: 2
      - name: centroid
        logicalType: geometry
        physicalType: wkt
        logicalTypeOptions:
          subType: Point
          crs: urn:ogc:def:crs:EPSG::28992
          dimensions: 2
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
        physicalType: wkb
        required: true
        description: "Full 3D flight path (longitude, latitude, altitude in metres)."
        logicalTypeOptions:
          subType: LineString
          crs: urn:ogc:def:crs:EPSG::4326
          dimensions: 3
          algorithm: vincenty
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
| GeoParquet                | `BYTE_ARRAY` (WKB, Parquet binary) |
| Oracle Spatial            | `SDO_GEOMETRY`                     |
| SQL Server                | `geometry` / `geography`           |

### Coordinate Reference Systems

The `crs` option accepts OGC URN identifiers as defined in OGC 07-092r3. Common values:

| CRS name                        | URN                                     | Short form      |
| ------------------------------- | --------------------------------------- | --------------- |
| WGS 84 (longitude/latitude)     | `urn:ogc:def:crs:EPSG::4326`           | `EPSG:4326`     |
| WGS 84 / Pseudo-Mercator        | `urn:ogc:def:crs:EPSG::3857`           | `EPSG:3857`     |
| Dutch RD New                    | `urn:ogc:def:crs:EPSG::28992`          | `EPSG:28992`    |
| UTM Zone 32N                    | `urn:ogc:def:crs:EPSG::32632`          | `EPSG:32632`    |

When `crs` is omitted, `urn:ogc:def:crs:EPSG::4326` (WGS 84) is assumed, matching the GeoParquet and GeoJSON conventions.

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

### Relationship to existing types

`geometry` and `geography` are not modelled as `string` or `binary` because:

- A string column gives no hint that the value encodes spatial data, its subtype, or its CRS.
- The binary / WKB encoding carries no standard metadata channel for CRS or subtype.
- Native geospatial types in target systems (PostGIS, BigQuery, Snowflake, Iceberg) are distinct from generic strings and binaries; the logical type should reflect that.

`physicalType` continues to carry the target-system column syntax and encoding format, keeping the logical/physical separation consistent with the rest of ODCS.

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
- **Validator impact**: Contract validators SHOULD warn when `logicalType` is `geometry` or `geography` and `physicalType` is a plain `string` without a note that WKT encoding is intended.
- **Binary support**: The original issue request for a binary logical type is addressed by combining `logicalType: geometry` (or `geography`) with `physicalType: wkb`.

## References

- [ISO 19125-1: Geographic information — Simple feature access — Part 1: Common architecture](https://www.iso.org/standard/40114.html)
- [OGC 07-092r3: Definition identifier URNs in OGC namespace](https://docs.ogc.org/is/07-092r3/07-092r3.html) — specifies the `urn:ogc:def:crs:EPSG::*` URN format
- [Apache Iceberg v3 spec — geometry and geography types](https://iceberg.apache.org/spec/#primitive-types)
- [GeoParquet specification](https://geoparquet.org/releases/v1.1.0/)
- [GeoArrow specification](https://geoarrow.org/)
- [PostGIS geometry/geography reference](https://postgis.net/docs/manual-3.5/using_postgis_dbmanagement.html#PostGIS_GeographyVSGeometry)
- [Snowflake GEOGRAPHY data type](https://docs.snowflake.com/en/sql-reference/data-types-geospatial)
- [BigQuery GEOGRAPHY type](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#geography_type)
- [RFC-0002 Data Types](approved/odcs-v3.0.0/0002-types.md) — ODCS logical/physical types foundation
- [RFC-0017 New Date Types](approved/odcs-v3.1.0/0017-new-date-types.md) — precedent for adding new logical types with `logicalTypeOptions`
- [RFC-0042 Vector Type](0042-vector-type.md) — precedent for a rich `logicalTypeOptions` structure alongside a new `logicalType`
