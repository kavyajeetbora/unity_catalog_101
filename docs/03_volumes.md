
## What are volumes, and when would you use one instead of a table?

<img src='https://docs.databricks.com/aws/en/assets/images/object-model-volume-d472dba24dca8002ff629cde896d4753.png' height=300/>

A volume is a Unity Catalog object that governs files (CSV, Parquet, PMTiles, JSON, PDFs, images) at catalog.schema.volume, accessed through a /Volumes/... path. Permissions come from Unity Catalog (READ VOLUME, WRITE VOLUME), with audit and lineage. Use a table when the data is structured and queried with SQL or Delta features. Use a volume when you need the data as files: landing zones, geospatial files, ML artifacts, or files other tools read by path. Managed vs external follows the same rule as tables.