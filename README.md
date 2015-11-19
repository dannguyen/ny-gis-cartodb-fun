
# Using CartoDB to analyze and map New York demographic and crime data

Some draft notes and documentation on using [CartoDB](https://cartodb.com) to do some data visualization and GIS, including how to collect that data from the source, and how to use SQL to wrangle it for analytical and visualization purposes.

- The [Maps](#maps) section contains example maps, as well as the SQL needed to create their datasets.
- The [Datasets](#datasets) section documents where on the Web I found the data as well as the actual file downloaded.
- The [Join/lookup tables](#joinlookup-tables) section shows the SQL for deriving the intermediary datasets needed to link the official datasets, e.g. how the [pretracts_lookup](https://dunnguyen.cartodb.com/tables/pretracts_lookup) is used to associate each Census tract to a NYPD precinct, which allows for the [calculation of Census population by NYPD precinct](http://bit.ly/1Qv4Lxr).


# Maps (so far)

## [Census total population by NYPD precinct](http://bit.ly/1Qv4Lxr)

<a href="http://bit.ly/1Qv4Lxr">
  <img src="http://i.imgur.com/GNytrbe.png" alt="Census total population by NYPD precinct">
</a>




#### Dataset and SQL

You can see the underlying dataset here: [nypd_precincts_and_2010_census_pop](https://dunnguyen.cartodb.com/tables/nypd_precincts_and_2010_census_pop)

The SQL to create it from my [source datasets](#datasets):

~~~sql
SELECT 
  precinct_id,
  precinct_pop,
  cartodb_id,
  the_geom_webmercator
FROM nypd_precincts
INNER JOIN  
    (SELECT 
      precinct_id,
      SUM(total_pop) AS precinct_pop
    FROM 
      pretracts_lookup
    INNER JOIN census_pop
      ON
        pretracts_lookup.tract_id = census_pop.census_tract_code
         AND 
        pretracts_lookup.borocode = census_pop.dcp_borough_code
    GROUP BY precinct_id) 
  AS tx
  ON tx.precinct_id = nypd_precincts.precinct;
~~~






# Datasets

- [census_pop](https://dunnguyen.cartodb.com/tables/census_pop) - contains 2010 Census population by race data
  - Source: [NYC Planning Department demographics page](http://www.nyc.gov/html/dcp/html/census/demo_tables_2010.shtml)
  - Download: [Total Population by Mutually Exclusive Race and Hispanic Origin, 2010 (xls)](http://www.nyc.gov/html/dcp/download/census/census2010/t_pl_p3a_ct.xlsx)
- [census_tracts](https://dunnguyen.cartodb.com/tables/census_tracts) - contains the shapes of the 2010 Census tracts 
  - Source: [NYC Planning Department: Political and Administrative Districts](http://www.nyc.gov/html/dcp/html/bytes/districts_download_metadata.shtml#cbt)
  - Download: [Census Tracts 2010 (Clipped to Shoreline) (.zip)](http://www.nyc.gov/html/dcp/download/bytes/nyct2010_15c.zip)
- [nypd_precincts](https://dunnguyen.cartodb.com/tables/nypd_precincts)
  - Source: [NYC Planning Department: Political and Administrative Districts](http://www.nyc.gov/html/dcp/html/bytes/districts_download_metadata.shtml#shfp) 
  - Download: [Police Precincts (Clipped to Shoreline)](http://www.nyc.gov/html/dcp/download/bytes/nypp_15c.zip) 
- [nypd_crimedata](https://dunnguyen.cartodb.com/tables/nypd_crimedata) - NYPD's historical crime statistics at the precinct level
  - Source: [I have a separate repo describing the stats compilation process](https://github.com/datahoarder/nypd-historical-crime-stats)
  - Download: [Compiled precinct-level crime stats 2000 to 2014 (.csv)](https://github.com/datahoarder/nypd-historical-crime-stats/raw/master/data/compiled/nypd-precinct-historical-crime-data.csv)

## Join/lookup tables

### [pretracts_lookup](https://dunnguyen.cartodb.com/tables/pretracts_lookup)

Requires a spatial join of the Census tracts to each precinct -- not knowing a lot about PostGIS but also knowing how annoying it is to define containment/intersection of real-life boundaries in general, I opted to join the tracts to precincts on the basis of whether the tract's centroid was within a precinct's boundary. 

~~~sql
SELECT
  borocode,
  ct2010 AS tract_id,
  precinct AS precinct_id
FROM census_tracts
INNER JOIN nypd_precincts 
  ON ST_WITHIN(
                ST_Centroid(census_tracts.the_geom), 
                nypd_precincts.the_geom);
~~~
