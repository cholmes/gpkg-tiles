# GeoPackage Raster Specification

This document will eventually be the full spec for using rasters in a GeoPackage. It 
is an extension of the Tile specification, with the conventions and additional metadata
for raster data sets. They will act as Tile data sets using Views, and offer additional
metadata on the storage of the actual raster.

For now it is a place to store the raster specific information and notes from the main
[specification] (spec.md).

## General strategy

Tiles spec should be just what's needed to get at the tiles. The raster spec should be 
able to represent everything in the tile spec with views and complementary tables.


## Raster notes to include

Following a convention used by [RasterLite] (https://www.gaia-gis.it/fossil/librasterlite/index), 
tables or views containing record-level metadata are named with a raster or tile table name prefix and 
a “_rt_metadata” suffix, e.g. {RasterTableName}_rt_metadata.

> The Section 2 below should be redone to gel with the tile specification, like how it interacts with tile_table_metadata

### 2	Raster Columns
A GeoPackage implementing Raster SHALL contain a `raster_columns` table or view as defined in this clause.  The `raster_columns` 
table or view SHALL contain one row record describing each tile column in any table in a GeoPackage.  
The `r_raster_column` in `r_table_name` SHALL be defined as a BLOB data type.  

The `compr_qual_factor` column value indicates the lowest image quality of any raster or tile in the 
associated column on a scale from 1 (lowest) to 100 (highest) for rasters compressed with a lossy 
compression algorithm. It is always 100 if all rasters or tiles are compressed with a lossless 
compression algorithm, or are not compressed.  The value -1 indicates "unknown" and is specified as 
the default value.

The `georectification` column value indicates the minimum level of georectification to areas on the 
earth for all rasters or tiles in the associated column are georectified. A value of -1 indicates "unknown" 
as is specified as the default value. A value of 0 indicates that no rasters or tiles are georectified. 
A value of 1 indicates that all rasters or tiles are georectified (but not necessarily orthorectified). 
A value of 2 indicates that all rasters or tiles are orthorectified (which implies georectified) to 
accurately align with real world coordinates, have constant scale, and support direct measurement of 
distances, angles, and areas.

The srid SHALL have a value contained in the `spatial_ref_sys` table defined in clause 9.2 above.

All GeoPackages SHALL support image/png and image/jpeg formats for rasters and tiles. GeoPackages may 
support image/x-webp and image/tiff formats for rasters and tiles. GeoPackage support for the image/tiff 
format [[31]] (#31) is limited to GeoTIFF [[32]] (#32) images that meet the requirements of the NGA 
Implementation Profile [[33]] (#33) for coordinate transformation case 3 where the position and scale of 
the data is known exactly, and no rotation of the image is required.

[[Note 2]] (implementation.md#note-2) and [[Note 3]] (implementation.md#note-3)

**Table 1** - `raster_columns` 

 + Table or View Name: `raster_columns`

| Column Name | ColumnType | Column Description | Null | Default | Key |
| ----------- | ---------- | ------------------ | ---- | ------- | --- | 
| `r_table_name` | text |	Name of the table containing the raster column, e.g. {FeatureTableName} OR {RasterLayerName}_tiles | no	|	| PK FK |
| `r_raster_column` | text | Name of a column in a table that is a raster column with a BLOB data type | no | |	PK |
| `compr_qual_factor` |	integer |	Compression quality factor: 1 (lowest) to 100 (highest) for lossy compression; always 100 for lossless or no compression, -1 if unknown. | no |	-1 | |
| `georectification` |	integer |	Is the raster georectified; 1=unknown, 0=not georectified, 1=georectified, 2=orthorectified |	no | -1 | |
| `srid` |	integer |	Spatial Reference System ID: spatial_ref_sys.srid |	no | | FK |

**Table 2** - `raster_columns` Table Definition SQL

```SQL
CREATE TABLE
  raster_columns
  (
    r_table_name TEXT NOT NULL,
    r_raster_column TEXT NOT NULL,
    compr_qual_factor INTEGER NOT NULL DEFAULT -1,
    georectification INTEGER NOT NULL DEFAULT -1,
    srid INTEGER NOT NULL DEFAULT 0,
    CONSTRAINT pk_rc PRIMARY KEY (r_table_name, r_raster_column) ON CONFLICT ROLLBACK,
    CONSTRAINT fk_rc_r_srid FOREIGN KEY (srid) REFERENCES spatial_ref_sys(srid),
    CONSTRAINT fk_rc_r_gc FOREIGN KEY (r_table_name) REFERENCES geopackage_contents(table_name)
  )
```

### From section 3

The `t_table_name` column value SHALL be a row value of `r_table_name` in the `raster_columns` 
table, enforced by a trigger.

> Raster columns should need this, but not all tiles. So can add it in as a constraint in this portion of the spec.

> Raster columns should implement a view of 'tile table metadata'. One question is what to do with SRID. 
Seems useful to have on the tile table metadata, can we make a view of it from the raster column? And/or where 
does the raster column get its is_times_two_zoom value from? I guess a raster could be the source of multiple tile sets.