# GeoPackage Tiles Specification

### 1 Tile Introduction
There are a wide variety of commercial and open source conventions for storing, indexing,
accessing and describing individual rasters and tiles in tile matrix pyramids. Unfortunately, 
no applicable existing consensus, national or international specifications have standardized 
practices in this domain. In addition, various image file formats have different 
representational capabilities, and include different self-descriptive metadata.  

The Tile Store data / metadata model and conventions and described below 
support direct use of tiles in a GeoPackage in two ways. First, they specify how 
existing applications may create SQL Views of the data /metadata model on top of existing 
application tables that follow different interface conventions. Second, they include 
and expose enough metadata information at both the dataset and record levels to allow 
applications that use GeoPackage data to discover its characteristics without having to parse 
all of the stored images. Applications that store GeoPackage raster and tile data, which are 
presumed to have this information available, SHALL store sufficient metadata to enable its 
intended use. 

Following a convention used by [MBTiles] (https://github.com/mapbox/mbtiles-spec), the
Tile Store data model may be implemented directly as SQL tables in a SQLite database for 
maximum performance, or as SQL views on top of tables in an existing SQLite Tile store
for maximum adaptability and loose coupling to enable widespread implementation. A GeoPackage 
can store multiple raster and tile pyramid data sets in different tables or views in the same 
container. 

Additionally an extension specification enables storage of raster blobs using the same data /
metadata model with Views to implement the Tile portion.  

The tables or views that implement the GeoPackage Tile Store data / metadata model are described 
and discussed individually in the following subsections.

[[Note 1]] (implementation.md#note-1)


### 2	Tile Table Metadata
A GeoPackage SHALL contain a `tile_table_metadata` table or view as defined in this clause. The 
`tile_table_metadata` table or view SHALL contain one row record describing each tile table in a 
GeoPackage.  The `is_times_two_zoom` column value SHALL be 1 if zoom level pixel sizes 
vary by powers of 2 between adjacent zoom levels in the corresponding tile table, or 0 if not.

[[Note 4]] (implementation.md#note-4) and [[Note 5]] (implementation.md#note-5)

**Table 2** - `tile_table_metadata`
Table or View Name: `tile_table_metadata`

| Column Name | Column Type	| Column Description | Null	| Default	Key |
|-------------|-------------|--------------------|------|-------------|
| t_table_name | text	| Name of the Tile Table	| no | PK |
| is_times_two_zoom	| integer	| Zoom level pixel sizes vary by powers of 2 (0=false,1=true)	| no | 1 |
| `srid` |	integer |	Spatial Reference System ID: spatial_ref_sys.srid |	no | | FK |


**Table 3** - `tile_table_metadata` Table Definition SQL

```SQL
CREATE TABLE
  tile_table_metadata
  (
    t_table_name TEXT NOT NULL PRIMARY KEY,
    is_times_two_zoom INTEGER NOT NULL DEFAULT 1,
    srid INTEGER NOT NULL DEFAULT 0
  )
```

### 3	Tile Matrix Metadata
A GeoPackage SHALL contain a `tile_matrix_metadata` table or view as defined in this clause.  
The `tile_matrix_metadata` table or view SHALL contain one row record for each zoom level that 
contains one or more tiles in each tiles table.  It may contain row records for zoom levels in 
a tiles table that do not contain tiles.  

The `tile_matrix_metadata` table documents the structure of the tile matrix at each zoom level 
in each tiles table. It allows GeoPackages to contain rectangular as well as square tiles (e.g. 
for better representation of polar regions). It allows tile pyramids with zoom levels that differ 
in resolution by powers of 2, irregular intervals, or regular intervals other than powers of 2. 
When the value of the `is_times_two_zoom` column in the `tile_table_metadata` record for a tiles 
table is 1 (true) then the pixel sizes for adjacent zoom levels in the `tile_matrix_metadata` table 
for that table SHALL only vary by powers of 2.

[[Note 6]] (implementation.md#note-6)  

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

[[Note 7]] (implementation.md#note-7), [[Note 8]] (implementation.md#note-8), [[Note 9]] (implementation.md#note-9), and [[Note 10]] (implementation.md#note-10) 

**Table 4** - `tile_matrix_metadata`
+ Table or View Name: `tile_matrix_metadata`

|Column Name | Column Type | Column Description |	Null | Default | Key |
|------------|-------------|--------------------|------|---------|-----|
| `t_table_name` |	text |	{TileLayerName}_tiles |no	| | PK, FK |
| `zoom_level`	| integer |	0 <= `zoom_level` <= max_level for `t_table_name`	| no |	0 |	PK |
| `matrix_width` |	integer |	Number of columns (>= 1) in tile matrix at this zoom level | no |	1 | |	
| `matrix_height` |	integer |	Number of rows (>= 1) in tile matrix at this zoom level |	no | 1 | |	
| `tile_width` |	integer |	Tile width in pixels (>= 1)for this zoom level |	no |	256	| |
| `tile_height` |	integer |	Tile height in pixels (>= 1) for this zoom level |	no |	256	| |
| `pixel_x_size` |	double |	In `t_table_name` srid units or default meters for srid 0 (>0) |	no |	1 | |
| `pixel_y_size` |	double |	In `t_table_name` srid units or default meters for srid 0 (>0) |	no |	1	| |

**Table 5** - `tile_matrix_metadata` Table Creation SQL

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



### 4	Tiles Table
Tiles in a tile matrix set with one or more zoom levels SHALL be stored in a GeoPackage in a tiles 
table or view with a unique name for every different tile matrix set in the GeoPackage.  Each tiles 
table or view SHALL be defined with the columns described in table 35 below.  Each tiles table or 
view SHALL contain tile matrices at one or more zoom levels of different spatial resolution (map scale).
All tiles at a particular zoom level must have the same `pixel_x_size` and `pixel_y_size` values 
specified in the `tile_matrix_metadata` row record for that tiles table and zoom level.  

When the value of the `is_times_two_zoom` column in the `tile_table_metadata` record for a tiles 
table row is 1 (true) then the pixel sizes for adjacent zoom levels in the tiles table SHALL only 
vary by powers of 2.

[[Note 11]] (implementation.md#note-11) and [[Note 12]] (implementation.md#note-12)

**Table 7** – tiles table
+ Table or View Name:   {TilesTableName} tiles table

|Column Name | Column Type | Column Description |  Null | Default | Key |
|------------|-------------|--------------------|------|---------|-----|
| `id` |	integer	| Autoincrement primary key |	no |	|	PK |
| `zoom_level` |	integer |	min(zoom_level) <= `zoom_level` <= max(zoom_level)  for t_table_name |	no |	0	 | UK |
| `tile_column` |	integer	| 0 to `tile_matrix_metadata` `matrix_width` – 1 |	no |	0	| UK |
| `tile_row` |	integer	| 0 to `tile_matrix_metadata` `matrix_height` - 1 |	no	| 0 |	UK |
| `tile_data` |	BLOB	| Of an image MIME type specified in clause 10.2 | no	| | |	

**Table 8** - EXAMPLE: `sample_matrix_tiles` Table Definition SQL

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

###6	Tiles Table Metadata

There SHALL be a {Tile TableName}_rt_metadata table or view for each tiles table 
in a GeoPackage defined with the columns described in table 40 below.  

[[Note 15]] (implementation.md#note-15)

The data in a row record in this table refers to the raster in the `r_raster_column` column in the 
{Tile TableName}table for the record with a rowed equal to the `row_id_value` primary key column value. 

The `min_x`, `min_y`, `max_x` and `max_y` column values define a bounding box that 
SHALL be the spatial extent of the area on the earth represented by the raster.

**Table 11** - `{TileLayerName}_rt_metadata`
+ Table or View Name: `{TileLayerName}_rt_metadata`

|Column Name | Column Type | Column Description |  Null | Default | Key |
|------------|-------------|--------------------|------|---------|-----|
| row_id_value |	integer |	rowid in tiles table |	no |	|	PK | 
| min_x	| double |	In tile_table_metadata.srid	| no |	-180.0	| |
| min_y	| double	| In tile_table_metadata.srid	| no	| -90.0	| |
| max_x	| double	| In tile_table_metadata.srid	| no	| 180.0	| |
| max_y	| double	| In tile_table_metadata.srid	| no	| 90.0	| |

**Table 12** - EXAMPLE: `{RasterLayerName}_rt_metadata` Table Definition SQL

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


####Footnotes

######[31]  
NGA Standardization Document: Implementation Profile for Tagged Image File Format (TIFF) and Geographic Tagged Image File Format (GeoTIFF), Version 2.0,  2001-10-26  https://nsgreg.nga.mil/doc/view?i=2224  
######[32]  
IETF RFC 3986 Uniform Resource Identifier (URI): Generic Syntax http://www.ietf.org/rfc/rfc3986.txt 
######[33]  
OGC08-131r3 The Specification Model — A Standard for Modular specifications  https://portal.opengeospatial.org/files/?artifact_id=34762 

