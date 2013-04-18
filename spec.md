# GeoPackage Tiles Specification

## 10  Raster Tile Store
### 10.1	Raster Tile Introduction
There are a wide variety of commercial and open source conventions for storing, indexing,
accessing and describing individual rasters and tiles in tile matrix pyramids. Unfortunately, 
no applicable existing consensus, national or international specifications have standardized 
practices in this domain. In addition, various image file formats have different 
representational capabilities, and include different self-descriptive metadata.  

The Raster / Tile Store data / metadata model, conventions and SQL functions described below 
support direct use of rasters and tiles in a GeoPackage in two ways. First, they specify how 
existing applications may create SQL Views of the data /metadata model on top of existing 
application tables that that follow different interface conventions. Second, they include 
and expose enough metadata information at both the dataset and record levels to allow 
applications that use GeoPackage data to discover its characteristics without having to parse 
all of the stored images. Applications that store GeoPackage raster and tile data, which are 
presumed to have this information available, SHALL store sufficient metadata to enable its 
intended use. 

Following a convention used by [MBTiles] (https://github.com/mapbox/mbtiles-spec), the Raster
/ Tile Store data model may be implemented directly as SQL tables in a SQLite database for 
maximum performance, or as SQL views on top of tables in an existing SQLite Raster / Tile store
for maximum adaptability and loose coupling to enable widespread implementation. A GeoPackage 
can store multiple raster and tile pyramid data sets in different tables or views in the same 
container. Following a convention used by [RasterLite] (https://www.gaia-gis.it/fossil/librasterlite/index), 
tables or views containing record-level metadata are named with a raster or tile table name prefix and 
a “_rt_metadata” suffix, e.g. {RasterTableName}{_rt_metadata.  

The tables or views that implement the GeoPackage Raster / Tile Store data / metadata model are described 
and discussed individually in the following subsections.

[[Note 1]] (#note-1)

### 10.2	Raster Columns
A GeoPackage SHALL contain a `raster_columns` table or view as defined in this clause.  The `raster_columns` 
table or view SHALL contain one row record describing each raster or tile column in any table in a GeoPackage.  The `r_raster_column` in `r_table_name` SHALL be defined as a BLOB data type.  

The `compr_qual_` factor column value indicates the lowest image quality of any raster or tile in the 
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

[[Note 2]] (#note-2) and [[Note 3]] (#note-3)

**Table 21** - `raster_columns` 

 + Table or View Name: `raster_columns`

| Column Name | ColumnType | Column Description | Null | Default | Key |
| ----------- | ---------- | ------------------ | ---- | ------- | --- | 
| `r_table_name` | text |	Name of the table containing the raster column, e.g. {FeatureTableName} OR {RasterLayerName}_tiles | no	|	| PK FK |
| `r_raster_column` | text | Name of the column in `r_table_name` that is a raster column with a BLOB data type | no | |	PK |
| `compr_qual_factor` |	integer |	Compression quality factor: 1 (lowest) to 100 (highest) for lossy compression; always 100 for lossless or no compression, -1 if unknown. | no |	-1 | |
| `georectification` |	integer |	Is the raster georectified; 1=unknown, 0=not georectified, 1=georectified, 2=orthorectified |	no | -1 | |
| `srid` |	integer |	Spatial Reference System ID: spatial_ref_sys.srid |	no | | FK |

**Table 1** - `raster_columns` Table Definition SQL

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

### 10.3	Tile Table Metadata
A GeoPackage SHALL contain a `tile_table_metadata` table or view as defined in this clause. The 
`tile_table_metadata` table or view SHALL contain one row record describing each tile table in a 
GeoPackage.  The `t_table_name` column value SHALL be a row value of `r_table_name` in the `raster_columns` 
table, enforced by a trigger.  The `is_times_two_zoom` column value SHALL be 1 if zoom level pixel sizes 
vary by powers of 2 between adjacent zoom levels in the corresponding tile table, or 0 if not.

[[Note 4]] (#note-4) and [[Note 5]] (#note-5)

**Table 25** - `tile_table_metadata`
Table or View Name: `tile_table_metadata`

| Column Name | Column Type	| Column Description | Null	| Default	Key |
|-------------|-------------|--------------------|------|-------------|
| t_table_name | text	| {RasterLayerName}{_tiles}	| no | PK |
| is_times_two_zoom	| integer	| Zoom level pixel sizes vary by powers of 2 (0=false,1=true)	| no | 1 |

**Table 2** - `tile_table_metadata` Table Definition SQL

```SQL
CREATE TABLE
  tile_table_metadata
  (
    t_table_name TEXT NOT NULL PRIMARY KEY,
    is_times_two_zoom INTEGER NOT NULL DEFAULT 1
  )
```

### 10.4	Tile Matrix Metadata
A GeoPackage SHALL contain a `tile_matrix_metadata` table or view as defined in this clause.  
The `tile_matrix_metadata` table or view SHALL contain one row record for each zoom level that 
contains one or more tiles in each tiles table.  It may contain row records for zoom levels in 
a tiles table that do not contain tiles. A `tile_matrix_metadata` row record SHALL be inserted 
for a zoom level for `t_table_name` before any tiles are inserted into the corresponding tiles 
table, so that triggers on that table specified in clause 10.5 below may reference `tile_matrix_metadata` 
column values for that zoom level to reject invalid data. 

The `tile_matrix_metadata` table documents the structure of the tile matrix at each zoom level 
in each tiles table. It allows GeoPackages to contain rectangular as well as square tiles (e.g. 
for better representation of polar regions). It allows tile pyramids with zoom levels that differ 
in resolution by powers of 2, irregular intervals, or regular intervals other than powers of 2. 
When the value of the `is_times_two_zoom` column in the `tile_table_metadata` record for a tiles 
table is 1 (true) then the pixel sizes for adjacent zoom levels in the `tile_matrix_metadata` table 
for that table SHALL only vary by powers of 2.

[[Note 6]] (#note-6)  

GeoPackages SHALL follow the most frequently used conventions of a tile origin at the upper left and
a zoom-out-level of 0 for the smallest map scale “whole world” zoom level view, as specified by 
[WMTS] (http://portal.opengeospatial.org/files/?artifact_id=35326). The tile coordinate (0,0) 
SHALL always refer to the tile in the upper left corner of the tile matrix at any zoom level, 
regardless of the actual availability of that tile.  Pixel sizes for zoom levels sorted in ascending 
order SHALL be sorted in descending order.  

GeoPackages SHALL not require that tiles be provided for level 0 or any other particular zoom level. 
This means that a tile matrix set can be sparse, i.e. not contain a tile for any particular position 
at a certain tile zoom level. This does not affect the spatial extent stated by the min/max x/y columns 
values in the `geopackage_contents` record for the same `t_table_name`, or the tile matrix width and 
height at that level.

[[Note 7]] (#note-7), [[Note 8]] (#note-8), [[Note 9]] (#note-9), and [[Note 10]] (#note-10) 

**Table 29** - `tile_matrix_metadata`
+ Table or View Name: `tile_matrix_metadata`

|Column Name | Column Type | Column Description |	Null | Default | Key |
|------------|-------------|--------------------|------|---------|-----|
| `t_table_name` |	text |	{RasterLayerName}_tiles |no	| | PK, FK |
| `zoom_level`	| integer |	0 <= `zoom_level` <= max_level for `t_table_name`	| no |	0 |	PK |
| `matrix_width` |	integer |	Number of columns (>= 1) in tile matrix at this zoom level | no |	1 | |	
| `matrix_height` |	integer |	Number of rows (>= 1) in tile matrix at this zoom level |	no | 1 | |	
| `tile_width` |	integer |	Tile width in pixels (>= 1)for this zoom level |	no |	256	| |
| `tile_height` |	integer |	Tile height in pixels (>= 1) for this zoom level |	no |	256	| |
| `pixel_x_size` |	double |	In `t_table_name` srid units or default meters for srid 0 (>0) |	no |	1 | |
| `pixel_y_size` |	double |	In `t_table_name` srid units or default meters for srid 0 (>0) |	no |	1	| |

**Table 3** - `tile_matrix_metadata` Table Creation SQL

```SQL
CREATE TABLE
  tile_matrix_metadata
  (
    t_table_name TEXT NOT NULL,
    zoom_level INTEGER NOT NULL,
    matrix_width INTEGER NOT NULL,
    matrix_height INTEGER NOT NULL,
    tile_width INTEGER NOT NULL,
    tile_height INTEGER NOT NULL,
    pixel_x_size DOUBLE NOT NULL,
    pixel_y_size DOUBLE NOT NULL,
    CONSTRAINT pk_ttm PRIMARY KEY (t_table_name, zoom_level) ON CONFLICT ROLLBACK,
    CONSTRAINT fk_ttm_t_table_name FOREIGN KEY (t_table_name) REFERENCES tile_table_metadata(t_table_name
  )
```



### 10.5	Tiles Table
Tiles in a tile matrix set with one or more zoom levels SHALL be stored in a GeoPackage in a tiles 
table or view with a unique name for every different tile matrix set in the GeoPackage.  Each tiles 
table or view SHALL be defined with the columns described in table 35 below.  Each tiles table or 
view SHALL contain tile matrices at one or more zoom levels of different spatial resolution (map scale).
All tiles at a particular zoom level must have the same `pixel_x_size` and `pixel_y_size` values 
specified in the `tile_matrix_metadata` row record for that tiles table and zoom level.  

When the value of the `is_times_two_zoom` column in the `tile_table_metadata` record for a tiles 
table row is 1 (true) then the pixel sizes for adjacent zoom levels in the tiles table SHALL only 
vary by powers of 2.

[[Note 11]] (#note-11) and [[Note 12]] (#note-12)

GeoPackages SHALL implement appropriate SQL triggers on each tiles table by executing the 
`add_tile_triggers()` routine specified in clause 10.8 below with the tiles table as a parameter 
value to ensure that 

1.	The `zoom_level` value is specified for the tiles table in the `tile_matrix_metadata` table

2.	The tile column value is between 0 and the `matrix_height` specified for the zoom_level in the `tile_matrix_metadata` table

3.	The `tile_row value is between 0 and the `matrix_width` specified for the `zoom_level` in the `tile_matrix_metadata` table

**Table 33** – tiles table
+ Table or View Name:   {TilesTableName} tiles table

|Column Name | Column Type | Column Description |  Null | Default | Key |
|------------|-------------|--------------------|------|---------|-----|
| `id` |	integer	| Autoincrement primary key |	no |	|	PK |
| `zoom_level` |	integer |	min(zoom_level) <= `zoom_level` <= max(zoom_level)  for t_table_name |	no |	0	 | UK |
| `tile_column` |	integer	| 0 to `tile_matrix_metadata` `matrix_width` – 1 |	no |	0	| UK |
| `tile_row` |	integer	| 0 to `tile_matrix_metadata` `matrix_height` - 1 |	no	| 0 |	UK |
| `tile_data` |	BLOB	| Of an image MIME type specified in clause 10.2 | no	| | |	

**Table 4** - EXAMPLE: `sample_matrix_tiles` Table Definition SQL

```SQL
CREATE TABLE
  sample_matrix_tiles
  (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    zoom_level INTEGER NOT NULL DEFAULT 0,
    tile_column INTEGER NOT NULL DEFAULT 0,
    tile_row INTEGER NOT NULL DEFAULT 0,
    tile_data BLOB NOT NULL DEFAULT (zeroblob(4)),
    UNIQUE (zoom_level, tile_column, tile_row)
  )
```

### 10.6 Raster Tables

Raster tables have raster columns defined as BLOB data types that contain rasters that are not part
of tile matrix sets.  Every table in a GeoPackage that is not a tiles tables as as described in 
clause 9.5 and that includes one or more raster columns is a raster table.  Raster tables are also 
feature tables as specified in clause 8.4 above that may or may not have geometry columns in addition 
to raster columns.

Every raster table in a GeoPackage shall have a primary key defined on one or more columns so that row
level metadata records may be linked to the rasters in it by rowid as described in clauses 10.7 and 11.3 below.

[[Note 13]] (#note-13)

**Table 39** - EXAMPLE: `sample_rasters` Table or View
+ Table or View Name: {TilesTableName} `sample_rasters`

|Column Name | Column Type | Column Description |  Null | Default | Key |
|------------|-------------|--------------------|------|---------|-----|
| `id` | integer	 | Autoincrement primary key |	no |	|	PK |
| `elevation`  | BLOB | Elevation coverage; of type raster_format_metadata.mime_type | no | | |
| `description` | text | Description of the area | no | 'no desc' | |
| `photo` | BLOB | Photograph of the area; of type_raster_format_metadata.mime_type | no | |

**Table 5** - EXAMPLE: `sample_rasters` Table Definition SQL

```SQL
CREATE TABLE
  sample_rasters
  (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    elevation BLOB NOT NULL,
    description TEXT NOT NULL DEFAULT 'no_desc',
    photo BLOB NOT NULL
  )
```

[[Note 14]] (#note-14)

###10.7	Rasters or Tiles Table Metadata

There SHALL be a {Raster|Tile TableName}_rt_metadata table or view for each rasters or tiles table 
in a GeoPackage defined with the columns described in table 40 below.  

[[Note 15]] (#note-15)

The data in a row record in this table refers to the raster in the `r_raster_column` column in the 
{Raster|Tile TableName}table for the record with a rowed equal to the `row_id_value` primary key column value. 

[[Note 16]] (#note-16)

The `compr_qual_factor` column value indicates the image quality of that raster on a scale from 1 
(lowest) to 100 (highest) for rasters compressed with a lossy compression algorithm. It is always 
100 for rasters compressed with a lossless compression algorithm, or with no compression. A value 
of -1 indicates "unknown" as is specified as the default value.  

The `georectification` column value indicates whether or not that raster is georectified to an area 
on the earth. A value of 0 indicates that the raster is not georectified. .  A value of -1 indicates 
"unknown" as is specified as the default value.  A value of 1 indicates that the raster is georectified 
(but not necessarily orthorectified). A value of 2 indicates that the raster is orthorectified (which 
implies georectified) to accurately align with real world coordinates, have constant scale, and support 
direct measurement of distances, angles, and areas

For a georectified raster (i.e. georectification is 1 or 2), the `min_x`, `min_y`, `max_x` and `max_y` column 
values define a bounding box that SHALL be the spatial extent of the area on the earth represented by the raster.  

[[Note 17]] (#note-17)

**Table 40** - `{RasterLayerName}_rt_metadata`
+ Table or View Name: `{RasterLayerName}_rt_metadata`

|Column Name | Column Type | Column Description |  Null | Default | Key |
|------------|-------------|--------------------|------|---------|-----|
| row_id_value |	integer |	rowid in rasters or tiles table |	no |	|	PK | 
| r_raster_column |	text |	“tile_data” for a tiles table, or the name of a raster column for a rasters table	| no |	raster_column_name |	PK |
| compr_qual_factor |	integer |	Compression quality factor: 1 (lowest) to 100 (highest) for lossy compression; always 100 for lossless or no compression, -1 if unknown. | no	| -1 | |
| georectification |	integer |	Is the raster georectified; -1=unknown, 0=not georectified, 1=georectified, 2=orthorectified	| no |	-1 | |
| min_x	| double |	In raster_columns.srid	| no |	-180.0	| |
| min_y	| double	| In raster_columns.srid	| no	| -90.0	| |
| max_x	| double	| In raster_columns.srid	| no	| 180.0	| |
| max_y	| double	| In raster_columns.srid	| no	| 90.0	| |

**Table 6** - EXAMPLE: `{RasterLayerName}_rt_metadata` Table Definition SQL

```SQL
CREATE TABLE
  sample_matrix_tiles_rt_metadata
  (
    row_id_value INTEGER NOT NULL,
    r_raster_column TEXT NOT NULL DEFAULT 'tile_data',
    compr_qual_factor INTEGER NOT NULL DEFAULT -1,
    georectification INTEGER NOT NULL DEFAULT -1,
    min_x DOUBLE NOT NULL DEFAULT -180.0,
    min_y DOUBLE NOT NULL DEFAULT -90.0,
    max_x DOUBLE NOT NULL DEFAULT 180.0,
    max_y DOUBLE NOT NULL DEFAULT 90.0,
    CONSTRAINT pk_smt_rm PRIMARY KEY (row_id_value, r_raster_column) ON CONFLICT ROLLBACK
  )
```


####Implementation Notes


######[Note 1]

Images of multiple MIME types may be stored in given table.  For example, in a tiles table, image/png 
format tiles without compression could be used for transparency where there is no data on the tile edges, and 
image/jpeg format tiles with compression could be used for storage efficiency where there is image data for all 
pixels.  Images of multiple bit depths of the same MIME type may also be stored in a given table, for example 
image/png tiles in both 8 and 24 bit depths.

######[Note 2]

A feature type may be defined to have 0..n raster attributes, so the corresponding feature 
table may contain from 0..n raster columns.

######[Note 3]

 A raster tile layer table has only one raster column named `tile_data`.

######[Note 4]

A row record for a tile table must be inserted into this table before row records can be inserted 
into the tile_matrix_metadata table described in clause 10.4 due to the presence of foreign key and other 
integrity constraints on that table.

######[Note 5]

GeoPackage applications that insert, update, or delete tiles (matrix set) table tiles row records 
are responsible for maintaining the corresponding descriptive contents of the tile_table_metadata table.

######[Note 6]

Most tile pyramids have an origin at the upper left, a convention adopted by the 
OGC [Web Map Tile Service (WMTS)] (http://portal.opengeospatial.org/files/?artifact_id=35326), 
but some such as [TMS] (http://wiki.osgeo.org/wiki/Tile_Map_Service_Specification) used by 
[MB-Tiles] (https://github.com/mapbox/mbtiles-spec) have an origin at the lower left. Most tile 
pyramids, such as [Open Street Map] (http://wiki.openstreetmap.org/wiki/Main_Page), 
[OSMDroidAtlas] (http://wiki.openstreetmap.org/wiki/Main_Page), and [FalconView] (http://www.falconview.org/trac/FalconView) 
use a zoom_out_level of 0 for the smallest map scale “whole world” zoom level view, another 
convention adopted by WMTS, but some such as [Big Planet Tracks] (http://code.google.com/p/big-planet-tracks/) 
invert this convention and use 0 or 1 for the largest map scale “local detail” zoom level view.

######[Note 7]

GeoPackage applications may query the `tile_matrix_metadata` table or the tiles (matrix set) 
table specified in clause 10.7 below to determine the minimum and maximum zoom levels for a given tile matrix table.  

######[Note 8]

GeoPackage applications may query the tiles (matrix set) table to determine which tiles are available at each zoom level.

######[Note 9]

GeoPackage applications that insert, update, or delete tiles (matrix set) table tiles row 
records are responsible for maintaining the corresponding descriptive contents of the `tile_matrix_metadata` table.

######[Note 10]

The `geopackage_contents` table (see clause 8.2 above) contains coordinates that define a
bounding box as the stated spatial extent for all tiles in a tile (matrix set) table. If the geographic 
extent of the image data contained in these tiles is within but not equal to this bounding box, then the
non-image area of matrix edge tiles must be padded with no-data values, preferably transparent ones.

######[Note 11]

The id primary key allows tiles table views to be created on [RasterLite] (https://www.gaia-gis.it/fossil/librasterlite/index) 
version 1 raster table implementations, where the tiles are selected based on a spatially indexed 
bounding box in a separate metadata table.

######[Note 12]

The zoom_level / tile_column / tile_row unique key allows tiles to be selected and accessed 
by “z, x, y”, a common convention used by [MBTiles] (https://github.com/mapbox/mbtiles-spec), 
[Big Planet Tracks] (http://code.google.com/p/big-planet-tracks/), and other implementations. 
In a SQLite implementation this unique key is automatically indexed. This table / view definition 
may also follow [RasterLite] (https://www.gaia-gis.it/fossil/librasterlite/index) version 1 conventions, 
where the tiles are selected based on a spatially indexed bounding box in a separate metadata table.

######[Note 13]

An integer column primary key is recommended for best performance.

######[Note 14]

The `sample_rasters` table created in SQL 5 above could be extended with one or more 
geometry columns by calls to the `addGeometryColumn()` routine specified in clause 9.4 to have 
both raster and geometry columns like the `sample_feature_table` shown in Figure3: GeoPackageTables above in clause 7.

######[Note 15]

This table naming convention is adopted from [RasterLite] (https://www.gaia-gis.it/fossil/librasterlite/index).

######[Note 16]

In an SQLite implementation, the rowid value is always equal to the value of a single-column 
primary key on an [integer column] (http://www.sqlite.org/lang_createtable.html#rowid).  Althought not 
stated in the SQLite documentation, testing has not revealed a case where rowed values on a table with 
any primry key column(s) defined are changed by a database reorganization performed by the VACUUM SQL command.

######[Note 17]

This data structure can be implemented as a table in the absence of geometry data types or spatial 
indexes. When implemented as a view, the min/max x/y columns could reference ordinates of a bounding box geometry 
in an underlying table when geometry data types are available, e.g. in [RasterLite] (https://www.gaia-gis.it/fossil/librasterlite/index).


####Footnotes

######[31]  
NGA Standardization Document: Implementation Profile for Tagged Image File Format (TIFF) and Geographic Tagged Image File Format (GeoTIFF), Version 2.0,  2001-10-26  https://nsgreg.nga.mil/doc/view?i=2224  
######[32]  
IETF RFC 3986 Uniform Resource Identifier (URI): Generic Syntax http://www.ietf.org/rfc/rfc3986.txt 
######[33]  
OGC08-131r3 The Specification Model — A Standard for Modular specifications  https://portal.opengeospatial.org/files/?artifact_id=34762 

