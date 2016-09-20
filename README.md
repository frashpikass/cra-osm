# CRA OSM: a tool to generate a Contribute Reliability Approximation map for a specific OpenStreetMap user without the need for feedbacks

In the context of geographical information systems (GIS), when analyzing volunteered geographical informations (VGI) it's natural to question their reliability. Moreover, the common lack of feedback systems in VGI services such as Open Street Map makes it even harder have a clear picture of which part of the map is reliable. Thus, different metrics such as contribute position, age and level of detail should be analyzed in order to produce a localized reliability evaluation. This value could be easily used as an initialization value for other VGI reliability algorithms.

The __cra-osm__ tool, given a user's name and the XML (.osm) dump of a portion of the Open Street Map (OSM) database, provides a way to calculate an initial reliability approximation for any new contribute of said user in the area of interest.

This project is a fork of Maurizio (@napo) Napolitano's [mvp-osm](https://github.com/napo/mvp-osm) project, also available on GitHub.

## Requirements
* Python >2.7
* Spatialite >2.4
* spatialite-tool (spatialite_osm_raw, spatialite_osm_map)
* QGIS with the _Interpolation_ plugin

## Theory
Some of the key concepts of _mvp-osm_ such as __pet-locations__ (an area looked after by an user which has been contributing to it for enough time, where enough means "for a time longer than a specific time frame") and __Good Tags__ (VGI tags, more specifically OSM tag keys, which show that the user has a direct knowledge of the area described by such tags) are critical for the inner working of _cra-osm_.

The logic behind _cra-osm_ goes like this:

1. The Open Street Map XML dump (.osm or .pbf file) is converted into a Spatialite database (e.g.: input.sqlite) with the _spatialite\_osm\_raw_ tool.
2. The Spatialite database is given as an input to _cra.py_. Some other parameters are:
	* the __username__ of the user to produce OSM reliability evaluations for;
	* the list of __Good Tags__ that determine what kind of detail quality we're looking for;
	* a __time frame__ size in days that discriminates how long it takes for VGI data to become stale, or for contributions to an area to become relevant as a _pet location_;
	* the spatial reference system (SRID) of the input and output database;
	* a __cell size__, which is the length of the side of each of the squares that make up the analysis grid; its measuring unit depends on the SRID.
	* You can see a full list of input parameters with their description by calling the cra.py script without arguments;
3. The _cra.py_ script generates a grid to cover the whole input map, looks for contributes by the given _username_, calculates _username_'s _pet-locations_ for every cell in the grid and evaluates their reliability with a value in the interval (0,1], where 0 means "total lack of reliability" and 1 means "maximum reliability". However, _cra-osm_ considers a _pet-location_ as an area which is always more reliable of any other non-pet-location area.
4. The output database (e.g.: output_username.sqlite)is given to QGIS as an input as a Spatialite vector layer, altogether with the original map (which can be processed from the original XML dump with __spatialite\_osm\_map__ for better representation). I suggest to load the _ln\_boundary_ layer from the original map database and the _usersgrid_ layer from the output database.
5. QGIS's [interpolation plugin](https://docs.qgis.org/2.6/en/docs/user_manual/plugins/plugins_interpolation.html) with Shepard's Inverse Distance Weighting algorithm is used to interpolate the reliability values over the chosen region. Apply such algorithm to the _reliability_ attribute of vector _usersgrid_ of the output database, picking the grid cell size chosen at point 2. as the X and Y cell size for the IDW interpolation. A customized _p_ parameter value for Shepard's interpolation can be obtained as an output from _cra.py_ by specifying the _epsilon_ and _limit distance_ values as arguments for the _cra.py_ script.
6. The raster layer generated in step 5. can be customized with pseudo colours to make it more easily understandable. See [this tutorial](http://www.qgistutorials.com/en/docs/interpolating_point_data.html) to learn how.
7. Every cell in the raster now contains the approximated reliability value.

## Pet location reliability evaluation table
Presence of contributes described by Good Tags | Age of the last contribute | Reliability
--------- | ------------------- | -----------
Yes | less than the time frame | 1
Yes | more than the time frame | 0.75
No | less than the time frame | 0.50
No | more than the time frame | 0.25

## CLI usage
To convert an OSM XML into a Spatialite database for _cra.py_ processing write:
```
spatialite_osm_raw -o input.osm -d input.sqlite
```

To convert an OSM XML into a Spatialite database to visualize it in QGIS write:
```
spatialite_osm_map -o input.osm -d map.sqlite
```

To launch _cra.py_ and get help on the available input arguments write:
```
python2 cra.py
```

To analyze an OSM spatialite database for user 'Goffredo' with a cell size of 1000 meters write:
```
python2 cra.py -i input.sqlite -u Goffredo -g 1000
```

Enjoy!
