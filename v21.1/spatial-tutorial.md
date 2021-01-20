---
title: Spatial Tutorial
summary: Tutorial for working with spatial data in CockroachDB.
toc: true
---

## Introduction

Explain what we are doing here

## First steps

Explain what this data is, where to get it, and how to load it.

## Data set overview

Explain the schemas, including diagrams or what-have-you

## Scouting loon locations

### (1) Where has the Common Loon been sighted by the NABBS in the years 2000-2019 in NY state?

{% include copy-clipboard.html %}
~~~ sql
WITH
	loon_sightings
		AS (
			SELECT
				st_collect(routes.geom) AS the_geom
			FROM
				birds.birds,
				birds.observations,
				birds.routes
			WHERE
				birds.name = 'Common Loon'
				AND birds.id = observations.bird_id
				AND observations.route_id = routes.id
		)
SELECT
	st_asgeojson(the_geom)
FROM
	loon_sightings
~~~

### (2) What is the area of the Loon's sightings range? (SRID: 4326)

{% include copy-clipboard.html %}
~~~ sql
WITH
	loon_sightings
		AS (
			SELECT
				st_collect(routes.geom) AS the_geom
			FROM
				birds.birds,
				birds.observations,
				birds.routes
			WHERE
				birds.name = 'Common Loon'
				AND birds.id = observations.bird_id
				AND observations.route_id = routes.id
		)
SELECT
	st_area(st_convexhull(the_geom))
FROM
	loon_sightings
~~~

### (3) How many Loon sightings have there been in NY state in the years 2000-2019?

{% include copy-clipboard.html %}
~~~ sql
SELECT
	birds.name, sum(observations.count) AS sightings
FROM
	birds.birds, birds.observations, birds.routes
WHERE
	birds.name = 'Common Loon'
	AND birds.id = observations.bird_id
	AND observations.route_id = routes.id
GROUP BY
	birds.name
~~~

### (4) How many Loon sightings were there in NY state in just the year 2019?

{% include copy-clipboard.html %}
~~~ sql
SELECT
	birds.name, sum(observations.count) AS sightings
FROM
	birds.birds, birds.observations, birds.routes
WHERE
	birds.name = 'Common Loon'
	AND birds.id = observations.bird_id
	AND observations.route_id = routes.id
	AND observations.year = 2019
GROUP BY
	birds.name
~~~

### (5) What is the density of Loon sightings per square <unit?> in its range as defined by this data set (that is, NY state) in 2019?

{% include copy-clipboard.html %}
~~~ sql
WITH
	loon_habitat
		AS (
			SELECT
				sum(observations.count) AS sightings,
				st_convexhull(st_collect(routes.geom))
					AS the_hull
			FROM
				birds.birds,
				birds.observations,
				birds.routes
			WHERE
				birds.name = 'Common Loon'
				AND birds.id = observations.bird_id
				AND observations.route_id = routes.id
				AND observations.year = 2019
		)
SELECT
	loon_habitat.sightings::FLOAT8
	/ (SELECT st_area(loon_habitat.the_hull))
FROM
	loon_habitat
~~~

## Things about bookstores near loons

### (6) What are the bookstores that lie within the Loon's habitat range in NY state?

SPATIAL JOIN (inverted join on geom index)

{% include copy-clipboard.html %}
~~~ sql
WITH
	loon_sightings
		AS (
			SELECT
				st_convexhull(st_collect(routes.geom))
					AS loon_hull
			FROM
				birds.birds,
				birds.observations,
				birds.routes
			WHERE
				birds.name = 'Common Loon'
				AND birds.id = observations.bird_id
				AND observations.route_id = routes.id
		)
SELECT
	name, street, city, state
FROM
	tutorial.bookstores, loon_sightings
WHERE
	st_contains(loon_hull, geom)
ORDER BY
	geom
~~~

### (7) How many different species of birds' habitats contain the location of the Book Nook in Saranac Lake, NY?

{% include copy-clipboard.html %}
~~~ sql
WITH
	the_book_nook
		AS (
			SELECT
				bookstores.name, street, city, state, geom
			FROM
				tutorial.bookstores
			WHERE
				state = 'NY' AND city = 'Saranac Lake'
		),
	local_birds
		AS (
			SELECT
				birds.name,
				st_convexhull(st_collect(routes.geom))
					AS the_hull
			FROM
				birds.birds,
				birds.observations,
				birds.routes
			WHERE
				birds.id = observations.bird_id
				AND observations.route_id = routes.id
			GROUP BY
				birds.name
		)
SELECT
	count(local_birds.name)
FROM
	local_birds, the_book_nook
WHERE
	st_contains(local_birds.the_hull, the_book_nook.geom)
~~~

### (8) Which birds were most often sighted within 5 miles of the Book Nook in Saranac Lake, NY during the 2000-2019 observation period?  (Limit to top 25)

SPATIAL INDEX (MAYBE - cross join on spatial predicate)

{% include copy-clipboard.html %}
~~~ sql
WITH
	the_book_nook
		AS (
			SELECT
				bookstores.name, street, city, state, geom
			FROM
				tutorial.bookstores
			WHERE
				state = 'NY' AND city = 'Saranac Lake'
		)
SELECT
	birds.name, sum(observations.count) AS sightings
FROM
	birds.birds,
	birds.observations,
	birds.routes,
	the_book_nook
WHERE
	birds.id = observations.bird_id
	AND observations.route_id = routes.id
	AND st_distance(
			the_book_nook.geom::GEOGRAPHY,
			routes.geom::GEOGRAPHY
		)
		< (1609 * 10)
GROUP BY
	birds.name
ORDER BY
	sightings DESC
LIMIT
	25
~~~

### (9) What does the convex hull of bookstore locations that lie within the Loon's habitat look like?

SPATIAL JOIN (inverted join on geom index)

{% include copy-clipboard.html %}
~~~ sql
WITH
	loon_habitat
		AS (
			SELECT
				st_convexhull(st_collect(routes.geom))
					AS the_hull
			FROM
				birds.birds,
				birds.observations,
				birds.routes
			WHERE
				birds.name = 'Common Loon'
				AND birds.id = observations.bird_id
				AND observations.route_id = routes.id
		)
SELECT
	st_asgeojson(st_convexhull(st_collect(geom)))
FROM
	tutorial.bookstores, loon_habitat
WHERE
	st_contains(the_hull, geom)
~~~

### (10) What is the area of the convex hull of bookstore locations that are in the Loon's habitat range within NY state?

SPATIAL JOIN (inverted join on geom index)

{% include copy-clipboard.html %}
~~~ sql
WITH
	loon_habitat
		AS (
			SELECT
				st_convexhull(st_collect(routes.geom))
					AS the_hull
			FROM
				birds.birds,
				birds.observations,
				birds.routes
			WHERE
				birds.name = 'Common Loon'
				AND birds.id = observations.bird_id
				AND observations.route_id = routes.id
		)
SELECT
	st_area(st_convexhull(st_collect(geom)))
FROM
	tutorial.bookstores, loon_habitat
WHERE
	st_contains(the_hull, geom)
~~~

## Trips between bookstores near loons

### (11) How long is the route from Mysteries on Main Street in Johnstown, NY to The Book Nook in Saranac Lake, NY?

{% include copy-clipboard.html %}
~~~ sql
SELECT ST_Length(geom) FROM bookstore_routes WHERE end_store_id = (SELECT ID FROM tutorial.bookstores WHERE city = 'Johnstown' AND STATE = 'NY') AND start_store_id = (SELECT ID FROM tutorial.bookstores WHERE city = 'Saranac Lake' AND STATE = 'NY');
~~~

### (12) What does the route from Mysteries on Main Street in Johnstown, NY to The Book Nook in Saranac Lake, NY look like? (hint: paste GeoJSON output into geojson.io)

{% include copy-clipboard.html %}
~~~ sql
SELECT ST_AsGeoJSON(geom) FROM bookstore_routes WHERE end_store_id = (SELECT ID FROM tutorial.bookstores WHERE city = 'Johnstown' AND STATE = 'NY') AND start_store_id = (SELECT ID FROM tutorial.bookstores WHERE city = 'Saranac Lake' AND STATE = 'NY');
~~~

### (13) What were the 25 most-commonly-sighted birds in 2019 within 10 miles of the route between Mysteries on Main Street in Johnstown, NY and The Bookstore Plus in Lake Placid, NY?

SPATIAL JOIN (MAYBE - cross join on spatial predicate)

{% include copy-clipboard.html %}
~~~ sql
WITH bookstore_trip AS (SELECT geom FROM bookstore_routes WHERE start_store_id = (SELECT ID FROM tutorial.bookstores WHERE city = 'Johnstown' AND STATE = 'NY') AND end_store_id = (SELECT ID FROM tutorial.bookstores WHERE city = 'Lake Placid' AND STATE = 'NY')) SELECT birds.birds.name,SUM(birds.observations.COUNT) AS sightings FROM birds.birds,birds.observations,birds.routes,bookstore_trip WHERE birds.birds.id = birds.observations.bird_id AND observations.route_id = routes.id AND observations.YEAR = 2019 AND ST_Distance(bookstore_trip.geom::GEOGRAPHY,birds.routes.geom::GEOGRAPHY) < (1609*10) GROUP BY birds.name ORDER BY sightings DESC LIMIT 25;
~~~

### (14) What are the 25 least often sighted (i.e., rarest) other species of birds per unit area in the Loon's sightings range?

SPATIAL JOIN - inverted join on routes@geom_idx

{% include copy-clipboard.html %}
~~~ sql
WITH loon_habitat AS (SELECT st_convexhull(st_collect(routes.geom)) AS loon_hull FROM birds.birds, birds.observations, birds.routes WHERE birds.name = 'Common Loon' AND birds.id = observations.bird_id AND observations.route_id = routes.ID) SELECT birds.name, SUM(birds.observations.COUNT) AS sightings FROM birds.birds,birds.observations,birds.routes,loon_habitat WHERE birds.id = observations.bird_id AND observations.route_id = routes.id AND ST_Contains(loon_hull,routes.geom) GROUP BY birds.name ORDER BY sightings ASC LIMIT 25;
~~~

## Road and travel information

### (15) What are the top 10 roads nearest to a Loon sighting location in NY?

SPATIAL JOIN (MAYBE - cross join on spatial predicate)

{% include copy-clipboard.html %}
~~~ sql
WITH loon_habitat AS (SELECT st_collect(routes.geom) AS geom FROM birds.birds, birds.observations, birds.routes WHERE birds.name = 'Common Loon' AND birds.id = observations.bird_id AND observations.route_id = routes.ID) SELECT ST_AsGeoJSON(roads.geom) FROM roads,loon_habitat WHERE roads.STATE = 'NY' AND ST_Distance(loon_habitat.geom, st_setsrid(roads.geom,4326)) < 1 ORDER BY ST_Distance(loon_habitat.geom,st_setsrid(roads.geom,4326)) ASC LIMIT 10;
~~~

### (16) How many miles of roads are contained within the portion of the Loon habitat that lies within NY state?

SPATIAL JOIN (MAYBE - cross join on spatial predicate)

{% include copy-clipboard.html %}
~~~ sql
WITH loon_habitat AS (SELECT ST_ConvexHull(st_collect(routes.geom)) AS hull FROM birds.birds, birds.observations, birds.routes WHERE birds.name = 'Common Loon' AND birds.id = observations.bird_id AND observations.route_id = routes.ID) SELECT sum(roads.miles) FROM roads,loon_habitat WHERE roads.STATE = 'NY' AND ST_Contains(loon_habitat.hull, st_setsrid(roads.geom,4326));
~~~

### (17) Which bookstore in the Loon's region in NY has the fewest miles of roads within a 10 mile radius of the store? (In other words, this may be a proxy for which store is the most remotely located)

SPATIAL JOIN (inverted join on bookstores@geom_idx)

{% include copy-clipboard.html %}
~~~ sql
WITH loon_habitat AS (SELECT st_convexhull(st_collect(routes.geom)) AS geom FROM birds.birds, birds.observations, birds.routes WHERE birds.name = 'Common Loon' AND birds.id = observations.bird_id AND observations.route_id = routes.ID), loon_bookstores AS (SELECT bookstores.name, address,bookstores.geom AS geom FROM bookstores,loon_habitat WHERE ST_Contains(loon_habitat.geom,bookstores.geom))  SELECT loon_bookstores.name,address,SUM(roads.miles)::INT AS nearby_road_miles FROM roads,loon_bookstores WHERE (ST_Distance(loon_bookstores.geom,st_setsrid(roads.geom, 4326)) < (10/69)) GROUP BY loon_bookstores.name,address ORDER BY nearby_road_miles ASC;
~~~

## More on other birds (hawks and owls)

### (18) What are the hawks sighted in the same region as the northern Loon? (assumption: they will have the family "Accipitridae")

SPATIAL JOIN (inverted join on routes@geom_idx)

{% include copy-clipboard.html %}
~~~ sql
WITH loon_habitat AS (SELECT st_collect(routes.geom) AS geom FROM birds.birds, birds.observations, birds.routes WHERE birds.name = 'Common Loon' AND birds.id = observations.bird_id AND observations.route_id = routes.ID) SELECT birds.name FROM birds.birds,birds.observations,birds.routes,loon_habitat WHERE birds.family = 'Accipitridae' AND birds.ID = observations.bird_id AND observations.route_id = routes.ID AND ST_Contains(loon_habitat.geom,routes.geom) GROUP BY birds.name;
~~~

### (19) What if we also want to look for owls as well as the hawks we just found above?

SPATIAL JOIN (inverted join on routes@geom_idx)

{% include copy-clipboard.html %}
~~~ sql
WITH loon_habitat AS (SELECT st_collect(routes.geom) AS geom FROM birds.birds, birds.observations, birds.routes WHERE birds.name = 'Common Loon' AND birds.id = observations.bird_id AND observations.route_id = routes.ID) SELECT birds.name FROM birds.birds,birds.observations,birds.routes,loon_habitat WHERE (birds.family = 'Accipitridae' OR birds.FAMILY = 'Strigidae') AND birds.ID = observations.bird_id AND observations.route_id = routes.ID AND ST_Contains(loon_habitat.geom,routes.geom) GROUP BY birds.name,birds.FAMILY;
~~~

## See also

- [Install CockroachDB](install-cockroachdb.html)
- [Working with Spatial Data](spatial-data.html)
- [Spatial Features](spatial-features.html)
- [Spatial indexes](spatial-indexes.html)
- [Spatial & GIS Glossary of Terms](spatial-glossary.html)
- [Working with Spatial Data](spatial-data.html)
- [Migrate from Shapefiles](migrate-from-shapefiles.html)
- [Migrate from GeoJSON](migrate-from-geojson.html)
- [Migrate from GeoPackage](migrate-from-geopackage.html)
- [Migrate from OpenStreetMap](migrate-from-openstreetmap.html)
- [Spatial functions](functions-and-operators.html#spatial-functions)
- [POINT](point.html)
- [LINESTRING](linestring.html)
- [POLYGON](polygon.html)
- [MULTIPOINT](multipoint.html)
- [MULTILINESTRING](multilinestring.html)
- [MULTIPOLYGON](multipolygon.html)
- [GEOMETRYCOLLECTION](geometrycollection.html)
- [Well known text](well-known-text.html)
- [Well known binary](well-known-binary.html)
- [GeoJSON](geojson.html)
- [SRID 4326 - longitude and latitude](srid-4326.html)
- [`ST_Contains`](st_contains.html)
- [`ST_ConvexHull`](st_convexhull.html)
- [`ST_CoveredBy`](st_coveredby.html)
- [`ST_Covers`](st_covers.html)
- [`ST_Disjoint`](st_disjoint.html)
- [`ST_Equals`](st_equals.html)
- [`ST_Intersects`](st_intersects.html)
- [`ST_Overlaps`](st_overlaps.html)
- [`ST_Touches`](st_touches.html)
- [`ST_Union`](st_union.html)
- [`ST_Within`](st_within.html)
- [Troubleshooting overview](troubleshooting-overview.html)
- [Support resources](support-resources.html)
