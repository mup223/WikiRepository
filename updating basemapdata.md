# Updating GeoCall Map Data

## Table of Contents

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Updating GeoCall Map Data](#updating-geocall-map-data)
	- [Table of Contents](#table-of-contents)
	- [Overview](#overview)
	- [General Workflow](#general-workflow)
	- [Detailed Workflow](#detailed-workflow)
	- [PostgreSQL Data Structures](#postgresql-data-structures)
	- [General QA Procedures](#general-qa-procedures)

<!-- /TOC -->

## Overview
This documentation is intended to guide you through updating the map data that is displayed in GeoCall. This process mostly revolves around PostgreSQL – getting the data into it, running the scripts to get it in the necessary format, and then exporting it out to be loaded into GeoCall. Since the data is given to us per County, that is typically the order in which they are updated.

## General Workflow
* Get the data from the SCGIS FTP.
* Copy the data to a local or network folder for your records.
* Import the data into PostgreSQL.
* Run scripts to format County data.
* Delete existing County data from staging table and insert newly formatted data
* Run command to export staging table to .shp format to desired location for GeoCall tool.
* Run GeoCall tool in development environment to get .shp data just export into the GeoCall PostgreSQL database.
* Create indexes on newly created GeoCall table.
* Repeat previous two steps in production and any other environments needed.


## Detailed Workflow
* Get the data from the SCGIS FTP.
  - ftp://www.gis.sc.gov/
  - username: gicget
  - password: gic.2013.coop!
* Copy that data to a local or network folder for your records. I have chosen to keep all of my records here:
  - `\\geostore\Scum_files\Projects\Basemap Updates`
* Import the data into PostgreSQL.
  - The database I currently use is located here:
    - Host: GEOSTORE
    - Port: 5433
    - Username: postgres
    - Password: SpiDee11
    - Database: BaseMapUpdates
  - Each County has a corresponding database schema.
  - There are four elements that usually need updating in each County and there are tables in the schema that correspond to each:
    - roads
    - addresses
    - parcels
    - municipal boundaries
* Run scripts to format County data before inserting into the staging table
  - I have created .sql scripts for every county and they are located here:
    - `\\GEOSPACE\GeoCallV3\GIS and Postgre Files\Queries\Base Data Updates`
* Delete existing County data from staging table and insert newly formatted data
  - The staging table is located in the public schema of the same database and there are tables for each of the corresponding features listed above:
    - ex: `public.addresses`
    - If you'll notice, in all of the .sql scripts this will be the last thing that is done for each of the features.
* Run command line script to export the data from the staging table
  - the script to export the data is located here:
    - `\\GEOSPACE\GeoCallV3\GIS and Postgre Files\scripts\BaseMap_PSQL_export.bat`
* Run the GeoCall tool `importBase.bat`
  - There is a batch file located on the desktop of the VM, but if you open up the file a look at where it is pointing, you will find it is pointing to the `import_base.txt` tool which is located here:
    - `M:\GeoCallV3\config\test\gis-config`
* After the tool has successfully run, it's time to open up the localhost PostgreSQL instance and create the indexes. Currently the best tool to accomplish this is PGAdmin 3 and running the `Create New Indexes.sql` file located here:
  - `M:\Queries`
* Once you've completed all of the steps and confirmed that GeoCall can search addresses and the data is displaying properly, move onto the next machines and repeat the previous two steps.

## PostgreSQL Data Structures
I included this section in here so that, if you want to take your own route to performing the updates you'll know outright what table structure GeoCall is looking for.  

Addresses:
~~~sql
-- Table: public.addresses

-- DROP TABLE public.addresses;

CREATE TABLE public.addresses
(
  id integer NOT NULL DEFAULT nextval('addresses_id_seq'::regclass),
  geom geometry(Point,4326),
  gid bigint,
  housenum character varying(30),
  prefix character varying(10),
  name character varying(100),
  type character varying(16),
  suffix character varying(16),
  placename character varying(100),
  countyname character varying(100),
  address character varying(150),
  unit character varying(50),
  CONSTRAINT addresses_pkey PRIMARY KEY (id)
)
WITH (
  OIDS=FALSE
);
ALTER TABLE public.addresses
  OWNER TO postgres;
~~~  

Building Footprints:
~~~sql
-- Table: public.footprint

-- DROP TABLE public.footprint;

CREATE TABLE public.footprint
(
  gid integer NOT NULL DEFAULT nextval('footprints_gid_seq'::regclass),
  geom geometry(MultiPolygon,4326),
  countyname character varying(100),
  feature character varying(100),
  CONSTRAINT footprints_pkey PRIMARY KEY (gid)
)
WITH (
  OIDS=FALSE
);
ALTER TABLE public.footprint
  OWNER TO postgres;
~~~  

Parcels:
~~~sql
-- Table: public.parcels

-- DROP TABLE public.parcels;

CREATE TABLE public.parcels
(
  gid integer NOT NULL DEFAULT nextval('parcels_new_gid_seq'::regclass),
  ___uid_ numeric,
  countyname character varying(50),
  geom geometry(MultiPolygon,4326),
  split character varying(5),
  x character varying(50),
  shap_leng numeric,
  shape_le_1 numeric,
  shape_area numeric,
  tms_acreag character varying(50),
  CONSTRAINT parcels_new_pkey PRIMARY KEY (gid)
)
WITH (
  OIDS=FALSE
);
ALTER TABLE public.parcels
  OWNER TO postgres;

-- Index: public.parcels_new_geom_idx

-- DROP INDEX public.parcels_new_geom_idx;

CREATE INDEX parcels_new_geom_idx
  ON public.parcels
  USING gist
  (geom);
~~~  

Places:
~~~sql
-- Table: public.places

-- DROP TABLE public.places;

CREATE TABLE public.places
(
  gid integer NOT NULL DEFAULT nextval('places_serial_gid_seq'::regclass),
  geom geometry(MultiPolygon,4326) NOT NULL,
  countyname character varying(100),
  placename character varying(250),
  CONSTRAINT places_serial_pkey PRIMARY KEY (gid)
)
WITH (
  OIDS=FALSE
);
ALTER TABLE public.places
  OWNER TO postgres;
~~~  

Roads:
~~~sql
-- Table: public.roads

-- DROP TABLE public.roads;

CREATE TABLE public.roads
(
  gid integer NOT NULL DEFAULT nextval('roads_gid_seq'::regclass),
  id numeric(10,0),
  __gid numeric,
  l_f_add character varying(30),
  l_t_add character varying(30),
  r_f_add character varying(30),
  r_t_add character varying(30),
  prefix character varying(10),
  fcc character varying(10),
  name character varying(100),
  type character varying(16),
  suffix character varying(16),
  strplace character varying(100),
  streetname character varying(100),
  placename character varying(100),
  countyname character varying(100),
  geom geometry(MultiLineString,4326),
  CONSTRAINT roads_pkey PRIMARY KEY (gid)
)
WITH (
  OIDS=FALSE
);
ALTER TABLE public.roads
  OWNER TO postgres;

-- Index: public.roads_geom_idx

-- DROP INDEX public.roads_geom_idx;

CREATE INDEX roads_geom_idx
  ON public.roads
  USING gist
  (geom);
~~~

## General QA Procedures

There have been a few conventions that should be mentioned when updating base map data for GeoCall to preserve cohesiveness for the CSR's who are querying the data endlessly.
* If the road name is Savannah Hwy then the name is 'Savannah' and the road type is 'Hwy'. If the road name is Hwy 17, then the name is 'Hwy 17' and the type is null. You'll see that each of the county .sql scripts will have some variation of commands to account for this.
* Many of the road types that are submitted to us from the counties do not adhere to the USPS suffix standardization. You will see in most of the .sql scripts there are some commands to bring the types up to this standard.
  - `https://pe.usps.com/text/pub28/28apc_002.htm`
* All of the numbered streets are stored in their numerical form:
  - First St = 1st St
  - The script to fix this is run prior to exporting and is located here:
    - `\\geospace\GeoCallV3\GIS and Postgre Files\Queries\Base Data Updates\generalQA.sql`
