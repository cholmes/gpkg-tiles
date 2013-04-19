#Implementation Notes

This document initially will serve as the place for the notes in the main spec document
to live, linked to from there.

In time it may expand to a fuller engineering report document as well, giving an overview
of the research done to get to this specification.

###Notes

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

